# **Volumetria e disponibilidade**

Ao abordar monitoramento no PostgreSQL geralmente são referidos dois tipos que são eles **contínuo** e **pontual**.

## **Monitoramento pontual**

* **Uptime**<br/>
  Tempo em que a base de dados está disponível, o mesmo pode ser obtido através do log (local definido em log_directory no arquivo postgresql.conf) ou através da função **pg_postmaster_start_time()** que retorna, o momento, data e hora do start da base de dados:

  ```sql
  SELECT date_trunc('hour', pg_postmaster_start_time() as start_date, date_trunc('second', current_timestamp - pg_postmaster_start_time)) as uptime;
  ```

  ![Consulta pg_postmaster_start_time()](./img/consulta_pg_postmaster_start_time.png "Consulta pg_postmaster_start_time")

- **Variação do tamanho da base**<br/>
  O tamanho da base de dados pode ser analisado com a seguinte consulta

  ```sql
  SELECT pg_size_pretty(sum(pg_database_size(oid))::BIGINT) FROM pg_database;
  ```
  ![Consulta tamanho de base](./img/consulta_tamanho_de_base.png "Consulta tamanho de base")

- **Número total de conexões**<br/>
  Para verificar o quão próximo o número de conexões esta de atingir o limite

  ```sql
  SELECT count(*) as total_conn FROM pg_stat_activity;
  ```

  ![Consulta total de conexões](./img/consulta_total_de_conexoes.png "Consulta total de conexões")

- **Monitoramento de usuários/sessões do cluster**<br/>
  Com **pg_stat_activity**, podemos monitorar as conexões ao cluster

  ```sql
  SELECT datid, datname, pid, application_name FROM pg_stat_activity;
  ```

  É possível repetir o select com \watch x; sendo **x** o número de segundos entre os intervalos de consulta

  ![Consulta conexões ao cluster](./img/consulta_conexoes_ao_cluster.png "Consulta de conexões ao cluster")

## **Eliminação de seções no cluster**

Por diversos motivos (execução de comandos muito demorados, locks em outras sessões etc...), podemos ter a necessidade de eliminar uma sessão. Uma boa prática é cancelar o comando SQL antes de tal eliminação. Isso pode ser realizado com a função **pg_cancel_backend(pid);** caso não seja possível, utilizamos a função **pg_terminate_backend(pid)**, como no exemplo a seguir:

```sql
SELECT datid, datname, pid, application_name FROM pg_stat_activity;
```

![Consulta seções](./img/consulta_secoes.png "Consulta de seções no cluster")

Encerrando as conexões com pid 3448 e 3424 correspondente as operacoes que estavam sendo realizadas com pgAdmin4 na base hardwork

```sql
SELECT pg_terminate_backend(3424);
```

```sql
SELECT pg_terminate_backend(3448);
```

Após encerramento ao tentar dar continuidade no pgAdmin4 o mesmo apresentou aviso relacionado a perca de conexão

![Aviso conexão](./img/aviso_conexao.png "Aviso conexão")

<br/>

[<<==](../capitulo_6/capitulo_6.md) |====| [Home](../README.md) |====| [==>>](../capitulo_8/capitulo_8.md)