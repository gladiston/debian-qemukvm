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

## PARTICIONAMENTO DURANTE A INSTALAÇÃO DO DEBIAN/UBUNTU
A instalação do Debian/Ubuntu não tem grandes mistérios — o ponto mais delicado é mesmo o **particionamento do disco**.  
No link abaixo explico melhor essa questão, é bom que leia antes de fazer a instalação. Segue:  

[Particionamento durante a instalação do Debian/Ubuntu](https://github.com/gladiston/debianlinux/blob/main/docs/debian_part.md)


---
## SERÁ QUE MEU COMPUTADOR É ELEGIVEL PARA VIRTUALIZAÇÃO NATIVA QEMU+KVM
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

## BACKUP DE REDE
Agora que verificamos que seu computador é elegível para virtualização QEMU+KVM, que tal fazer o backup da sua configuração de rede original?  Isso é importante porque assim que instalar o `libvirt` ele vai criar uma interface NAT para virtualizar e isso vai mudar o ambiente inicial, se quiser saber como fazer isso, siga as instruções no link a seguir:  
[Fazendo um backup das configurações de rede](debian_backup_restore_network.md)  

---
## VIRTUALIZAÇÃO NATIVA QEMU+KVM
O Linux é capaz de criar máquinas virtuais e ele mesmo ser o hypervisor. Será um servidor de virtualização nivel 1, o mais rápido possivel, no entanto com algumas ausencia de recursos que facilitam a configuração que existem no VirtualBox e VMWare, por exemplo, criar redes virtuais com vários tipos de topologias,  clipboard e transferencia de arquivos entre host e anfitrião e outras coisas.  
Siga os tópicos e saiba que eles estão na ordem que deveriam ser seguidos, e fica a vontade para pular os que não precisa, de maneira geral não há dependencias aqui, ou seja, para um tópico funcionar precisa ter feito o tópico anterior. O único ponto fixo é realmente é a **INSTALAÇÃO**, sem ela, nada funcionará, obviamente.  
Vamos agora executar a instalação do QEMU+KVM, siga o link:  

[Instalando o QEMU+KVM](debian_qemu_kvm.md)

---
## VIRTUALIZAÇÃO NATIVA QEMU+KVM - WINDOWS
A virtualização com QEMU+KVM oferece desempenho quase nativo, aproveitando recursos do processador e integração direta com o kernel Linux. É uma opção poderosa e leve para executar Windows dentro do Linux, especialmente quando usada com o Virt-Manager, que simplifica a criação e o gerenciamento de VMs.  

Neste guia, você verá como criar uma máquina virtual Windows otimizada, usando o pacote de drivers virtio-win para melhor desempenho de disco, rede e vídeo, além de boas práticas sobre discos dinâmicos, rede bridge e configurações essenciais.  

Agora que você entende o contexto, vamos começar preparando o ambiente com o pacote de drivers VirtIO. Siga as instruções no link abaixo:  

[Criando e ajustando VM Windows](debian_qemu_kvm_windows.md)

## VIRTUALIZAÇÃO NATIVA QEMU+KVM - Criando máquinas virtuais pelo Virt-Manager
Instruções de como usar o virt-manager encontra-se na página:  
[Criando máquinas virtuais pelo Virt-Manager](https://sempreupdate.com.br/como-configurar-e-usar-o-virt-manager-para-kvm-no-fedora-ubuntu-debian-e-derivados/#google_vignette)   

---
## INSTALANDO PROGRAMAS BÁSICOS NA VM WINDOWS
Instale os [programas básicos](debian_qemu_kvm_windows_apps.md).  

---
## VIRTUALIZAÇÃO NATIVA QEMU+KVM - CRIANDO UMA INTERFACE BRIDGE
Para trabalhos extensos e mais profissionais com VMs é impossivel viver apenas com NAT porque na maioria dos ambientes de desenvolvimento ou corporativos uma VM precisa enxergar o anfitrião e também as outras VMs, então siga o tutorial a seguir para criar uma conexão do tipo bridge em seu sistema:  

[Criando conexões bridge pelo terminal](debian_qemu_kvm_bridge.md)   

É claro que se você está lendo este tutorial e apenas quer usar o Windows com o NAT, não precisará de uma conexão de bridge.  

---
## COMPARTILHAMENTO DE ARQUIVOS ENTRE HOSPEDERIRO E ANFITRIÃO

Você precisa do suporte ao VirtioFS, siga as instruções no link a seguir:  
[Compartilhamento de arquivos entre hospedeiro e anfitrião](debian_qemu_kvm_windows_virtiofs.md).  

---
## EXECUTANDO MAQUINAS VIRTUAIS VIA VIRT-VIEWER
Você provavelmente está usando o `virt-manager` para iniciar e encerrar suas VMs e não tem nada de errado com isso, no entanto, há um aplicativo mais otimizado para isso, ele se chama `virt-viewer`.  
Alguns serviços, especialmente para VM Windows, esperam que você use o `virt-viewer` sem o `virt-manager` acionado.  
Veja no link abaixo como usufruir do `virt-viewer` ao invés do virt-manager:

[Executando VM usando virt-viewer](debian_qemu_kvm_virtviewer.md)  

---
### OTIMIZANDO O DISCO QCOW2
O QCOW2 é um formato copy-on-write com recursos como snapshots, compressão e alocação sob demanda. Esses recursos trazem overhead e, com o tempo, geram fragmentação interna. Mas máquinas Windows são muito mais afetadas do que as demais porque o Windows gera memória virtual, arquivos temporários a todo instante. Então o link a seguir descreve como podemos otimizar e compactar o disco virtual para que o desempenho - especialmente para VMs Windows - fique sempre máximizado.  

[Instruções para melhorar o desempenho](debian_qemu_kvm_otimizar_disco.md)

---
## Backup de Máquinas Virtuais em QEMU+KVM
**Backup de máquina virtual** é a replicação sistemática dos arquivos de disco (imagens QCOW2, RAW, VDI, etc.) e metadados de configuração (arquivos XML do libvirt) para um local independente, garantindo recuperação em caso de corrupção, falha de hardware, exclusão acidental ou desastre. Diferencia-se de snapshots, que são pontos de restauração locais; backups são cópias isoladas em mídia ou storage separado.  

Para entender melhor e aprender como fazer siga o link:  
[Realizando backups de maquinas virtuais](debian_qemu_kvm_backup.md)  

---
## Convertendo máquinas virtuais do VirtualBox para QEMU+KVM
O VirtualBox é um hipervisor do tipo 2 amplamente utilizado para virtualização em desktops. As vezes é preciso migrar máquinas virtuais desta plataforma para o QEMU+KVM. A conversão funciona, mas levamos junto o lixo da instalação anterior e nem sempre temos os melhores resultados, no entanto, é melhor do que criar tudo do zero. Siga o link abaixo se precisar migrar máquinas virtuais do VirtualBox:  

[Convertendo máquinas virtuais do VirtualBox para QEMU/KVM](debian_qemu_kvm_vbox.md)  
