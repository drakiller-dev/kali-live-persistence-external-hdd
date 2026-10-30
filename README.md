# Kali Linux Live persistente em HD externo de 1 TB (UEFI/GRUB + LUKS preservado)

**Relato técnico de uma configuração que funcionou** — outubro de 2026. O objetivo foi inicializar o Kali Linux Live diretamente de um HD USB, com persistência, **sem depender do pendrive Ventoy usado na preparação** e sem apagar uma partição LUKS preexistente de ~600 GB.

> ⚠️ **Atenção:** este é um registro do procedimento aplicado a um HD específico, **não um roteiro universal para executar cegamente**. Redimensionar e formatar partições pode destruir arquivos. Faça backups verificados e confirme modelos, capacidades e UUIDs dos dispositivos antes de qualquer alteração. Nunca formate a partição LUKS existente nem use `dd` na unidade inteira neste cenário.

## Resultado obtido

- O firmware UEFI do notebook Dell iniciou o **GRUB do HD externo** (`BOOTX64.EFI`).
- A opção **Kali Live Persistence** iniciou o Kali Linux Live 2026.3, imagem `kali-linux-2026.3-live-everything-amd64.iso`.
- Um arquivo criado na pasta pessoal permaneceu disponível após reiniciar: **persistência testada**.
- A partição LUKS original de ~600 GB foi mantida separada.
- O SSD interno do notebook não foi escolhido como destino da instalação do GRUB.

## Hardware e layout final

Preparação realizada a partir de **outro pendrive** com Ventoy e Kali Live. HD externo Western Digital USB de 1 TB (~931,5 GiB no Linux).

| Partição do HD USB | Sistema | Tamanho | Uso |
|---|---|---:|---|
| `/dev/sdX1` | `crypto_LUKS` | 558,79 GiB | Volume criptografado preexistente, preservado |
| `/dev/sdX2` | ext4 | 24 GiB | ISO, kernel, initrd e GRUB |
| `/dev/sdX3` | FAT32 (`boot,esp`) | 1 GiB | Partição EFI, `KALI_EFI` |
| `/dev/sdX4` | ext4, label `persistence` | 347,72 GiB | Persistência do Kali |

**`sdX` não é um caminho real.** O mesmo HD apareceu como `/dev/sdc` numa sessão e `/dev/sdb` após reiniciar. **Confirme sempre:**

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,MODEL,MOUNTPOINTS
sudo parted -l
```

## 1 — Reorganização das partições

O HD já possuía uma partição LUKS e outra ext4 de ~372,72 GiB. No GParted, após **desmontar** a ext4 e conferir seu sistema de arquivos com `e2fsck`, a segunda partição foi reduzida a **24 GiB**. No espaço liberado foram criadas:

1. FAT32 de **1 GiB** chamada `KALI_EFI`, com flags **boot, esp**.
2. ext4 com o restante (~**347,72 GiB**) e rótulo de **sistema de arquivos** `persistence`.

As operações foram efetivamente **aplicadas** no GParted. A partição LUKS permaneceu intocada. **Essa etapa apresenta risco real de perda de dados se não houver backup**, mesmo ao redimensionar apenas outra partição do mesmo disco. Os valores são específicos deste projeto.

## 2 — Preparação da persistência

A partição ext4 rotulada `persistence` precisa ter, **em sua raiz**, o arquivo `persistence.conf` com uma linha:

```text
/ union
```

Após identificar com segurança a quarta partição:

```bash
sudo mkdir -p /mnt/kali-persistence
sudo mount /dev/sdX4 /mnt/kali-persistence
sudo sh -c 'echo "/ union" > /mnt/kali-persistence/persistence.conf'
cat /mnt/kali-persistence/persistence.conf
sync
sudo umount /mnt/kali-persistence
```

`/dev/sdX4` é ilustrativo: substitua **somente depois** de validar o dispositivo. Neste caso, o arquivo foi criado e confirmado. **Não é necessário formatar outra vez uma partição já preparada.**

## 3 — ISO e pontos de montagem

A ISO foi colocada na **raiz da partição ext4 de 24 GiB**, com nome exato:

```text
/kali-linux-2026.3-live-everything-amd64.iso
```

Baixe a ISO de fonte oficial e confira o SHA256 disponibilizado para **a mesma imagem e versão**. Na sessão Live iniciada pelo Ventoy, os pontos de montagem usados foram:

```bash
sudo mkdir -p /mnt/kali-boot /mnt/kali-efi
sudo mount /dev/sdX2 /mnt/kali-boot
sudo mount /dev/sdX3 /mnt/kali-efi
findmnt /mnt/kali-boot
findmnt /mnt/kali-efi
```

Os resultados devem apontar para a **ext4 de 24 GiB** e a **FAT32 de 1 GiB** do HD externo, respectivamente. Na primeira sessão do projeto, os nomes efetivos eram `/dev/sdc2` e `/dev/sdc3`.

## 4 — Instalação UEFI do GRUB no HD externo

Na sessão Kali Live, foram instaladas as ferramentas `grub-efi-amd64-bin` e `grub2-common` (pacotes instalados em uma sessão Live sem persistência não sobrevivem ao reboot). **Com as montagens verificadas**, o comando que funcionou foi:

```bash
sudo grub-install --target=x86_64-efi --efi-directory=/mnt/kali-efi --boot-directory=/mnt/kali-boot/boot --removable --no-nvram
```

Saída observada:

```text
Installing for x86_64-efi platform.
Installation finished. No error reported.
```

Foi confirmada a presença de `/mnt/kali-efi/EFI/BOOT/BOOTX64.EFI`. A opção `--removable` prepara o caminho padrão de inicialização de mídia removível; `--no-nvram` evita cadastrar nova entrada de boot no firmware. **Não usar o SSD interno como destino.**

## 5 — Erro do `loopback` e correção decisiva

A primeira configuração tentava carregar o kernel dentro da ISO usando `loopback loop` (e depois `loopback kalilive`). O GRUB aparecia, mas a inicialização falhava:

```text
error: device name already exists.
error: file '/live/vmlinuz' not found.
error: you need to load the kernel first.
```

A alteração do nome do dispositivo virtual não resolveu a localização do kernel. A **solução efetivamente testada** foi **copiar o kernel e o initrd para fora da ISO**, na raiz da ext4 de 24 GiB:

```bash
sudo mkdir -p /mnt/kali-iso
cd /mnt/kali-boot
sudo mount -o loop,ro *.iso /mnt/kali-iso
sudo cp /mnt/kali-iso/live/vmlinuz /mnt/kali-boot/
sudo cp /mnt/kali-iso/live/initrd.img /mnt/kali-boot/
ls -lh /mnt/kali-boot/vmlinuz /mnt/kali-boot/initrd.img
```

Foram observados aproximadamente **14 MB** para `vmlinuz` e **215 MB** para `initrd.img` (podem variar entre versões).

## 6 — `grub.cfg` final que funcionou

Arquivo: `/mnt/kali-boot/boot/grub/grub.cfg`

```cfg
set timeout=10

menuentry "Kali Live Persistence" {
    search --file --set=root /kali-linux-2026.3-live-everything-amd64.iso
    linux /vmlinuz boot=live components findiso=/kali-linux-2026.3-live-everything-amd64.iso persistence
    initrd /initrd.img
}
```

O `search` encontra a partição da ISO; `linux /vmlinuz` e `initrd /initrd.img` carregam os arquivos **diretamente da partição ext4**. O parâmetro `findiso` permite que o estágio seguinte localize a imagem Live, e `persistence` solicita o armazenamento persistente preparado.

Após salvar, foi executado o teste de sintaxe:

```bash
sudo grub-script-check /mnt/kali-boot/boot/grub/grub.cfg
```

**Nenhuma mensagem** indica sintaxe aceita, **não** garante que o boot vá funcionar. O nome e os caminhos da ISO devem corresponder ao conteúdo real.

## 7 — Desmontagem e teste de boot

```bash
cd ~
sync
sudo umount /mnt/kali-iso
sudo umount /mnt/kali-boot
sudo umount /mnt/kali-efi  # somente se estiver montada
```

Desligamos a sessão Live do Ventoy, retiramos o pendrive e mantivemos o HD externo conectado. No notebook Dell, usamos **F12** e escolhemos o boot UEFI do HD USB. O menu **Kali Live Persistence** iniciou corretamente.

### Verificação de persistência

```bash
echo FUNCIONOU > ~/teste.txt
sudo reboot
```

Após iniciar outra vez pelo HD:

```bash
cat ~/teste.txt
```

O resultado foi `FUNCIONOU`, confirmando que o arquivo permaneceu após reiniciar.

## Erros e aprendizados

| Problema | Resolução ou aprendizado |
|---|---|
| `e2fsck` recusou a operação | Verificar apenas a partição ext4 **desmontada** |
| GParted mostrava mudanças, mas nenhum efeito | Era necessário **aplicar operações pendentes** |
| `grub-install: unrecognized option` | Corrigir a grafia exata de `--efi-directory` |
| `grub-script-check: command not found` após reboot Ventoy | Pacote instalado na sessão Live não persistente não foi conservado |
| Terminal exibiu `zsh: bad pattern: ^[[200~` | Caracteres de controle de colagem; cancelar e redigitar |
| Kernel não encontrado em `(loop)/live/` | Extrair `vmlinuz` e `initrd.img` e alterar o `grub.cfg` |
| `/dev/sdc` mudou para `/dev/sdb` | Nomes de dispositivos **não são identificadores estáveis** |

## Segurança e limitações

- O armazenamento persistente aqui está em **ext4 não criptografado**. O LUKS de ~600 GB protege **somente sua própria partição**, quando bloqueada.
- Kali **não é anônimo por padrão**: a instalação em HD externo não oculta IP, identidade ou registros de rede.
- A ESP e os arquivos do GRUB não têm proteção adicional contra adulteração física configurada neste projeto.
- O procedimento foi testado em **um equipamento UEFI**; não foi validado em diversos computadores nem com Secure Boot ativo. Habilitá-lo sem preparo pode impedir que o GRUB personalizado inicie.
- Uma atualização convencional via `apt` não garante a atualização do kernel e initrd extraídos da ISO. Ao trocar a imagem Live, planeje atualizar ambos e revisar os caminhos de boot.
- Faça testes de segurança somente em ambientes próprios ou com permissão.

## Próximos aprimoramentos (ainda não realizados)

1. Backup externo do GRUB, ISO, dados persistentes e backup sensível/protegido do cabeçalho LUKS existente.
2. Migração planejada da persistência para **LUKS**, com backup e restauração, **sem tocar na primeira partição criptografada**.
3. Firewall, atualização controlada e isolamento dos laboratórios de testes.
4. Verificação de integridade da inicialização e análise de compatibilidade com Secure Boot.

## Referências oficiais

- [Kali — Persistência em Live USB](https://www.kali.org/docs/usb/usb-persistence/)
- [Kali — Persistência criptografada](https://www.kali.org/docs/usb/usb-persistence-encryption/)
- [Kali — Live USB no Linux](https://www.kali.org/docs/usb/live-usb-install-with-linux/)
- [Kali — Atualizações em USB](https://www.kali.org/docs/usb/updating-kali-on-usb/)
- [GNU GRUB — `grub-install`](https://www.gnu.org/software/grub/manual/grub/html_node/Invoking-grub_002dinstall.html)

---

**Observação:** este projeto demonstra uma solução validada para um cenário específico e não substitui um procedimento de migração com backup, teste de recuperação e identificação rigorosa dos discos.
