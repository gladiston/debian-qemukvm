# Virtualização nativa QEMU/KVM

O Linux pode atuar como **hypervisor tipo 1**: as máquinas virtuais rodam com desempenho próximo do hardware, mas o ecossistema **libvirt** + **QEMU/KVM** é mais “baixo nível” que produtos como VirtualBox ou VMware em alguns confortos (topologias de rede prontas, clipboard integrado, pastas compartilhadas sem configurar Virtio-FS/SPICE, etc.). Este guia parte do Debian e derivações com `apt`.

## Pacotes principais do hypervisor

```bash
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients
sudo apt install -y libguestfs-tools
```

O `apt` costuma instalar **dependências** (rede virtual, firmware UEFI, utilitários). A tabela resume o que cada nome costuma representar na prática:

| Pacote | Explicação |
|:------- |:---------- |
| **qemu-kvm** | Emulação de CPU/dispositivos e aceleração **KVM** quando o hardware suporta (`/dev/kvm`). É o núcleo que de fato executa a VM. |
| **libvirt-daemon-system** | Serviço **libvirtd**: API estável para criar, iniciar e gerenciar domínios (VMs), redes e armazenamento. |
| **libvirt-clients** | Ferramentas de linha de comando (`virsh`, `virt-install`, entre outras) que falam com o libvirt. |
| **libguestfs-tools** | Utilitários **libguestfs** para trabalhar **em imagens de disco** sem necessariamente **ligar** a VM: inspecionar sistemas de arquivos (`virt-filesystems`, `virt-inspector`), copiar arquivos para dentro/fora da imagem (`virt-copy-in`, `virt-copy-out`), redimensionar disco (`virt-resize`), abrir **guestfish** para navegar partições e arquivos, preparar imagem para compactação (`virt-sparsify`), etc. Úteis para manutenção, backup e scripts. |
| **dnsmasq-base** *(dependência comum)* | Base para **DNS/DHCP** leve; o libvirt usa isso na rede **NAT padrão** (`virbr0`) para dar IP às VMs. |
| **ovmf** *(dependência comum)* | Firmware **EDK II/OVMF** para arranque **UEFI** em VM — quase obrigatório para **Windows** recente e muitos Linux em modo EFI. |
| **isc-dhcp-client** *(dependência comum)* | Cliente DHCP no hospedeiro; aparece na cadeia de dependências do ecossistema de rede. |

## Permitir uso sem root

Para administrar VMs com o **usuário** normal (sem `sudo` constante), o **hospedeiro** precisa pertencer aos grupos **`kvm`** (acesso a `/dev/kvm`) e **`libvirt`** (API do libvirt). No Debian e Ubuntu o grupo **`kvm`** costuma existir; em outras famílias o nome ou o mecanismo pode diferir.

Confira se existe o grupo `kvm`:

```bash
getent group kvm
```

Se aparecer uma linha do tipo `kvm:x:993:`, inclua o usuário:

```bash
sudo usermod -aG kvm $USER
```

Valide de novo:

```bash
getent group kvm
```

O ideal é ver o seu login no fim da linha (por exemplo `kvm:x:993:gsantana`). Em seguida faça o mesmo raciocínio para **`libvirt`**:

```bash
getent group libvirt
```

Deve existir algo como `libvirt:x:117:`. Inclua o usuário:

```bash
sudo usermod -aG libvirt $USER
getent group libvirt
```

Confirme que o login aparece em ambos os grupos. **Feche a sessão gráfica ou reinicie** (ou use `newgrp`) para o shell atual herdar os grupos novos.

Sem estes grupos, o usuário não **acessa** os mesmos *sockets* e dispositivos que o daemon usa, e o **virt-manager** / **virsh** podem falhar ou pedir senha de administrador.   

## Desktop: interface gráfica e integração com o convidado

Em estação de trabalho vale instalar também o **virt-manager**, o **virtiofsd** (compartilhamento **Virtio-FS** com convidados que suportam), o **virt-viewer** (janela SPICE/VNC sem o painel completo do virt-manager) e os agentes **SPICE** / **qemu-guest-agent** — estes últimos **dentro do convidado** Linux (via `apt` no guest), para clipboard, resolução e relógio.

No **hospedeiro**:

```bash
sudo apt install -y virt-manager virtiofsd virt-viewer
```

No **Linux convidado** (quando aplicável), instale os agentes e confira o serviço:

```bash
sudo apt install -y spice-vdagent spice-webdavd qemu-guest-agent
systemctl status spice-vdagentd
```

Se estiver inativo (`Active: inactive`), suba o daemon:

```bash
sudo systemctl start spice-vdagentd
```

| Pacote | Explicação |
|:------- |:---------- |
| **virt-manager** | Interface gráfica para criar e gerenciar domínios, redes e volumes; ponto de partida habitual no desktop. |
| **virtiofsd** | Daemon **Virtio-FS** no hospedeiro: compartilhar pastas com baixa latência (convidado precisa de driver e, no Windows, **WinFsp**). |
| **virt-viewer** | Visualização SPICE/VNC em janela dedicada (útil para testes ou segundo monitor). |
| **spice-vdagent** *(guest)* | Clipboard integrado e redimensionamento automático da resolução do guest com o tamanho da janela SPICE. |
| **spice-webdavd** *(guest)* | Canal **WebDAV** via SPICE para transferir arquivos (alternativa a Virtio-FS). |
| **qemu-guest-agent** *(guest)* | Canal **qemu-ga** para operações coordenadas (snapshot consistente, relógio, rede, etc.). |

## Ativar o libvirt no arranque

Com KVM e pacotes instalados, inicie o daemon e ative-o no boot:

```bash
sudo systemctl start libvirtd
sudo systemctl enable libvirtd
```

O `enable` pode mostrar linhas sobre *SysV* — é normal. Confirme:

```bash
sudo systemctl status libvirtd
```

Procure **`Active: active (running)`** e serviço **enabled**. Em versões recentes também pode haver *sockets* `libvirtd.socket`; o conjunto deve permitir `virsh list` sem erro de permissão (após grupos e nova sessão).

### Primeira execução do virt-manager

Abra o **virt-manager**. Na árvore à esquerda, em **QEMU/KVM**, clique com o botão direito e escolha **Conectar**:  
![Executando o virt-manager pela primeira vez](img/debian_qemu_kvm01.png)

Na primeira conexão, o libvirt cria a árvore sob **`/var/lib/libvirt`**. Vale inspecionar donos e grupos (útil antes de mudar o pool para outro disco):

```bash
sudo tree -ug --dirsfirst /var/lib/libvirt
```

Exemplo de saída:

```
[root     root    ]  /var/lib/libvirt
├── [root     root    ]  boot
├── [root     root    ]  dnsmasq
│   ├── [root     root    ]  default.addnhosts
│   ├── [root     root    ]  default.conf
│   ├── [root     root    ]  default.hostsfile
│   └── [root     root    ]  virbr0.status
├── [root     root    ]  images
├── [libvirt-qemu kvm     ]  qemu
│   ├── [libvirt-qemu kvm     ]  checkpoint
│   ├── [libvirt-qemu kvm     ]  dump
│   ├── [libvirt-qemu kvm     ]  nvram
│   ├── [libvirt-qemu kvm     ]  ram
│   ├── [libvirt-qemu kvm     ]  save
│   └── [libvirt-qemu kvm     ]  snapshot
└── [root     root    ]  sanlock

12 directories, 4 files
```

Mais abaixo, se você mover o armazenamento para `/home`, será preciso **reproduzir donos e permissões**; esta árvore é a referência.

### Onde o pool `default` guarda discos por omissão

Depois da primeira ligação ao **QEMU/KVM**, o libvirt expõe o *pool* **`default`**, normalmente em **`/var/lib/libvirt/images`**. Em **servidor**, `/var` muitas vezes está num volume grande e dedicado; em **desktop** costuma **compartilhar o mesmo disco** que `/` e ficar **apertado** se `/home` estiver em outro disco com muito mais espaço.

**Nota:** ter **`/home` numa partição separada** facilita reinstalar o sistema sem perder dados do usuário.

Se quiser as imagens **num caminho com mais espaço** (ex.: **`/home/libvirt`**) sem link simbólico (o **AppArmor** e políticas de segurança tratam *bind mount* de forma mais previsível que symlinks para `/var/lib/libvirt`), use **bind mount**: o kernel mostra o conteúdo de `/home/libvirt` **no sítio** `/var/lib/libvirt`, transparente para o libvirt.

#### Bind mount: `/var/lib/libvirt` → `/home/libvirt`

**Parar tudo** — desligue as VMs e pare o libvirt:

```bash
sudo systemctl stop libvirtd.service libvirtd.socket
```

**Replicar dados** para o destino definitivo:

```bash
sudo mkdir -p /home/libvirt
sudo rsync -aX /var/lib/libvirt/ /home/libvirt/
```

**Preparar o ponto de montagem** — guarde a árvore antiga e renomeie para não perder nada até confirmar:

```bash
sudo mv /var/lib/libvirt /var/lib/libvirt.bak
sudo mkdir /var/lib/libvirt
```

**`/etc/fstab`** — adicione a linha do bind (ajuste o editor se preferir `nano`):  

```bash
sudo editor /etc/fstab
```

```
# bind: conteúdo real em /home/libvirt, visível em /var/lib/libvirt
/home/libvirt  /var/lib/libvirt  none  bind  0  0
```

Guarde, recarregue unidades e monte:

```bash
sudo systemctl daemon-reload
sudo mount /var/lib/libvirt
```

Quando confirmar que tudo funciona, pode apagar **`/var/lib/libvirt.bak`**. **AppArmor** (ou equivalente) costuma **confiar** em bind mount ao caminho **canônico** esperado; por isso evitamos symlink aqui.

Reinicie o libvirt:  

```bash
sudo systemctl start libvirtd.service libvirtd.socket
```

### Pools de armazenamento (VMs e ISOs)

Cada **pool** tem um nome e aponta para um diretório (ou outro tipo de armazenamento). Liste com detalhe:

```bash
sudo virsh pool-list --all --details
```

Exemplo de saída (nomes de estado dependem do idioma da instalação):

```
 Nome      Estado       Auto-iniciar   Persistente   Capacidade   Alocação     Disponível
-------------------------------------------------------------------------------------------
 default   executando   sim            sim           937,82 GiB   143,51 GiB   794,31 GiB
```

**Observação:** se não aparecer nenhum pool, verifique se o **libvirt** está ativo e se o **virt-manager** já conectou ao **QEMU/KVM** pelo menos uma vez.

**Capacidade** e **disponível** referem-se ao volume onde o *target* do pool está montado (por exemplo `/home` após o bind mount acima).
Se o estado do pool **`default`** não for **running** (ou **executando** no locale em português), inicie:

```bash
sudo virsh pool-start default
```

Se **autostart** não estiver ativo, habilite:  

```bash
sudo virsh pool-autostart default
```

### Onde ficam as imagens das VMs?

Com o cenário deste guia, os discos `.qcow2` (ou `.img`) costumam ficar em:

> `/home/libvirt/images`

É possível definir **outro** diretório como pool:

```bash
sudo mkdir -p /outro/lugar/images
sudo chmod g+s /outro/lugar/images
sudo chown -R libvirt-qemu:libvirt-qemu /outro/lugar/images
```

Registre o pool no libvirt:

```bash
sudo virsh pool-define-as nome-do-pool dir --target /outro/lugar/images
sudo virsh pool-autostart nome-do-pool
sudo virsh pool-start nome-do-pool
```

Para uso só em desktop, um pool **`default`** bem dimensionado costuma bastar.

Ao **importar** uma imagem copiada de fora para o pool **`default`**, ajuste dono e modo para o QEMU conseguir abrir:  

```bash
# Ajusta o dono apenas para o arquivo da imagem específica
sudo chown libvirt-qemu:libvirt-qemu /var/lib/libvirt/images/sua-vm-importada.qcow2

# Garante que o grupo tenha permissão de escrita/leitura
sudo chmod 660 /var/lib/libvirt/images/sua-vm-importada.qcow2
```

Em vez de **`chown`** a cada importação, use **ACL** no diretório **`images`**: regra **default** para arquivos e pastas **novos** e regra para o que **já existe**.

```bash
sudo setfacl -R -d -m u:libvirt-qemu:rwx /var/lib/libvirt/images
sudo setfacl -R  -m u:libvirt-qemu:rwx /var/lib/libvirt/images
```

(Ajuste o caminho se o *target* do pool for outro; após bind mount, `/var/lib/libvirt/images` e `/home/libvirt/images` apontam para o mesmo conteúdo.)

### Pool em sistema de arquivos Btrfs

Se o *target* do pool estiver em **Btrfs**, vale ajustar *copy-on-write* e desempenho conforme o guia dedicado:

[Virtualização nativa QEMU/KVM com Btrfs](debian_qemu_kvm_btrfs.md)

### Pool de ISOs

ISOs de instalação são grandes e pouco usadas depois da instalação; muita gente **as guarda** em disco **mais barato** (HDD) e deixa **SSDs** para imagens de VM — é sugestão, não regra.

Este exemplo usa **`/home/libvirt/isos`**:

```bash
mkdir -p /home/libvirt/isos
sudo virsh pool-define-as isos dir - - - - "/home/libvirt/isos"
```

Ative e deixe no boot:

```bash
sudo virsh pool-build isos
sudo virsh pool-start isos
sudo virsh pool-autostart isos
```

Confira:

```bash
sudo virsh pool-list --all
```

Confirme o caminho XML:

```bash
sudo virsh pool-dumpxml "isos" | grep -oP '(?<=<path>).*(?=</path>)'
```

Para **redefinir** o pool (os arquivos na pasta **não** são apagados pelo libvirt):

```bash
virsh pool-list --all
sudo virsh pool-destroy isos
sudo virsh pool-undefine isos
```

Apague os `.iso` manualmente na pasta se quiser liberar espaço.  
