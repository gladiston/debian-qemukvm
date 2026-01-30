# Configuração de Rede Bridge no Debian (QEMU/KVM)

Este guia descreve como configurar uma ponte de rede (Bridge) real no Debian. Diferente do `macvtap`, a Bridge permite que o Host e a Máquina Virtual (VM) se comuniquem entre si, além de permitir que a VM seja vista como um dispositivo físico na rede local.

Falamos em artigo anterior sobre uma bridge do tipo **macvtap**, mas ela tem o seguinte problema,e ela isola o Host da VM por design. Para resolver isso, precisamos usar o **Linux Bridge (`br0`)**. E é sobre este outro tipo de bridge que iremos falar.

---
## 1. Identificação da Interface Física

Os nomes das interfaces variam conforme o hardware (ex: `eth0`, `enp5s0`, `eno1`). Antes de configurar, identifique a sua:

```bash
ip link show
```
Haverá um resultado como:  
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp5s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 10:ff:e0:05:81:ad brd ff:ff:ff:ff:ff:ff
    altname enx10ffe00581ad
4: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000
    link/ether 52:54:00:3d:67:a4 brd ff:ff:ff:ff:ff:ff
```

Procure a interface que está no estado `UP`, em nosso exemplo foi identificado como `enp5s0`, então daqui em diante chamaremos essa interface de **SUA_INTERFACE**. Substitua este termo pelo nome real que você encontrou.

## 2. Instalação das Ferramentas

Você precisará do pacote que gerencia pontes no Linux:

```bash
sudo apt update && sudo apt install bridge-utils -y

```

## 3. Configurando o `/etc/network/interfaces`

É aqui que a mágica acontece. Vamos "escravizar" sua placa física à ponte `br0`. O IP passará a pertencer à ponte, e não mais à placa física.

1. Faça um backup da configuração atual:
```bash
sudo cp /etc/network/interfaces /etc/network/interfaces.bak
```
3. Edite o arquivo:
```bash
sudo editor /etc/network/interfaces
```
5. Configure conforme o exemplo abaixo se quiser um IP estaticopara sua bridge do tipo 'ponte':
```text
# Interface Loopback
auto lo
iface lo inet loopback

# Interface Física (Sem IP)
allow-hotplug SUA_INTERFACE
iface SUA_INTERFACE inet manual

# Interface Bridge (Configurada via DHCP)
auto br0
iface br0 inet dhcp
    bridge_ports SUA_INTERFACE
    bridge_stp off
    bridge_fd 0
    bridge_maxwait 0

```
Mas se por alguma razão, você precisar de IP fixo para essa ponte, então o arquivo de configuração ficaria assim:  
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
A vantagem em usar IP fixo é que ele é mais rápido do que por DHCP, mas se for usá-la então avise seu administrador para que ele faça a reserva no pool de IPs, senão poderá existir dois IPs iguais na rede e isso dá um problemão e então **você** será identificado como o causador e sofrerá as penalidades.  

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

