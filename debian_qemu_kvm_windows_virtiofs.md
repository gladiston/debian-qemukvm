# VIRT-MANAGER - COMPARTILHANDO ARQUIVOS VIA Virtio-FS + WinFsp
Para compartilhar arquivos entre o sistema hospedeiro e o convidado, você pode usar o Virtio-FS com o WinFsp. Esse é o método com melhor desempenho entre os que costumamos usar.  
Siga as instruções abaixo.  

Depois de iniciar a VM, acesse a página do WinFsp no link abaixo:  
[https://github.com/winfsp/winfsp/releases](https://github.com/winfsp/winfsp/releases)  
Baixe a última versão disponível:   
![página do WinFsp](img/debian_qemu_kvm_windows59.png)  

Execute `services.msc` e veja se os serviços estão habilitados:  
**VirtIO-FS Service**  
![VirtIO-FS Service](img/debian_qemu_kvm_windows60.png)   

**WinFsp.Launcher**  
![WinFsp.Launcher Service](img/debian_qemu_kvm_windows61.png)   

Depois de conferir que os serviços aparecem corretamente, **desligue a VM Windows** — por enquanto não precisamos dela.  

## Uma pasta no Linux vira um “volume” no Windows

**O que você precisa entender antes de configurar**

1. **Uma exportação por vez (no sentido prático)**  
   Com WinFsp + Virtio-FS, o Windows costuma expor **uma pasta do hospedeiro** como **uma letra de unidade** (por exemplo `Z:`). Não é como montar dez pastas diferentes e ganhar dez letras automaticamente com o mesmo fluxo simples — por isso, no virt-manager, pense em **uma pasta raiz** que o Windows enxergará como um “disco” ou volume.

2. **“Volume” aqui é o nome da configuração, não um disco físico**  
   No Linux você escolhe um diretório (ex.: `~/work`). No assistente do virt-manager isso entra como **volume/pool** de sistema de arquivos. No Windows, esse conteúdo aparece como **uma unidade**; não é necessário confundir com partições reais do disco.

3. **Se precisar de várias pastas “de verdade” (Downloads, documentos, projetos)**  
   No **hospedeiro** você contorna essa limitação assim: mantém **uma única pasta exportada** (ex.: `~/work`) e, **dentro dela**, junta o que precisa com **`bind mount`**: você monta pastas reais do sistema em subpastas vazias dentro de `~/work`. Para o Windows continua sendo **uma** unidade; por baixo, no Linux, são vários diretórios reunidos num só lugar.  
   **Diferença rápida em relação a um link simbólico:** o kernel trata `bind mount` como o próprio conteúdo daquele caminho; já um symlink é outro tipo de arquivo. Para o Virtio-FS enxergar tudo de forma previsível, o fluxo com **uma árvore sob `~/work` + bind mounts** costuma ser o mais seguro.

4. **Por que isso importa em VM**  
   Assim você **não exporta o `$HOME` inteiro** e ainda assim acessa Downloads, projetos etc., tudo sob **um** `Source Path` coerente.

Neste tutorial usamos a pasta `~/work` no Linux como única pasta exportada para o Windows. Vamos criá-la:
```bash
mkdir -p ~/work            # pasta vazia
```
Vamos criar um arquivo na pasta só para lembrar que ela é um volume exportado (e não um disco comum):
```bash
echo Este é um volume exportado do Linux > ~/work/readme.txt
echo ~/work >> ~/work/readme.txt
```
Os exemplos com `mount --bind` para várias pastas vêm na seção **OPÇÃO 1**; por ora, foque em fazer esta primeira exportação funcionar.

## Configurar o volume no virt-manager
Com a VM desligada, usando o virt-manager, vá em **Mostrar os detalhes do hardware virtual**:  
![Mostrar os detalhes do hardware virtual](img/debian_qemu_kvm_windows_virtiofs01.png)   

Você precisará criar um **pool**. Para isso, adicione um novo hardware e escolha **Sistema de arquivos (File system)**:  
![Sistema de arquivos (File system)](img/debian_qemu_kvm_windows_virtiofs02.png)   

Na tela acima, verifique se o **Driver** está como **virtiofs**.  

Em **Caminho de origem** você precisará clicar em **Navegar** e então selecionar um **pool**. Um **pool**, neste exemplo, é o “recipiente” no libvirt que aponta para onde estão os arquivos que você quer ver na VM — em geral uma pasta no seu sistema. Como `~/work` fica dentro do seu `$HOME` (ex.: `/home/gsantana`), é esse o **pool** a criar:  
![Criando um novo volume](img/debian_qemu_kvm_windows_virtiofs03.png)   

Uma vez criado o volume, a parte seguinte é apenas selecionar a pasta `~/work`:   
![Criando um novo volume](img/debian_qemu_kvm_windows_virtiofs04.png)   

Agora que os parâmetros corretos apareceram, confira se estão assim:  

| Campo | Valor | Observação |
|-------|-------|------------|
| **Nome** | `work` | É apenas um nome; poderia ser qualquer um, mas vamos padronizar para usar o mesmo nome da pasta — isto facilita. |
| **Tipo** | `dir` | Indica que o destino é um diretório; há outras opções que você poderá estudar depois. |
| **Caminho de destino** | sua pasta `~/work` | Tem que ser um caminho direto; não use links. |

Depois clique em **Concluir**.  

**Por que não dá para só digitar `/home/gsantana/work` e pronto?**

Quando você termina o assistente, o **Caminho de origem** aparece como **`/home/gsantana/work`** — e parece redundante ter passado por “criar pool” e “escolher pasta”. O ponto é este:

1. **O libvirt não pensa só em “caminho solto”.** Ele organiza armazenamento em **pools** (pastas de armazenamento, LVM, etc.). O Virtio-FS precisa de um **caminho que o libvirt já conheça** como parte de um desses pools.
2. **O assistente existe para registrar esse vínculo.** Você cria o pool (no exemplo, ligado ao seu `$HOME`) e **só então** escolhe `~/work` **dentro** dele. Assim o XML da VM fica consistente: origem válida para o hypervisor, não um texto digitado à mão que o libvirt não validou.
3. **Se você pular o pool e apontar para qualquer pasta**, pode até “passar” em versões novas do virt-manager, mas é exatamente aí que costumam aparecer **VMs que não sobem**, **compartilhamentos que somem** ou **caminhos que o QEMU não resolvem** — porque a configuração foge do modelo que o libvirt espera.

**Regra prática:** sempre que o virt-manager oferecer o fluxo **pool → pasta dentro do pool**, use esse fluxo. É o caminho suportado e previsível.  

Na janela final, fica assim:  
![Criando um novo volume](img/debian_qemu_kvm_windows_virtiofs05.png)   

| Campo | Valor | Observação |
|-------|-------|------------|
| **Driver** | `virtiofs` | É o nome do driver |
| **Caminho de origem** | `/home/gsantana/work` | O conteúdo que a VM passa a acessar no Windows |
| **Caminho de destino** | `work` | Nome arbitrário; para facilitar, usamos o mesmo nome da pasta. |

Se quiser **só leitura** no Windows (a VM não grava nessa pasta), marque a opção **Exportar sistema de arquivos como montagem somente leitura**.

Depois clique em **Concluir** para finalizar.   

### Vamos ao Windows

Inicie a VM Windows.  
Em seguida, execute `services.msc` e verifique se os serviços estão habilitados:  
**VirtIO-FS Service**  
![VirtIO-FS Service](img/debian_qemu_kvm_windows60.png)   

**WinFsp.Launcher**  
![WinFsp.Launcher Service](img/debian_qemu_kvm_windows61.png)   

Se esses serviços estiverem corretamente inicializados, você verá a unidade `Z:` no Windows:   
![Letra Z:](img/debian_qemu_kvm_windows_virtiofs06.png)      

O Windows costuma atribuir letras de unidade a partir de `Z:`; por isso a unidade aparece como `Z:`. Dá para mudar? Sim, é **possível**: no **Prompt de Comando** ou no **Terminal** do Windows, execute:
```cmd
"C:\Program Files\Virtio-Win\VioFS\virtiofs.exe" work X:
```
Assim, o volume **work** passará a usar a letra **X:**.  
Se quiser que isso ocorra na inicialização, **neste** caso você precisará colocar o comando acima num arquivo `.bat` e programá-lo para rodar no logon.   

E se não aparecer nenhuma unidade? Revise o processo de configuração. A causa mais comum é o `Source Path` **não estar** associado a um **pool** no libvirt. Outra causa frequente é o **WinFsp** não estar instalado: abra `services.msc` e confira se o serviço Virtio-FS sobe; sem o WinFsp, ele costuma falhar.

### CONSOLIDE TUDO NUM ÚNICO PONTO DE ENTRADA
Consolide todas as pastas necessárias numa **única** pasta maior. Para isso, use **`bind mount`** (não confunda com atalho de link simbólico no Explorer). No exemplo a seguir, vamos agregar a pasta `Downloads` ao volume **work**. Primeiro, crie uma pasta vazia `~/work/downloads`:  

```bash
mkdir -p ~/work/downloads  # pasta vazia
```
Se você olhar de dentro da VM Windows, verá uma pasta `downloads` vazia:  
![Volume com a pasta downloads vazia](img/debian_qemu_kvm_windows_virtiofs07.png)    

Agora monte a pasta `~/Downloads` em `~/work/downloads` com `bind mount`:
```bash
sudo mount --bind /home/gsantana/Downloads /home/gsantana/work/downloads
```
Dentro da VM, abra de novo `Z:\downloads` e confira:  
![Volume com a pasta downloads vazia](img/debian_qemu_kvm_windows_virtiofs08.png)    

Se quiser tornar esse `bind mount` **permanente** (ou seja, mantê-lo após reiniciar o computador), edite o `/etc/fstab` e inclua a linha de montagem. Por exemplo:  
```bash
sudo editor /etc/fstab
```
Inclua a seguinte linha ao final do arquivo:  
```text
# bind mounts importantes: 
/home/gsantana/Downloads  /home/gsantana/work/downloads  none  bind,defaults  0  0
```
Salve o arquivo e saia do editor.  
Para testar o `/etc/fstab` sem reiniciar, na VM a pasta `Z:\downloads` não pode estar em uso; caso contrário, o `umount` falha. Às vezes o Windows mantém o diretório aberto — nesses casos, só desligando a VM. Certifique-se de que `~/work/downloads` não está em uso no hospedeiro e desmonte assim:
```bash
sudo systemctl daemon-reload  # isso atualiza /etc/fstab
sudo umount /home/gsantana/work/downloads  # desmonta
```
Em seguida, monte tudo de novo usando o `/etc/fstab` como referência:
```bash
sudo mount -a
```
Se não houver mensagem de erro, provavelmente funcionou; para confirmar, liste o conteúdo de `~/work/downloads` ou execute:
```bash
ls -lh ~/work/downloads
```
Então você verá os mesmos arquivos que estão em `~/Downloads`.  
Por fim, acrescente outras pastas do mesmo modo e vá **agregando-as** em **`/home/gsantana/work`**.   

## SEGURANÇA
Para a segurança do seu sistema hospedeiro e do convidado:  
1. Você pode criar **um** pool para o seu `$HOME`, mas não deve exportar o `$HOME` inteiro para dentro de uma VM.  
2. Exporte como `Source Path` apenas as pastas de que aquela VM precisar. Além de **ser** mais seguro, isso limita programas que fazem **telemetria** — no fundo, **spywares** que ficam **bisbilhotando** o seu sistema.   
3. Minha preferência: crie uma pasta no estilo **work** e use *bind mounts* para reunir só o que essa VM precisa dentro desse `Source Path`.
4. Se você usa Delphi, trabalhe direto na unidade **work** (seja qual for a letra que o Windows der), mas evite que o projeto dependa de **links simbólicos** apontando para essa unidade — o Delphi costuma falhar ao resolver arquivos por symlink nesse caso, mesmo com os dados presentes. **Por outro lado**, links simbólicos para **compartilhamento de rede** costumam funcionar bem com o Delphi.  


## Dicas do YouTube

Este vídeo mostra o uso do **Virtio-FS** no Proxmox, reforça o ganho de desempenho e compara com métodos mais lentos, como WebDAV:

[COMPARTILHANDO ARQUIVOS ENTRE VMs NO PROXMOX? VEJA O PODER DO VIRTIO-FS\!](https://www.youtube.com/watch?v=1kGtxAVFIqc)  

---

[Voltar à página de virtualização nativa com QEMU/KVM — VM Windows](debian_qemu_kvm_windows.md)   





