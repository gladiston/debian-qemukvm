# Como abrir arquivos qcow

Primeiro vamos listar as imagens:

```bash
ls -lh /var/lib/libvirt/images
```

Digamos que o resultado seja:

```context
-rw-rw----+ 1 libvirt-qemu libvirt-qemu 45G abr 16 16:09 win11-dx11.qcow2
```

Agora você precisa decorar estes comandos:

## Descobrindo o sistema operacional

Isso nem sempre é possivel, por isso é muito prático ter um prefixo ou sufixo que começe com `win11-nonono` para tipificar `Windows 11` ou `debian-nonono`  para um Linux/Debian. 

Então, vamos ao comando:

```bash
sudo guestfish --ro -a /var/lib/libvirt/images/win11-dx11.qcow2
```

Daí você entra no modo de prompt dentro deste arquivo, os comandos que você tem disponível podem ser descobertos com o comando:

```bash
help --list
```

São dezenas de comandos, mas os mais básicos de que precisa são:

```bash
rn
list-partitions
```

E lhe será mostrado:

```
/dev/sda1
/dev/sda2
/dev/sda3
/dev/sda4
```

Depois de listar as partições, podemos pedir para ele listar apenas as partições que ele reconhece:

```bash
list-filesystems
```

E lhe será mostrado:

```
/dev/sda1: vfat
/dev/sda4: ntfs
```

E agora que sabemos que `/dev/sda4` é a partição como `ntfs` então:

```bash
mount-ro /dev/sda3 /
```

Agora modelos listar com o comando `ll`:

```bash
ll /
```

Para sair, basta usar o comando `quit` .

Agora, estamos prontos para a montagem por fora do prompt do comando `guestfish`:

```bash
sudo mkdir -p /mnt/qcow2
sudo guestmount -a /var/lib/libvirt/images/win11-dx11.qcow2 -m /dev/sda4 --ro /mnt/qcow2
```

Agora que está montado, vamos listar o conteúdo:

```bash
sudo ls -lh /mnt/qcow2
```

E então o seu conteúdo será exibido.

## Descobrindo o sistema de arquivos dentro de cada disco virtual usando  o virt-filesystems

Este é outro método, ele não é melhor ou pior, mas é mais simples, apenas execute:

```bash
sudo virt-filesystems --all --long -h -a /var/lib/libvirt/images/win11-dx11.qcow2
```

O resultado será algo como:

```context
Name       Type        VFS   Label  MBR  Size  Parent
/dev/sda1  filesystem  vfat  -      -    196M  -
/dev/sda4  filesystem  ntfs  -      -    749M  -
/dev/sda1  partition   -     -      -    200M  /dev/sda
/dev/sda2  partition   -     -      -    16M   /dev/sda
/dev/sda3  partition   -     -      -    199G  /dev/sda
/dev/sda4  partition   -     -      -    749M  /dev/sda
/dev/sda   device      -     -      -    200G  -
```

Agora, temos uma lista de partições, embora algumas foram identificadas como `vfat` e `ntfs`, outras foram apresentadas como '-' ou seja, sem identificação. Isso não quer dizer que a partição interna esteja com problemas, apenas que o utilitário não foi capaz de identificar, mas provavelmente você já sabe que se é um disco com Windows 11 instalado então qualquer que seja a partição, será `ntfs`. Agora que já sabemos disso, vamos escolher uma dessas partições para montar, minha escolha pelo tamanho é:

```text
/dev/sda3  partition   -     -      -    199G  /dev/sda
```

 Como você viu, no exemplo anterior, usei `/dev/sda4` como `ntfs`, mas neste outro exemplo, temos muitas outras e daí pelo tamanho, podemos observar que a partição de 199G `/dev/sda3` é a mais provável . Então, vamos montar da mesma forma que o fizemos da vez anterior:

```bash
sudo mkdir -p /mnt/qcow2
sudo guestmount -a /var/lib/libvirt/images/win11-dx11.qcow2 -m /dev/sda3 --ro /mnt/qcow2
```




