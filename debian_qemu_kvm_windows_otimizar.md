# Otimização da VM Windows

Depois da instalação, o Windows traz muitos recursos em segundo plano que consomem CPU, disco e memória. O passo a passo abaixo ajuda a **reduzir esse custo** em VMs — sempre avaliando o que faz sentido no **seu** uso (servidor de testes, desktop isolado, etc.).

---

## Remover o Gerenciador do servidor da inicialização (edições Server)

Se você usa **Windows Server**, o **Gerenciador do Servidor** pode abrir a cada logon. Para desativar essa abertura automática:

1. No próprio Gerenciador, vá em **Gerenciar** → **Propriedades do Gerenciador do Servidor**.
2. Marque **Não iniciar o Gerenciador do Servidor automaticamente no logon**.

![Desativar início automático do Gerenciador do Servidor](img/debian_qemu_kvm_windows51.png)

---

## Otimizando o Windows - Menu do Windows
No painel de menu, remova os recursos que não precisa como caixa de pesquisa e visão de tarefas:   
![Remova os recursos que não precisa como caixa de pesquisa e visão de tarefas](img/debian_qemu_kvm_windows52.png)    

No menu/barra de tarefas, **desligue** o que não precisar — por exemplo **caixa de pesquisa** na barra e **visão de tarefas**, se não usar.

![Personalizar itens da barra de tarefas](img/debian_qemu_kvm_windows52.png)

**Regra simples:** tudo o que anima, indexa ou fica consultando o sistema sem necessidade real pode ser desligado em VM.

---

## Papel de parede

Use **fundo sólido** (por exemplo preto) em vez de papel de parede. Em ambiente virtualizado isso costuma **aliviar** trabalho de composição do ambiente gráfico.

![Fundo sólido em vez de papel de parede](img/debian_qemu_kvm_windows53.png)

---

## Plano de energia

Em **VM** você normalmente **não** precisa economizar energia no sentido de notebook em bateria. Ajuste para **Alto desempenho** ou equivalente:

1. **Configurações** → busque por **Energia** (ou **Energia e bateria**).
2. Desligue economia agressiva e prefira desempenho máximo, se a opção existir na sua edição.

![Ajuste de energia](img/debian_qemu_kvm_windows54.png)

---

## Otimizando o Windows - Proteção contra vírus e ameaças
O Windows Server e a versao Desktop incluem um sistema integrado de vigilância e proteção chamado de **Proteção contra vírus e ameaças** que fazem muito sentido num desktop, e que tem como compromisso periodicamente varrer todos os seus arquivos, além disso, cada arquivo executado, criado ou copiado também será vasculhado imediatamente. Isso parece bom, mas não faz tanto sentido assim em ambientes controlados como VMs, então é bom você desativá-lo para que o desempenho da VM fique ainda melhor.   

O Windows (Server e desktop) inclui **Proteção contra vírus e ameaças**, que faz varreduras e analisa arquivos — útil em desktop exposto; em **VM isolada** e **controlada**, desligar ou reduzir o escopo pode **melhorar** a responsividade.

**Só faça isso** se a VM tiver **baixo risco**: sem downloads dúbios, sem abrir anexos externos, rede limitada, snapshot/backup. **Não** confunda com desligar o **Firewall** — o firewall continua recomendado.

No menu Iniciar, abra **Segurança do Windows** → **Proteção contra vírus e ameaças** e ajuste ou desative conforme sua política.

![Proteção contra vírus e ameaças](img/debian_qemu_kvm_windows_otimizar01.png)

**Meio-termo:** mantenha o antivírus, mas **exclua pastas “internas”** de varredura agendada (por exemplo diretórios de build), desde que **mantenha** áreas sensíveis (como **Users**) cobertas.

![Exclusões ou escopo de varredura](img/debian_qemu_kvm_windows_otimizar04.png)

**Observação:** após **atualizações** do Windows, o Defender e opções relacionadas podem ser **reativados**. Vale **revisar** de tempos em tempos (comum no Windows 11).

---

## Programas dispensáveis

Se não usa **Microsoft 365** nessa VM, evite instalar **OneDrive** e apps do ecossistema que sincronizam em segundo plano — consomem CPU, disco e rede.

---

## Efeitos visuais e responsividade

O que mais incomoda em VM muitas vezes não é o processador em si, e sim a **responsividade** da interface (menus, janelas, animações).

Se você usa **GPU passthrough**, o cenário muda; **este guia** assume vídeo “virtual” (por exemplo **QXL** / SPICE), onde **efeitos 3D** e animações pesam.

1. Abra **Propriedades do sistema** (por exemplo: `sysdm.cpl` ou **Configurações** → **Sistema** → **Sobre** → **Configurações avançadas do sistema**).
2. Aba **Avançado** → **Desempenho** → **Configurações**.
3. Prefira **Ajustar para melhor desempenho** ou desmarque efeitos que não precisar (sombras, animações, etc.).

![Ajuste de efeitos visuais](img/debian_qemu_kvm_windows_otimizar02.png)

---

## Agendador de Tarefas (apps de terceiros)

Depois de instalar o que precisar, abra o **Agendador de Tarefas** e revise tarefas de **atualização automática** de software (ex.: Java, Adobe). Programar verificação diária “quando ocioso” em VM costuma ser **desnecessário** e consome recursos.

---

## Relógio (HPET)

O tempo da VM em KVM costuma ser tratado pelo **hypervisor**. Para não forçar o **HPET** como base principal no convidado, em **cmd** **como administrador**:

```cmd
bcdedit /set useplatformclock No
```

**Importante:** use o **Prompt de Comando (cmd)** administrativo, não confunda com o **PowerShell** para este comando específico.

---

## Desativando o *Shutdown Event Tracker* no Windows Server

Por padrão, o **Windows Server** exibe uma janela chamada **Shutdown Event Tracker**, que solicita ao usuário o **motivo do desligamento ou reinicialização**.
Esse recurso foi criado para registrar eventos de parada no **Event Viewer** (ID 1074, origem USER32), sendo útil em ambientes com auditoria, mas desnecessário em VMs de teste ou servidores pessoais.

![Aplicativos na inicialização](img/debian_qemu_kvm_windows55.png)

1. Pressione **Win + R** e digite:

   ```
   gpedit.msc
   ```
2. Navegue até:

   ```
   Configuração do Computador >
   Modelos Administrativos >
   Sistema >
   Exibir Controlador de eventos de desligamento
   ```
3. Dê duplo clique em **Exibir rastreador de eventos de desligamento**.
4. Marque **Desabilitado** e confirme com **OK**.
5. Reinicie o servidor (ou apenas encerre e entre novamente) para aplicar a alteração.

Após aplicar essa configuração, o **Shutdown Event Tracker** deixará de ser exibido, permitindo que o Windows Server **desligue diretamente**, sem solicitar justificativas.

  
---

## Serviços dispensáveis

Alguns serviços são **candidatos** a desativar em VM de laboratório. Abra `services.msc`, avalie cada caso e **mude o tipo de inicialização** ou use o script abaixo **com cautela**.

| Nome em `services.msc` | Nome do serviço (`sc`) | Função | Desativar? |
|------------------------|-------------------------|--------|------------|
| **Windows Search** | `WSearch` | Indexação de arquivos e e-mail | Sim |
| **SysMain** *(Superfetch)* | `SysMain` | Cache e pré-carregamento | Sim |
| **Optimize Drives** | `defragsvc` | Desfragmentação (HDD) | Sim (VM em disco virtual costuma usar outra estratégia no hospedeiro) |
| **Windows Error Reporting** | `WerSvc` | Relatórios de erro à Microsoft | Sim |
| **Diagnostic Policy Service** | `DPS` | Diagnósticos | Sim |
| **Connected User Experiences and Telemetry** | `DiagTrack` | Telemetria | Sim |
| **Windows Update Medic Service** | `WaaSMedicSvc` | Tenta restabelecer o Windows Update quando alterado | Sim *(avaliação de risco)* |
| **Remote Registry** | `RemoteRegistry` | Registro remoto | Sim |
| **Fax** | `Fax` | Fax | Sim |
| **Print Spooler** | `Spooler` | Impressão | Sim *(se não imprimir nessa VM)* |
| **Bluetooth Support Service** | `bthserv` | Bluetooth | Sim |
| **Smart Card** | `SCardSvr` | Smart card | Sim |
| **Secondary Logon** | `seclogon` | “Executar como outro usuário” | *Opcional — avalie* |
| **Microsoft Defender Antivirus** | `WinDefend` | Antivírus integrado | *Só se VM isolada e política aceitar* |
| **Offline Files** | `CscService` | Arquivos offline | Sim |
| **Program Compatibility Assistant** | `PcaSvc` | Assistente de compatibilidade | Sim |
| **Security Center** | `wscsvc` | Central de segurança (alertas) | Sim |

A tabela reflete sobretudo **Windows Server**; o **Windows 11** tem mais serviços — complemente conforme necessidade.

### Script em lote (revisar antes de rodar)

Para **não** editar um por um no `services.msc`, você pode usar um `.cmd` **como administrador**. **Remova** da lista os serviços que **precisa** (por exemplo **Print Spooler**, **Secondary Logon**).

```cmd
@echo off
echo === Otimizando VM Windows para melhor desempenho ===
echo.

for %%S in (
  WSearch
  SysMain
  defragsvc
  WerSvc
  DPS
  DiagTrack
  WaaSMedicSvc
  RemoteRegistry
  Fax
  bthserv
  SCardSvr
  WinDefend
  CscService
  PcaSvc
  wscsvc
) do (
  echo Desativando %%S ...
  sc stop "%%S" >nul 2>&1
  sc config "%%S" start= disabled >nul 2>&1
)

echo.
echo === Concluído! Reinicie o Windows para aplicar todas as alterações. ===
pause
```
Aproveite para remover da lista acima os nomes de serviços que na sua definição lhe são útes, afinal, a lista acima desativa todos os serviços que detalhei na tabela. Eu por exemplo, removo da lista os serviços a serem desativados: **Print Spooler** e **Secondary Logon** porque eles me são uteis, por isso eles estão fora da lista do arquivo .cmd acima.    
Salve o conteúdo acima como `agilizar_vm.cmd`, então clique com o botão direito sobre ele e **“Executar como Administrador”**.    

Caso se arrependa de ter desativado algum serviço em particular, execute `servces.msc` e ative-o.  

**OBSERVAÇÃO**: Você está desativando o `defragsvc`, o que lhe impossibilitará a desfragmentação do disco e deve estar pensando se isso é uma boa idéia, sim, é uma boa idéia porque caso precisemos desfragmentar o disco, não usaremos o desfragmentador do Windows, mas a ferramenta de otimização para arquivos qcow2 que é muito mais eficiente e limpa espaços vazios do disco fazendo recuar o tamanho do arquivo da VM.  

**De novo:** **atualizações** podem **reativar** serviços. Revise periodicamente.

---

## Otimizando o Windows - Tarefas agendadas desnecessárias
Vamos remover todas as tarefas agendadas desnecessárias, abra o terminal **PS(PowerShell)** como administrador e execute:  
Primeiro vamos listá-las, execute:
```cmd
Get-ScheduledTask -TaskName '*schedule*'
```
Isso listará algo como:
```
TaskPath                                       TaskName                          State
--------                                       --------                          -----
\Microsoft\Windows\Defrag\                     ScheduledDefrag                   Ready
\Microsoft\Windows\Diagnosis\                  Scheduled                         Ready
\Microsoft\Windows\UpdateOrchestrator\         Schedule Maintenance Work         Disabled
\Microsoft\Windows\UpdateOrchestrator\         Schedule Scan                     Ready
\Microsoft\Windows\UpdateOrchestrator\         Schedule Scan Static Task         Ready
\Microsoft\Windows\UpdateOrchestrator\         Schedule Wake To Work             Disabled
\Microsoft\Windows\UpdateOrchestrator\         Schedule Work                     Disabled
\Microsoft\Windows\Windows Defender\           Windows Defender Scheduled Scan   Ready
\Microsoft\Windows\WindowsUpdate\              Scheduled Start                   Ready
```
E então terá uma ideia do que está no estado de `Ready`, ou seja, pronto para ser executado em momentos especificados pela própria Microsoft. Se desejar, poderá desativá-los, mas terá de fazer um por um, dessa forma:
```cmd
Disable-ScheduledTask -TaskPath '\Microsoft\Windows\Defrag\' -TaskName 'ScheduledDefrag'
```

Exemplos que costumam fazer sentido **revisar** em VM de laboratório (ajuste ao seu cenário):

Abaixo temos a sintaxe para desabilitar os que eu recomendo por considerar inapropriado para uma VM:  
```cmd
Disable-ScheduledTask -TaskPath '\Microsoft\Windows\Defrag\' -TaskName 'ScheduledDefrag'
Disable-ScheduledTask -TaskPath '\Microsoft\Windows\Diagnosis\' -TaskName 'Scheduled'
Disable-ScheduledTask -TaskPath '\Microsoft\Windows\Windows Defender\' -TaskName 'Windows Defender Scheduled Scan'
Disable-ScheduledTask -TaskPath '\Microsoft\Windows\WindowsUpdate\' -TaskName 'Scheduled Start'
```
Não sei se percebeu, mas até mesmo o 'Windows Update' esta na lista para ser desativado, então para atualizar seu Windows, só indo diretamente nas configurações e mandando atualizar manualmente.

**ATENÇÃO**: Após alguma atualização, pode acontecer de esta ou aquela programação agendada seja religada, então, periodicamente reveja esta configuração.  

---

[Retornar à página de Virtualização nativa com QAEMU+KVM Usando VM/Windows](debian_qemu_kvm_windows.md)   



