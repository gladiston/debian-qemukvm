# Compactar o Arquivo QCOW2
Arquivos que representam o disco da VM usam o formato qcow2, que vai inxando com o tempo.
Então vez ou outra voce precisa compactá-las, mas não é uma compactação do estilo zip, essa é uma compactação que envolve a desfragmentação eliberando os espaços de setores vazios do disco.  
Para começar o processo, desligue a VM.  
Para reduzir o tamanho do disco, eliminando blocos vazios, use:

```bash
$ sudo virt-sparsify --in-place /home/libvirt/images/win2k25.qcow2
[   2.6] Trimming /dev/sda1
[   2.7] Trimming /dev/sda2
[   4.0] Trimming /dev/sda3
[   4.1] Sparsify in-place operation completed with no errors
```
O comando realiza uma desfragmentação lógica da imagem QCOW2, consolidando os espaços vazios para o final do arquivo enquanto mantém seu tamanho original. Durante este processo, operações de trimming sinalizam ao formato QCOW2 quais blocos estão realmente vazios, permitindo que o Windows reconheça este espaço como efetivamente disponível para novas alocações de arquivo. Isso otimiza significativamente a performance da VM porque, com os espaços vazios consolidados e sinalizados, o SO convidado pode alocar novos arquivos sem que o QEMU precise realizar custosas operações de growing — o processo onde a imagem QCOW2 precisa ser expandida para armazenar mais dados, consumindo recursos e aumentando latência. Embora o arquivo permaneça no mesmo tamanho, essa otimização de trimming é suficiente para melhorar a performance do Windows, eliminando o overhead desnecessário de expansão de imagem e tornando as operações de I/O mais previsíveis e eficientes.  


---

## 🧩 Conclusão

A conversão de discos **VDI>QCOW2** é o caminho mais prático para migrar VMs do VirtualBox para o QEMU/KVM.
Com essa abordagem:

* você mantém todos os dados intactos,
* aproveita o desempenho nativo do KVM,
* e ainda ganha recursos avançados como snapshots e backup incremental.

Essa técnica é ideal tanto para **migrações definitivas** quanto para **testes de performance** em ambientes Linux modernos.

