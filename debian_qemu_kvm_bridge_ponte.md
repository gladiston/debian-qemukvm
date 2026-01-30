# Configuração de Rede Bridge no Debian-Like com NetworkManager (QEMU/KVM)

Este guia descreve como configurar uma ponte de rede (Bridge) real utilizando o **NetworkManager** (`nmcli`). Diferente do `macvtap`, a Bridge permite que o Host e a Máquina Virtual (VM) se comuniquem entre si (acesso ao Samba, SSH, etc), além de permitir que a VM seja vista como um dispositivo físico na rede local.

Este método é o recomendado para versões Desktop (Debian, Ubuntu, Mint), pois evita conflitos de DNS e rotas que ocorrem ao editar arquivos manuais.

---

## 1. Identificação e Limpeza (Importante)

Antes de criar a nova ponte, precisamos garantir que o ambiente esteja limpo para evitar conflitos de IP e UUID.

### Passo A: Identificar a interface física e a conexão ativa
```bash
nmcli device

```

Procure a interface `ethernet` que está `conectada` (ex: `enp5s0`). Chamaremos ela de **SUA_INTERFACE**.
Anote também o nome da conexão listada na coluna `CONNECTION` (geralmente `Wired connection 1`).

### Passo B: Limpar conexões de bridge antigas

Verifique se já existe uma conexão chamada `br0` e remova-a para evitar duplicidade de UUID:

```bash
# Se houver algo chamado 'br0' no comando 'nmcli connection show', remova:
sudo nmcli connection delete br0

```

### Passo C: Preparar o arquivo interfaces

Certifique-se de que o arquivo `/etc/network/interfaces` não possui configurações para a sua placa física. Deixe-o apenas com o básico:

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

### Passo A: Criar a interface da Bridge

```bash
sudo nmcli connection add type bridge autoconnect yes con-name br0 ifname br0

```

### Passo B: Escravizar a placa física à Bridge

Substitua `SUA_INTERFACE` pelo nome real (ex: `enp5s0`).

```bash
sudo nmcli connection add type bridge-slave autoconnect yes con-name bridge-slave-SUA_INTERFACE ifname SUA_INTERFACE master br0

```

### Passo C: Desativar a conexão Ethernet antiga (Crucial)

O NetworkManager não consegue subir a Bridge se a placa física estiver "presa" à conexão Ethernet comum. Substitua o nome entre aspas pelo que você encontrou no Passo 1A:

```bash
sudo nmcli connection down "Wired connection 1"

```

### Passo D: Configurar e Ativar a Bridge

**Opção 1: DHCP (Recomendado)**

```bash
sudo nmcli connection modify br0 ipv4.method auto
sudo nmcli connection up br0

```

**Opção 2: IP Fixo (Estático)**

```bash
sudo nmcli connection modify br0 ipv4.addresses 192.168.1.100/24 ipv4.gateway 192.168.1.1 ipv4.dns "192.168.1.5, 8.8.8.8" ipv4.method manual
sudo nmcli connection up br0

```

---

## 4. Testando a Conectividade do Host

Antes de abrir o Virt-Manager, verifique se o seu Debian está navegando corretamente:

1. **Verificar IP:** O comando `ip addr show br0` deve mostrar o IP recebido.
2. **Testar Rota (Ping IP):** `ping -c 4 8.8.8.8` (Se funcionar, o gateway está ok).
3. **Testar DNS (Ping Nome):** `ping -c 4 google.com` (Se funcionar, o DNS está ok).
4. **Verificar Bridge:** O comando `brctl show` deve listar `SUA_INTERFACE` dentro de `br0`.

---

## 5. Configuração no Virtual Machine Manager (Virt-Manager)

Agora, configure sua VM para entrar neste switch virtual:

1. Abra o **Virt-Manager** e acesse os detalhes da sua VM.
2. Vá em **NIC** (Placa de Rede).
3. **Network Source**: Selecione `Bridge device...`
4. **Device name**: Digite manualmente `br0`.
5. **Device model**: Escolha `virtio`.

---

## 6. Solução de Problemas: Host não responde à VM?

Se a VM navegar mas não acessar o Samba ou serviços do Host (Debian), o Kernel pode estar filtrando o tráfego interno da ponte. Execute:

```bash
echo "net.bridge.bridge-nf-call-iptables = 0" | sudo tee /etc/sysctl.d/bridge.conf
echo "net.bridge.bridge-nf-call-arptables = 0" | sudo tee -a /etc/sysctl.d/bridge.conf
sudo sysctl -p /etc/sysctl.d/bridge.conf

```

---

## 7. Dica para o Samba

Garanta que o Samba responda na interface correta editando o `/etc/samba/smb.conf`:

```ini
interfaces = lo br0
bind interfaces only = yes

```

## Conclusão

Gerenciar a Bridge pelo **NetworkManager** garante que o sistema lide corretamente com a troca de rotas e DNS, mantendo a estabilidade tanto do Host quanto das Máquinas Virtuais.

---

[Retornar](https://github.com/gladiston/debian-qemukvm)
