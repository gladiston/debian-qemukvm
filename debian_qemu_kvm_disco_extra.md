# CRIANDO E ANEXANDO DISCOS EXTRAS (QCOW2) EM VMs QEMU+KVM

Este guia mostra como **criar** um novo disco virtual no pool `default` usando o terminal e como **anexar** esse disco em uma VM existente (via virt-manager ou via `virsh`). Ao final, mostro como **inicializar/formatar** o disco dentro do Windows (e uma nota rápida para Linux).

---

## 1) Criando um novo disco (volume) no pool `default`

O método recomendado aqui é criar o volume direto pelo `virsh`:

```bash
sudo virsh vol-create-as default disco-extra.qcow2 200G --format qcow2
```

Para confirmar que foi criado:

```bash
sudo virsh vol-list default
sudo virsh vol-info --pool default disco-extra.qcow2
```

Em instalações comuns do libvirt, esse arquivo costuma ficar em **`/home/libvirt/images/`** (pool `default`), mas o caminho exato depende da configuração do seu host.

---

## 2) Anexando o disco extra em uma VM (virt-manager)

1. Desligue a VM (recomendado para evitar confusão de barramentos e para o Windows detectar com mais previsibilidade).
2. Abra o **virt-manager** → selecione a VM → **Abrir** → **Mostrar detalhes**.
3. Clique em **Adicionar hardware** → **Armazenamento (Storage)**.
4. Em **Selecione ou crie um armazenamento personalizado** → **Gerenciar...** → escolha o volume `disco-extra.qcow2` no pool `default`.
5. **Barramento**: prefira **VirtIO** para desempenho.
6. Clique em **Concluir** → **Aplicar** (se aplicável).

Depois inicie a VM.

---

## 3) Anexando o disco extra em uma VM (linha de comando com `virsh`)

Descubra o nome da VM (domain):

```bash
sudo virsh list --all
```

Descubra o caminho do volume no pool `default` (para anexar pelo caminho):

```bash
sudo virsh vol-path --pool default disco-extra.qcow2
```

Anexe como disco VirtIO (exemplo usando `vdb`) e persistente:

```bash
sudo virsh attach-disk NOME_DA_VM "$(sudo virsh vol-path --pool default disco-extra.qcow2)" vdb --cache none --persistent --targetbus virtio
```

Se quiser conferir como ficou no XML:

```bash
sudo virsh dumpxml NOME_DA_VM | less
```

Observações:
- `vdb`, `vdc`, `vdd`… são destinos comuns para discos extras VirtIO (o principal geralmente fica em `vda`).
- Se o seu disco principal já usa VirtIO, manter os extras em VirtIO tende a ser o melhor caminho.

---

## 4) Inicializando e formatando o disco dentro do Windows

Após anexar e iniciar a VM:

1. Abra **Gerenciamento de Disco** (`diskmgmt.msc`).
2. O Windows deve detectar um **novo disco não inicializado**.
3. Inicialize como **GPT** (recomendado na maioria dos cenários modernos/UEFI).
4. Crie um **Novo Volume Simples**, escolha letra de unidade e formate (ex.: **NTFS**).

Pronto: o novo disco aparecerá no Explorador de Arquivos.

### Nota importante (Windows 11 + TPM = criptografia automática)

A partir do **Windows 11** (principalmente com **TPM** ativo e “Device Encryption/BitLocker” disponível), é comum que **novos volumes** (inclusive discos extras recém-adicionados) acabem sendo **criptografados automaticamente**.  
Para VMs isso costuma ser **ruim**: há **perda de desempenho** (overhead de criptografia em I/O), aumento de tempo em operações e mais atrito em tarefas como **backup/migração/otimização** de disco (`qcow2`).

Opções para evitar isso (use a que fizer mais sentido no seu cenário):

- **Desabilitar a criptografia no volume (depois de criado)**:
  - Se o Windows criptografar o disco extra automaticamente, você pode desabilitar no volume recém-criado (ex.: `D:`). Abra o **Prompt de Comando** como **Administrador** e verifique o estado:

```cmd
manage-bde -status
```

  - Se o volume extra aparecer como criptografado, desabilite a criptografia nele:

```cmd
manage-bde -off D:
```

  - Aguarde a descriptografia concluir. **Não desligue a VM** durante o processo. Para acompanhar, execute `manage-bde -status` periodicamente até o volume mostrar algo como **Totalmente descriptografado** (e porcentagem 0%).

- **Desligar a “Device encryption” / criptografia automática (antes de formatar o novo disco)**:
  - Em algumas edições, existe o caminho **Configurações → Privacidade e segurança → Criptografia do dispositivo (Device encryption)**. Se aparecer, desative.
  - Em ambientes corporativos/Pro/Enterprise, também é comum controlar isso por política; se você administra a VM via GPO, procure por políticas de **BitLocker / Device Encryption** que forçam criptografia automática.

- **Alternativa prática para VM: preparar o disco via LiveCD Linux**:
  - Dê boot por um LiveCD (ex.: GParted Live ou uma ISO Linux qualquer), particione o disco e **formate fora do Windows**.
  - Se você precisa que o Windows leia o volume, pode criar GPT e formatar em NTFS via Linux (exemplo genérico; ajuste o device certo, como `/dev/vdb1`):

```bash
sudo parted -s /dev/vdb mklabel gpt mkpart primary ntfs 1MiB 100%
sudo mkfs.ntfs -f -L "DADOS" /dev/vdb1
```

  - Ao voltar para o Windows, o volume já estará pronto e você reduz as chances de “assistentes” do Windows ativarem criptografia automática durante o fluxo de inicialização/formatação.

- **Opção de contorno**: **remover temporariamente o TPM**, inicializar/formatar o disco extra, e depois **re-adicionar o TPM** (útil quando o seu objetivo é só evitar o gatilho de criptografia automática durante o provisionamento do volume).

---

## 5) Nota rápida para VMs Linux (opcional)

Em guests Linux, após anexar o disco, você normalmente:

- identifica o dispositivo (`lsblk`, `blkid`)
- particiona (`fdisk`/`parted`)
- formata (`mkfs.ext4`/`mkfs.xfs`)
- monta e, se quiser, fixa no `/etc/fstab`

---

[Retornar ao índice](README.md)

