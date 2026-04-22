# Configuração de Rede Bridge no Debian com `/etc/network/interfaces` (QEMU/KVM)

Este guia descreve como configurar uma ponte de rede (**Bridge**) real no Debian utilizando o método clássico com `/etc/network/interfaces`.

Diferente do `macvtap`, a Bridge permite que o Host e a Máquina Virtual (VM) se comuniquem entre si (acesso ao Samba, SSH, etc), além de permitir que a VM seja vista como um dispositivo físico na rede local.

Neste exemplo, considere o seguinte cenário:

* Rede: `192.168.1.0/24`
* Gateway: `192.168.1.1`
* IP atual do host: `192.168.1.178`
* Nome da bridge que iremos criar: `br0`

A ideia é simples: sua interface física deixará de ter IP, e quem passará a responder pelo IP será a bridge `br0`.

---

## 1. Identificação e Definição de Variáveis (Crucial)

Para evitar erros de digitação, vamos identificar o nome da interface e trabalhar com variáveis.

### Identificar a interface

Execute:

```bash
ip -br a
```

Exemplo de saída:

```
lo               UNKNOWN        127.0.0.1/8
enp5s0           UP             192.168.1.178/24
```

Aqui identificamos que a interface é `enp5s0`. O nome pode mudar conforme o release da distro Debian-Like que estiver usando, as vezes pode ser `eth0`  ou outro nome, por isso, memorize o resultado, pois usaremos o nome obtido dentro de arquivos de configuração a seguir.

---

## 2. Backup da Configuração Atual

Antes de qualquer alteração:

```bash
sudo cp /etc/network/interfaces /etc/network/interfaces.bak
```

---

## 3. Configuração da Bridge

Agora vamos editar o arquivo principal:

```bash
sudo editor /etc/network/interfaces
```

---

### Opção 1: DHCP (Recomendado)

```bash
auto lo
iface lo inet loopback

auto br0
iface br0 inet dhcp
    bridge_ports enp5s0
    bridge_stp off
    bridge_fd 0
```

Note que no lugar de `enp5s0` voce deve colocar o que encontrou no passo #1.

---

### Opção 2: IP Fixo (Estático)

Ajuste conforme sua rede:

```bash
auto lo
iface lo inet loopback

auto br0
iface br0 inet static
    address 192.168.1.178
    netmask 255.255.255.0
    gateway 192.168.1.1
    bridge_ports enp5s0
    bridge_stp off
    bridge_fd 0
```

Note que no lugar de `enp5s0` voce deve colocar o que encontrou no passo #1.

---

## 4. Aplicando a Configuração

Neste momento, sua interface física deixará de responder diretamente pela rede.

Se estiver acessando remotamente, tenha cuidado.

Aplicar:

```bash
sudo systemctl restart networking
```

---

## 5. Testando a Conectividade com a Rede Local

Verifique o IP:

```bash
ip a
```

Agora o IP deve aparecer em `br0`.

---

Verifique a rota:

```bash
ip route
```

Exemplo esperado:

```
default via 192.168.1.1 dev br0 proto dhcp src 192.168.1.178
```

---

Teste o gateway:

```bash
ping -c 3 192.168.1.1
```

Se houver resposta, a bridge está funcionando na rede local.

---

## 6. Testando a Conectividade com a Internet

```bash
ping -c 4 8.8.8.8
ping -c 4 google.com
```

Se o teste por IP funcionar mas o DNS falhar, revise a configuração de DNS.

---

## 7. Configuração no Virtual Machine Manager (Virt-Manager)

Agora configure sua VM:

1. Abra os detalhes da VM
2. Vá em **NIC (Placa de Rede)**
3. **Network Source**: `Bridge device`
4. **Device name**: `br0`
5. **Device model**: `virtio`

---

## Conclusão

A configuração de bridge no Debian via `/etc/network/interfaces` é direta, previsível e altamente estável.

Após configurada corretamente, a bridge permite:

* Comunicação direta entre host e VM
* Integração completa com a rede local
* Comportamento equivalente a uma máquina física

---

## Retornar

[Retornar](README.md)
