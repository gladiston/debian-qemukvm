# Virtualização nativa QEMU/KVM

O Linux pode atuar como **hypervisor tipo 1**: as máquinas virtuais rodam com desempenho próximo do hardware, mas o ecossistema **libvirt** + **QEMU/KVM** é mais “baixo nível” que produtos como VirtualBox ou VMware em alguns confortos (topologias de rede prontas, clipboard integrado, pastas compartilhadas sem configurar Virtio-FS/SPICE, etc.). Este guia parte do Debian e derivações com `apt`.

## Conferência inicial

Por padrão, o virtualizador deposita suas máquinas virtuais e agregados em: 

```
/var/lib/libvirt
```
A pasta acima já existe em seu sistema?
Se sim, o qem+kvm já foi instalado. Isso pode ter sido feito durante a instalação limpa ou depois, mas não importa, neste caso você terá de pular as instruções e seguir o artigo:
[Movendo QEMU+KVM pre-instalado para /home](debian_qemu_kvm_home.md)  

## Criando a pasta-base

E isso está correto para servidores onde **/var** é uma partição/disco em separado, porém nossa máquina é um desktop e geralmente não usamos **/var** porque ele é um agregadinho do **/(root)** onde deixamos um espaço livre minimo para apenas a instalação do Linux. Assim, eu recomendo que façamos uma mudança para:

```
/home/libvirt
```
Nosso **/home** , geralmente foi formatado para ser uma partição em separado e durar para sempre e por isso, geralmente é a partição que terá bem mais espaço disponível, então vamos, criar as pastas:

```bash
sudo mkdir -p /var/lib/libvirt
sudo mkdir -p /home/libvirt
```
Nossa pasta `/home/libvirt` precisa de permissõs adequadas:
```bash
sudo chmod 755 /home/libvirt
```
Agora precisaremos de um **bind mount,** isto é, fazer com que **/var/lib/libvirt** seja um ponto de montagem para **/home/libvirt**, assim, quando instalarmos nossa ferramenta de virtualização, a mesma seguirá seu roteira padrão pensando que seus arquivos estão sendo depositados em **/var/lib/libvirt**, mas na verdade estará em **/home/libvirt**. Execute:

```bash
sudo editor /etc/fstab
```

E adicione a seguinte linha ao final deste arquivo:

```
# bind: conteúdo real em /home/libvirt, visível em /var/lib/libvirt
/home/libvirt  /var/lib/libvirt  none  bind  0  0
```
Salve o arquivo e saia do editor.
Porque usar `bind mount`? Porque programas como o **AppArmor** (ou equivalente) costuma **confiar** em bind mount ao caminho **canônico** esperado, teríamos problemas se usassemos  symlink aqui.

Depois vamos recarregar a configuração do /etc/fstab, execute:

```bash
sudo systemctl daemon-reload
```

Agora vamos montá-lo:

```bash
sudo mount /var/lib/libvirt
```

E assim, teremos uma pasta com bind mount, criando a ilusão para o libvirt de que ele está usando /var/lib/libvirt. Mas agora precisamos nos precaver de nós mesmos, quando alguém olhar /var/lib/libvirt e olhar /home/libvirt poderá achar que se trata de uma duplicação, então para minimizar esse dano, vamos criar um arquivo de aviso, por isso, seu nome estará em maiusculo, execute:

```bash
sudo editor /var/lib/libvirt/NAO_ME_APAGUE.txt
```

E cole o seguinte conteúdo:

```
Este é um ponto de montagem de/para:
/var/lib/libvirt para /home/libvirt
NÃO SÃO DUAS PASTAS DUPLICADAS, MAS ESPELHADAS
Se apagar daqui, estará apagando de lá.
```

Salve o arquivo e saia para o terminal.

Pode parecer um aviso idiota, mas a mente de cada um funciona de um jeito diferente e avisos assim são importantes.

## Pacotes principais do hypervisor

```bash
sudo apt install tree libvirt-daemon-system libvirt-clients \
  libguestfs-tools ovmf isc-dhcp-client dnsmasq-base swtpm swtpm-tools \
  qemu-system-modules-spice qemu-utils qemu-system-gui -y
```

O `apt` costuma instalar **dependências** (rede virtual, firmware UEFI, utilitários). A tabela resume o que cada nome costuma representar na prática:

| Pacote                                    | Explicação                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **qemu-kvm**                              | Emulação de CPU/dispositivos e aceleração **KVM** quando o hardware suporta (`/dev/kvm`). É o núcleo que de fato executa a VM.                                                                                                                                                                                                                                                                                                                             |
| **libvirt-daemon-system**                 | Serviço **libvirtd**: API estável para criar, iniciar e gerenciar domínios (VMs), redes e armazenamento.                                                                                                                                                                                                                                                                                                                                                   |
| **libvirt-clients**                       | Ferramentas de linha de comando (`virsh`, `virt-install`, entre outras) que falam com o libvirt.                                                                                                                                                                                                                                                                                                                                                           |
| **libguestfs-tools**                      | Utilitários **libguestfs** para trabalhar **em imagens de disco** sem necessariamente **ligar** a VM: inspecionar sistemas de arquivos (`virt-filesystems`, `virt-inspector`), copiar arquivos para dentro/fora da imagem (`virt-copy-in`, `virt-copy-out`), redimensionar disco (`virt-resize`), abrir **guestfish** para navegar partições e arquivos, preparar imagem para compactação (`virt-sparsify`), etc. Úteis para manutenção, backup e scripts. |
| **dnsmasq-base** *(dependência comum)*    | Base para **DNS/DHCP** leve; o libvirt usa isso na rede **NAT padrão** (`virbr0`) para dar IP às VMs.                                                                                                                                                                                                                                                                                                                                                      |
| **ovmf** *(dependência comum)*            | Firmware **EDK II/OVMF** para arranque **UEFI** em VM — quase obrigatório para **Windows** recente e muitos Linux em modo EFI.                                                                                                                                                                                                                                                                                                                             |
| **isc-dhcp-client** *(dependência comum)* | Cliente DHCP no hospedeiro; aparece na cadeia de dependências do ecossistema de rede.                                                                                                                                                                                                                                                                                                                                                                      |

## Permitir uso sem root

Para administrar VMs com o **usuário** normal (sem `sudo` constante), o **hospedeiro** precisa pertencer aos grupos **`kvm`** (acesso a `/dev/kvm`) e **`libvirt`** (API do libvirt). No Debian e Ubuntu o grupo **`kvm`** costuma existir; em outras famílias o nome ou o mecanismo pode diferir.

Confira se existe o grupo `kvm`:

```bash
getent group kvm
```

Verá algo como: 

> kvm:x:993  
ou  
> kvm:x:991:libvirt-qemu  

O comando acima indica quais usuários estão no grupo `kvm`, note que nosso usuário não está nele, então vamos incluí-lo:


```bash
sudo usermod -aG kvm $USER
sudo newgrp
```
Agora, vamos confirmar:
```bash
getent group kvm
```
O resultado agora incluirá o nosso usuário(ex gsantana):  
> kvm:x:993,**gsantana**    
ou  
> kvm:x:991:libvirt-qemu,**gsantana**  
  
Ao ver o seu login no fim da linha (por exemplo `gsantana`) significa que tudo deu certo. 
Em seguida aplique o mesmo raciocínio para **`libvirt`**:

```bash
getent group libvirt
```
Listará algo como:
> libvirt:x:975  
Se não apareceu seu usuário(ex `gsantana`), então precisará incluí-lo:
  
```bash
sudo usermod -aG libvirt $USER
sudo newgrp
```
Depois confira novamente:
```bash
getent group libvirt
```
Listará algo como:
> libvirt:x:975:gsantana  
Ao ver o seu login no fim da linha (por exemplo `gsantana`) significa que tudo deu certo. 

Sem acesso a estes grupos, o usuário não **acessa** os mesmos recursos que o daemon usa, e o **virt-manager** / **virsh** podem falhar ou pedir senha de administrador.

## Desktop: interface gráfica e integração com o convidado

O hypervisor funciona em forma de backend e serviço, ou seja, sua interatividade com o serviço de virtualização é apenas pelo terminal e para alguns de nós isso é uma 'sofrência' que dá dó. Mesmo em servidores usamos um sistema de gerenciamento com um frontend agradável como o **Proxmox**  para gerenciá-lo sem precisar requerer ao terminal.

Em estação de trabalho - como nosso caso - há outros como `gnome-boxes` e `Cockpit`, porém, o mais popular é o `virt-manager`, vamos instalá-lo:

```bash
sudo apt install virt-manager spice-client-gtk gir1.2-spiceclientgtk-3.0 -y
```

Com o `virt-manager` voce cria, altera e exclui suas VMs. Ele acompanha um viewer embutido, assim, ao criar por exemplo uma VM com o Windows, você poderá interagir com ele, como formatar e instalar o sistema. Mas existe outro viewer que é um pouco melhor, e na realidade, alguns serviços de interatividade só funcionam com ele, chama-se `virt-viewer`, vamos instalá-lo:

```bash
sudo apt install virt-viewer -y
```

### OBRIGATÓRIO: qemu-guest-agent

Canal **qemu-ga** para operações coordenadas (snapshot consistente, relógio, rede, etc.).  Instale:

```bash
sudo apt install qemu-guest-agent -y
```

### OBRIGATÓRIO: SPICE-VDAGENT

Ao criar uma VM, em algum momento você configurará a monitor como do tipo SPICE, daí voce poderá ter um clipboard integrado e redimensionamento automático da resolução da tela do convidado com o tamanho da janela.  Instale:

```bash
sudo apt install spice-vdagent -y
```

Depois de instalá-lo, confira se o serviço está habilitado:

```bash
sudo systemctl status spice-vdagentd
```

E caso esteja inoperante, você o inicia:

```bash
sudo systemctl start spice-vdagentd
```

Se o serviço não estiver ativo, tampouco o `spice-vdagentd` funcionará nas VMs.

### OPCIONAL: VIRTIOFSD

O pacote virtiofsd serve para compartilhar uma pasta do hospedeiro lINUX com a máquina virtual (Windows e Linux) usando virtio-fs. Em termos simples, ele é o daemon do lado do host que implementa esse compartilhamento para o QEMU/KVM. 

Não, ele não é obrigatório em toda instalação de QEMU/KVM.
Um detalhe importante: o virtiofsd é do lado do host, mas a VM também precisa de suporte ao virtio-fs para montar esse compartilhamento. Em alguns casos, recursos como migração/snapshot com memória podem ter restrições dependendo da versão do virtiofsd

No **hospedeiro**:

```bash
sudo apt install virtiofsd -y
```

Se vocÊ instalá-lo, lembre-se de que no Windows precisará também dos drivers `virtiofsd` para que ele possa enxergar as pastas que compartilhou com o host. Pessoalmente, no Windows, eu acho ele problemático com algumas aplicações. Como arquivos Linux são case sensitive, isso significa que podem existir arquivos de mesmo nome com maiusculas e minusculas diferente, e alguns aplicativos Windows se perdem com isso, por exemplo o **Rad  Studio Delphi e C++ Builder**.

### OPCIONAL: WEBDAV

Nas configurações da VM, você pode criar um canal  **WebDAV** via SPICE para transferir arquivos entre hospedeiro e convidado, é similar ao Virtio-FS, mas para fazer este compartilhamento usa-se o protocolo HTTP/HTTPS. Este tipo de compartilhamento é conhecido pelos programadores, ele é bem mais lento que o `Virtio-FS` e geralmente você só usaria ele com projetos de programação bem estruturados que funcione muito bem offline, mas que no final, precise sincronizar seus arquivos. Usá-lo como unidade de rede é praticamente inviável. Para tê-lo, instale:

```bash
sudo apt install spice-webdavd -y
```

Depois de instalá-lo, confira se o serviço está habilitado:

```bash
sudo systemctl status spice-vdagentd
sudo systemctl start spice-webdavd # caso esteja desativado
```

Ele também irá requerer o driver para convidado na máquina Windows.

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

E mostrará algo como:

```textile
● libvirtd.service - libvirt legacy monolithic daemon
     Loaded: loaded (/usr/lib/systemd/system/libvirtd.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-04-15 18:29:06 -03; 17s ago
 Invocation: 66bf2356ca0a4da78d45f1edb7ea6dda
TriggeredBy: ● libvirtd.socket
             ● libvirtd-ro.socket
             ● libvirtd-admin.socket
       Docs: man:libvirtd(8)
             https://libvirt.org/
   Main PID: 11927 (libvirtd)
      Tasks: 21 (limit: 32768)
     Memory: 6.1M (peak: 9.4M)
        CPU: 177ms
     CGroup: /system.slice/libvirtd.service
             └─11927 /usr/sbin/libvirtd --timeout 120

abr 15 18:29:06 ti-01 systemd[1]: Starting libvirtd.service - libvirt legacy monolithic daemon...
abr 15 18:29:06 ti-01 systemd[1]: Started libvirtd.service - libvirt legacy monolithic daemon.
```

Procure **`Active: active (running)`** e serviço **enabled**. Em versões recentes também pode haver *sockets* `libvirtd.socket`; o conjunto deve permitir `virsh list` sem erro de permissão (após grupos e nova sessão). Neste momento, é criado a pasta `/var/lib/libvirt` com toda a estrutura necessária para a virtualização.

### Primeira execução do virt-manager

Abra o **virt-manager**. Na árvore à esquerda, em **QEMU/KVM**, clique com o botão direito e escolha **Conectar**:  
![Executando o virt-manager pela primeira vez](img/debian_qemu_kvm01.png)

Dê uma olhada na árvore sob **`/var/lib/libvirt`**. Vale inspecionar donos e grupos (útil antes de mudar o pool para outro disco):

```bash
sudo tree -ug --dirsfirst /var/lib/libvirt
```

Exemplo de saída:
```
[root     root    ]  /var/lib/libvirt  
├── [root     root    ]  boot  
├── [root     root    ]  dnsmasq  
│   ├── [root     root    ]  default.addnhosts  
│   ├── [root     root    ]  default.conf  
│   ├── [root     root    ]  default.hostsfile  
│   ├── [root     root    ]  virbr0.macs  
│   └── [root     root    ]  virbr0.status  
├── [libvirt-qemu libvirt-qemu]  images  
├── [libvirt-qemu libvirt-qemu]  qemu  
│   ├── [libvirt-qemu libvirt-qemu]  checkpoint  
│   ├── [libvirt-qemu libvirt-qemu]  dump  
│   ├── [libvirt-qemu libvirt-qemu]  nvram  
│   │   └── [libvirt-qemu libvirt-qemu]  win11-dx11_VARS.fd  
│   ├── [libvirt-qemu libvirt-qemu]  ram  
│   ├── [libvirt-qemu libvirt-qemu]  save  
│   └── [libvirt-qemu libvirt-qemu]  snapshot  
└── [root     root    ]  NAO_ME_APAGUE.txt  
```
Mais abaixo, se você mover o armazenamento para `/home`, será preciso **reproduzir donos e permissões**; esta árvore é a referência.

### Onde o pool `default` guarda discos por omissão

Depois da primeira ligação ao **QEMU/KVM**, o libvirt expõe o *pool* **`default`**, normalmente em **`/var/lib/libvirt/images`**.  Mas vamos confirmar, execute:

```bash
sudo virsh pool-list --all --details
```

A saída será similar a essa:

```context
 Name      State     Autostart   Persistent   Capacity     Allocation   Available
------------------------------------------------------------------------------------
 default   running   yes         yes          937,82 GiB   750,83 GiB   187,00 GiB
```

O pool `default` onde está sendo gravado? Vamos ver, execute:

```bash
sudo virsh pool-dumpxml default
```

Vai mostrar dentro de um texto longo algo como:

```textile
  <target>
    <path>/var/lib/libvirt/images</path>
  </target>
```

Isso indica exatamente onde nossas imagens serão criadas com o pool `default`. E se você fez isso com o bind mount então notará que a pasta `/var/lib/libvirt` é na verade `/home/libvirt`.

Caso não tenha criado o `bind mount` porque você já tinha máquinas virtuais pré-existentes em `/var/lib/libvirt/images` então o tópico MAQUINAS VIRTUAIS PRÉ-EXISTENTES é para você

## Máquinas virtuais pré-existentes

Se você conseguiu o `bind ount` no inicio do artigo e fez a instalação seguindo este procedimento então pode pular este tópico, ele é apenas para quem que já tinha qemu+kvm já instalado e por isso já tinha `/var/lib/libvirt` e não pôde criar o `bind mount` do jeito que fizemos.

#### Bind mount para nosso home: `/var/lib/libvirt` → `/home/libvirt`

**Parar tudo** — desligue as VMs e pare o libvirt:

```bash
sudo systemctl stop libvirtd.service libvirtd.socket
```

**Replicar dados existentes **para o destino definitivo:

```bash
sudo mkdir -p /home/libvirt
sudo rsync -aX /var/lib/libvirt/ /home/libvirt/
```

**Preparar o ponto de montagem** — guarde a árvore antiga e renomeie para não perder nada até confirmar:

```bash
sudo mv /var/lib/libvirt /var/lib/libvirt.bak
```

Se o comando acima falhar com  a mensagem:  

> mv: não foi possível mover '/var/lib/libvirt' para '/var/lib/libvirt.bak': Dispositivo ou recurso está ocupado

Então use o `fuse` para detectar quem ou o que está bloqueando a pasta:

```bash
fuser -v /var/lib/libvirt
```

E finalmente, depois de ter conseguido desbloquear, então repita o comando:

```bash
sudo mv /var/lib/libvirt /var/lib/libvirt.bak
```

Após ter mantido o backup desta pasta, vamos recriá-la, porém vazia:

```bash
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

Para testar, crie uma nova VM com um disco associado, depois observe se a imagem desse disco foi parar em `/home/libvirt/images/`. Se existir, tá tudo certo.

## O pool está no inicio automático?

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
Se o estado do pool **`default`** não for **running** (ou **executando** ), inicie:

```bash
sudo virsh pool-start default
```

Se **autostart** (ou **autoiniciar**) não estiver ativo, habilite:  

```bash
sudo virsh pool-autostart default
```

## Movendo VMs para cá
Para reutilizar as VMs antigas neste novo servidor é simples, mas antes de copiar as VMs, faça o seguinte ajuste:
```bash
sudo chown -R libvirt-qemu:kvm /var/lib/libvirt/images
sudo chmod -R 660 /var/lib/libvirt/images
```
As permissões acima é o padrão a ser usada para a pasta que contêm as imagens de VMs, agora basta copiar as VMs que precisa - geralmente arquivos .qcow2 - para a nova pasta, exemplo:
```bash
sudo mv /home/libvirt.old/images/*.qcow2 /var/lib/libvirt/images
```
Mas ao copiar VMs para lá, os donos dos arquivos que forem para lá provavelmente serão o `root` porque você usou o `sudo` nestas cópias e daí vamos repetir as permissões:
```bash
sudo chown -R libvirt-qemu:kvm /var/lib/libvirt/images
sudo chmod -R 660 /var/lib/libvirt/images
```
Agora, podemos usar o gerenciador de virtualização e importar essas VMs.  

## Criando novos pools

O pool `defalt` para desktops é suficiente para o armazenamento de imagens de VMs. Mas caso queira criar novos pools para seprar VMs por grupo, exemplo, dekstops, servirores, isos, etc... o procedimento é o seguinte:

```bash
sudo virsh pool-define-as nome-do-pool dir --target /outro/lugar/images
sudo virsh pool-autostart nome-do-pool
sudo virsh pool-start nome-do-pool
```

Para uso só em desktop, um pool **`default`** bem dimensionado costuma bastar.

Ao **importar** uma imagem copiada de fora para o pool **`default`**, ajuste dono e modo para o QEMU conseguir abrir:

```bash
sudo chown libvirt-qemu:kvm /outro/lugar/images/*.qcow2
sudo chmod 660 /outro/lugar/images/*.qcow2
```
(Ajuste o caminho se o *target* do pool for outro; após bind mount, `/var/lib/libvirt/images` e `/home/libvirt/images` apontam para o mesmo conteúdo.)

## Permissões de pasta

As imagens criadas pelo qemu+kvm tem sua extensão `.qcow2` (ou `.img`) e costumam ficar no pool `default`, mas onde fica mesmo este pool?  Existe mais de um? As vezes, isso confunde. Você pode usar o `virt-manager` para descobrir para onde os pools  apontam, mas caso esteja no terminal, execute:

```bash
sudo virsh pool-dumpxml default
```

Vai mostrar um texto, procure por:

```textile
  <target>
    <path>/var/lib/libvirt/images</path>
  </target>
```

Como sabemos que `/var/lib/libvirt` é um `bind mount` para `/home/libvirt` então a pasta real é:

> `/home/libvirt/images`

Garanta que o dono/grupo `libvirt-qemu` tenha permissão de escrita/leitura(chmod 660):

```bash
sudo chown -R libvirt-qemu:kvm /var/lib/libvirt/images
sudo chmod -R 660 /var/lib/libvirt/images
```

Agora, podemos usar as VMs e importá-las para nosso gerenciador de virtualização.  
Sei que é tentador dar permissão a si mesmo, mas a verdade é que vocÊ não precisa, tudo que fizer dentro da VM estará sendo feito por um usuário/grupo chamado `libvirt-qemu` e também porque você precisa se proteger de si mesmo, isto é, evitando que por acidente possa apagar o que não deve


## Pool de ISOs

ISOs de instalação são grandes e pouco usadas depois da instalação; muita gente **as guarda** em disco **mais barato** (HDD) e deixa **SSDs** para imagens de VM — é sugestão, não regra.

Este exemplo usa **`/home/libvirt/isos`**:

```bash
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

## Pool em sistema de arquivos Btrfs

Se o *target* do pool estiver em **Btrfs**, vale ajustar *copy-on-write* e desempenho conforme o guia dedicado:

[Virtualização nativa QEMU/KVM com Btrfs](debian_qemu_kvm_btrfs.md)

## Rede

Até mesmos as redes, dentro do sistema de virtualização são representados por nome. Sempre haverá um `default`, note:

```bash
sudo virsh net-list --all
```

O resultado provavelmente será:

```
 Name      State      Autostart   Persistent
----------------------------------------------
 default   inactive   no          yes
```

No exemplo acima, o campo **State** (Estado) esta marcado como **inactive** (inativo), também o campo **Autostart**(auto-inicio). Nesta situação, esta rede `default` está inoperável. Para ligar:  

```bash
sudo virsh net-start default
sudo virsh net-autostart default
```

O resultado do comando acima, seria:

```
sudo virsh net-autostart default
Network default started

Network default marked as autostarted
```

Agora, repetimos o comando, e veja:

```bash
sudo virsh net-list --all
```

O resultado provavelmente será:

```
 Name      State      Autostart   Persistent
----------------------------------------------
default   active   yes         yes
```

Agora temos a rede `default` ligada.  
