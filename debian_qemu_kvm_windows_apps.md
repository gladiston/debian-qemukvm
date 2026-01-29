# WINDOWS BÁSICO

Um sistema Windows básico em minha opinião, não sobrevive sem estes programas:

---

## Autologon

**Autologon** é uma ferramenta utilitária Microsoft Sysinternals que automatiza o login no Windows, permitindo que uma máquina inicie e efetue autenticação sem intervenção manual. Funciona armazenando credenciais de forma cifrada no registro do Windows, ideal para cenários de servidor, VM de laboratorio e automação de infraestrutura.  

Vale instalar em ambientes onde é necessário **boot desatendido** (unattended deployment), automação de testes em VMs de desenvolvimento, inicialização de serviços críticos que dependem de autenticação prévia, ou recuperação de ambientes após reinicializações não planejadas. Particularmente relevante em sua stack de **QEMU+KVM com Windows Server**, facilitando deployments ágeis, automação de laboratorios de teste, e gerenciamento centralizado de máquinas virtuais sem necessidade de interação via console gráfico.  

A ferramenta oferece interface CLI simples, criptografia segura das senhas no registro, e integração nativa com Group Policy, tornando-a solução prática para ambientes corporativos e de infraestrutura virtualizada. Durante o processo de ciação, já tinhamos sugerido sua instalação, mas caso ainda não tenha feito, é bastante recomendável.  

Download:  
[https://learn.microsoft.com/pt-br/sysinternals/downloads/autologon](https://learn.microsoft.com/pt-br/sysinternals/downloads/autologon)  

---

## Win7z
7-Zip é um utilitário de compressão de código aberto que implementa o formato 7z, oferecendo taxa de compressão superior a ZIP e RAR através do algoritmo LZMA2. Compacta arquivos, reduzindo espaço em disco e banda de rede, enquanto suporta criptografia AES-256 e múltiplos formatos.  
Vale instalar por ser leve, versátil, disponível nos repositórios (ex: apt install p7zip-full) e essencial em workflows de backup e distribuição de artefatos—particularmente em cenários onde margem de espaço é crítica.  

Download:  
[http://www.7-zip.org/download.html](http://www.7-zip.org/download.html)    

Não tem instalação, basta descompactar e executá-lo e então ele perguntará os dados para o logon e se tudo estiver correto, no próximo boot, o Windows se logará sozinho com as credenciais fornecidas.  

---

## Adobe Reader
**Adobe Reader** é o aplicativo oficial da Adobe para visualização de documentos **PDF** (Portable Document Format). Oferece leitura confiável com preservação de formatação, fontes e layouts complexos, além de funcionalidades como anotações, preenchimento de formulários interativos e assinatura digital.  
Vale instalar por garantir **compatibilidade total com PDFs proprietários**, especialmente ao lidar com **formulários interativos e campos dinâmicos**—onde é praticamente indispensável. Embora não seja o mais rápido para abrir PDFs simples (existem alternativas mais ágeis como Okular ou Evince), é o **mais compatível** quando o documento exige fidelidade total a recursos avançados e funcionalidades proprietárias.  

Download:  
[http://get.adobe.com/br/reader/enterprise/](http://get.adobe.com/br/reader/enterprise/)  

**ALERTA**:  O link acima é para baixar a versão offline, não tente usar o instalador online (websetup) porque este instalará junto outros programas que você não pediu. A versão online é uma das versões mais traiçoeiras que existe, ela pode incluir quantos outros programas a Oracle desejar em teu computador.    

---

## Visual Studio Code
**Visual Studio Code** (VS Code) é um **editor de código-fonte leve e extensível** desenvolvido pela Microsoft, baseado em Electron. Oferece suporte nativo a múltiplas linguagens de programação, debugging integrado, controle Git embutido e um ecossistema massivo de extensões que adaptam a ferramenta a qualquer workflow técnico.  
Vale instalar por ser **gratuito, multiplataforma** (Windows, Linux, macOS) e extremamente performático mesmo em máquinas com recursos limitados. Sua curva de aprendizado reduzida, comunidade ativa e integração com ferramentas DevOps (Docker, Kubernetes, SSH remoto) o tornaram **padrão de facto** entre desenvolvedores e SysAdmins. É ideal para codificação em PHP, JavaScript, Python, C/C++ e scripting—essencial em ambientes de infraestrutura e desenvolvimento.  
Durante a instalação, não esqueça de marcar as opções:  
* Adicione a ação "Abir com Code" ao menu de contexto de arquivos do Windows Explorer.
* Adicione a ação "Abir com Code" ao menu de contexto de diretório do Windows Explorer.

Download:  
[https://code.visualstudio.com/](https://code.visualstudio.com/)    

**ALERTA**:  Muito cuidado com o download, a maioria das pessoas vai institivamente na primeira opção que é **User Installer** que se autoinstala no perfil do usuário, no entanto, é melhor baixar a versão **System Installer** que beneficia todos os perfís existentes:  
![Instale o VSCode System Installer](../img/debian_qemu_kvm_windows_vscode1.png)   

---

## Notepad++

**Notepad++** é um editor de texto leve baseado em Scintilla, com suporte a syntax highlighting para múltiplas linguagens, busca avançada e manipulação de arquivos em lote. Funciona como substituto aprimorado do Notepad padrão do Windows com recursos de produtividade básicos.  
Vale instalar apesar de ser **inferior ao VS Code em praticamente todos os aspectos** (extensibilidade, debugging, integração DevOps, comunidade). Porém, para **abrir arquivos simples** como .txt, .ini, .log e configurações de baixa complexidade, o Notepad++ é **significativamente mais rápido**—inicialização quase instantânea versus overhead do VS Code. Recomenda-se mantê-lo instalado em cenários onde velocidade de abertura de arquivos textuais é prioridade, complementando (não substituindo) o VS Code no arsenal técnico.  

Download:  
[https://notepad-plus-plus.org/downloads/](https://notepad-plus-plus.org/downloads/)   

Ao instalar o programa, escolha o idioma em português logo no inicio de sua instalação.  
Depois de instalá-lo, vá em tipos conhecidos de arquivos como .txt e .ini e em suas propriedades escolha "abrir com" e indique o notepad++.   

---

## LinkShell Extension

**LinkShell Extension** é uma extensão de contexto do Windows Explorer que facilita a criação de **hard links, symbolic links (junctions) e copy-on-write** diretamente via menu de clique direito. Oferece interface gráfica intuitiva para operações de linking que normalmente exigiriam comandos de linha de comando complexos.  
Vale instalar por permitir **gerenciamento eficiente de espaço em disco** através de hard links (múltiplas referências ao mesmo arquivo físico), essencial em cenários de backup deduplicated, repositórios de desenvolvimento e arquivamento. É particularmente útil para administradores que trabalham com armazenamento otimizado ou ambientes onde a economia de espaço é crítica—oferecendo alternativa visual robusta à manipulação manual via CMD ou PowerShell.  
Depois de sua instalação é recomendado reiniciar sua sessão do Windows.  

Download:  
[http://schinagl.priv.at/nt/hardlinkshellext/linkshellextension.html](http://schinagl.priv.at/nt/hardlinkshellext/linkshellextension.html)   

Serão dois downloads, o runtime [vc_redist.x64.exe](https://aka.ms/vs/15/release/vc_redist.x64.exe) e o programa principal.  
Ao instalar o programa principal, escolha o idioma em português logo no inicio de sua instalação.  

---

## VLC Media Player
**VLC Media Player** é um reprodutor multimídia de código aberto que suporta uma ampla variedade de formatos de vídeo, áudio e streaming (MPEG, H.264, HEVC, MP3, FLAC, OGG, entre outros). Funciona como solução completa para reprodução, conversão e manipulação de mídia sem dependência de codecs externos.  
Vale instalar por ser **multiplataforma** (Windows, Linux, macOS), leve, sem publicidade intrusiva e praticamente **compatível com qualquer formato multimídia**—eliminando a necessidade de múltiplos players especializados. Oferece funcionalidades avançadas como streaming de rede, captura de tela, conversão de formatos e suporte a legendas, tornando-o essencial em ambientes onde flexibilidade e compatibilidade de mídia são requisitos operacionais.  

Download:  
[https://www.videolan.org/vlc/](https://www.videolan.org/vlc/)   

Ao instalar o programa, escolha o idioma em português logo no inicio de sua instalação.  

---

## RAMMap
**RAMMap** é uma ferramenta de diagnóstico desenvolvida pela Microsoft (Sysinternals) que oferece **visualização detalhada e em tempo real da alocação de memória RAM** no Windows. Mapeia como a memória física é distribuída entre processos, cache, kernel e outros componentes do sistema operacional.  
Vale instalar por ser **essencial para troubleshooting de performance** e identificação de vazamentos de memória, processos consumidores excessivos e comportamento anômalo de cache. Para um **gerente de TI e administrador de sistemas**, RAMMap é ferramenta indispensável em análise de gargalos, planejamento de capacidade e diagnóstico de instabilidades—oferecendo granularidade que Task Manager não fornece. Permite decisões baseadas em dados concretos sobre alocação de recursos e identificação de problemas antes de impactarem produção.  

Download:  
[https://learn.microsoft.com/en-us/sysinternals/downloads/rammap](https://learn.microsoft.com/en-us/sysinternals/downloads/rammap)

A instalação dele não é óbvia, é um arquivo .zip contendo 3 arquivos:
* RAMMap.exe: Você copia para **C:\Windows\SysWOW64**, mas apenas que você for debugar uma aplicação 32bits, você não usará.     
* RAMMap64.exe: Você copia para **C:\Windows\System32**, será a versão mais usada.  
* RAMMap64a.exe: Para processadores diferentes de Intel/AMD, ignore-o.

Estou considerando que voce esteja usando um Windows 64 bits.  
Quando precisar executar o `RAMMap`, simplesmente Win+R e digite `RAMMap64`.  

---

## PuTTY
**PuTTY** é um cliente SSH/Telnet leve e multiplataforma que oferece acesso seguro a servidores remotos via protocolo SSH, além de suporte legado a Telnet, Rlogin e serial connections. Funciona como ferramenta essencial para administração remota de infraestrutura Unix/Linux e dispositivos de rede.  

Vale instalar por ser **gratuito, portável e extremamente leve**, sem dependências complexas—ideal para acesso rápido a servidores Debian, RedHat e outras distribuições Linux. Oferece autenticação por chave pública/privada, port forwarding local/remoto, suporte a SOCKS proxy e gerenciamento de sessões salvas, tornando-o **padrão de facto** para SysAdmins em ambientes Windows que precisam administrar infraestrutura Linux/Unix. Embora mais recente que alternatives como Windows Terminal + OpenSSH, PuTTY permanece relevante por sua compatibilidade e ausência de overhead.  

Download:  
[https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)

---

## FileZilla

**FileZilla** é um cliente FTP/SFTP/FTPS multiplataforma de código aberto que oferece transferência segura de arquivos para servidores remotos. Funciona como solução completa para gerenciamento de deployments, backup e sincronização de arquivos em infraestrutura distribuída.  

Vale instalar por ser **gratuito, intuitivo e suportar múltiplos protocolos** (FTP, SFTP com chaves SSH, FTPS com TLS/SSL), essencial para administradores que precisam transferir arquivos entre ambientes Windows e servidores Linux/Unix. Oferece gerenciamento de sites salvos, sincronização bidirecional, fila de transferências paralelas e suporte a proxy, tornando-o ferramenta indispensável em cenários de manutenção de servidores, deployment de aplicações e gerenciamento de conteúdo. Particularmente útil em workflows que não justificam overhead de ferramentas mais complexas como rsync ou git.  

**ADVERTÊNCIA**:  O Link principal do FileZilla há uma advertência dizendo que o **instalador pode conter programas promocionais**, por essa razão, estou te passando um link alternativo na mesma página, que geralmente é livre de **programas promocionais**, mesmo assim, seja cauteloso, durante a instalação observe se há opções marcadas para instalação de programas adicionais, se houverem, desmarque-os.  

Download: 
[https://filezilla-project.org/download.php?show_all=1](https://filezilla-project.org/download.php?show_all=1)   

---

## Git para Windows
**Git** é um **sistema de controle de versão distribuído** que rastreia alterações em arquivos e projetos, permitindo histórico completo de modificações, ramificações paralelas (branches) e colaboração entre múltiplos desenvolvedores. Cada repositório Git funciona como uma cópia independente e auditável do projeto.  

Vale instalar por ser o **padrão da indústria** em versionamento de código, integrar-se com plataformas como GitHub e GitLab, facilitar **reversão de mudanças** e **merge de branches**, além de ser essencial em pipelines CI/CD modernos. É indispensável para qualquer desenvolvedor ou administrador que trabalhe com infraestrutura como código (IaC).  
O instalador sugere instalar em **C:\Program Files\Git**, no entanto, para configurar outras ferramentas de programação e ajustar o PATH nestas ferramentas, eu sugiro instalar e **C:\Git** e não criar atalho no menu ou área de trabalho.  

Durante a instalação do git, ele faz uma pergunta sobre qual editor de texto usar em casos especiais, por exemplo para `commit`, `diff` entre outras, as opções geralmente são: **vim**, **notepad+++**, **vscode** ou até mesmo o tradicional bloco de noas do Windows, se você não faz idéia qual deles usará, então escolha o **notepad++**. Também durante a instalação, o **git** pergunta sobre qual é o terminador de linha, essa questão é relevante porque no mundo da Web, Linux, Mac e outros sistemas, o terminador de linha é simplesmente o ENTER(LF), porém no Windows são dois caracteres e por isso chamado de CRLF. Se voce lida com os dois tipos e geralmente é, responda **Checkout as-is, commit as-is** para que o git nunca faça nenhuma conversão. Muito novato já bagunçou arquivos de outros por cauda de uma péssima decisão aqui.

Download:  
[https://git-scm.com/downloads/win](https://git-scm.com/downloads/win)  

---

## Windows Update
Aproveite este momento para fazer todas as atualizações do Windows, faça-o até não haver mais nenhuma.  


---

## PC Manager para Windows
**PC Manager** é um **utilitário de otimização e limpeza de sistema** que oferece gerenciamento centralizado de recursos, monitoramento de desempenho, limpeza de arquivos temporários e malware, além de controle de inicialização e processos em background. Funciona como um painel unificado para manutenção preventiva do Windows, consolidando funcionalidades espalhadas entre ferramentas nativas do sistema operacional.

Vale instalar por ser uma **ferramenta oficial da Microsoft** com integração nativa ao Windows 11 e Windows Server, otimizar o desempenho da máquina removendo bloatwares e processos desnecessários, facilitar o **monitoramento em tempo real** de CPU, memória e disco, oferecer limpeza de startup para acelerar boot, além de incluir proteção contra malware e PUPs (Potentially Unwanted Programs). É especialmente relevante para administradores de TI que gerenciam múltiplas estações e precisam manter a integridade e desempenho da frota de máquinas.

Durante a instalação do PC Manager, ele se integra automaticamente à Configurações do Windows e cria atalhos no menu Iniciar. Diferentemente de ferramentas legacy, a Microsoft o mantém atualizado via Microsoft Store, garantindo que você sempre tenha a versão mais recente com correções de segurança. Recomenda-se **fixar a aplicação na barra de tarefas** para acesso rápido durante rotinas de manutenção, especialmente em cenários de troubleshooting de performance.

Ao executar pela primeira vez, o PC Manager oferece uma **varredura completa do sistema** recomendando ações prioritárias. Preste atenção às categorias: **Limpeza** (arquivos temporários, cache, downloads antigos), **Startup** (processos de inicialização desnecessários), **Apps** (desinstalação de software redundante) e **Segurança** (verificação de malware e PUPs). Para ambientes corporativos ou máquinas críticas, execute um **diagnóstico prévio** antes de aplicar limpezas agressivas, documentando o estado inicial para auditoria.

Uma prática recomendada é executar o PC Manager **mensalmente em estações de trabalho** e **semanalmente em servidores de desenvolvimento**, configurando as exclusões apropriadas para não remover arquivos críticos de aplicações específicas (como dados de bancos de dados, arquivos de projeto ou configurações de ferramentas de CI/CD). Se você gerencia múltiplas máquinas, integre o PC Manager em scripts de manutenção preventiva via PowerShell ou Task Scheduler, aproveitando sua CLI para automação.

Download:  
[https://apps.microsoft.com/detail/9pm860492szd?hl=pt-BR&gl=BR](https://apps.microsoft.com/detail/9pm860492szd?hl=pt-BR&gl=BR)

Aproveite este momento para fazer uma **Limpeza profunda** usando o PC Manager:  

![Realizando a primeira Limpeza Profunda](../img/debian_qemu_kvm_windows_pcmanager1.png)  



---

[Retornar à página de Virtualização nativa com QAEMU+KVM Usando VM/Windows](debian_qemu_kvm_windows.md)   
