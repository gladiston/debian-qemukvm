# Configuração de Rede Bridge no Debian-Like com NetworkManager (QEMU/KVM)

Este guia descreve como configurar uma ponte de rede (Bridge) real utilizando o **NetworkManager** (`nmcli`). Diferente do `macvtap`, a Bridge permite que o Host e a Máquina Virtual (VM) se comuniquem entre si (acesso ao Samba, SSH, etc), além de permitir que a VM seja vista como um dispositivo físico na rede local.

Diferente do método via arquivo `/etc/network/interfaces`, o uso do NetworkManager é o ideal para versões Desktop, pois ele gerencia corretamente o DNS e as rotas de navegação.

---

## 1. Identificação da Interface Física

Os nomes das interfaces variam conforme o hardware. Identifique qual interface está ativa no seu sistema:

```bash
nmcli device

```

Procure a interface do tipo `ethernet` que está `conectada`.

> **IMPORTANTE:** Neste guia, chamaremos essa interface de **SUA_INTERFACE** (ex: `enp5s0`). Substitua este termo pelo nome real que você encontrou.

## 2. Instalação das Ferramentas

Embora o NetworkManager gerencie a rede, o sistema ainda precisa dos utilitários de ponte:

```bash
sudo apt update && sudo apt install bridge-utils -y

```

## 3. Criando a Bridge via Terminal (nmcli)

O uso do `nmcli` garante que o NetworkManager crie a bridge e mova o DNS automaticamente para ela.

### Passo A: Criar a interface da Bridge

```bash
sudo nmcli connection add type bridge autoconnect yes con-name br0 ifname br0

```

### Passo B: Escravizar a placa física à Bridge

```bash
sudo nmcli connection add type bridge-slave autoconnect yes con-name bridge-slave-SUA_INTERFACE ifname SUA_INTERFACE master br0

```

### Passo C: Configurar a Bridge (DHCP ou Estático)

**Opção 1: Configuração via DHCP (Recomendado)**

```bash
sudo nmcli connection modify br0 ipv4.method auto

```

**Opção 2: Configuração com IP Fixo (Estático)**
*(Substitua pelos dados da sua rede)*

```bash
sudo nmcli connection modify br0 ipv4.addresses 192.168.1.100/24 ipv4.gateway 192.168.1.1 ipv4.dns "192.168.1.5,8.8.8.8" ipv4.method manual

```

### Passo D: Ativar as conexões

```bash
sudo nmcli connection up br0

```

*Nota: Sua conexão física cairá por alguns segundos e voltará através da ponte `br0`.*

## 4. Configuração no Virtual Machine Manager (Virt-Manager)

Para que a sua VM utilize esse novo switch virtual:

1. Abra o **Virt-Manager** e acesse os detalhes da sua VM (ícone da lâmpada).
2. Vá em **NIC** (Placa de Rede).
3. **Network Source**: Selecione `Bridge device...`
4. **Device name**: Digite manualmente `br0`.
5. **Device model**: Escolha `virtio`.

* *Dica: Se a VM for Windows, certifique-se de ter os drivers `virtio-win` instalados.*

## 5. Solução de Problemas: O Host não responde à VM?

Se a VM navegar mas não acessar o Samba do Host, o Kernel pode estar filtrando o tráfego da ponte. Execute:

```bash
# Cria o arquivo de configuração persistente
echo "net.bridge.bridge-nf-call-iptables = 0" | sudo tee /etc/sysctl.d/bridge.conf
echo "net.bridge.bridge-nf-call-arptables = 0" | sudo tee -a /etc/sysctl.d/bridge.conf

# Aplica as mudanças imediatamente
sudo sysctl -p /etc/sysctl.d/bridge.conf

```

## 6. Verificação Final

Verifique se a ponte está ativa e com o IP correto:

```bash
ip addr show br0

```

E o comando `brctl show` deve exibir a interface física acoplada à `br0`:

```text
bridge name     bridge id               STP enabled     interfaces
br0             8000.10ffe00581ad       yes             SUA_INTERFACE
                                                        vnet0 (sua VM ativa)

```

---

### Dica para o Samba

Certifique-se de que o Samba está ouvindo na interface da ponte no arquivo `/etc/samba/smb.conf`:

```ini
interfaces = lo br0
bind interfaces only = yes

```

## Conclusão

Ao utilizar o **NetworkManager** para gerenciar sua Bridge, você mantém a facilidade de uso do ambiente Desktop (como a gestão automática de DNS) com o poder de virtualização profissional do KVM.

### Dicas Finais:

* **Interface Gráfica:** Você também pode gerenciar essas conexões através do ícone de rede do seu sistema, mas o `nmcli` via terminal é mais preciso para evitar erros de configuração.
* **DNS:** Caso a navegação falhe, o NetworkManager permite ajustar o DNS facilmente via `nmcli connection modify br0 ipv4.dns "8.8.8.8"`.
* **Reboot:** Embora o `nmcli` aplique as mudanças na hora, um `reboot` é recomendado para testar se a bridge subirá automaticamente no início do sistema.

---

[Retornar](https://github.com/gladiston/debian-qemukvm)

