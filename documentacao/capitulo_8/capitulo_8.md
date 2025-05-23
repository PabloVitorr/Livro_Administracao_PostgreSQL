# **Executando backups e restaurações no cluster PostgreSQL**

**Existem vários tipos de backups que podem ser aplicados ao cluster PostgreSQL, sendo que cada um cobre cenários diferentes:**

- **Cold backup**
  Backup físico em que o database é desligado antes da cópia de seus arquivos.

- **Hot backup**
  Backup realizado com o database em funcionamento. Pode apresentar problemas de consistência, mas não gera indisponibilidades ao sistema. É de dois tipos, físico e lógico:
    
    - **Backup físico**
      São copiados os arquivos do banco de seus diretorios de origem. Esse tipo de backup normalmente é realizado com o databaase desligado, mas pode ser implementado com ***hot backup*** com mecanismos de geração e gerenciamento de **WAL**. Esse tipo de backup não é utilizado ara migrações de versão. Permite a recuperação **PITR** e Total usando os arquivos de vetores transacionais.

    - **Backup lógico**
      Realizado com o banco aberto, em que são gerados arquivos com os comandos SQL para recriação do ambiente, independe de ambiente de hardware e pode ser usado para migração de dados. Permite a cópia de objetos individuais, são extraídos os dados e os metadados, gerando scripts que podem ser aplicados para restaurar um banco de dados a uma determinada posição.

- **Arquivos de carga**
  Backup onde são utilizadas ferramentas **ETL**, sendo gerados arquivos de dados e scripts de criação, armazenados como cópia de segurança ou para migração.

- **Backup dos arquivos de vetores de alteração**
  **Ao realizar o backup completo de um database, estamos realizando a cópia de uma posição no tempo. Um banco ativo, depois de algum tempo, terá uma quantidade enorme de registros alterados ou inseridos que, em caso de falha do servidor, poderão ser perdidos. Com base no exemplo apresentado pode ser necessário salvar os arquivos de *vetores de alteração*. Esses arquivos são chamados no PostgreSQL de *archives WAL*. Os vetores de alteração registram todas as modificações, inserções, deleções e alterações no database. Com um backup base realizado, podemos aplicar essas alterações a ele e conseguir recuperar todos os dados até o momento do último vetor de transação gravado, ou realizar uma restauração PITR e voltar a algum momento anterior ao problema.**

<br/>

---

<br/>

## **Write-ahead logging**

**Write-ahead logging** (registro prévio da escrita), é um conjunto de métodos para garantir a integridade dos dados que realiza a gravação dos vetores de atualização (operações de **insert**, **update** e **delete**), antes que sejam definitivamente salvos nos arquivos de dados, em um arquivo de log. **Com esses métodos, caso aconteça uma falha, os dados salvos no WAL podem ser utilizados com a finalidade de levar o banco de dados a um estado íntegro, outro ponto é referente a gravação em disco que é uma operação trabalhosa que exige dedicação do sistema, pode ser realizada em lote, minimizando acesso para ela**.

O PostgreSQL mantém um ***cache do buffer***, no qual são lidos os blocos de dados, quando os dados devem ser recuperados. As modificações de dados não são feitas diretamente no disco, mas na cópia em memória dos blocos de dados no ***buffer cache***. A modificação realizada não é gravada no disco/armazenamento até que ocorra um **checkpoint** no banco de dados. A gravação desses blocos modificados do ***buffer cache*** no disco é chamada de **liberação de blocos** onde um bloco modificado no cache, mas ainda não gravado no disco, é chamado de **bloco sujo**.

O uso de **WAL** reduz o número de gravações de disco, onde apenas alterações são gravadas nos arquivos de log, em vez de todos os arquivos de dados modificados pela transação. O arquivo de log é escrito sequencialmente e, portanto o custo da sincronização dele é muito menor que o custo de sincronização das páginas de dados. **WAL** permite suportar o backup on-line e a recuperação **PITR** e serve como suporte para a implementação de replicação. 

<br/>

---

<br>

## **Métodos de backup e restauração**

### **Backup consistente a frio (físico)** <br>
  Basicamente, o procedimento é **parar** o cluster PostgreSQL, copiar o diretório ***data***, o ***tablespace*** e os arquivos de configuração (postgresql.conf, pg_hba.conf) e ativar novamente o cluster.

### **Backup pontual a quente (lógico)** <br/>
  Consiste na realização de um **dump**, lógico, em que o arquivo gerado é na verdade uma sequência de comandos SQL que deve retomar o cluster no momento de início da operação. Esse processo permite cópias de bases individuais e objetos isolados, é popular em migrações entre versões e plataformas. É um backup realizado com o servidor ativo, não gerando portanto, indisponibilidade.

- **pg_dump** <br/>
    É possível realizar backups remotos de várias formas, informando o host (-h) e a porta (-p), é possível também compactar o arquivo (-z) ou apenas os schemas (-s) sem qualquer dado, entre outras formas:

    ```bash
    pg_dump -h <host> <dbname> > outfile
    ```

    - **`pg_dump`** <br/>
      É o comando para iniciar a ferramenta **pg_dump**.
    
    - **`-h host`** <br/>
      Especifica o host onde o banco de dados esta localizado.
    
    - **`dbname`** <br/>
      Nome do banco de dados a ser feito o backup.

    - **`> outfile`** <br/>
      Redireciona a saída do comando para um arquivo especifico.

	    ![1.png](./img/1.png)

		ao acessarmos o arquivo gerado e possível vermos a estrutura do banco de dados e os dados nas instruções SQL. 

		![2.png](./img/2.png)

		![3.png](./img/3.png)

    ```bash
    pg_dump mydatabase > database.sql
    ```

    - **`pg_dump mydatabase > database.sql`** <br/>
      Comando semelhante ao anterior, porém não especifica o host. Se nenhum host é fornecido, o **pg_dump** assume que o banco de dados está localizado no **host local** (*localhost*).
    
    ```bash
    pg_dump -h $HOST -U postgres -F -d mydatabase > /backups/mydatabase.sql
    ```
    
    - **`-U postgres`** <br/>
      Especifica o usuário do PostgreSQL para se conectar ao banco de dados.
    
    - **`-F`** <br/>
      Especifica o formato do arquivo de saída.
    
    - **`-d mydatabase`** <br/>
      Especifica o nome do banco de dados a ser feito o **backup**.

    - **`> /backups/mydatabase.sql`** <br/>
      Redireciona a saída do comando para o arquivo **`/backups/mydatabase.sql`**

    **Dando sequencia é possível também exportar apenas os schemas, sem dados:**

    ```bash
    pg_dump -h $HOST -U postgres -F -s -d mydatabase > /backups/mydatabase_without_data.sql
    ```

- **pg_dumpall** <br/>
Semelhante ao **pg_dump** porem utilizado para realizar a copia do cluster inteiro (todas as bases de dados do cluster).

### **Backup a quente (fisico) com arquivamento**

Neste processo e realizado a copia dos arquivos fisicos. **O utilitario **pg_basebackup** possibilita a realizacao de copias binarias apenas da base de dados completa**. <br/>
Este utilitario pode ser utilizado para copias simples, recuperacao pontual, recuperacao **PITR**, copia inicial para para clonar um cluster, servidor standby etc...

#### **Realizando um processo de exemplo com a criacao das areas de armazenamento atraves dos seguintes diretorios (neste caso ja sendo criados no diretorios `/home/hw` utilizado anteriormente, e tambem ja com o usuario `postgres` por questoes de permissoes de acesso) e configuracao do `archiving`**

1. No servidor principal (servidor utilizado ate o momento), realizado a criacao dos seguintes diretorios
   1. /home/hw/

    ```bash
    mkdir pg_backup
    ```

   2. `/home/hw/pg_backup/`

    ```bash
    mkdir archive_wal
    ```

   3. `/home/hw/pg_backup/`

    ```bash
    mkdir base
    ```

2. Estrutura dos diretorios:

      ![](./img/4.png)

3. Configurando agora o PostgreSQL, alterando as seguintes configuracoes
   1. `wal_level` = archive
   2. `max_wal_senders` = 10
   3. `archive_mode` = on
   4. `archive_command` = 'cp %p /home/hw/pg_backup/archive_wal/%f </dev/null'
   5. `archive_timeout` = 180

4. Validando as configuracoes do PostegreSQL antes das alteracoes

  ```sql
  SELECT name, context, setting FROM pg_settings WHERE name = 'wal_level' OR name = 'max_wal_senders' OR name = 'archive_mode' OR name = 'archive_command' OR name = 'archive_timeout';
  ```

  ![](./img/5.png)

5. Alterando

    ```sql
    ALTER SYSTEM SET wal_level = archive;
    ```

    ```sql
    ALTER SYSTEM SET max_wal_senders = 10;
    ```

    ```sql
    ALTER SYSTEM SET archive_mode = on;
    ```

    ```sql
    ALTER SYSTEM SET archive_command = 'cp %p /home/hw/pg_backup/archive_wal/%f </dev/null';
    ```

    ```sql
    ALTER SYSTEM SET archive_timeout = 180;
    ```

    ![](./img/6.png)

6. Apos estas configuracoes realizado o restart do banco

    ```sql
    systemctl restart postgresql-14
    ```

     1. Validado alteracoes realizadas

        ![](./img/8.png) 

7. Realizando o backup com o utilitario **pg_basebackup** atraves do seguinte comando:

    ```bash
    pg_basebackup -h localhost -U postgres -D /home/hw/pg_backup/base/bkp_one.dmp -P -Ft
    ```

    ![](./img/9.png)

    ![](./img/10.png)

#### **Recuperacao completa com arquivos de WAL**

Realizando processo de restauracao em outro servidor **_usuario_standby_**. Neste caso o servidor de destino devera ter a mesma estrutura e versao do servidor de origem...

1. Realizado o processo de instalacao padrao do S.O e do PostgreSQL no novo servidor. Para facilitar o processo a seguir fora realizado a conexao em ambos os servidores por SSH.

    ![](./img/20.png)

2. Recriado os diretorios no servidor **standby**
   
   ![](./img/11.png)

3. Recriado o usuario **hw** no servidor **standby** e a tablespace **tbs_hw**

    ```sql
    CREATE ROLE hw WITH LOGIN PASSWORD '<password>' CREATEDB CREATEROLE;
    ```

    ```sql
    ALTER USER hw SUPERUSER;
    ```

    ```sql
    CREATE TABLESPACE tbs_hw LOCATION '/home/hw/tablespaces/pg_tbs_hw';
    ```

4. Configurado o utilitario **pg_ctl**

5. Realizado a transferencia completa do diretorio **/base** para o servidor **standby** e alterado as permissoes do mesmo...

    ```bash
    chmod 700 bkp_one.dmp/
    ```

    ```bash
    chmod 600 bkp_one.dmp/*
    ```

    ```bash
    chown -R postgres:postgres bkp_one.dmp/
    ```

    ![](./img/12.png)

6. Restaurando o backup
    1. Parando o banco de dados no servidor **standby**

        ```bash
        sudo service postgresql-14 stop
        ```

        ```bash
        sudo service postgresql-14 status
        ```

        ![](./img/13.png)
      
    2. Armazenando por seguranca o diretorio `$PGDATA`

        ```bash
        mkdir bkp_database_before
        ```

        ```bash
        cp -r /var/lib/pgsql/14/data/* /home/hw/pg_backup/base/bkp_database_before/
        ```

        ![](./img/14.png)

    3. Removendo os arquivos da antiga base de dados do diretorio `$PGDATA`para descompactar e restaurar os dadaos da nova base

        ```bash
        cd /var/lib/pgsql/14/data/
        ```

        ```bash
        rm -rf ./*
        ```

        ![](./img/15.png)

        ```bash
        tar -xvf /home/hw/pg_backup/base/bkp_one.dmp/base.tar -C /var/lib/pgsql/14/data/
        ```

        ![](./img/16.png)

        Validado descompactacao

        ![](./img/17.png)

    4. Como temos a tablespace realizar a recuperacao da mesma forma 

        ```bash
        tar -xvf /home/hw/pg_backup/base/bkp_one.dmp/16402.tar -C /home/hw/tablespaces/pg_tbs_hw/
        ```

        ![](./img/18.png)

  7. Realizando agora a copia alteracao das permissoes dos arquivos de **WAL** para relaizar a sincronizacao com os dados restantes

      ![](./img/21.png)

      ![](./img/22.png)

  8. Neste momento o livro nos instruem a realizar a criação do arquivo `recovery.conf` no diretório `/var/lib/pgsql/14/data/` para instruir o banco na inicialização, porem foi identificado que apos a versão 12 o PostgreSQL não possui mais suporte a esse método, onde em uma das tentativas de iniciar o servico fora retornado o erro **FATAL: using recovery command file "recovery.conf" is not supported**.

      Foi necessario realizar uma mudanca no processo por conta de mundancas que ocorreram ate chegarmos a versao 14 do PostgreSQL

       1. Alterar os parametros `archive_mode`, `restore_command` e `recovery_target_timeline` no `postgresql.conf` 

          ```bash
          archive_mode = on
          ```

          ```bash
          restore_command = 'cp /home/hw/pg_backup/archive_wal/%f %p'
          ```

          ```bash
          recovery_target_timeline = 'immediate'
          ```

          ![](./img/23.png)
        
      1. Apenas com as alteracoes no `postgresql.conf` o servico nao iniciou retornando o seguinte log: 
          
          ```log
          2025-03-15 22:43:48.058 -03 [37097] FATAL:  could not locate required checkpoint record
          2025-03-15 22:43:48.058 -03 [37097] HINT:  If you are restoring from a backup, touch "/var/lib/pgsql/14/data/recovery.signal" and add required recovery options.
                  If you are not restoring from a backup, try removing the file "/var/lib/pgsql/14/data/backup_label".
                  Be careful: removing "/var/lib/pgsql/14/data/backup_label" will result in a corrupt cluster if restoring from a backup.
          ```



          ![](./img/25.png)
      
          onde foi necessario criar o arquivo de sinalizacao `recovery.signal` no diretorio `$PGDATA` com os seguintes parametros

          ```bash
          restore_command = 'cp /home/hw/pg_backup/archive_wal/%f %p'
          ```

          ```bash
          archive_mode = on
          ```
        
  9. Iniciado servico

      ![](./img/26.png)

  10. Realizado a conexao e consulta na base de dados `hardwork` e validado dados correspondentes ao banco de origem

      ![](./img/27.png)

<br/>

---

<br/>

[**<<==**](../capitulo_7/capitulo_7.md) |====| [**Home**](../../README.md) |====| [**==>>**](../capitulo_9/capitulo_9.md)

<br/>