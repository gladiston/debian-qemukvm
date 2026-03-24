# VIRT-MANAGER - COMPARTILHANDO ARQUIVOS VIA Virtio-FS+WinSFP
Para compartilhar arquivos entre o sistema hospedeiro e convidado, voce pode usar o virtiofs+WinSFP. Esse é o método mais performático que existe.  
Siga as instruções abaixo:  

Depois de iniciar a VM, acesse a a página WinSFP no link abaixo:  
[https://github.com/winfsp/winfsp/releases](https://github.com/winfsp/winfsp/releases)  
E baixe a ultima versão disponivel:   
![página WinSFP](img/debian_qemu_kvm_windows59.png)  

Execute `services.msc` e veja se os serviços estão habilitados:  
**VirtIO-FS Service**  
![VirtIO-FS Service](img/debian_qemu_kvm_windows60.png)   

**WinFsp.Launcher**  
![WinFsp.Launcher Service](img/debian_qemu_kvm_windows61.png)   

Assim que os serviços estiverem sido confirmados, **DESLIGUE A VM WINDOWS**, não precisaremos dela agora.  

## Elegendo uma pasta, ops volume.
Assim como no filme Highlander só pode haver um, até o momento atual, o winsfp só funciona com uma única pasta exportada do ambiente hospedeiro para a VM. Esta pasta n Linux será chamada de `volume` no Windows. Ou seja, o Windows a identificará como uma unidade ou disco.  

Por algum motivo tecnológico, quando exportamos um volume para o Windows, só podemos ter um. Essa é uma infeliz limitação. Mas isso pode ser resolvido usando `bind mounts`, um recurso que permite agregar pastas de outros lugares a este volume como será chamado no Windows, é bem similar aos links simbólicos com uma única diferença, o sistema operacional consegue saber se um caminho é um link simbólico ou não, mas `bind mounts` não, então tudo que for montado dentro de `~\work` estará visivel no Windows. E para um ambiente virtualizado isso é basante importante.  

Neste tutorial vou escolher a pasta ~/work no Linux para ser exportada via winsfp para a VM Windows. Então vamos criá-la:
```bash
mkdir -p ~/work            # pasta vazia
```
Vamos criar um arquivo dentro apenas para que VMs windows saibam que esta não será uma unidade comum:
```bash
echo Este é um volume exportado do Linux > ~/work/readme.txt
echo ~/work >> ~/work/readme.txt
```
Vamos falar do `bind mounts` mais tarde, por ora, vamos fazer esta exportação de volume funcionar.

## Apareça volume
Com a VM desligada, usando o virt-manager, vá em **Mostrar os detalhes do hardware virtual**:  
![Mostrar os detalhes do hardware virtual](img/debian_qemu_kvm_windows_virtiofs01.png)   

Você precisará criar um **pool**, para isso adicione um novo hardware e escolha um novo hardware **Sistema de arquivos(FileSystem)**:  
![Sistema de arquivos(FileSystem)](img/debian_qemu_kvm_windows_virtiofs02.png)   

Na tela acima, perceba que **Driver** é **virtiofs**.  

Em **Caminho de origem** você precisará clicar em **Navegar** e então selecionar um **pool**. Um **pool** é na prática - neste exemplo - uma pasta que contem arquivos/pastas que desejo usufruir. Como a pasta `~/work` está dentro do meu $HOME(/home/gsantana) então este será o **pool** a ser criado:  
![Criando um novo volue](img/debian_qemu_kvm_windows_virtiofs03.png)   

Uma vez criado o volume, a parte seguinte é apenas selecionar a pasta `~/work`:   
![Criando um novo volue](img/debian_qemu_kvm_windows_virtiofs04.png)   

Agora que os parametros corretos retornaram, tenha certeza de que sejam:  

| Campo | Valor | Observação |
|-------|-------|------------|
| **Nome** | `work` | É apenas um nome; poderia ser qualquer um, mas vamos padronizar para usar o mesmo nome da pasta — isto facilita. |
| **Tipo** | `dir` | Indica que se aponta para um diretório; há outras opções que você poderá estudar mais tarde. |
| **Caminho de destino** | sua pasta `~/work` | Tem que ser um caminho direto; não use links. |

depois clique em **Concluir**.  

Daí você voltará atrás e **Caminho de origem** ficou **/home/gsantana/work** e a essa altura você está se perguntando porque eu não digitei esse caminho diretamente e precise criar um  **volume**? A resposta simples é, **porque este é o jeito certo**. Você não deve selecionar uma pasta onde você não tem um volume previamente criado para ela. Em versões mais antigas, o virt-manager nem aceita pastas que não estejam dentro de um pool e o fato das versões mais novas aceitarem, não quer dizer que funcionará.  

Nos janela conclusiva, ficará assim:
![Criando um novo volue](img/debian_qemu_kvm_windows_virtiofs05.png)   

| Campo | Valor | Observação |
|-------|-------|------------|
| **Driver** | `virtiofs` | É o nome do driver |
| **Caminho de origem** | `/home/gsantana/work` | O conteúdo a ser usufruido no Windows |
| **Caminho de destino** |`work` | É apenas um nome qualquer, para facilitar damos o mesmo nome da pasta. |

Se quiser impedir da estação windows escrever nesta pasta você pode marcar a opão **Exportar sistema de arquivo como montagem somente leitura**.

Depois clique em **Concluir** para finalizar.   

** Vamos ao Windows
Inicie sua VM Windows.
Depois, execute `services.msc` e veja se os serviços estão habilitados:  
**VirtIO-FS Service**  
![VirtIO-FS Service](img/debian_qemu_kvm_windows60.png)   

**WinFsp.Launcher**  
![WinFsp.Launcher Service](img/debian_qemu_kvm_windows61.png)   

Sem O `WinFsp`, o serviço `Virtio-FS` não inicia, e portanto, a unidade que representa o `Source Path` escolhido não aparecerá. Voce até pode chamar o `services.msc` no menu do Windows e verá que este serviço não inicializa sem o `WinFsp` instalado. 

Isso acontece porque o programa `WinFsp` é convocado assim que o serviço `Virtio-FS`  é iniciado, e então ele lê o `Source Path` e cria então a unidade como Z:, mas temos um problema, ele lê o primeiro `Source Path`, mas não executa os demais se existirem. Caso crie um outro `Source Path` com o nome de `docs` apontando para algum lugar do hospedeiro, essa unidade não será mostrada. Isso corre porque dentro do Windows, o `WinFsp` só consegue executar uma instância de cada vez quando o serviço `Virtio-FS` é iniciado. Assim, caso precise de mais unidades, temos duas soluções alternativas:  



### OPÇÃO 1 - CONSOLIDE TUDO NUM ÚNICO PONTO DE ENTRADA
Consolide todas as pastas necessarias numa unica pasta maior, poderá tentar usar links simbolicos usando a opção de `bind`:
```
mkdir -p /home/gsantana/work            # pasta vazia
mkdir -p /home/gsantana/work/downloads  # pasta vazia
mkdir -p /home/gsantana/work/docs       # pasta vazia

# usando `bind mounts`:
sudo mount --bind /home/gsantana/Downloads /home/gsantana/work/downloads
sudo mount --bind /home/gsantana/docs /home/gsantana/work/docs
```
Onde:  
* **/home/gsantana/downloads**  é uma pasta real com arquivos dentro que será montada (mount --bind) na pasta vazia **/home/gsantana/work/downloads**.  
* **/home/gsantana/docs**  é uma pasta real com arquivos dentro que será montada (mount --bind) na pasta vazia **/home/gsantana/work/docs**.  
E você vai *linkando* (mount --bind) dessa forma todas as pastas de que precisa para dentro de **/home/gsantana/work**.   
E agora voce exporta o `Source Path` como `/home/gsantana/work` e obterá todas as pastas dentro de uma unica que poderá ser usada dentro da VM Windows.  
Essa é a minha opção preferida, inclusive já tenho um script para montar as todas as pastas que preciso dentro de **~/work/** .


### OPÇÃO 2 - MAPEIE UNIDADES POR DENTRO DO WINDOWS
Essa é uma opção que reluto em usar porque vai criando letras de drive para cada `Source Path`.  
Dentro do windows, execute outra instancia manualmente com o comando:  
```
   "C:\Program Files\Virtio-Win\VioFS\virtiofs.exe" work X:
```
Isso criará a letra Z: (ou que estiver disponível) para `Source Path` definido como **docs**.  Você terá de criar um `.bat` para mapear cada letra de drive para cada `Source Path` e depois colocá-lo na auto inicialização do seu perfil, assim não precisará executar estes comandos todas as vezes.  

Assim, que estes serviços forem iniciados, olhe novamente para o explorer e notará que as pastas que foram exportadas, em nosso exemplo apenas a pasta `Downloads` serão reconhecidas como unidades:  

![Novas iunidades no Windows](img/debian_qemu_kvm_windows62.png)   

## SEGURANÇA
Para a segurança de seu sistema hospedeiro e convidado:  
1. Você pode criar uma pool para seu $HOME, mas não deve exportar seu $HOME inteiro para dentro de uma VM.  
2. Aprenda a exportar como `Source Path` apenas as pastas de que aquela VM precisará. Além de mais seguro, limita programas que fazem **telemetria**, mas que no fundo são **spywares** que ficam bisbilhotando seu sistema.   
3. Minha preferencia, crie uma pasta similar ao exemplo **work** e use *bind mounts* para indicar apenas as pastas desejadas dentro desse `Source Path`.
4. Se você trabalha com Delphi, trabalhe diretamente na unidade **work** seja lá qual for a letra de drive que o Windows der para ela, mas evite fazê-lo usar links simblicos apontando para esta unidade, o Delphi ao usar links simbolicos parece falhar em achar os arquivos, embora eles estejam lá. Curiosamente, links simbolicos apontem para compartilhamentos de rede, o Delphi funciona perfeitamente.  




## Dicas do YouTube

Este vídeo demonstra o poder e o uso do **Virtio-FS** no Proxmox, reforçando a performance da tecnologia e mostrando a diferença de velocidade em relação a métodos mais lentos como o WebDAV:

[COMPARTILHANDO ARQUIVOS ENTRE VMs NO PROXMOX? VEJA O PODER DO VIRTIO-FS\!](https://www.youtube.com/watch?v=1kGtxAVFIqc)  

---

[Retornar à página de Virtualização nativa com QAEMU+KVM Usando VM/Windows](debian_qemu_kvm_windows.md)   





