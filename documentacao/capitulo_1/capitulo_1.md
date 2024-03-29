# **Instalações e orientações sobre suporte e versões**

<br/>

## **Instalação do CentOS**

- [**Download**](https://www.centos.org/download/ "CentOS download")

- [**Mirror List**](https://www.centos.org/download/mirrors/ "List of CentOS official mirrors")

- [**Isos CentOS-7**](http://mirror.ci.ifes.edu.br/centos/7.9.2009/isos/x86_64/ "Isos Instituto Federal Espírito Santo")

### **Atualizar o sistema**
```bash
yum update
```

### **Configurar timezone**

```bash
timedatectl set-timezone America/Sao_Paulo
```

### **Desabilitando *firewall* e o *Selix* (conjunto de restrições de segurança extra em cima das ferramentas de segurança normais do linux) em ambiente de teste**

- ### **Acesse o usuário root:**

  ```bash
  su
  ```
  
  ```bash
  service firewalld stop
  ```
  
  ```bash
  systemctl disable firewalld
  ```
  
  ```bash
  vim /etc/sysconfig/selinux
  # SELINUX=disable
  ```

<br/>

## **Instalação do cluster PostgreSQL**

Seguir passo a passo conforme descrito em documentação oficial

- [**PostgreSQL Downloads**](https://www.postgresql.org/download/ "Packages and Installers")
<br/>

### **Pacote *contrib***

Comando responsável por realizar a instalação do servidor PostgreSQL juntamente com o pacote contrib, que permite a instalação das *extensions*, ou seja resumindo muito, se trata de um pacote contendo uma série de opcionais que podem ser instalados sobre demanda. 

```bash
yum install -y postgresql14-server postgresql14-contrib
```

<br/>

## **Configurações iniciais**

Depois da instalação para que possamos acessar o cluster PostgreSQL, e necessario realizar algumas configurações nos arquivos **postgresql.conf** e **pg_hba.conf**.

O arquivo **postgresql.conf** esta ajustado inicialmente apenas para configurações locais. Podemos alterar isso modificando o parâmetro **listen_addresses**:


- **Primeiramente o arquivo *postgresql.conf***<br/>
Acessando arquivo:

  ```bash
  var/lib/pgsql/14/data/postgresql.conf
  ```

- **Configuração padrão**
  ```conf
  #------------------------------------------------------------------------------
  # CONNECTIONS AND AUTHENTICATION
  #------------------------------------------------------------------------------

  # - Connection Settings -

  #listen_addresses = 'localhost'         # what IP address(es) to listen on;
  ```

- **Alterando parâmetro permitindo conexões remotas**
  ```conf
  #------------------------------------------------------------------------------
  # CONNECTIONS AND AUTHENTICATION
  #------------------------------------------------------------------------------

  # - Connection Settings -

  #listen_addresses = '*'         # what IP address(es) to listen on;
  ```

- **Habilitando as conexões no *pg_hba.conf***<br/>
Acessando arquivo:

  ```bash
  vim /var/lib/pgsql/14/data/pg_hba.conf
  ```

- **Configuração padrão**
  ```conf
  # TYPE  DATABASE        USER            ADDRESS                 METHOD

  # "local" is for Unix domain socket connections only
  local   all             all                                     peer
  # IPv4 local connections:
  host    all             all             127.0.0.1/32            scram-sha-256
  # IPv6 local connections:
  host    all             all             ::1/128                 scram-sha-256
  # Allow replication connections from localhost, by a user with the
  # replication privilege.
  local   replication     all                                     peer
  host    replication     all             127.0.0.1/32            scram-sha-256
  host    replication     all             ::1/128                 scram-sha-256
  ```

- **Alterando parâmetro para habilitar conexão**
  ```conf
  # TYPE  DATABASE        USER            ADDRESS                 METHOD

  # "local" is for Unix domain socket connections only
  local   all             all                                     peer
  # IPv4 local connections:
  host    all             all             127.0.0.1/32            scram-sha-256
  # IPv6 local connections:
  host    all             all             ::1/128                 scram-sha-256
  # Allow replication connections from localhost, by a user with the
  # replication privilege.
  local   replication     all                                     peer
  host    replication     all             127.0.0.1/32            scram-sha-256
  host    replication     all             ::1/128                 scram-sha-256
  host    all             all             0.0.0.0/0               scram-sha-256
  ```

  Com isso, depois do processo de ***reload*** do cluster, e possivel conectar ao database remotamente com qualquer ferramenta (por exemplo, PgAdmin). **Esses valores liberam a conexão com senha scram-sha-256 a qualquer host, não sendo, recomendados para ambiente de produção**.

- **Definindo senha do user *postgres***<br/>
  Inserindo senha ao user postgres por meio o utilitário ***psql***, que é instalado juntamente com o **PostgreSQL**.

  ```bash
  psql
  ```

  ou

  ```bash
  sudo -i -u postgres psql
  ```

  ```sql
  alter user postgres with encrypted password '<password>';
  ```

<br/>

 [**Home**](../../README.md) |====| [**==>>**](../capitulo_2/capitulo_2.md)

<br/>
