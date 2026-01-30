# Configuração de Rede Bridge no Debian-Like (QEMU/KVM)

Este guia descreve como configurar uma ponte de rede (Bridge) real no Debian-Like. Diferente do `macvtap`, a Bridge permite que o Host e a Máquina Virtual (VM) se comuniquem entre si, além de permitir que a VM seja vista como um dispositivo físico na rede local.

Falamos em artigo anterior sobre uma bridge do tipo **macvtap**, mas ela possui um limitador: ela isola o Host da VM por design. Para resolver isso e permitir que a VM acesse compartilhamentos Samba ou serviços no próprio Host, precisamos usar o **Linux Bridge (`br0`)**.

---

## 1. Identificação da Interface Física

Os nomes das interfaces variam conforme o hardware (ex: `eth0`, `enp5s0`, `eno1`). Antes de configurar, identifique qual interface está ativa no seu sistema:

```bash
ip link show

```

Haverá um resultado similar a este:

```text
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
2: enp5s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ... state UP ...
4: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 ... state DOWN ...

```

**Importante:** Ignore as interfaces `lo` (loopback) e `virbr0` (ponte padrão do NAT). Procure a interface física que está no estado **UP**. No exemplo acima, é a `enp5s0`.

> Daqui em diante, chamaremos essa interface de **SUA_INTERFACE**. Substitua este termo pelo nome real que você encontrou.

## 2. Instalação das Ferramentas

Você precisará do pacote que gerencia pontes no Linux:

```bash
sudo apt update && sudo apt install bridge-utils -y

```

## 3. Configurando o `/etc/network/interfaces`

Nesta etapa, vamos "escravizar" sua placa física à ponte `br0`. O endereço IP passará a pertencer à ponte, e não mais à placa física diretamente.

1. Faça um backup da configuração atual por segurança:

```bash
sudo cp /etc/network/interfaces /etc/network/interfaces.bak

```

2. Edite o arquivo com seu editor preferido:

```bash
sudo editor /etc/network/interfaces
```
Seu arquivo `/etc/network/interfaces` deve estar assim:
```
source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback
```
Abaixo dessas linhas voce deve incluir uma das opções a seguir:  


**Opção A: Configuração via DHCP (Recomendado para iniciantes)**

```text
# ( mantenha as linhas anteriores )

# Interface Física 
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

**Opção B: Configuração com IP Fixo (Estático)**
Use esta opção se o seu servidor precisar manter sempre o mesmo endereço IP.
*Nota: Se usar esta opção, certifique-se de que o IP escolhido não está em uso para evitar conflitos de rede.*

```text
# ( mantenha as linhas anteriores )

# Interface Física 
allow-hotplug SUA_INTERFACE
iface SUA_INTERFACE inet manual

# Interface Bridge (IP Estático)
auto br0
iface br0 inet static
    bridge_ports SUA_INTERFACE
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1
    bridge_stp off
    bridge_fd 0
    bridge_maxwait 0

```

## 3. Aplicando e Reiniciando

Diferente de redes simples, a Bridge altera rotas profundas do Kernel. Para limpar caches de ARP e garantir que as novas rotas sejam assumidas corretamente, reinicie o computador:

```bash
sudo reboot

```

## 4. Configuração no Virtual Machine Manager (Virt-Manager)

Para que a sua VM utilize esse novo switch virtual:

1. Abra o **Virt-Manager** e acesse os detalhes da sua VM (ícone da lâmpada).
2. Vá em **NIC** (Placa de Rede).
3. **Network Source**: Selecione `Bridge device...`
4. **Device name**: Digite manualmente `br0`.
5. **Device model**: Escolha `virtio`.
* *Dica: Se a VM for Windows, certifique-se de carregar a ISO dos drivers `virtio-win` para que o sistema reconheça a placa.*



## 5. Solução de Problemas: O Host não responde à VM?

Se a VM navega na internet mas não consegue acessar o Samba ou dar Ping no Host, o Kernel pode estar filtrando o tráfego interno da ponte. Execute os comandos abaixo para desativar esse filtro:

```bash
# Cria o arquivo de configuração persistente
echo "net.bridge.bridge-nf-call-iptables = 0" | sudo tee /etc/sysctl.d/bridge.conf
echo "net.bridge.bridge-nf-call-arptables = 0" | sudo tee -a /etc/sysctl.d/bridge.conf

# Aplica as mudanças imediatamente
sudo sysctl -p /etc/sysctl.d/bridge.conf

```

## 6. Verificação Final

No terminal do Debian-Like, o comando `brctl show` deve exibir a interface física e a interface virtual da VM (vnet) conectadas à `br0`:

```text
bridge name     bridge id               STP enabled     interfaces
br0             8000.10ffe00581ad       no              SUA_INTERFACE
                                                        vnet0 (sua VM ativa)

```

---

### Dica para o Samba

Para garantir que o serviço de arquivos responda corretamente através da ponte, verifique seu arquivo `/etc/samba/smb.conf`:

```ini
interfaces = lo br0
bind interfaces only = yes

```

## Conclusão

Ao configurar uma **Linux Bridge**, você elimina a barreira invisível entre Host e Guest. Isso permite que serviços de rede funcionem de forma transparente, facilitando o desenvolvimento e a administração de sistemas.

### Dicas Finais:

* **Reserva de IP:** Se usar DHCP, faça a reserva do IP pelo endereço MAC no seu roteador. Isso evita que o Host mude de IP e você perca o mapeamento das unidades de rede na VM.
* **Performance:** O driver `virtio` é essencial para atingir altas velocidades de transferência de dados com baixo uso de CPU.
* **DNS:** Caso o Host perca a resolução de nomes após a configuração, verifique se o arquivo `/etc/resolv.conf` aponta para um DNS válido.

---

[Retornar](https://github.com/gladiston/debian-qemukvm)

