# Controle do VirtualBox atraves da Linha de Comando

Durante pratica do conteudo apresentado no livro, em certos casos utilizei mais de uma VM ao mesmo tempo, sendo assim para poupar processamento as mesmas foram iniciadas em modo `headless`. 

Sendo assim a partir de certo ponto nao fora utilizado o ambiente grafico e sim realizado a conexao SSH para realizacao dos processos abordados no livro... 

Abaixo alguns comandos basicos para gerenciar VM's no VirtualBox por linha de comando

### Listando VMs existentes em sua maquina

```bash
VBoxManage list vms
```

<br/>

### Listando VMs em execucao

```bash
VBoxManage list runningvms
```

<br/>

### Iniciando VM para acesso remoto, onde a mesma e iniciada porem sem interface grafica

```bash
VBoxManage startvm 1-OracleLinux9.3 --type headless
```

<br/>

### Desligando VM's de maneira limpa

```bash
VBoxManage controlvm 1-OracleLinux9.3 acpipowerbutton
```

<br/>

### Desligando VM's a forca 

```bash
VBoxManage controlvm 1-OracleLinux9.3 poweroff
```

<br/>

---

<br/>

[**Ambiente**](../ambiente.md)

<br/>