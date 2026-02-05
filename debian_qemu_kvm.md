# VIRTUALIZAÇÃO NATIVA QEMU+KVM
O Linux é capaz de criar máquinas virtuais e ele mesmo ser o hypervisor. Será um servidor de virtualização nivel 1, o mais rápido possivel, no entanto com algumas ausencia de recursos que facilitam a configuração que existem no VirtualBox e VMWare, por exemplo, criar redes virtuais com vários tipos de topologias,  clipboard e transferencia de arquivos entre host e anfitrião e outras coisas.  

## 
## Vamos instalar os pacotes principais:  
```bash
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients
sudo apt install -y libguestfs-tools
```
|Pacote|Explicação|
|:--|:--|
|libvirt-daemon-system|Configura o daemon libvirtd para gerenciar VMs via KVM.|  
|libvirt-clients|Ferramentas CLI (, virt-install, etc.).|  
|dnsmasq-bas|Fornece DHCP/NAT automáticos para redes virtuais.|  
|ovmf|Permite boot UEFI em VMs (necessário para Windows modernos).| 
|isc-dhcp-client|Utilitário para cliente DHCP|

## Permitir uso sem root
Adicione seu usuário ao grupo libvirt (e kvm):
Observe se existe  o grupo 'kvm', ele não é necessário em algumas distros, execute:
```bash
getent group kvm
```
Se ele existir, aparecerá algo como:
> kvm:x:993:

Apareceu `kvm`? Entãovamos incluir o nosso usuário no grupo 'kvm', execute:
```bash
sudo usermod -aG kvm $USER
```
Depois, verifique se realmente estou no grupo `kvm`, execute:  
```bash
getent group kvm
```  
Você deve ver:
>kvm:x:993:gsantana    

Se aparecer `kvm` e `gsantana`, então tá tudo certo e podemos prosseguir. 

>**CURIOSIDADE**: Em algumas distros, o grupo 'kvm' não existe porque não é necessário, a distro cuida disto de forma diferente, mas no caso do Debian/Ubuntu e derivações, ele precisa existir.  

Depois disso, então repetimos a operação para observar se o grupo 'libvirt' existe:  
```bash
getent group libvirt
```
É **obrigatório o grupo libvirt existir**, se não existir, algo deu muito errado nos passos anteriores, deverá aparecer algo como:
> libvirt:x:117:   

Apareceu `libvirt`? Então agora que sabemos que ele existe, então incluímos nosso usuário no grupo 'libvirt', execute:
```bash 
sudo usermod -aG libvirt $USER
```
Depois, verifique se realmente estou nestes grupos:  
```bash 
getent group libvirt
```  
Você deve ver:
>libvirt:x:117:gsantana
 
Se aparecer `libvirt` e `gsantana`, então tá tudo certo e podemos prosseguir.  

Esses acessos são dados para que o usuário possa ter acesso a arquivos e pastas que apenas o `libvirt` e `kvm` teriam.   


## USANDO VIRTUALIZAÇÃO EM MEU DESKTOP
Por tratar-se de um desktop, faça a instalação mais completa:
```bash
sudo apt install -y virt-manager virtiofsd 
sudo apt install -y virt-viewer
sudo apt install -y spice-vdagent spice-webdavd qemu-guest-agent

```
Depois:
```bash
systemctl status spice-vdagentd
```
Se o serviço estiver inativo  `Active: inactive`, então inicie ele:
Depois:
```bash
sudo systemctl start spice-vdagentd
```

|Pacote|Explicação|
|:--|:--| 
|virt-manager|Para uso em desktop ou estação de trabalho, o virt-manager é praticamente indispensável.|   
|virtiofsd|O pacote virtiofsd fornece o daemon do Virtio-FS, que é o método moderno (e mais rápido) para compartilhar pastas entre host e VMs Linux.|  
|spice-vdagent|copiar/colar e ajuste automatico de resolução|  
|spice-webdavd|arrastar e soltar arquivos (SPICE-WebDAV)|  
|qemu-guest-agent|sincronização de tempo|    

## ATIVAR O SERVIÇO DE VIRTUALIZAÇÃO NO BOOT
Se os módulos do 'kvm' aparecem então agora é o momento de prepará-los para iniciar-se como serviço durante o boot, assim, inicie o serviço do libvirtd com:  
```bash
sudo systemctl start libvirtd
```
E para iniciar o serviço durante o boot, execute:
```bash
sudo systemctl enable libvirtd
```
A resposta aguardada é:
```
Synchronizing state of libvirtd.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable libvirtd
```
Confira que o serviço esta ativado:
```bash
sudo systemctl status libvirtd
```
Um resultado assim, aparecerá:
```bash
● libvirtd.service - libvirt legacy monolithic daemon
     Loaded: loaded (/usr/lib/systemd/system/libvirtd.service; enabled; preset: enabled)
     Active: active (running) since Wed 2025-10-08 16:26:55 -03; 41s ago
 Invocation: dfc2bb59b8ae4ab3929c9385a657e489
TriggeredBy: ● libvirtd-ro.socket
             ● libvirtd-admin.socket
             ● libvirtd.socket
       Docs: man:libvirtd(8)
             https://libvirt.org/
   Main PID: 3956 (libvirtd)
      Tasks: 21 (limit: 32768)
     Memory: 6.2M (peak: 8M)
        CPU: 199ms
     CGroup: /system.slice/libvirtd.service
             └─3956 /usr/sbin/libvirtd --timeout 120

out 08 16:26:55 ti-01 systemd[1]: Starting libvirtd.service - libvirt legacy monolithic daemon...
out 08 16:26:55 ti-01 systemd[1]: Started libvirtd.service - libvirt legacy monolithic daemon.
```
Se retornou `Active: active` então tá tudo certo.

### EXECUTANDO O VIRT-MANAGER
Vamos executar o frontend para gerenciamento de nossas maquinas virtuais, o virt-manager. No menu de seu ambiente gráfico, procure por `virt-manager` e execute-o pela primeira vez. Procure na árvode de dados, QEMU/KVM, clique com o botão direito e escolha **Conectar**:  
![Executando o virt-manager pela primeira vez](img/debian_qemu_kvm01.png)

Ao fazer isso pela primeira vez, serão criadas as pastas e arquivos necessários para seu funcionamento em /var/lib/libvirt, veja:
```bash
sudo tree -ug --dirsfirst /var/lib/libvirt
```
E serão exibidas as pastas criadas e notem os donos:  
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
Porque é importante você conhecer isso? Porque num passo mais adiante precisaremos mudar esta pasta de lugar e precisaremos copiar as pastas e permissões originais.  

### VIRTUALIZAÇÃO NATIVA QEMU+KVM - PASTA PARA ARMAZENAR AS VMs e ISOs
Após, iniciar o virt-manager pela primeira vez então teremos o pool chamado **default** usado pelo libvirt que aponta para **/var/lib/libvirt/images**, essa é a localização padrão.  
A localização padrão é otima se estivessemos falando de servidores porque a pasta **/var** geralmente seria um ponto de montagem separada da `/home`, mas estamos em um desktop e de maneira geral a partição `/(root)` inclui `/var` como agregado e aí nem sempre tem espaço suficiente se o camarada usou `/home` como partição separada. 
**IMPORTANTE**: Deixar o /home como partição separada é uma boa estratégia para que novas reinstalações e upgrades, não perca os dados e reaproveite configurações anteriores.  

Então se você tem o `/(root)` com espaço curto e/ou `/home` como partição separada, recomendo que suas VMs estejam na partição com mais espaço, por exemplo, o seu `/home/libvirt`. Antigamente usariamentos um link simbolico apontando `/var/lib/libvirt/images` -> `/home/libvirt`, mas isso nos dias de hoje criaria alguns problemas com o **AppArmor**, então o que iremos fazer se chama **Bind Mount**.

#### Solução para colocar `/var/lib/libvirt/images` em `/home/libvirt`: Bind Mount
Em vez de um link simbólico, o método mais robusto e "transparente" para o sistema é o Bind Mount. Ele faz com que um diretório em outra partição apareça exatamente como se estivesse no local original, sem que o software (ou o AppArmor) precise lidar com a interpretação de links.  

1. Vamos momentaneamente desligue todas as VMs e pare o serviço do libvirt:  
```bash
sudo systemctl stop libvirtd.service libvirtd.socket
```

2. Vamos criar nossa pasta `libvirt`:  
```bash
sudo mkdir -p /home/libvirt
```
Vamos usar o comando `rsync` para replicar a pasta `libvirt` para nosso local desejado:  
```bash
sudo rsync -aX /var/lib/libvirt/ /home/libvirt/
```

3. Configurar o Bind Mount no /etc/fstab  
Renomeie o diretório antigo (como backup temporário) e crie um ponto de montagem vazio:
```bash
sudo mv /var/lib/libvirt /var/lib/libvirt.bak
```
E depois criamos a pasta vazia:  
```bash
sudo mkdir /var/lib/libvirt
```

Agora, um ponto critico, vamos adicionar ao nosso `/etc/fstab` uma instrução para montar `/var/lib/libvirt` com o conteúdo em `/home/libvirt`:  

```bash
sudo editor /etc/fstab
```
E então acrescente a linha:  
```
# bind para a pasta libvir em /home
/home/libvirt  /var/lib/libvirt  none  bind  0  0
```
Salve o arquivo e saia do editor.  
Agora, você já pode montá-la:  
```bash
sudo systemctl daemon-reload
sudo mount /var/lib/libvirt
```
**IMPORTANTE**: Nas distros atuais, boa parte delas tem o **App Armor** ou programa similar que atua com perfis rigidos de segurança que sabe diferenciar links simbolicos de pastas comuns e se ele achar que estão tentando enganá-lo então bloqueia a ação. Por essa razão estamos usadno o `bind mount`.

Agora podemos iniciar novamente o serviço:  
```bash
sudo systemctl start libvirtd.service libvirtd.socket
```

### VIRTUALIZAÇÃO NATIVA QEMU+KVM - POOLS PARA ARMAZENAR AS VMs E ISOs
O sistema trabalha no que chamamos de 'pools', o conceito é que cada pool tem um nome e aponta para uma pasta ou dispositivo, você pode ter todas as VMs no mesmo lugar, ou criar pools diferentes para cada contexto, vamos ver agora quantos pools temos no sistema, execute:  
```bash
sudo virsh pool-list --all --details
```
E então será exibo algo assim: 
```
 Nome      Estado       Auto-iniciar   Persistente   Capacidade   Alocação     Diposnível
-------------------------------------------------------------------------------------------
 default   executando   sim            sim           937,82 GiB   143,51 GiB   794,31 GiB
```
** OBSERVAÇÃO** Se o ultimo comando não listou nada, é porque voce ainda não usou o `virt-manager` e iniciou o serviço.  

Note que a capacidade exibida e disponivel, ou seja, `937,82 GiB` e `794,31 GiB` representam a nossa unidade `/home`.
Se o pool `default` estiver com o estado diferente de **executando**, então execute também:  
```bash
sudo virsh pool-start default
```
Se o pool `default` estiver com o `auto-iniciar` diferente de **sim**, então execute também:  
```bash
sudo virsh pool-autostart default
```

### Onde ficam as imagens?
As imagens, ou máquinas virtuais ficam em:  
> /home/libvirt/images    

Esse será nosso padrão, mas pode-se ter outros lugares adicionados com o comando:  
```bash
mkdir -p /outro/lugar/images
sudo chmod g+s /outro/lugar/images
sudo chown libvirt-qemu:libvirt-qemu -R /outro/lugar/images
```
Depois formalizar com:  
```bash
sudo virsh pool-define-as <nome-do-pool> dir --target /outro/lugar/images
sudo virsh pool-autostart <nome-do-pool>
sudo virsh pool-start <nome-do-pool>
```
E então teremos um `nome-do-pool` que armazena suas imagens em `/outro/lugar/images`.  
Mas gerenciar varios pools diferentes não é algo importante para o nosso caso onde o uso é Desktop. 
Algo importante de lembrar é ao importar/copiar máquinas virtuais de outros lugares para o nosso pool `default` vamos precisar mudar a permissão dela, algo como:  
```bash
# Ajusta o dono apenas para o arquivo da imagem específica
sudo chown libvirt-qemu:libvirt-qemu /var/lib/libvirt/images/sua-vm-importada.qcow2

# Garante que o grupo tenha permissão de escrita/leitura
sudo chmod 660 /var/lib/libvirt/images/sua-vm-importada.qcow2
```
Mas ficar usando chown/chmod para toda vez que importamos imagens não é muito produtivo então vamos usar ACLs no Linux. ACLs no linux são similares as DACLs no Windows, você pode fazer muito mais que as permissões primitivas octais, veja o exemplo abaixo:  

Nesta permissão ACL, definiremos que o usuário libvirt-qemu sempre terá rwx (leitura/escrita/execução) em qualquer arquivo NOVO criado na pasta `images`:  
```bash
sudo setfacl -R -m d:u:libvirt-qemu:rwx /var/lib/libvirt/images
```
Agora, a próxima ACL aplica para os arquivos que já estão lá agora:
```bash
sudo setfacl -R -m u:libvirt-qemu:rwx /var/lib/libvirt/images
```
Com uso de ACLs damos adeus a chown/chmod constantes.


### VIRTUALIZAÇÃO EM UNIDADES DO TIPO BTRFS

Se você apontou o pool `default` para um local que usa partição Btrfs, então você terá uma perda de performance se não aplicar as instruções no link abaixo:   

[Virtualização nativa do qemu+kvm usando Btrfs](debian_qemu_kvm_btrfs.md)


### VIRTUALIZAÇÃO NATIVA QEMU+KVM - Localização das ISOs  
Também precisaremos de um repositório para guardar nossas isos - arquivos de instalação de sistemas operacionais - escolha o diretorio que desejar, mas o mais bacana é não ter arquivos .iso dentro de unidades caras e rápidas como ssd, o interessante é armazená-las em discos mecânicos que são mais baratos, mas isso é apenas uma sugestão, caso você use o sistema de virtualização apenas para máquinas Windows e terá poucos isos, talvez não faça diferença onde criar este _pool_ porque apagará estes .iso depois de terminada a instalação e é este exemplo que faço abaixo.   

Nos passos anteriores, assumimos que nossas ISOs serão gravadas em:  
```
/home/libvirt/isos
```
Então criamos o pool 'isos', execute:
```bash
mkdir -p /home/libvirt/isos
```
Depois, vamos defini-la como um pool(repositório) do virtualizador:  
```bash
sudo virsh pool-define-as isos dir - - - - "/home/libvirt/isos"
```

O pool 'isos' está definido, mas precisa ser construído e iniciado(também autoiniciado após o boot), então execute:  
```
sudo virsh pool-build isos
sudo virsh pool-start isos
sudo virsh pool-autostart isos
```

Agora, vamos conferir, execute:  
```bash
sudo virsh pool-list --all
```
E será exibido algo como:
```
 Nome      Estado   Auto-iniciar
----------------------------------
 default   ativo    sim
 isos      ativo    sim
```
Ótimo, o pool **isos** está pronto, vamos ver mais detalhes dele, execute:
```bash
sudo virsh pool-dumpxml "isos" | grep -oP '(?<=<path>).*(?=</path>)'
```
E verá algo como:
```
/home/libvirt/isos
```


Se no futuro quiser mudar o lugar desse pool, você pode excluir e criar de novo, assim:
```bash
virsh pool-list --all # para listar todos e ter certeza do nome a excluir
sudo virsh pool-destroy isos  # parar com o uso do pool
sudo virsh pool-undefine isos # remover a definição do pool
```
Essa remoção do pool não mexe com os arquivos que estavam na pasta, e caso precise disso, remova-os manualmente.  
