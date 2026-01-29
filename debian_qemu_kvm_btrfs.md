# VIRTUALIZAÇÃO NATIVA QEMU+KVM E BTRFS
Se a partição onde guarda as VMs é do tipo ext4, ignore este tópico, pule para o próximo.  
Mas, se a partição for do tipo Btrfs, então carece de alguns acertos, por que? Porque este tipo de partição faz uma série de operações no disco e algumas delas são anti-performaticas para carregamento de VMs, são elas:  

* **CoW**: O Copy-on-Write(CoW) é um recurso do Btrfs que (1) quando um arquivo é modificado, ele não é alterado diretamente e (2) o sistema cria uma nova cópia dos blocos modificados e só depois descarta os antigos e isso protege contra corrupção e permite que snapshots instantaneos sejam criados, mas também significa que cada gravação cria fragmentação e sobrecarga de I/O. E agora? Uma coisa interessante é que o CoW pode ser desligado por pastas, então vamos fazer isso à pasta onde as imagens serão armazenadas, mas atenção, a pasta deve estar vazia, agora execute:
```
sudo chattr +C /home/libvirt
```
* **Compressão de dados**: No seu tempo ocioso, o Btrfs vai compactar seus arquivos e ele faz isso de maneira efetiva sem você perceber, não se preocupe, ele não faz isso nos arquivos em uso, mas em maquinas virtuais que são arquivos grandes e são modificados a todo instante, a ideia de compactar não é boa idéia porque gera mais processamento e I/O que rouba recursos que poderiam estar indo para as VMs em uso, então o que fazer? A solução é (1) você configurar no virtualizador que crie arquivos seguimentados, ao inves de uma única VM de tamanho contiguo. Ativando este recurso, o programa irá separá-los em vários arquivos menorescomo continuação da sessão anterior sem nunca sobregravá-los, o lado ruim desse método é que ele vai ocupar muito, mas muito espaço. A outra solução, (2) é desabilitando a compressão na partição onde as VMs estão localizadas, e nesse caso, vamos pelo jeito mais simples, quando você for mais experiente, crie volumes separados para VMs para não ter que desligar a compressão para a partição/disco inteiro como faremos agora, edite o arquivo /etc/fstab e procure pela representação do seu disco/partição Btrfs, no meu exemplo, esta assim:
  
```
UUID=c045fd1f-7c4f-4ec3-84d9-ec79f8859adf /               btrfs   defaults,subvol=@rootfs 0       0
```
Agora, junto com as opções 'default', você acrescenta ',compress=no', ficando assim:
```
UUID=c045fd1f-7c4f-4ec3-84d9-ec79f8859adf /               btrfs   defaults,subvol=@rootfs,compress=no 0       0
```
Salve e feche o arquivo (`Ctrl+O`, `Enter`, `Ctrl+X`).    
Notou o **compress=no** na linha acima? Ela desligará a compressão na partição de montagem, agora execute:  
```
sudo systemctl daemon-reload
```
Note que agora, a *compressão zstd* para a unidade inteira esta desligada, significando que todos os arquivos ocuparão mais espaços.  
Recomendo que *reinicie o computador* antes de prosseguir.   

Depois de reiniciar o computador, abra o terminal e execute:  
```
$ sudo btrfs filesystem df /
Data, single: total=19.01GiB, used=15.59GiB
System, DUP: total=8.00MiB, used=16.00KiB
Metadata, DUP: total=2.00GiB, used=333.94MiB
GlobalReserve, single: total=35.06MiB, used=0.00B
```
Se não aparecer a palavra “*Compressed*” então significa que nenhum dado comprimido está sendo escrito — a compressão está efetivamente desativada.  

**DESFRAGMENTAÇÃO DE PASTA BTRFS**  
Algo também muito recomendado é a desfragmentação da pasta, pois a propriedades CoW do Btrfs causam fragmentação, o que não é nenhum problema para arquivos normais já que o algoritimo do CoW resolve isso, mas as imagens de VMs costumam ser grandes e mudam seu conteúdo a cada uso então nesta situação é bom desfragmentar de tempos em tempos. Isso pode ser feito com o comando:   
```
sudo btrfs filesystem defragment -r "/home/$USER/libvirt/images"
```
Se for possivel, use o agendador de tarefas do Linux para rodá-lo num horário programado, execute o comando **crontab -e** e adicione a linha:  
```
0 12 * * * /bin/bash -c '/usr/bin/btrfs filesystem defragment -r "/home/gsantana/libvirt/images"'
```
O comando acima, no horário 12:00 (almoço) fará a desfragmentação da pasta mencionada.  
