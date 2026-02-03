# Configuração de Rede Bridge no Debian-Like com NetworkManager (QEMU/KVM)

Este guia descreve como configurar uma ponte de rede (Bridge) real utilizando o **NetworkManager** (`nmcli`). Diferente do `macvtap`, a Bridge permite que o Host e a Máquina Virtual (VM) se comuniquem entre si (acesso ao Samba, SSH, etc), além de permitir que a VM seja vista como um dispositivo físico na rede local.

---

## 1. Identificação e Definição de Variáveis (Crucial)

Para evitar erros de digitação, vamos identificar os nomes e exportá-los como variáveis de sistema.

### Passo A: Identificar a interface e a conexão
Execute o comando:
```bash
nmcli device
```
O resultado será algo similar a este resultado:
```
DEVICE  TYPE      STATE                   CONNECTION         
enp5s0  ethernet  conectado               Wired connection 1 
lo      loopback  connected (externally)  lo  
```
Onde a coluna **DEVICE** anotamos a identificação da placa de rede, isto é, `enp5s0`.  
E na outra coluna **CONNECTION**, anotamos o nome daa conexão ativa, isto é, `Wired connection 1`.  

### Passo B: Exportar as variáveis

Substitua os valores abaixo pelos que você encontrou no passo anterior. **Estes nomes serão usados em todos os comandos seguintes.**
Exportamos o nome da identificação da placa de rede:  
```bash
export my_iface="enp5s0"
```
E depois exportamos o nome da conexão:  
```bash
export my_con_name="Wired connection 1"
```
Isso será importante porque nos passos anteriores não queremos um copiar/colar para dentro do seu terminal que seja literal.  

---

## 2. Limpeza e Preparação

Antes de criar a nova ponte, vamos garantir que o ambiente esteja limpo.

### Remover configurações preexistentes intituladas como "br0"
Vamos criar uma conexão chamada **br0**, mas se ela existir por causa de tentativas frustadas anteriores então irá falhar, para impedir isso, testamos a sua existencia e a removemos:  
```bash
# Remove a conexão br0 e o escravo anterior, se existirem
sudo nmcli connection delete br0 2>/dev/null
sudo nmcli connection delete bridge-slave-$my_iface 2>/dev/null
```
Não tem problema se der erro porque a conexão "br0" não existir quando executar pela primeira vez.  

---

## 3. Criando a Bridge via Terminal (nmcli)

Vamos criar a interface mestre
```bash
sudo nmcli connection add type bridge autoconnect yes con-name br0 ifname br0
```
Então será gerada a mensagem:  
> A conexão “br0” (612432f8-5386-4e18-b12c-1922ca82ddf6) foi adicionada com sucesso.  

A mensagem acima é indicação de sucesso, então agora vamos criar a interface escrava vinculada à sua placa física:  
```bash
sudo nmcli connection add type bridge-slave autoconnect yes con-name bridge-slave-$my_iface ifname $my_iface master br0
```

### Neutralizar a conexão antiga
Atente-se aqui que você desligou uma placa de rede - $my_con_name - e esta prestes a iniciar a bridge recém criada `br0` então, ela irá pegar um IP novo e se este IP não tiver navegação com a Internet, você não navegará.  
Então se tiver que salvar alguma coisa, salve-a antes dos próximos passos.  

Precisamos impedir que a conexão padrão "roube" a placa física da Bridge:

```bash
sudo nmcli connection modify "$my_con_name" connection.autoconnect no
```
E depois, execute:  
```bash
sudo nmcli connection down "$my_con_name"
```
Uma mensagem de confirmação será exibida, muito parecida com essa:  
> Conexão “Wired connection 1” desativada com sucesso (caminho D-Bus ativo: /org/freedesktop/NetworkManager/ActiveConnection/3)



### Ativar a Bridge

**Opção 1: Via DHCP (Recomendado)**

```bash
sudo nmcli connection modify br0 ipv4.method auto
```
Se quiser definir um Mac Address antes de subir a bridge, execute também:  
```bash
sudo nmcli connection modify br0 bridge.mac-address be:ba:c0:ca:00:50
```
Agora vamos subir a interface bridge, execute:  
```bash
sudo nmcli connection up bridge-slave-$my_iface
```
```bash
sudo nmcli connection up br0
```

**Opção 2: Via IP Fixo (Estático)**
*(Ajuste o IP e o DNS conforme sua necessidade antes de colar)*

```bash
sudo nmcli connection modify br0 ipv4.addresses 192.168.1.100/24 ipv4.gateway 192.168.1.1 ipv4.dns "192.168.1.1, 8.8.8.8" ipv4.method manual
```
```bash
sudo nmcli connection up bridge-slave-$my_iface
```
```bash
sudo nmcli connection up br0
```

---

### Testando a Conectividade com a rede local
Execute:  
```bash
ip route
```
E lhe será exibido algo como:  
```
default via 192.168.1.1 dev br0 proto dhcp src 192.168.1.178 metric 425 
192.168.1.0/24 dev br0 proto kernel scope link src 192.168.1.178 metric 425 
```
Se aparecer os parametros **default via** e **src** então parece tudo certo porque descobrimos neste comando que sua conexão tem o ip `192.168.1.178` (parametro src) e o seu gateway é `192.168.1.1` e agora que sabemos disso, vamos pingar o seu gateway(default via), executamos o ping:  
```bash
ping -c 3 192.168.1.1
```
E aguardamos a resposta, depois de 3 respostas, os pings serão interrompidos:  
```
PING 192.168.1.1 (192.168.1.1) 56(84) bytes of data.
64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=0.381 ms
64 bytes from 192.168.1.1: icmp_seq=2 ttl=64 time=0.554 ms
64 bytes from 192.168.1.1: icmp_seq=3 ttl=64 time=0.314 ms

--- 192.168.1.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2026ms
rtt min/avg/max/mdev = 0.314/0.416/0.554/0.101 ms
```
O sucesso no ping indica que nossa bridge do tipo ponte está preparada para máquinas virtuais.  
Se falhar, não adianta prosseguir, reveja os passos anteriores.

---

### Testando a conectividade com o mundo exterior
Uma vez que tenha passado no teste de rede local, verifique se o mundo exterior(a internet) está acessível, execute:

1. **Ping IP:** `ping -c 4 8.8.8.8`
2. **Ping DNS:** `ping -c 4 google.com`
3. **Verificar Bridge:** `brctl show` (A `$my_iface` deve aparecer dentro de `br0`).

Se o comando `ping` falhar, veja sua configuração de rede, é bem provavel que o firewall, gateway ou proxy tenha bloqueio para o novo IP que a `br0` tenha recebido.   

---

## 4. Configuração no Virtual Machine Manager (Virt-Manager)

Agora, configure sua VM:

1. No **Virt-Manager**, abra os detalhes da VM -> **NIC (Placa de Rede)**.
2. **Network Source**: Selecione `Bridge device...`
3. **Device name**: Digite manualmente `br0`.
4. **Device model**: Escolha `virtio`.

Pronto, agora poderá iniciar a VM e estará usando a conexão do tipo bridge.

---

[Retornar](https://github.com/gladiston/debian-qemukvm)

