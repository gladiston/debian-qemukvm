# PREPARAÇÃO DO AMBIENTE UBUNTU PARA VIRTUALIZAÇÃO USANDO QEMU+KVM

Este documento tem como objetivo preparar um ambiente Debian-Like(Ubuntu) sólido e funcional para desenvolvedores, administradores de sistemas e demais profissionais de TI que queiram usar máquinas virtuais em seus computadores.  
Não se trata apenas de uma sequência de comandos que irei cuspir e você irá repetir em seu computador, meu objetivo é que você também entenda o **como e por que** cada coisa é feita.

A ideia é que, ao final, você não apenas tenha um ambiente pronto para trabalhar com máquinas virtuais, especialmente virtualizar o Windows onde geralmente há maior dificuldade. Sistemas operacionais como BSD e as distros com kernel Linux foram feitos para serem virtualizados e praticamente não há nada de especial a ser feito depois que a máquina virtual é criada, porém no Windows é completamente diferente, depois da VM Windows ser criada e o sistema estar instalado é praticado o que chamamos de _full-virtualization_, um tipo de virtualização que engloba a emulação de cada circuito do hardware e o que desejamos é a _paravirtualização_ onde tanto hospedeiro como a VM compartilham o hardware de uma forma mais inteligente evitando emulações.  

Cada seção foi escrita para ser **autoexplicativa e prática**, apresentando o comando, o motivo de usá-lo e o que esperar de seu resultado.  
Assim, mesmo que você esteja acostumado a copiar instruções da internet, este guia vai além — ele busca **fortalecer seu entendimento** sobre o funcionamento do sistema Linux e seus principais utilitários.

As instruções foram testadas em versões recentes do Ubuntu (25.10 e derivadas), mas também servem como referência para outras distribuições baseadas em Debian.  
O foco é a eficiência, a estabilidade e o domínio do ambiente de desenvolvimento, sem depender de ferramentas gráficas para tarefas que podem (e devem) ser entendidas pelo terminal.

Prepare-se para não apenas configurar o seu sistema, mas **aprender o Linux de verdade**, enquanto monta um ambiente de trabalho robusto e otimizado para o dia a dia.  

---

## Sobre o particionamento (Btrfs vs ext4)
Se o seu foco for virtualização e você pretende usar snapshots (recurso em que o Btrfs brilha), o Btrfs pode ser excelente — mas há nuances para VMs (desempenho, CoW, layout de subvolumes) que exigem atenção.
Se você não precisa de snapshots ou prefere o caminho mais simples, ext4 é uma escolha direta e estável. No tópico específico de Btrfs explico quando e por que usá-lo (e como ajustar para uso de VMs).  
Se possivel, todas as partições que contêm dados importantes devem ter um *label* como #dados1, #dados2, #disco1, #home e assim por diante, sempre sendo fáceis de serem identificados quando executarmos o comando **lsblk -f**. Colocar labels em disco é vida!  

---

## Como usar este guia
A proposta deste guia não é **decorar comandos**, mas servir como um **repositório pessoal de referência**.  
Você lê este guia passo a passo e além de instalar, aprende para quê servem o que está instalando ou configurando.  

Embora você possa ler este Guia passo a Passo diretamente do github, existe a opção de baixá-lo para lê-lo offline caso precise, eu recomendo fortemente que você **clone este projeto** localmente porque eu usei imagens nela e o github em planos gratuítos tem um limite mensal para a exibição desses objetos então se por acaso este limite for ultrapassado, não irá ver as imagens. Para clonar, execute:  
```bash
git clone https://github.com/gladiston/debianlinux.git
```
Para ler, execute:  
```bash
sudo apt install -y grip
cd debian-qemukvm
grip
```
Você verá algo como:  
> * Serving Flask app 'grip.app'  
> * Debug mode: off  
> * Running on http://localhost:6419  

Então no seu navegador, aponte para o endereço acima e verá o conteúdo desse guia.  Pessoalmente eu gosto do jeito que o github renderiza porque eles colocam um botão de copiar que economiza tempo e a renderização local não faz isso. Em contrapartida, numa cópia local, você verá sempre as imagens sem depender dos limites impostos pelo github.  

A propósito, sinta-se à vontade para **adaptar os scripts ao seu cenário** pulando o que for indesejado.  

---

## Resultado esperado
Um sistema previsível e repetível, com configurações documentadas, pronto para trabalho diário, testes e virtualização.

---

## Os padrões usados neste guia
Para o correto entendimento deste guia, usarei alguns padrões:  
**Nome do host**: ti-01  
**Nome do usuário**: gsantana  
**Nome do dominio local**: localdomain.lan  
**Debian-Like**: É o termo que uso para distro Linux baseadas em Debian que pode se referir aos vários sabores do Ubuntu e também Linux Mint, Zorin OS,... quando algo for específico para o Debian, irei realçar.   
**Ubuntu-Like**: É o termo que uso para distro Linux baseadas em Ubuntu que pode se referir aos vários sabores do Ubuntu e também Linux Mint, Zorin OS,...   
**HowTo**: É o termo que designamos para este guia passo-a-passo.    

As vezes, comandos que precisam ser executados no terminal são mesclados com o texto da saída do comando, quando isso acontecer, para que você diferencie, qual que é o comando e qual é a saída de texto dele, os comandos serão precedidos de "$", por exemplo:  
```
$ sudo apt update -y
Obter:1 https://dl.google.com/linux/chrome/deb stable InRelease [1.825 B]
Obter:2 https://dl.google.com/linux/chrome/deb stable/main amd64 Packages [1.210 B]                                           
Atingido:3 http://archive.ubuntu.com/ubuntu questing InRelease                                             
Atingido:4 http://security.ubuntu.com/ubuntu questing-security InRelease
Atingido:5 http://archive.ubuntu.com/ubuntu questing-updates InRelease
Atingido:6 http://archive.ubuntu.com/ubuntu questing-backports InRelease
Obter:7 http://archive.ubuntu.com/ubuntu questing/main Translation-pt_BR [349 kB]
Obter:8 http://archive.ubuntu.com/ubuntu questing/main Translation-pt [161 kB]
Obter:9 http://archive.ubuntu.com/ubuntu questing/universe Translation-pt [823 kB]
Obter:10 http://archive.ubuntu.com/ubuntu questing/universe Translation-pt_BR [1.668 kB]
Obter:11 http://archive.ubuntu.com/ubuntu questing/restricted Translation-pt_BR [584 B]
Obter:12 http://archive.ubuntu.com/ubuntu questing/restricted Translation-pt [588 B]
Obter:13 http://archive.ubuntu.com/ubuntu questing/multiverse Translation-pt [7.720 B]
Obter:14 http://archive.ubuntu.com/ubuntu questing/multiverse Translation-pt_BR [18,3 kB]
Obtidos 3.031 kB em 4s (848 kB/s)                                              
Todos os pacotes estão atualizados.         
Nota: Algumas fontes podem ser modernizadas. Execute 'apt modernize-sources' para fazer isso.
```

---
## UBUNTU vs DEBIAN: QUAL ESCOLHER PARA SUA PRODUTIVIDADE
As duas distribuições mais influentes no ecossistema Linux são **Debian** e **Ubuntu**, sendo o Ubuntu uma derivação direta do Debian otimizada para usuários domésticos e corporativos.  Você conhece as diferenças? Qual escolher para sua produtividade?  No link abaixo, respondo esta pergunta.  


[UBUNTU vs DEBIAN: Qual Escolher para Sua Produtividade?](https://github.com/gladiston/debianlinux/blob/main/docs/debian_ubuntu.md)  

---

## GNOME ou KDE PLASMA: QUAL O MELHOR DESKTOP?
Em sistemas Linux, o **ambiente de trabalho** (ou **Desktop Environment - DE**) é a camada gráfica que interage diretamente com o usuário. Diferentemente do Windows ou macOS, que possuem uma interface fixa, **Linux oferece múltiplas opções de ambientes gráficos**, cada um com filosofia, funcionalidade e estética próprias. As mais populares no Linux são: KDE e GNOME, mas qual as diferenças?

Curioso para saber minha opinião? Clique  no link abaixo:  
[GNOME ou KDE PLASMA: Qual o melhor desktop?](https://github.com/gladiston/debianlinux/blob/main/docs/debian_kde_gnome.md)  


---

## PARTICIONAMENTO DURANTE A INSTALAÇÃO DO DEBIAN/UBUNTU
A instalação do Debian/Ubuntu não tem grandes mistérios — o ponto mais delicado é mesmo o **particionamento do disco**.  
No link abaixo explico melhor essa questão, é bom que leia antes de fazer a instalação. Segue:  

[Particionamento durante a instalação do Debian/Ubuntu](https://github.com/gladiston/debianlinux/blob/main/docs/debian_part.md)


---

# VIRTUALIZAÇÃO NATIVA QEMU+KVM
O Linux é capaz de criar máquinas virtuais e ele mesmo ser o hypervisor. Será um servidor de virtualização nivel 1, o mais rápido possivel, no entanto com algumas ausencia de recursos que facilitam a configuração que existem no VirtualBox e VMWare, por exemplo, criar redes virtuais com vários tipos de topologias,  clipboard e transferencia de arquivos entre host e anfitrião e outras coisas.  

## Vamos fazer o backup?
Antes de prosseguirmos, que tal fazer o backup da sua configuração de rede original?  Isso é importante porque assim que instalar o `libvirt` ele vai criar uma interface NAT para virtualizar e isso vai mudar o ambiente inicial, se quiser saber como fazer isso, siga as instruções no link a seguir:  
[Fazendo um backup das configurações de rede](debian_backup_restore_network.md)  

## VIRTUALIZAÇÃO NATIVA QEMU+KVM - CONFERINDO MÓDULOS
Agora, verifique se os módulos do KVM estão carregados no kernel:
```bash
lsmod | grep kvm
```
Uma saída aceitável seria:  
```
kvm_amd               217088  0
kvm                  1396736  1 kvm_amd
irqbypass              12288  1 kvm
ccp                   163840  1 kvm_amd
```
Se constar na lista o módulo `kvm` e `kvm_amd` ou `kvm_intel`, então tá tudo certo e pode prosseguir para o próximo tópico.    
Caso não apareça nada, então é hora de ir na BIOS de seu computador e habilitar o suporte à virtualização. Geralmente essa configuração fica escondida na BIOS sob um desses nomes: Advanced, Processor, CPU Configuration ou Chipset. O suporte para virtualização é chamado de nomes diferentes entre AMD e Intel, veja:  

|Fabricante|Nome da Opção na BIOS|
|:--|:--|
|Intel|Intel Virtualization Technology ou VT-x|
|AMD|SVM Mode (Secure Virtual Machine) ou AMD-V|

Boa sorte, nem sempre é fácil encontrá-los na BIOS. Depois que conseguir um teste positivo, então prossiga.  


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
![Executando o virt-manager pela primeira vez](../img/debian_qemu_kvm01.png)

Ao fazer isso pela primeira vez, serão criadas as pastas e arquivos necessários para seu funcionamento em /var/lib/libvirt, veja:
```bash
cd /var/lib/libvirt
sudo  sudo tree -ug --dirsfirst
```
E serão exibidas as pastas criadas e notem os donos:  
```
[root     root    ]  .
├── [root     root    ]  boot
├── [root     root    ]  images
└── [libvirt-qemu libvirt-qemu]  qemu
    ├── [libvirt-qemu libvirt-qemu]  checkpoint
    ├── [libvirt-qemu libvirt-qemu]  dump
    ├── [libvirt-qemu libvirt-qemu]  nvram
    ├── [libvirt-qemu libvirt-qemu]  ram
    ├── [libvirt-qemu libvirt-qemu]  save
    └── [libvirt-qemu libvirt-qemu]  snapshot

10 directories, 0 files
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
sudo mkdir /var/lib/libvirt
```

Agora, um ponto critico, vamos adicionar ao nosso `/etc/fstab` uma instrução para montar `/var/lib/libvirt` com o conteúdo em `/home/libvirt`:  

```bash
sudo editor /etc/fstab
```
E então acrescente a linha:  
```
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
sudo chown -R /outro/lugar/images
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
```
mkdir -p /home/libvirt/isos
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

### VIRTUALIZAÇÃO NATIVA QEMU+KVM - CRIANDO UMA INTERFACE BRIDGE
Para trabalhos extensos e mais profissionais com VMs é impossivel viver apenas com NAT porque na maioria dos ambientes de desenvolvimento ou corporativos uma VM precisa enxergar o anfitrião e também as outras VMs, então siga o tutorial a seguir para criar uma conexão do tipo bridge em seu sistema:  

[Criando conexões bridge pelo terminal](debian_qemu_kvm_bridge.md)   

É claro que se você está lendo este tutorial e apenas quer usar o Windows com o NAT, não precisará de uma conexão de bridge.  
  

### VIRTUALIZAÇÃO NATIVA QEMU+KVM - Windows
A virtualização com QEMU+KVM oferece desempenho quase nativo, aproveitando recursos do processador e integração direta com o kernel Linux. É uma opção poderosa e leve para executar Windows dentro do Linux, especialmente quando usada com o Virt-Manager, que simplifica a criação e o gerenciamento de VMs.  

Neste guia, você verá como criar uma máquina virtual Windows otimizada, usando o pacote de drivers virtio-win para melhor desempenho de disco, rede e vídeo, além de boas práticas sobre discos dinâmicos, rede bridge e configurações essenciais.  

Agora que você entende o contexto, vamos começar preparando o ambiente com o pacote de drivers VirtIO. Siga as instruções no link abaixo:  

[Criando e ajustando VM Windows](debian_qemu_kvm_windows.md)


### VIRTUALIZAÇÃO NATIVA QEMU+KVM - Criando máquinas virtuais pelo Virt-Manager
Instruções de como usar o virt-manager encontra-se na página:  
[Criando máquinas virtuais pelo Virt-Manager](https://sempreupdate.com.br/como-configurar-e-usar-o-virt-manager-para-kvm-no-fedora-ubuntu-debian-e-derivados/#google_vignette)   


### EXECUTANDO MAQUINAS VIRTUAIS VIA VIRT-VIEWER
Você provavelmente está usando o `virt-manager` para iniciar e encerrar suas VMs e não tem nada de errado com isso, no entanto, há um aplicativo mais otimizado para isso, ele se chama `virt-viewer`.  
Alguns serviços, especialmente para VM Windows, esperam que você use o `virt-viewer` sem o `virt-manager` acionado.  
Veja no link abaixo como usufruir do `virt-viewer` ao invés do virt-manager:

[Executando VM usando virt-viewer](debian_qemu_kvm_virtviewer.md)  


### OTIMIZANDO O DISCO QCOW2
O QCOW2 é um formato copy-on-write com recursos como snapshots, compressão e alocação sob demanda. Esses recursos trazem overhead e, com o tempo, geram fragmentação interna. Mas máquinas Windows são muito mais afetadas do que as demais porque o Windows gera memória virtual, arquivos temporários a todo instante. Então o link a seguir descreve como podemos otimizar e compactar o disco virtual para que o desempenho - especialmente para VMs Windows - fique sempre máximizado.  

[Instruções para melhorar o desempenho](debian_qemu_kvm_otimizar_disco.md)

### Backup de Máquinas Virtuais em QEMU+KVM
**Backup de máquina virtual** é a replicação sistemática dos arquivos de disco (imagens QCOW2, RAW, VDI, etc.) e metadados de configuração (arquivos XML do libvirt) para um local independente, garantindo recuperação em caso de corrupção, falha de hardware, exclusão acidental ou desastre. Diferencia-se de snapshots, que são pontos de restauração locais; backups são cópias isoladas em mídia ou storage separado.  

Para entender melhor e aprender como fazer siga o link:  
[Realizando backups de maquinas virtuais](debian_qemu_kvm_backup.md)  


### Convertendo máquinas virtuais do VirtualBox para QEMU+KVM
O VirtualBox é um hipervisor do tipo 2 amplamente utilizado para virtualização em desktops. As vezes é preciso migrar máquinas virtuais desta plataforma para o QEMU+KVM. A conversão funciona, mas levamos junto o lixo da instalação anterior e nem sempre temos os melhores resultados, no entanto, é melhor do que criar tudo do zero. Siga o link abaixo se precisar migrar máquinas virtuais do VirtualBox:  

[Convertendo máquinas virtuais do VirtualBox para QEMU/KVM](debian_qemu_kvm_vbox.md)  
