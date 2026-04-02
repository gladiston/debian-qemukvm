# Criando uma interface de rede do tipo bridge no Debian e Ubuntu (QEMU/KVM)

Diferente do `macvtap`, a Bridge permite que o Host e a Máquina Virtual (VM) se comuniquem entre si (acesso ao Samba, SSH, etc), além de permitir que a VM seja vista como um dispositivo físico na rede local.

Debian e Ubuntu divertem no modo de como configurar este tipo de rede, não é que as instruções para Debian não funcionem no Ubuntu, elas funcionam, no entanto, o Ubuntu abandonou o jeito de configurar a rede via edição de arquivos, nesta distro, usa-se framework de rede chamada de **NetworkManager**.

O Debian, por outro lado, mantêm o jeito classico de sempre, editando arquivos manualmente.

---

## Criando uma interface bridge no Debian

Siga as instruções no link a seguir:

[Instruções para criar uma interface do tipo bridge no Debian](debian_qemu_kvm_bridge_ponte_debian.md)

## Criando uma interface bridge no Ubuntu

[Instruções para criar uma interface do tipo bridge no Ubuntu](debian_qemu_kvm_bridge_ponte_ubuntu.md)



---

[Retornar](https://github.com/gladiston/debian-qemukvm)


