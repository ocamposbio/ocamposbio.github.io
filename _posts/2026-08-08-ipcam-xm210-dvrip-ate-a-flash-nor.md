---
layout: post
title: "IPCam XM210 — Do DVRIP à flash NOR (WIP)"
subtitle: "Explorando uma câmera Xiongmai barata de 2MP sem consertar o firmware, ou: o que fazer quando a rootfs vem criptografada."
date: 2026-08-08
status: draft
---

> **Status: work in progress.** O dump direto da flash NOR ainda depende de um programador CH341A (a caminho). Este post atualiza conforme a exploração avança.

## tl;dr

Uma câmera IP `IPC_XM210_X2-WR-T_S38` (Xiongmai, SoC RISC-V T-Head C906, flash NOR 8MB) chegou às minhas mãos. Login DVRIP com `admin`/senha vazia, escrita arbitrária de partições via `OPSystemUpgrade`, cramfs customizado gravado com sucesso em `mtd4` — mas a rootfs é criptografada, o kernel não fala no UART, e o hook `extapp.sh` não disparou. A saída final está no hardware: **dump da flash SPI NOR com CH341A**.

## Hardware

| Componente | Valor |
|---|---|
| Modelo | IPC_XM210_X2-WR-T_S38 |
| SoC | XM210 (Xiongmai custom) |
| CPU | RISC-V T-Head C906, RV64GC |
| RAM | DDR3 16MB @ 300MHz |
| Flash | NOR 8MB (SPI) |
| WiFi | SSV6158M |
| Sensor | SC2331 (2MP) |
| Firmware | V5.07.R02.20260311 |

## Superfície de ataque

Port scan: **só `34567/TCP` (DVRIP) aberto**. Nada de HTTP, RTSP, telnet, 9527, 9530. Toda a superfície remota é o protocolo DVRIP da Xiongmai.

### MTD layout (da partição "app")

```
mtd0: boot      16 KB    u-boot SPL
mtd1: app       ~4 MB    u-boot principal
mtd2: version   4 KB
mtd3: download  ~2.4 MB  kernel + rootfs (ENCRIPTADA)
mtd4: custom    832 KB   cramfs (configs/scripts) ← vetor de escrita
mtd5: mtd       768 KB
```

O detalhe que importa: a partição `custom` é **writable via DVRIP** e monta um cramfs. É o vetor para injetar código — sem precisar quebrar a criptografia da rootfs.

## O que funcionou

### 1. Auth DVRIP com admin vazio

O hash XMMD5 da Xiongmai é trivial:

```python
def xmmd5(password):
    md5 = hashlib.md5(password.encode()).digest()
    magic = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'
    return ''.join(magic[(a+b) % 62] for a, b in zip(md5[0::2], md5[1::2]))[:8]
```

`admin` + senha vazia → `tlJwpbo6`. Login direto.

### 2. Escrita arbitrária de flash via OPSystemUpgrade

O comando `OPSystemUpgrade` aceita um ZIP com `InstallDesc` + imagens `.img` e flasha partições por índice. Com isso:

- **cramfs custom** construído e gravado em `mtd4` (com `extapp.sh` tentando abrir shell);
- **u-boot.env** (env_console / env_initsh) gravado em `mtd0` — incluindo uma tentativa de `init=/bin/sh` para pular o init normal;
- camera reinicia ~114s depois do flash (porta cai, processo confirmado por monitoramento).

### 3. UART 115200 — SPL boot visível

```
WDT: 0x00000014s
csi_mpu_config_region
Check Version OK
Boot...
flash init...ok
pll init.PLLC-2MPsensor.00003010.ok
board init osmem[7538K]
Decompress fail:00000001, status:00000000
Decompress finish.
[ silêncio — kernel não tem console neste UART ]
```

## O que NÃO funcionou (e por quê)

| Tentativa | Resultado |
|---|---|
| CVE-2022-45045 (upgrade auth bypass) | Bloqueado — validação criptográfica Mx8Q (Ret=513) |
| `extapp.sh` → telnet/revshell | Nenhum port (23, 2323, 4444, 8080) abriu — hook não chamado ou busybox sem applets |
| `env_initsh` (`init=/bin/sh`) | UART segue mudo — bootargs provavelmente hardcoded no u-boot binário |
| Kernel console via UART0 | Kernel usa **UART1** (`xmuart=1`), pinos ainda não tracejados na PCB |
| Rootfs desencriptada | **download.img criptografada** — chave vive dentro do RTOS criptografado |

O beco sem saída é circular: rootfs criptografada → precisa de shell → shell depende do kernel → kernel mudo no UART → preciso de dump da flash para análise estática. **É aí que entra o CH341A.**

## Próximos passos (quando o CH341A chegar)

1. **Dump da flash SPI NOR 8MB** com o CH341A (chip provavelmente uma Winbond/GigaDevice 8MB, SOIC-8 — pode precisar de clamp/SOIC-8 clip);
2. Identificar partições reais no dump cru e extrair `download.img` para análise estática **fora da câmera** (binwalk, unpack);
3. Localizar o u-boot real (`app`) e os bootargs hardcoded — desbloquear console de kernel via UART1;
4. Confirmar o hook `extapp.sh` no `rcS` real e reflash do cramfs corrigido.

## Lições até aqui

- **Câmera de $10 da Xiongmai = superfície enorme, mas a criptografia salva o firmware.** A barreira real não foi o código, foi a *falta de acesso ao dado estático*.
- **CVE-2022-45045 não é universal**: o bypass conhecido morre na validação Mx8Q de firmware mais novo. Fingerprint da versão primeiro.
- **Flash writable via protocolo de câmera é comum e subestimado**: quando a rootfs é inacessível, a partição de config é o caminho.
- **UART mudo ≠ sem UART**: console de debug pode estar em outra porta (UART1), baud ou pinos.

*Continua quando o hardware chegar.*
