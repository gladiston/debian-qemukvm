# Configuração de Rede Bridge no Debian-Like com NetworkManager (QEMU/KVM)

Este guia descreve como configurar uma ponte de rede (Bridge) real utilizando o **NetworkManager** (`nmcli`). Diferente do `macvtap`, a Bridge permite que o Host e a Máquina Virtual (VM) se comuniquem entre si (acesso ao Samba, SSH, etc), além de permitir que a VM seja vista como um dispositivo físico na rede local.

Este método é o recomendado para versões Desktop (Debian, Ubuntu, Mint), pois evita conflitos de DNS e rotas que ocorrem ao editar arquivos manuais.

---

## 1. Identificação e Limpeza (Importante)

Antes de criar a nova ponte, precisamos garantir que não existam conexões antigas com o mesmo nome e identificar sua placa física.

### Passo A: Identificar a interface física
```bash
nmcli device

```

Procure a interface do tipo `ethernet` que está `conectada` (ex: `enp5s0`). Chamaremos ela de **SUA_INTERFACE**.

### Passo B: Limpar conexões antigas

Para evitar erros de UUID duplicado, verifique se já existe uma conexão chamada `br0` ou escravos antigos e remova-os:

```bash
# Lista as conexões
nmcli connection show

# Se houver algo chamado 'br0', remova:
sudo nmcli connection delete br0

```

### Passo C: Preparar o arquivo interfaces

Certifique-se de que o arquivo `/etc/network/interfaces` não possui configurações para a sua placa física ou para a bridge. Deixe-o apenas com o básico:

```text
auto lo
iface lo inet loopback
source /etc/network/interfaces.d/*

```

---

## 2. Instalação das Ferramentas

Mesmo usando o NetworkManager, os utilitários de ponte do sistema são necessários:

```bash
sudo apt update && sudo apt install bridge-utils -y

```

---

## 3. Criando a Bridge via Terminal (nmcli)

Siga esta sequência exata de comandos para criar a ponte de forma limpa:

### Passo A: Criar a interface da Bridge

```bash
sudo nmcli connection add type bridge autoconnect yes con-name br0 ifname br0

```

### Passo B: Escravizar a placa física à Bridge

Substitua `SUA_INTERFACE` pelo nome que você identificou no Passo 1 (ex: `enp5s0`).

```bash
sudo nmcli connection add type bridge-slave autoconnect yes con-name bridge-slave-SUA_INTERFACE ifname SUA_INTERFACE master br0

```

### Passo C: Configurar a Bridge (Escolha uma opção)

**Opção 1: Configuração via DHCP (Recomendado)**
O NetworkManager cuidará do IP, Gateway e DNS automaticamente.

```bash
sudo nmcli connection modify br0 ipv4.method auto

```

**Opção 2: Configuração com IP Fixo (Estático)**

```bash
sudo nmcli connection modify br0 ipv4.addresses 192.168.1.100/24 ipv4.gateway 192.168.1.1 ipv4.dns "192.168.1.5, 8.8.8.8" ipv4.method manual

```

### Passo D: Ativar a conexão

```bash
sudo nmcli connection up br0

```

*Sua conexão cairá por um instante e voltará operando através da ponte.*

---

## 4. Configuração no Virtual Machine Manager (Virt-Manager)

Agora, configure sua VM para entrar neste switch virtual:

1. Abra o **Virt-Manager** e acesse os detalhes da sua VM.
2. Vá em **NIC** (Placa de Rede).
3. **Network Source**: Selecione `Bridge device...`
4. **Device name**: Digite manualmente `br0`.
5. **Device model**: Escolha `virtio`.

---

## 5. Solução de Problemas: Host não responde à VM?

Se a VM navegar mas não acessar o Samba do Host, execute estes comandos para desativar o filtro de firewall interno da ponte:

```bash
echo "net.bridge.bridge-nf-call-iptables = 0" | sudo tee /etc/sysctl.d/bridge.conf
echo "net.bridge.bridge-nf-call-arptables = 0" | sudo tee -a /etc/sysctl.d/bridge.conf
sudo sysctl -p /etc/sysctl.d/bridge.conf

```

---

## 6. Verificação Final

Verifique se a interface física foi corretamente acoplada à ponte:

```bash
brctl show

```

A saída deve mostrar a `SUA_INTERFACE` (e as interfaces `vnet` das VMs ligadas) listadas sob a ponte `br0`.

---

### Dica para o Samba

Garanta que o Samba responda na interface correta editando o `/etc/samba/smb.conf`:

```ini
interfaces = lo br0
bind interfaces only = yes

```

## Conclusão

Gerenciar a Bridge pelo **NetworkManager** garante que o sistema lide corretamente com a troca de rotas e DNS, algo que costuma falhar em ambientes Desktop quando configurado manualmente por arquivos de texto.

---

[Retornar](https://github.com/gladiston/debian-qemukvm)

