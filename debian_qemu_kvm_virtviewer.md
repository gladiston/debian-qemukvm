# EXECUTANDO MAQUINAS VIRTUAIS VIA VIRT-VIEWER
Você provavelmente está usando o `virt-manager` para iniciar e encerrar suas VMs e não tem nada de errado com isso, no entanto, há um aplicativo mais otimizado para isso, ele se chama `virt-viewer`.  
Alguns serviços, especialmente para VM Windows, esperam que você use o `virt-viewer` sem o `virt-manager` acionado, é o caso do serviço **Spice webdav proxy** dentre outros.  

Então vamos conferir se o `virt-viwer` está instalado, execute:
```bash
virt-viewer --version
```
Se o comando acima falhar com uma mensagem:  
> virt-viewer: comando não encontrado

É porque o mesmo não está instalado, então vamos instalá-lo:  
```bash
sudo apt install virt-viewer
```

Nossas VMs sempŕe que possivel irão ser abertas usando `virt-viewer` porque a mesma foi otimizada para isso.  
Agora, precisamos identificar a nossa VM, cada VM tem um **NOME** \(que será chamada de Dominio pelo sistema\) e um **UUID** próprio, em algumas operações podemos usar o nome/dominio, mas em outras - especialmente as mais perigosas - precisamos usar o UUID. Em nosso exemplo, o nome/dominio de nossa VM é **win2k25-dx**, então para descobrir o UUID, executamos:  
```bash
sudo virsh domuuid win2k25-dx
```
E algo similar a isso será exibido:  
> ca25da76-55eb-45f7-8102-f54864480d4e  

O codigo acima é o **UUID** para a VM/dominio **win2k25-dx**, agora vamos iniciar a VM, execute:
```bash
sudo virsh start win2k25-dx
```
E algo similar a isso será exibido:  
> Domain 'win2k25-dx' started  

Isso indica que nossa VM já está rodando, agora podemos abrir a VM usando o `virt-viwer`, execute:  
```bash
virt-viewer --connect qemu:///system --wait win2k25-dx
```

Você pode colocar esta sequencia de comandos num shell script para automatizar:
```bash
mkdir ~/.local/bin
sudo editor ~/.local/bin/vm_win2k25-dx.sh 
```
E colar o seguinte conteúdo:
```
#!/bin/bash

VM_NAME="win2k25-dx"

# Garante que há ambiente gráfico antes de chamar o virt-viewer
if [ -z "$DISPLAY" ]; then
    echo "Erro: nenhuma sessão gráfica detectada (DISPLAY vazio)."
    echo "Abra uma sessão gráfica (ex.: KDE, GNOME) e rode o script novamente."
    exit 1
fi

# Descobre como rodar o virsh com permissão administrativa
# primeiro tenta sudo (melhor para linha de comando) e depois
# tenta pkexec, mas só se estivermos em ambiente gráfico
if command -v sudo >/dev/null 2>&1; then
    VIRSH_RUN=(sudo virsh -c qemu:///system)
elif command -v pkexec >/dev/null 2>&1; then
    VIRSH_RUN=(pkexec virsh -c qemu:///system)
else
    echo "Erro: nem sudo nem pkexec encontrados. Não é possível executar virsh com permissões administrativas."
    exit 1
fi

echo "Verificando estado da VM..."
STATUS=$("${VIRSH_RUN[@]}" domstate "$VM_NAME" 2>/dev/null)

if [[ "$STATUS" == "running" ]]; then
    echo "A VM '$VM_NAME' já está em execução."
else
    echo "A VM '$VM_NAME' não está ligada. Iniciando..."
    "${VIRSH_RUN[@]}" start "$VM_NAME"
    if [[ $? -ne 0 ]]; then
        echo "Erro: não foi possível iniciar a VM."
        exit 1
    fi
    echo "VM iniciada com sucesso."
fi

echo "Abrindo o virt-viewer como usuário normal..."
virt-viewer --connect qemu:///system --wait "$VM_NAME"
```

Talvez você queira executá-la no ambiente gráfico, neste caso, apenas crie um atalho:  
```bash
editor ~/.local/share/applications/vm_win2k25-dx.desktop 
```
E colar o seguinte conteúdo:
```
[Desktop Entry]
Type=Application
Name=Iniciar VM Windows Win2k25-DX
Comment=Inicia a VM win2k25-dx e abre no virt-viewer
Exec=/home/gsantana/.local/bin/vm_win2k25-dx.sh
Icon=computer
Terminal=false
Categories=System;
```
Onde `/home/gsantana/.local/bin/vm_win2k25-dx.sh` é o shell script que criamos. O comando é precedido por `pkexec` porque precisamos elevar as permissões, senão falhará.  
Agora você tem um atalho no seu sistema para esta maquina virtual, procure por `Windows`e ela aparecerá e se acaso você precisa dela muitas vezes, coloque-a na seção de favoritos de seu ambiente gráfico.  

---

[Retornar ao artigo sobre VIRTUALIZAÇÃO NATIVA QEMU+KVM](debian_qemu_kvm.md#executando-maquinas-virtuais-via-virt-viewer)
