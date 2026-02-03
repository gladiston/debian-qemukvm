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

Precisamos impedir que a conexão padrão "roube" a placa física da Bridge:

```bash
sudo nmcli connection modify "$my_con_name" connection.autoconnect no
```
E depois, execute:  
```bash
sudo nmcli connection down "$my_con_name"
```

### Ativar a Bridge

**Opção 1: Via DHCP (Recomendado)**

```bash
sudo nmcli connection modify br0 ipv4.method auto
sudo nmcli connection up bridge-slave-$my_iface
sudo nmcli connection up br0

```

**Opção 2: Via IP Fixo (Estático)**
*(Ajuste o IP e o DNS conforme sua necessidade antes de colar)*

```bash
sudo nmcli connection modify br0 ipv4.addresses 192.168.1.100/24 ipv4.gateway 192.168.1.1 ipv4.dns "192.168.1.5, 8.8.8.8" ipv4.method manual
sudo nmcli connection up bridge-slave-$my_iface
sudo nmcli connection up br0

```

---

## 4. Testando a Conectividade do Host

Verifique se o Host está online antes de prosseguir para a VM:

1. **Ping IP:** `ping -c 4 8.8.8.8`
2. **Ping DNS:** `ping -c 4 google.com`
3. **Verificar Bridge:** `brctl show` (A `$my_iface` deve aparecer dentro de `br0`).

---

## 5. Configuração no Virtual Machine Manager (Virt-Manager)

Agora, configure sua VM:

1. No **Virt-Manager**, abra os detalhes da VM -> **NIC (Placa de Rede)**.
2. **Network Source**: Selecione `Bridge device...`
3. **Device name**: Digite manualmente `br0`.
4. **Device model**: Escolha `virtio`.

---

## 6. Solução de Problemas: Host não responde à VM?

Se a VM tiver internet, mas não acessar o Samba do Host, desative o filtro da ponte:

```bash
echo "net.bridge.bridge-nf-call-iptables = 0" | sudo tee /etc/sysctl.d/bridge.conf
echo "net.bridge.bridge-nf-call-arptables = 0" | sudo tee -a /etc/sysctl.d/bridge.conf
sudo sysctl -p /etc/sysctl.d/bridge.conf

```

---

## 7. Configuração do Samba

Certifique-se de que o Samba ouça na interface `br0` em `/etc/samba/smb.conf`:

```ini
interfaces = lo br0
bind interfaces only = yes

```

---

[Retornar](https://github.com/gladiston/debian-qemukvm)

