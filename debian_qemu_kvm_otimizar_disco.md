# Otimizando imagens de VM (QCOW2) no qemu+kvm

O **QCOW2** é um formato *copy-on-write* que oferece recursos como snapshots, compressão e alocação sob demanda. Com o tempo, esses recursos introduzem fragmentação interna e perda de desempenho. Máquinas Windows são especialmente afetadas, pois criam e apagam arquivos temporários e de paginação continuamente.

Este guia mostra como **otimizar e compactar** discos QCOW2, mantendo desempenho máximo e reduzindo o espaço ocupado — **sem precisar executar comandos dentro da VM**.

> **Objetivo:** reduzir o tamanho em disco, melhorar I/O e preservar a integridade das imagens virtuais.

---

> ⚠️ **Nota sobre o termo “compactar”**
>
> Neste artigo, as palavras **“compactar”** ou **“comprimir”** não se referem à compressão de dados como `zip`, `gzip` ou `bzip2`.
> Aqui, elas indicam o **processo de redução do tamanho físico ocupado por uma imagem QCOW2** após a remoção de blocos não utilizados e reorganização dos dados no disco virtual.
> Em outras palavras, “compactar” significa **otimizar o espaço interno** da imagem, e não aplicar compressão de dados com perda ou custo de CPU.

---

## O que é e por que otimizar

* **Por que otimizar**

  * Reduz o tamanho físico no datastore (melhor aproveitamento de espaço).
  * Reorganiza clusters e metadados, melhorando desempenho de I/O.
  * Remove snapshots antigos e cadeias de backing desnecessárias.
  * Corrige metadados e inconsistências de refcount.
  * A primeira instalação do Windows ele criará cluster de 64K no disco, esse tamanho economiza espaço, mas aumenta a fragmentação e o Windows se beneficia muito de leituras e gravações sequencias. Em servidores baremetal Windows, onde o disco era muito maior do que eu precisava eu pré-formatava todos os discos usando o `gparted` com cluster de 1M e depois instalava o Windows neste disco sem formatar e fazia isso por causa da performance.

* **Como otimizar**

  * No host, usando ferramentas como `qemu-img` e `virt-sparsify` para reescrever ou enxugar a imagem.
  * Ajustando parâmetros (compat, cluster_size, lazy_refcounts) para equilibrar desempenho e espaço.

---

## Quando aplicar

* Após exclusões grandes de dados no guest.
* Depois de criar/remover vários snapshots.
* Periodicamente em ambientes com muita escrita.
* Antes de migrar ou consolidar VMs em outro storage.

---

## Ajustes no libvirt (via virt-manager)

Os ajustes mencionados a seguir foram feitos nos passos anteriores deste guia, no entanto, caso tenha caído nesta página por qualquer outra razão, o que iremos fazer é conferir se o nosso disco virtual foi ajustado para ter máxima performance com o Windows, esses ajustes garantem que operações de “descartar blocos” (TRIM/UNMAP) do convidado cheguem até o hospedeiro, permitindo que futuras exclusões dentro da VM liberem espaço real.

### 🔧 Pelo virt-manager (GUI)

1. **Abra o virt-manager** e selecione a VM desejada.
2. Clique no ícone **⚙️ “i” (Mostrar detalhes da máquina virtual)**.
3. No painel esquerdo, clique em **Disco VirtIO 1(VirtIO Disk)** — ou o nome do disco principal.
4. Na aba Detalhes, expanda **Opções avançadas** (Advanced options).
5. Ele mostrará:
   * **Modo do cachê(Cache mode):** `none`
   * **Modo de descarte(Discard):** `unmap`
   Mas em algumas distros - e Debian incluso - não mostrará essas opções:  
   * **IO mode:** `native`  
   * **Detect zeroes:** `unmap` *(ou `on`, se `unmap` não estiver disponível)*
Então você precisa adicioná-las manualmente, vá na aba **XML**, e localize o bloco `<disk …>` e provavelmente estará assim:
```xml
<disk type="file" device="disk">
  <driver name="qemu" type="qcow2" cache="none" discard="unmap"/>
  (...)
</disk>
```
Sugere-se acrescentar na opçõa destacada acima também as opções: `io="native" detect_zeroes="unmap"`, ficando assim:  
> <driver name="qemu" type="qcow2" cache='none' discard="unmap"  **io="native" detect_zeroes="unmap"**/>

Tome muito cuidado a sintaxe, aspas simples no lugar de aspas duplas ou a falta delas ou qualquer outro erro de sintaxe fará com que a VM não inicie.
 
6. Clique em **Aplicar** para salvar as alterações. É possivel que ao salvar, o editor visual mude a ordem dos parametros, ele realmente faz isso e não precisa se preocupar.  
7. Inicie a VM normalmente — as novas flags serão aplicadas no próximo boot, caso elas não não funcionem, reverta as alterações. Essas alterações são especificas para disco usando api "VirtIO" e possivelmente você não as utilizou quando criou sua VM.   


## Pré-requisitos e cuidados

* Faça **backup** da imagem QCOW2.
* Certifique-se de que a **VM esteja desligada**.
* Garanta espaço suficiente no filesystem do host.

Exemplo de caminho da VM:

```
/home/libvirt/images/win2k25.qcow2
```

Parar a VM:

```bash
sudo virsh shutdown win2k25
sudo virsh destroy win2k25  # se necessário
```

---

## Passo 1 — Verificar tamanho da imagem

```bash
cd /home/libvirt/images/
ls -lh *.qcow2
```

Exemplo:

```
-rw------- 1 root kvm 28G nov  5 14:38 win2k25.qcow2
```

---

## Passo 2 — Verificar integridade

```bash
sudo qemu-img check -r all /home/libvirt/images/win2k25.qcow2
```

Saída típica:

```
No errors were found on the image.
335077/3276800 = 10.23% allocated, 17.39% fragmented, 0.00% compressed clusters
Image end offset: 29463674880
```

---

## Passo 3 — Otimização com virt-sparsify

A ferramenta `virt-sparsify` do pacote `libguestfs-tools` remove blocos não utilizados e pode reduzir o tamanho físico da imagem.

### Opção A — Otimização **in-place** (mantém o mesmo arquivo)

```bash
cd /home/libvirt/images/
sudo virt-sparsify --in-place win2k25.qcow2
```
O seguinte resultado é o esperado:  
```
[   2.0] Trimming /dev/sda1
[   2.1] Trimming /dev/sda2
[   3.9] Trimming /dev/sda3
[   4.0] Sparsify in-place operation completed with no errors
```
O comando realiza uma **desfragmentação lógica** da imagem QCOW2, consolidando os espaços vazios para o final do arquivo enquanto mantém seu tamanho original. Durante este processo, operações de **trimming** sinalizam ao formato QCOW2 quais blocos estão realmente vazios, permitindo que o Windows reconheça este espaço como efetivamente disponível para novas alocações de arquivo. Isso otimiza significativamente a performance da VM porque, com os espaços vazios consolidados e sinalizados, o SO convidado pode alocar novos arquivos sem que o QEMU precise realizar custosas operações de **growing** — o processo onde a imagem QCOW2 precisa ser expandida para armazenar mais dados, consumindo recursos e aumentando latência. Embora o arquivo permaneça no mesmo tamanho, essa otimização de trimming é suficiente para melhorar a performance do Windows, eliminando o overhead desnecessário de expansão de imagem e tornando as operações de I/O mais previsíveis e eficientes.  

Essa é a melhor opção para mim, pois melhora suficientemente a performance do sistema Windows convidado sem alterar o tamanho alocado do arquivo dinâmico. Porém, durante certas operações é comum a imagem **crescer artificialmente**, um exemplo prático: descompactar um arquivo grande várias vezes dentro do Windows convidado e depois deletá-lo. Embora o arquivo tenha sido removido do Windows convidado, o tamanho da imagem QCOW2 nunca recuará — permanecerá no tamanho máximo atingido. Para recuperar esse espaço desperdiçado e realmente compactar a imagem, temos a Opção **"B"** a seguir.

### Opção B — Criar cópia **compactada**
Tem como usar `virt-sparsify` para compactar uma imagem, é assim:  
```bash
cd /home/libvirt/images/
sudo virt-sparsify --compress win2k25.qcow2 win2k25-optimized.qcow2
```
Mas eu não gosto de usá-la porque demora muito tempo e tem restrições de espaço que se não forem compreendidas, o comando nunca termina, então para compactar vamos ao jeito mais burro que existe que será usando o utilitário `qwmu-img`, eu uso a palavra `burro` porque se trata de reconstituir um arquivo novo a partir do velho removendo a desfragmentação. Contudo, é muito mais rápido do que usando o `virt-sparsify` para a mesma tarefa, vamos então usar o `qwmu-img` para compactar nossa imagem, execute:    
```bash
cd /home/libvirt/images/
sudo qemu-img convert -p -O qcow2 -c -o compat=1.1,cluster_size=1M,lazy_refcounts=on \
  win2k25.qcow2 win2k25-otimized.qcow2
```
Ele vai gerar uma novo arquivo `win2k25-optimized.qcow2` otimizado e compactado, veja a redução de tamanho:  
```bash
$ ls -lh *.qcow2
-rw-r--r-- 1 root kvm 17G nov  6 10:03 win2k25-otimized.qcow2
-rw------- 1 root kvm 28G nov  6 09:42 win2k25.qcow2
```
De 28G para 17G, nada mal hein?  
A opção `cluster_size=1M` para mim é a mais significativa porque ela transforma o tamanho de página de 64K para 1M, parece que isso o fará inchar, mas o efeito é inverso, menos fragmentação interna, leituras e gravações sequenciais mais rápidas e reduz metadados. O objetivo lembre-se é maximizar performance, se a situação fosse economizar espaço, um cluster de 4K iria fazer uma eonomia ainda maior, mas muito menos performático.

. Daí executamos um *swap atômico*, um termo que significa trocar o arquivo, mas caso falhe depois ainda podemos retornar o original:  
```bash
mv win2k25.qcow2 win2k25.qcow2.bak
mv win2k25-optimized.qcow2 win2k25.qcow2
```
E novamente, depois de validar o boot, remova o `.bak`, mas se falhar renomeie o `.bak` para o nome do arquivo original.  
O primeiro boot após a otimização usando este método será mais lento, mas não se preocupe, a culpa disso, é o `growing`, como o tamanho final da imagem  `qcow2` representa os dados sem nenhum espaço livre, o sistema irá criá-los na primeira execução.  Depois disso, a velocidade se normalizará até que haja um novo `growing`. Por isso, não use a opção **"B"** diariamente, use-a apenas depois de situações que descrevi no inicio. Agendar esse tipo de operação para ser diário ou mensal é perda de tempo, novamente, use-a apenas em situações que descrevi no inicio.   
  
---

## Passo 4 — Snapshots e cadeias de backing

* Listar snapshots:

  ```bash
  qemu-img snapshot -l /home/libvirt/images/win2k25.qcow2
  ```
* Remover snapshot interno:

  ```bash
  qemu-img snapshot -d <ID> /home/libvirt/images/win2k25.qcow2
  ```
* “Flatten” (quando há overlay/backing):

  ```bash
  qemu-img convert -p -O qcow2 overlay.qcow2 flattened.qcow2
  ```

---

## Validação e métricas

```bash
du -h /home/libvirt/images/win2k25*.qcow2
qemu-img info /home/libvirt/images/win2k25.qcow2
```

Ver mapa de alocação:

```bash
qemu-img map --output=json /home/libvirt/images/win2k25.qcow2 | jq .
```

Benchmark rápido:

```bash
qemu-img bench -c 4k -d 1G -f qcow2 /home/libvirt/images/win2k25.qcow2
```

---
## Permissões nos arquivos
Se fez ajustes e criou novos arquivos, então é razoável conferir se as permissões estão corretas, execute:  
```bash
sudo find /home/libvirt -type f -exec chmod 666 {} \; -o -type d -exec chmod 777 {} \;
sudo find /home/libvirt -type f -exec chown $USER:kvm {} \; -o -type d -exec chown $USER:kvm {} \;
```

## Boas práticas operacionais

* Sempre mantenha backups antes de qualquer otimização.
* Execute com a VM desligada.
* Padronize parâmetros QCOW2 conforme o tipo de carga:

  * **cluster_size=1M** → I/O sequencial intenso (DBs, backup).
  * **cluster_size=128K–256K** → I/O aleatório pequeno (sistemas).
* Monitore espaço e fragmentação periodicamente com `qemu-img info`.

---

[Retornar à página de Virtualização nativa com QAEMU+KVM Usando VM/Windows](debian_qemu_kvm_windows.md)   



