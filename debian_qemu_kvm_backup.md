# Backup de Máquinas Virtuais em QEMU+KVM

## O que é Backup de VM

**Backup de máquina virtual** é a replicação sistemática dos arquivos de disco (imagens QCOW2, RAW, VDI, etc.) e metadados de configuração (arquivos XML do libvirt) para um local independente, garantindo recuperação em caso de corrupção, falha de hardware, exclusão acidental ou desastre. Diferencia-se de snapshots, que são pontos de restauração locais; backups são cópias isoladas em mídia ou storage separado.

No contexto de QEMU+KVM, o backup preserva o estado completo da máquina virtual, incluindo sistema operacional, dados, configurações e histórico de inicialização, permitindo restauração total em caso de perda ou dano do repositório primário.

---

## Por Que Fazer Backup de VMs

É fundamental manter uma estratégia de backup robusta para proteger o ambiente virtual contra uma série de sinistros e garantir a continuidade das operações.

### 1. Prevenção de Perdas de Dados

O backup é a primeira linha de defesa contra a perda de dados. Ele se torna essencial nas seguintes situações:

* **Falhas de Hardware:** Discos rígidos (HDDs/SSDs) podem falhar subitamente. O backup isola o risco, garantindo que os dados da VM estejam seguros em outro local. Storages compartilhados (como TrueNAS ou NFS) também estão sujeitos a corrupção do sistema de arquivos.
* **Recuperação de Desastres:** Ataques de ransomware, malware ou intrusões podem comprometer tanto a máquina virtual quanto seu armazenamento primário. Um **backup offline** ou com retenção temporal (versões antigas) oferece defesa em profundidade.
* **Reversão de Mudanças Catastróficas:** Aplicações mal configuradas ou atualizações problemáticas podem corromper o sistema operacional da VM. Ter um backup anterior permite restaurar a máquina rapidamente para um estado operacional conhecido.

### 2. Conformidade e Mobilidade

Além da prevenção contra falhas, o backup atende a requisitos operacionais e legais:

* **Compliance e Auditoria:** Muitos ambientes corporativos exigem um histórico de backups documentado para demonstrar conformidade com políticas de continuidade de negócios e auditorias legais.
* **Mobilidade e Migração:** Backups de imagens de disco facilitam a replicação de VMs para outro host, data center ou para arquivamento. Isso é crucial para testes, clonagem e estratégias de recuperação de desastres (Disaster Recovery).

---

## Como Fazer Backup: Estratégias

* Para backups em ambientes de **desktop/laboratório**, a estratégia mais simples é o **Backup a Frio**, onde a VM é desligada rapidamente para garantir que os dados estejam consistentes antes da cópia.
    * O método mais manual usa o utilitário `qemu-img convert` diretamente para a cópia eficiente.
    * O método mais inteligente usa um **script de automação** que lida com o desligamento, a conversão via `qemu-img convert`, montagem de disco e verificação de integridade.

### 1. Backup a Frio (Máquina Desligada)

O backup a frio garante a consistência dos dados da VM, sendo a abordagem mais segura e simples para a maioria dos ambientes. Utilizar o utilitário nativo **`qemu-img convert`** para a cópia é crucial para eficiência.

#### Vantagens
   
* **Consistência de Dados:** Garante a integridade e consistência dos dados, pois não há operações de I/O em progresso durante a cópia.
* **Eficiência de Espaço:** A ferramenta `qemu-img convert` copia **apenas** os blocos de dados em uso, ignorando o espaço livre no disco da VM, o que minimiza o tamanho final do arquivo de backup.
* **Velocidade Aprimorada:** Devido à eficiência de espaço, o processo de transferência é mais rápido. Por exemplo, uma imagem de VM de **80 GB** pode ser transferida em apenas **12 minutos** com `qemu-img convert`, comparado a 16 minutos de métodos simples.
* **Simples de Usar:** Ideal para ambientes de desenvolvimento/teste e laboratórios.

#### Desvantagens

* **Downtime:** A máquina virtual precisa ser desligada durante o processo de cópia.
* **Inviável para Discos Extremos:** Pode ser ineficiente para VMs com discos muito grandes se o tempo de inatividade for um problema, embora o `qemu-img` minimize essa desvantagem.

#### Quando Usar
   
* VMs não-críticas ou em horários de baixa utilização.
* Laboratórios de teste.
* Máquinas que podem ficar offline.

### 2. Backup a Quente (Máquina Ligada)

#### Vantagens
   
* **Zero Downtime:** A máquina virtual continua operacional durante todo o processo de backup.
* **Ideal para Produção:** É a abordagem ideal para ambientes de produção e sistemas críticos.

#### Desvantagens
   
* **Complexidade:** Requer o uso de snapshots internos ou ferramentas avançadas como `virsh blockcommit`, sendo mais complexo de implementar corretamente.
* **Impacto na Performance:** O processo de backup pode impactar ligeiramente a performance da VM enquanto a cópia está em andamento.

#### Quando Usar
   
* VMs em produção.
* Servidores que não podem parar (alta disponibilidade).
* Ambientes com Acordos de Nível de Serviço (SLA) críticos.

### 3. Backup Incremental

O Backup Incremental é uma estratégia avançada que visa economizar espaço de armazenamento e tempo de execução, copiando apenas as alterações feitas desde o último backup.

#### Conceito
    
* **Primeira Execução:** É realizada uma cópia completa da VM (full backup).
* **Execuções Subsequentes:** São copiados apenas os blocos de dados alterados ou novos (o delta).
* **Benefício:** Reduz drasticamente o tempo de backup e o espaço total consumido no storage.

#### Implementação
    
* O QEMU suporta o backup incremental usando a ferramenta `qemu-img convert` em conjunto com a funcionalidade de snapshots.
* Ferramentas de gerenciamento como `virt-manager` podem abstrair a complexidade desse processo.

---

## Implementação Prática: Backup a Frio com Qemu-img (Método Manual)

Este é o método manual para um backup único, utilizando a eficiência do `qemu-img convert` para economizar espaço e tempo.

### Passo 1: Desligar a Máquina Virtual

Para garantir a consistência do disco, desligue a VM antes de qualquer operação de cópia.

```bash
virsh shutdown win2k25
````

Aguarde a máquina desligar completamente (verificar com `virsh list --all`).

### Passo 2: Exportar a Imagem Usando `qemu-img convert`

Use o `qemu-img convert` para criar a cópia, garantindo que apenas os blocos de dados usados sejam copiados (compressão de espaço não alocado).

```bash
qemu-img convert -p -O qcow2 \
  /home/libvirt/images/win2k25.qcow2 \
  /media/backup-vm/win2k25.qcow2.backup-$(date +%Y%m%d-%H%M%S).qcow2
```

**Explicação dos Parâmetros:**

  - `-p`: Mostra o progresso da conversão.
  - `-O qcow2`: Define o formato de saída como QCOW2 (mantendo o formato eficiente para armazenamento).
  - `win2k25.qcow2.backup-...`: Adiciona o timestamp para evitar a sobrescrita de backups anteriores.

### Passo 3: Verificar Integridade do Backup

Verifique o novo arquivo de backup para confirmar que ele está íntegro e pode ser lido pelo QEMU.

```bash
qemu-img check /media/backup-vm/win2k25.qcow2.backup-*
```

Saída esperada mostra tamanho virtual e tamanho real em disco.

### Passo 4: Reiniciar a Máquina Virtual

Religue a VM após a conclusão e verificação do backup.

```bash
virsh start win2k25
```

-----

## Backup Automatizado com Script e Montagem de Disco (Recomendado)

O script **`backup-vm.sh`** automatiza toda a rotina de backup a frio, tornando o processo mais seguro e fácil de replicar. Ele não apenas faz a cópia, mas também gerencia o ambiente:

  * **Gerenciamento da VM:** Desliga a VM de forma suave (`virsh shutdown`) antes de copiar e a religa ao final, garantindo a integridade dos dados.
  * **Destino Inteligente:** Aceita o *label* de um disco (ex: `backup-vms` ou `#hist`) como destino, localiza, monta o disco em um ponto temporário e o desmonta de forma segura ao concluir.
  * **Eficiência de Espaço:** Usa `qemu-img convert` para copiar apenas os dados que realmente estão em uso na VM, ignorando o "espaço em branco" e resultando em um arquivo de backup com o menor tamanho possível.
  * **Verificação:** Executa `qemu-img check` no arquivo de backup gerado, confirmando que ele pode ser lido e está íntegro.

> **Nota sobre Discos:** Se usar um disco externo para backup, ele precisa ter um **label** (rótulo) definido para que o script possa identificá-lo e montá-lo automaticamente. Use ferramentas como `gparted` para nomear seus discos.

Aqui está o script `backup-vm.sh` com o conteúdo completo:

```bash
#!/bin/bash
# backup-vm.sh - Backup automatizado de máquinas virtuais QEMU+KVM
# Autor: Gladiston Santana <gladiston.santana[at]gmail.dot.com>
# Uso: sudo ./backup-vm.sh <caminho-da-vm> <label-ou-caminho-destino>
# Exemplo:
#   sudo ./backup-vm.sh /home/libvirt/images/win2k25.qcow2 "#hist"
#   sudo ./backup-vm.sh /home/libvirt/images/win2k25.qcow2 /mnt/backup
# Licença: MIT (Permissiva)
# Criado em: 06/11/2025
# Ult. Atualização: 07/11/2025
#
# Descrição:
#   Backup de VM QEMU/KVM com destino inteligente (label ou diretório),
#   cópia via qemu-img convert (eficiência de espaço) e verificação.
#########################################################

set -euo pipefail
START_TS=$(date +%s)

# --- CONFIGURAÇÕES GERAIS ---
BACKUP_ROOT_NAME="libvirt-bak"  # nome da pasta principal de destino
OUTPUT_FORMAT="qcow2"           # Formato de saída. 'qcow2' é recomendado para eficiência e recursos.

# --- VALIDAÇÃO DE PARÂMETROS ---
if [ $# -lt 2 ]; then
  echo "ERRO: Parâmetros insuficientes."
  echo "Uso: sudo $0 <caminho-da-vm> <label-ou-caminho-destino>"
  echo "Exemplos de execução:"
  echo "  # 1. Destino usando o LABEL do disco (ex: 'backup-vms')"
  echo "  sudo $0 /home/libvirt/images/win2k25.qcow2 \"backup-vms\""
  echo ""
  echo "  # 2. Destino usando o CAMINHO para o diretório"
  echo "  sudo $0 /home/libvirt/images/win2k25.qcow2 /media/disco2/"
  exit 1
fi

VM_PATH_RAW="${1}"
DEST_PARAM="${2}"

# monta timestamp no formato YYYYMMDD-HHhMM (Ex: 20251107-13h26)
TIMESTAMP=$(date +%Y%m%d-%Hh%M)

MOUNT_POINT=""
VM_STATE="undefined"
WAS_RUNNING=0

# --- FUNÇÕES ---
log() { echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*"; }

error_exit() { log "--- ERRO FATAL: $1 ---"; exit 1; }

cleanup() {
  log "--- Iniciando rotina de limpeza ---"
  if [ -n "${MOUNT_POINT}" ] && mountpoint -q "${MOUNT_POINT}" 2>/dev/null; then
    log "Desmontando ${MOUNT_POINT}..."
    umount "${MOUNT_POINT}" || log "AVISO: falha ao desmontar ${MOUNT_POINT}"
  fi
  if [ -n "${MOUNT_POINT}" ] && [ -d "${MOUNT_POINT}" ]; then
    rmdir "${MOUNT_POINT}" 2>/dev/null || true
  fi
  log "--- Limpeza concluída ---"
}

fmt_dur() {
  local s="$1"
  local h=$(( s/3600 ))
  local m=$(( (s%3600)/60 ))
  local ss=$(( s%60 ))
  local out=""
  [ $h -gt 0 ] && out="${out}${h}h "
  [ $m -gt 0 ] && out="${out}${m}m "
  out="${out}${ss}s"
  echo "$out"
}

trap cleanup EXIT INT TERM

# ===== 1. PERMISSÕES =====
log "Verificando permissões..."
[ "${EUID}" -ne 0 ] && error_exit "Este script precisa ser executado como root."

# ===== 2. EXPANSÃO DE CAMINHO =====
if [[ "${VM_PATH_RAW}" == "~/"* ]]; then
  SUDO_HOME=$(getent passwd "${SUDO_USER:-root}" | cut -d: -f6)
  VM_PATH="${SUDO_HOME}/${VM_PATH_RAW:2}"
else
  VM_PATH="${VM_PATH_RAW}"
fi
[ ! -f "${VM_PATH}" ] && error_exit "Arquivo da VM não encontrado: ${VM_PATH}"

VM_FILE=$(basename "${VM_PATH}")
VM_NAME="${VM_FILE%.*}"
VM_EXT="${VM_FILE##*.}"
log "VM: ${VM_NAME} (${VM_PATH})"

# ===== 3. DESTINO =====
if [ -d "${DEST_PARAM}" ]; then
  DEST_DIR="${DEST_PARAM}"
  log "Destino detectado como diretório: ${DEST_DIR}"
else
  log "Procurando disco com label '${DEST_PARAM}'..."
  DEVICE_PATH=$(findfs "LABEL=${DEST_PARAM}" 2>/dev/null || true)
  [ -z "${DEVICE_PATH}" ] && error_exit "Destino '${DEST_PARAM}' não é diretório nem label válido."
  MOUNT_POINT="/mnt/backup-temp-${RANDOM}"
  mkdir -p "${MOUNT_POINT}"
  log "Montando ${DEVICE_PATH} em ${MOUNT_POINT}..."
  mount "${DEVICE_PATH}" "${MOUNT_POINT}"
  DEST_DIR="${MOUNT_POINT}"
  log "Disco montado com sucesso em ${MOUNT_POINT}"
fi

# ===== 4. ESTRUTURA DE PASTAS E PERMISSÕES =====
BACKUP_ROOT="${DEST_DIR}/${BACKUP_ROOT_NAME}"
BACKUP_SUBDIR="${BACKUP_ROOT}/${VM_NAME}"
mkdir -p "${BACKUP_SUBDIR}"
chmod -R 664 "${BACKUP_ROOT}"
log "Destino final: ${BACKUP_SUBDIR}"

# ===== 5. GERENCIAMENTO DE VM =====
if command -v virsh >/dev/null 2>&1 && virsh dominfo "${VM_NAME}" >/dev/null 2>&1; then
  VM_STATE=$(virsh domstate "${VM_NAME}" | head -n1)
  log "Estado atual da VM: ${VM_STATE}"
  if echo "${VM_STATE}" | grep -qi running; then
    WAS_RUNNING=1
    log "Solicitando desligamento de '${VM_NAME}'..."
    virsh shutdown "${VM_NAME}" || true
    for i in $(seq 1 60); do
      sleep 1
      state_now=$(virsh domstate "${VM_NAME}" | head -n1 || true)
      if echo "${state_now}" | grep -qi "shut"; then
        log "VM desligada."
        break
      fi
      if [ "${i}" -eq 60 ]; then
        log "Forçando desligamento (destroy)."
        virsh destroy "${VM_NAME}" || true
      fi
    done
  fi
fi

# ===== 6. BACKUP (QEMU-IMG CONVERT) =====
# Formato do arquivo: VM_NAME.backup-YYYYMMDD-HHhMM.OUTPUT_FORMAT
BACKUP_FILE="${BACKUP_SUBDIR}/${VM_NAME}.backup-${TIMESTAMP}.${OUTPUT_FORMAT}"
log "Iniciando conversão/cópia com qemu-img convert..."

qemu-img convert \
  -p \
  -O "${OUTPUT_FORMAT}" \
  "${VM_PATH}" \
  "${BACKUP_FILE}"

sync
log "Conversão/cópia concluída."

# ===== 7. VERIFICAÇÃO =====
# Sempre verificamos o novo arquivo de backup.
log "Verificando integridade do arquivo de backup..."
if qemu-img check "${BACKUP_FILE}" >/dev/null 2>&1; then
  log "Integridade ${OUTPUT_FORMAT} OK."
else
  error_exit "Falha na verificação QEMU-IMG do backup."
fi

# ===== 8. REINICIAR VM =====
if command -v virsh >/dev/null 2>&1 && [ "${WAS_RUNNING}" -eq 1 ]; then
  log "Reiniciando VM '${VM_NAME}'..."
  virsh start "${VM_NAME}" || log "Falha ao iniciar VM."
fi

# ===== 9. RESULTADO + DURAÇÃO =====
ELAPSED=$(( $(date +%s) - START_TS ))
log "================================================="
log "=== BACKUP CONCLUÍDO COM SUCESSO ==="
log "================================================="
log "Arquivo: ${BACKUP_FILE}"
log "Tamanho: $(du -h "${BACKUP_FILE}" | cut -f1)"
log "Duração total: $(fmt_dur "${ELAPSED}")"
```

Em servidores, voce provavelmente precisará de um esquema de script diferente, discos externos por dia da semana, backup incremental/diferencial, teste de checksum o qual não é nosso proposito aqui em demonstrar. Nosso uso tá voltado para virtualizar em desktop e por isso, o script acima é suficiente.

O script gera um arquivo final assim:

```
[destino]/libvirt-bak/win2k25/win2k25.qcow2.backup-20251106-14h29
```

O script agora usa **`qemu-img convert`** para a cópia. Essa ferramenta garante a eficiência de espaço ao copiar apenas os blocos de dados úteis da VM. Como o timestamp inclui hora e minuto (`YYYYMMDD-HHhMM`), se você executar o backup da mesma VM em um intervalo de menos de um minuto, o arquivo será sobrescrito. Portanto, garanta intervalos de execução adequados.

## Uso do Script

Tornar executável:

```bash
chmod +x backup-vm.sh
```

Executar backup (sintaxe: sudo ./backup-vm.sh /local/da/vm.qcow2 [label-disco|caminho])
Se o segundo parametro for omitido, ele procurará como destino um disco que tenha o label **backup-vms**.

Mas dá para usar discos com outros *labels* também, veja:

```bash
$ lsblk -f
NAME        FSTYPE FSVER LABEL   UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda                                                                                  
└─sda1      ext4   1.0   #hist   943b591a-35de-4079-8b65-4bd81b1e32ed    1,5T    13% /media/hist
sdb                                                                                  
└─sdb1      ext4   1.0   #dados2 b2154643-7b94-42a1-8146-267bb88ba833  565,4G    33% /home/gsantana/work/WinSrv
                                                                                     /mnt/dados2
(...)
```

Agora vamos fazer o backup para o disco com o label **\#hist**, executando:

```bash
sudo ./backup-vm.sh /home/libvirt/images/win2k25.qcow2 "#hist"
```

Mas ao inves de usar o label, também podemos especificar o caminho executando:

```bash
sudo ./backup-vm.sh /home/libvirt/images/win2k25.qcow2 /mnt/dados2
```

-----

## Estrutura de Diretórios Resultante

Após executar o script, a organização de backups será:

```
/media/backup-vm/libvirt-bak/
├── win2k25/
│   ├── win2k25.qcow2.backup-20250206
│   └── win2k25.qcow2.backup-20250207
├── ubuntuserver/
│   └── ubuntuserver.qcow2.backup-20250206
└── debian12/
    └── debian12.qcow2.backup-20250206
```

Cada VM possui sua própria subpasta isolada, facilitando **retenção seletiva**, **políticas de limpeza por VM** e **auditoria granular** de backups corporativos.

-----

## Agendamento com Cron

Para backup automático diário às 2h da manhã:

```bash
sudo editor /etc/crontab
```

E adicionar linha, especificando o usuário **root**:

```
0 2 * * * root /caminho/para/backup-vm.sh /local/da/vm/win2k25.qcow2 backup-vms >> /var/log/backup-vm-cron.log 2>&1
```

Para múltiplas VMs em sequência:

```bash
0 2 * * * root /caminho/para/backup-vm.sh /local/da/vm/win2k25.qcow2 backup-vms >> /var/log/backup-vm-cron.log 2>&1
30 2 * * * root /caminho/para/backup-vm.sh /local/da/vm/ubuntuserver.qcow2 backup-vms >> /var/log/backup-vm-cron.log 2>&1
0 3 * * * root /caminho/para/backup-vm.sh /local/da/vm/debian12.qcow2 backup-vms >> /var/log/backup-vm-cron.log 2>&1
```

No exemplo acima, você deve ter certeza de que os intervalos de backup para o outro são suficientes, porque o script ejeta o disco no final e não poderia fazer isso se outro backup tiver sido iniciado.

-----

## Verificação e Restauração de Backup

### Listar backups disponíveis

```bash
ls -lh /media/backup-vm/libvirt-bak/win2k25/
```

Saída esperada:

```
-rw-r--r-- 1 root root 50G Feb  6 14:30 win2k25.qcow2.backup-20250206
-rw-r--r-- 1 root root 50G Feb  7 02:00 win2k25.qcow2.backup-20250207
```

### Restaurar a partir de backup

A restauração envolve substituir a imagem de disco atual da VM pela cópia de backup.

#### 1\. Parar a VM em produção

```bash
virsh destroy win2k25
```

#### 2\. Fazer backup da versão atual comprometida (opcional, recomendado)

Para ter uma cópia de segurança do disco atual (mesmo que corrompido) antes de restaurar:

```bash
mv /home/libvirt/images/win2k25.qcow2 /home/libvirt/images/win2k25.qcow2.corrupted-$(date +%Y%m%d)
```

#### 3\. Copiar backup para origem

Substitua o arquivo de disco principal pelo arquivo de backup verificado.

```bash
cp /media/backup-vm/libvirt-bak/win2k25/win2k25.qcow2.backup-20250206-143022 /home/libvirt/images/win2k25.qcow2
```

#### 4\. Verificar integridade

Uma verificação rápida da cópia restaurada é sempre recomendada.

```bash
qemu-img check /home/libvirt/images/win2k25.qcow2
```

#### 5\. Reiniciar VM

Inicie a máquina virtual usando o disco restaurado.

```bash
virsh start win2k25
```

-----

## Manutenção e Limpeza de Backups Antigos

### Listar backups por antigüidade

```bash
ls -lht /media/backup-vm/libvirt-bak/win2k25/
```

### Remover backups com mais de 30 dias

```bash
find /media/backup-vm/libvirt-bak/win2k25/ -name "*.backup-*" -mtime +30 -delete
```

### Calcular espaço utilizado por VM

```bash
du -sh /media/backup-vm/libvirt-bak/*/
```

-----

## Monitoramento e Alertas

### Script de verificação de saúde de backups

```bash
#!/bin/bash

# Verificar integridade de todos os backups
for backup in /media/backup-vm/libvirt-bak/*/*.qcow2.backup-*; do
    [ -f "${backup}" ] || continue
    
    echo "Verificando: ${backup}"
    
    if qemu-img check "${backup}" > /dev/null 2>&1; then
        echo "✓ OK"
    else
        echo "✗ ERRO: Backup corrompido"
    fi
done
```

Executar semanalmente via cron:

```bash
0 3 * * 0 /caminho/para/verify-backups.sh | mail -s "Verificação de Backups" admin@empresa.com
```

-----

[Retornar à página de Virtualização nativa com QAEMU+KVM Usando VM/Windows](https://www.google.com/search?q=debian_qemu_kvm_windows.md)  

[Retornar à página de Virtualização nativa com QAEMU+KVM](https://www.google.com/search?q=debian_qemu_kvm.md)  
