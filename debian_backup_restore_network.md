# BACKUP/RESTORE DA CONFIGURAÇÃO ORIGINAL DE REDE
Vamos ser cautelosos e fazer um backup de nossa configuração de rede atual, assim se algo der errado, restauramos.
Veja como está sua configuração de rede atual, execute:
Depois observe o status, execute:
```bash
nmcli general status
```
E observe o resultado:  
```
STATE      CONNECTIVITY  WIFI-HW  WIFI        WWAN-HW  WWAN        METERED          
conectado  completa      missing  habilitado  missing  habilitado  não (adivinhado) 
```
Observe suas conexões atuais:  
```bash
nmcli con show
```
E observe os resultados:  
```
NAME                UUID                                  TYPE      DEVICE 
Wired connection 1  aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee  ethernet  enp8s0 
lo                  bbbbbbbb-cccc-dddd-eeee-ffffffffffff  loopback  lo  
```

Agora que viu sua configuração de rede original, escolha uma pasta para guardar o backup dessa configuração de rede, minha sugestão é **~/net-backups**, mas você pode escolher outra. Primeiro vamos apontar a variavel para a localização de seus backups:  
```bash
export BKP=~/net-backups
```

Depois criamos a pasta:
```bash
mkdir -p $BKP
```

Depois, vamos definir o nome do arquivo de backup com o seu nome:
```bash
export BKPFILE=$BKP/netcfg-$(date +%F_%H%M%S).tgz 
```

Depois, geramos um relatorio em formato .txt para posterior consulta e usamos o tar para compactar os arquivos de configurações atuais no local de destino:  
```bash
nmcli general status >$BKPFILE.txt
nmcli con show >>$BKPFILE.txt
ip -br a >>$BKPFILE.txt
sudo tar -C / -czf $BKPFILE   etc/network etc/NetworkManager etc/systemd/network
sudo chown -R $USER:$USER $BKP
```

Agora, verifique o arquivo que foi gerado:
```bash
ls -lh $BKP/
```
E verá algo similar a isso:
```
-rw-r--r-- 1 gsantana gsantana 2,3K out 23 16:57 netcfg-2025-10-22_165719.tgz
-rw-rw-r-- 1 gsantana gsantana  399 out 23 17:10 netcfg-2025-10-22_165719.tgz.txt
```
Como pôde ver, a composição dos arquivos que foram compactados são apenas alguns arquivos textos pequenos, quase nada em termos de espaço. Também criamos uma arquivo `.txt` que armazena os resultados dos comandos que você viu no inicio desse tutorial e mais tarde será possível comparar o antes e o depois. Se você não vê nenhum dos dois arquivos, o `.tgz` e o `.txt` ou estão vazios, então algo deu errado nos passos anteriores e é melhor revisá-los e não prosseguir.

### RESTAURAÇÃO BACKUP
Se mais tarde ocorrer alguma pane e precisar restaurar a configuração de rede, faça o seguinte, primeiro vamos apontar a variavel `BKP` para a localização de seus backups:  
```bash
export BKP=~/net-backups
```

Depois liste todos os seus backups e escoha o que for apropriado:  
```bash
$ ls -1 $BKP/
netcfg-2025-10-22_165719.tgz
```

Determinado qual o arquivo de backup desejado, execute:
```bash
sudo tar -C / -xzf $BKP/netcfg-2025-10-22_165719.tgz
```

Os arquivos de configuração serão substituidos pelo backup, porém o gerenciador que os controla precisa ser reiniciado:   

Vamos reiniciar, execute:  
```bash
sudo systemctl restart NetworkManager 
```
----

[Clique aqui para retornar a página principal](../README.md#backuprestore-da-configura%C3%A7%C3%A3o-original-de-rede)
