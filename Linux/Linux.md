# Linux Internals

Uma análise profunda de como o Linux funciona por baixo dos panos — do botão de ligar ao processo em execução.

Isso não é sobre distros. É sobre o sistema operacional em si: como ele inicializa, como o kernel gerencia hardware, memória, processos e tudo mais.

---

## O Processo de Boot

O que acontece desde o power on até o prompt de login:

```
Power On
  │
  ▼
BIOS/UEFI ──► POST (Power-On Self-Test)
  │              checks CPU, RAM, devices
  ▼
Bootloader (GRUB2)
  │   loads kernel + initramfs into memory
  ▼
Kernel Initialization
  │   decompresses, sets up memory, detects hardware
  ▼
initramfs (Initial RAM Filesystem)
  │   temporary root filesystem in memory
  │   loads essential drivers to mount real root
  ▼
Init System (systemd / PID 1)
  │   first userspace process
  │   starts all services and targets
  ▼
Login Prompt / Desktop
```

### BIOS vs UEFI

- **BIOS** (legado): 16-bit, tabela de partição MBR, máximo de 2TB por disco, boot sequencial
- **UEFI**: 64-bit, tabela de partição GPT, sem limite de tamanho, Secure Boot, mais rápido
- O UEFI armazena bootloaders na **ESP** (EFI System Partition), geralmente em `/boot/efi`

### GRUB2

- Grand Unified Bootloader — o bootloader Linux mais comum
- Configuração em `/boot/grub/grub.cfg` (gerado automaticamente, não edite diretamente)
- Edite `/etc/default/grub` → execute `grub-mkconfig -o /boot/grub/grub.cfg`
- Pode fazer chainload de outros bootloaders de SO (dual boot)

### initramfs

- Arquivo comprimido carregado na RAM junto com o kernel
- Contém um filesystem mínimo com os drivers necessários para montar o root real
- Por quê? A imagem do kernel não pode conter todos os drivers possíveis de disco/filesystem
- Após o root real ser montado, o initramfs é descartado

## Tópicos

- [[Linux/Kernel]]
- [[Linux/ProcessManagement]]
- [[Linux/MemoryManagement]]
- [[Linux/FileSystem]]
- [[Linux/SystemCalls]]
- [[Linux/Networking]]
- [[Linux/ProcessesAndThreads]]

## Recursos

- https://0xax.gitbooks.io/linux-insides/content/ (Linux Insides — análise profunda)
- https://tldp.org/LDP/tlk/tlk.html (The Linux Kernel)
- https://man7.org/linux/man-pages/ (man pages online)
- Linux Kernel Development — Robert Love (livro)
- Operating Systems: Three Easy Pieces — livro didático gratuito online

#### Meus comentários
- 
