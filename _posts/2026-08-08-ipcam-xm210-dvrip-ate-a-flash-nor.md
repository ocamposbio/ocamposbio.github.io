---
layout: post
title: "IPCam XM210 — do DVRIP até a flash NOR (WIP)"
subtitle: "O que acontece quando você compra uma câmera barata de 2MP, consegue escrever na flash dela... e a rootfs vem criptografada."
date: 2026-08-08
status: draft
---

> **Status: trabalho em andamento.** O dump direto da flash NOR ainda depende de um programador CH341A que está a caminho. Vou atualizando este post conforme a exploração avança.

## tl;dr

Comprei uma `IPC_XM210_X2-WR-T_S38` (Xiongmai, SoC RISC-V T-Head C906, flash NOR 8MB) para estudo de segurança de firmware. Consegui login no DVRIP com `admin` e senha vazia, descobri que o protocolo de upgrade grava qualquer coisa na flash, e cheguei a flashear um cramfs customizado na partição `mtd4`. Só que a rootfs é criptografada, o kernel não aparece no UART e meu hook de shell nunca disparou. O próximo passo é físico: dump da flash com CH341A.

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

## Por onde começar

Port scan: **só `34567/TCP` (DVRIP) aberto**. Nada de HTTP, RTSP, telnet, 9527, 9530 — a câmera não expõe mais nada. Toda a superfície remota é o protocolo DVRIP da Xiongmai, que é bem conhecido pela comunidade.

### Layout de partições (da partição "app")

```
mtd0: boot      16 KB    u-boot SPL
mtd1: app       ~4 MB    u-boot principal
mtd2: version   4 KB
mtd3: download  ~2.4 MB  kernel + rootfs (ENCRIPTADA)
mtd4: custom    832 KB   cramfs (configs/scripts) ← vetor de escrita
mtd5: mtd       768 KB
```

O que mudou o rumo do projeto foi descobrir que a partição `custom` é **gravável via DVRIP** e monta um cramfs. Ou seja: era possível injetar código sem quebrar a criptografia da rootfs.

## O que funcionou

### 1. Login DVRIP com admin vazio

O hash XMMD5 da Xiongmai é notavelmente fraco:

```python
def xmmd5(password):
    md5 = hashlib.md5(password.encode()).digest()
    magic = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'
    return ''.join(magic[(a+b) % 62] for a, b in zip(md5[0::2], md5[1::2]))[:8]
```

`admin` com senha vazia gera `tlJwpbo6` e entra direto.

### 2. Escrita arbitrária de flash via OPSystemUpgrade

O comando `OPSystemUpgrade` aceita um ZIP com `InstallDesc` + imagens `.img` e grava partições por índice. Com isso consegui:

- construir e gravar um **cramfs customizado** em `mtd4`, com um `extapp.sh` tentando abrir shell;
- gravar **u-boot.env** em `mtd0` (env_console / env_initsh), incluindo uma tentativa de `init=/bin/sh` para pular o init normal;
- observar a câmera reiniciar ~114s depois do flash (a porta cai, o processo de reboot é visível).

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

## O que travou (e por quê)

| Tentativa | Resultado |
|---|---|
| CVE-2022-45045 (upgrade auth bypass) | Bloqueado — validação criptográfica Mx8Q (Ret=513) |
| `extapp.sh` → telnet/revshell | Nenhum port (23, 2323, 4444, 8080) abriu — hook não chamado ou busybox sem applets |
| `env_initsh` (`init=/bin/sh`) | UART segue mudo — bootargs provavelmente hardcoded no u-boot binário |
| Kernel console via UART0 | Kernel usa **UART1** (`xmuart=1`), pinos ainda não tracejados na PCB |
| Rootfs desencriptada | **download.img criptografada** — chave vive dentro do RTOS criptografado |

A situação ficou circular: rootfs criptografada → precisava de shell → shell depende do kernel → kernel mudo no UART → precisava do dump da flash para análise estática. Cada caminho bloqueava o anterior. O que resta é o acesso físico: ler a flash direto no hardware.

## Próximos passos (quando o CH341A chegar)

1. **Dump da flash SPI NOR 8MB** com o CH341A (chip provavelmente uma Winbond/GigaDevice 8MB, SOIC-8 — pode precisar de clamp/SOIC-8 clip);
2. Identificar partições no dump cru e extrair `download.img` para análise estática **fora da câmera** (binwalk, unpack);
3. Localizar o u-boot real (`app`) e os bootargs hardcoded — desbloquear console de kernel via UART1;
4. Confirmar o hook `extapp.sh` no `rcS` real e reflash do cramfs corrigido.

## Notas soltas

- A criptografia foi a maior barreira, mais do que qualquer código ofuscado. Sem acesso ao dado estático, era tentativa e erro no escuro.
- CVE-2022-45045 não é universal: o bypass conhecido morre na validação Mx8Q dos firmwares mais novos. Vale conferir a versão antes de gastar tempo.
- Flash gravável via protocolo de câmera é comum e pouco explorado: quando a rootfs é inacessível, a partição de config vira o caminho.
- UART mudo não significa UART inexistente. Console de debug pode estar em outra porta, baud ou pinos — essa câmera tem um segundo UART esperando.

*Continua quando o hardware chegar.*
