# Configuração de Rede Bridge no Debian (QEMU/KVM)

Este guia descreve como configurar uma ponte de rede (Bridge) real no Debian. Diferente do `macvtap`, a Bridge permite que o Host e a Máquina Virtual (VM) se comuniquem entre si, além de permitir que a VM seja vista como um dispositivo físico na rede local.

Falamos em artigo anterior sobre uma bridge do tipo **macvtap**, mas ela tem o seguinte problema,e ela isola o Host da VM por design. Para resolver isso, precisamos usar o **Linux Bridge (`br0`)**. E é sobre este outro tipo de bridge que iremos falar.


# Configuração de Rede Bridge no Debian (QEMU/KVM)

Este guia ensina como configurar uma **Linux Bridge** real. Ao contrário do `macvtap`, que isola o Host da Máquina Virtual, a Bridge cria um switch virtual onde o Debian (Host) e as VMs Windows/Linux podem trocar arquivos (Samba), compartilhar impressoras e se comunicar livremente.

---

## 1. Identificação da Interface Física

Os nomes das interfaces variam conforme o hardware (ex: `eth0`, `enp5s0`, `eno1`). Antes de configurar, identifique a sua:

```bash
ip link show

```

> **NOTA:** Procure a interface que está no estado `UP`. Neste guia, chamaremos essa interface de **SUA_INTERFACE**. Substitua este termo pelo nome real que você encontrou.

## 2. Instalação das Ferramentas

Você precisará do pacote que gerencia pontes no Linux:

```bash
sudo apt update && sudo apt install bridge-utils -y

```

## 3. Configurando o `/etc/network/interfaces`

É aqui que a mágica acontece. Vamos "escravizar" sua placa física à ponte `br0`. O IP passará a pertencer à ponte, e não mais à placa física.

1. Faça um backup da configuração atual:
`sudo cp /etc/network/interfaces /etc/network/interfaces.bak`
2. Edite o arquivo:
`sudo nano /etc/network/interfaces`
3. Configure conforme o exemplo abaixo:

```text
# Interface Loopback
auto lo
iface lo inet loopback

# Interface Física (Sem IP)
allow-hotplug SUA_INTERFACE
iface SUA_INTERFACE inet manual

# Interface Bridge (O Servidor Debian usará este IP)
auto br0
iface br0 inet static
    bridge_ports SUA_INTERFACE
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1
    bridge_stp off       # Spanning Tree Protocol (off para uso doméstico/small office)
    bridge_fd 0          # Forward Delay
    bridge_maxwait 0

```

## 4. Aplicando e Reiniciando

Diferente de redes simples, a Bridge altera rotas profundas do Kernel. Recomenda-se reiniciar o host para limpar os caches de ARP e rotas antigas:

```bash
sudo reboot

```

## 5. Configuração no Virtual Machine Manager (Virt-Manager)

Para que a sua VM Windows "entre" nesse switch virtual:

1. Abra o **Virt-Manager** e acesse os detalhes da sua VM.
2. Vá em **NIC** (Placa de Rede).
3. **Network Source**: Selecione `Bridge device...`
4. **Device name**: Digite `br0`.
5. **Device model**: Escolha `virtio` (instale os drivers *virtio-win* no Windows para melhor performance).

## 6. Solução de Problemas: O Host não responde à VM?

Se após configurar tudo, a VM navegar na internet mas não acessar o Samba do Host, o culpado é o filtro da Bridge no Kernel.

Execute estes comandos para desativar o filtro de firewall dentro da ponte:

```bash
# Cria o arquivo de configuração
echo "net.bridge.bridge-nf-call-iptables = 0" | sudo tee /etc/sysctl.d/bridge.conf
echo "net.bridge.bridge-nf-call-arptables = 0" | sudo tee -a /etc/sysctl.d/bridge.conf

# Aplica as mudanças imediatamente
sudo sysctl -p /etc/sysctl.d/bridge.conf

```

## 7. Verificação Final

No terminal do Debian, o comando `brctl show` deve exibir algo parecido com isto:

```text
bridge name     bridge id               STP enabled     interfaces
br0             8000.10ffe00581ad       no              SUA_INTERFACE
                                                        vnet0 (sua VM aqui)

```

---

**Dica para Samba:** No seu `smb.conf`, garanta que o Samba está ouvindo na interface `br0`:
`interfaces = lo br0`
`bind interfaces only = yes`

