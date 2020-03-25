# Migrando dados

## Banco de Dados Compartilhado: uma boa ideia?

Mesmo depois de extrairmos os serviços de Pagamentos e Distância, mantivemos o mesmo MySQL monolítico.

![BD Compartilhado no Caelum Eats {w=73}](imagens/03-extraindo-servicos/distancia-service-extraido.png)

Há vantagens em manter um BD Compartilhado, como as mencionadas por Chris Richardson na página [Shared Database](https://microservices.io/patterns/data/shared-database.html) (RICHARDSON, 2018b):

- os desenvolvedores estão familiarizados com BD relacionais e soluções de ORM
- há o reforço da consistência dos dados com as garantias ACID (Atomicidade, Consistência, Isolamento e Durabilidade) do MySQL e de outros BDs relacionais
- é possível fazer consultas complexas de maneira eficiente, com joins de dados dos múltiplos serviços
- um único BD é mais fácil de operar e monitorar

Era comum em adoções de SOA que fosse mantido um BD Corporativo, que mantinha todos os dados da organização.

Porém, há diversas desvantagens, como as discutidas por Richardson na mesma página:

- dificuldade em escalar BDs, especialmente, relacionais
- necessidade de diferentes paradigmas de persistência para alguns serviços, como BDs orientados a grafos, como Neo4J, ou BDs bons em armazenar dados pouco estruturados, como MongoDB
- acoplamento no desenvolvimento, fazendo com que haja a necessidade de coordenação para a evolução dos schemas do BD monolítico, o que pode diminuir a velocidade dos times
- acoplamento no _runtime_, fazendo com que um lock em uma tabela ou uma consulta pesada feita por um serviço afete os demais

No livro [Building Microservices](https://learning.oreilly.com/library/view/building-microservices/9781491950340/) (NEWMAN, 2015), Sam Newman foca bastante no acoplamento gerado pelo Shared Database. Newman diz que essa Integração pelo BD é muito comum no mercado. A facilidade de obter e modificar dados de outro serviço diretamente pelo BD explica a popularidade. É como se o schema do BD fosse uma API. O acoplamento é feito por detalhes de implementação e uma mudança no schema do BD quebra os "clientes" da integração. Uma migração para outro paradigma de BD fica impossibilitada. A promessa de autonomia de uma Arquitetura de Microservices seria uma promessa não cumprida. Ficaria difícil evitar mudanças que quebram o contrato, o que inevitavelmente levaria a medo de qualquer mudança.

Sam Newman conta, no livro [Monolith to Microservices](https://learning.oreilly.com/library/view/monolith-to-microservices/9781492047834/) (NEWMAN, 2019), sobre uma experiência em um banco de investimento em que o time chegou a conclusão que uma reestruturação do schema do BD iriam aumentar drasticamente a performance do sistema. Então, descobriram que outras aplicações tinham acesso de leitura, e até de escrita, ao BD. Como o mesmo usuário e senha eram utilizados, era impossível saber quais eram essas aplicações e o que estava sendo acessado. Por uma análise de tráfego de rede, estimaram que cerca de 20 outras aplicações estavam usando integração pelo BD. Eventualmente, as credenciais foram desabilitadas e o time esperou o contato das pessoas que mantinham essas aplicações. Então, descobriram que a maioria das aplicações não tinha uma equipe para mantê-las. Ou seja, o schema antigo teria que ser mantido. O BD passou a ser uma API pública. O time de Newman resolveu o problema criando um schema privado e projetando os dados em Views públicas com informações limitadas, para que os outros sistemas acessassem.

Em [um tweet](https://twitter.com/rponte/status/1186337012724441089) (PONTE, 2019), Rafael Ponte, deixa claro que usar um Shared Database é integração de sistemas e que, nesse cenário, é preciso manter um contrato bem definido. Dessa maneira, a evolução dos schemas e a manutenção de longo prazo ficam facilitadas. Segundo Ponte, o problema é que o mercado utilizado o que ele chamada de _orgia de dados_, em que os sistemas acessando diretamente os dados, sem um contrato claro. E BDs permitem diversas maneiras de definir contratos:

- Grants a schemas
- API via procedures
- API via views
- API via tabelas de integração
- Eventos e signals

Rafael Ponte explica que tabelas de integração são especialmente úteis, pois são simples de implementar e provêem um contrato bem definido. Nenhum sistema conhece a estrutura de tabelas do outro, os detalhes de implementação do BD original. Há, portanto, _information hiding_ e encapsulamento. Um sistema produz dados para a tabela de integração e outro sistema consome esses dados, na cadência em que desejar.

Ponte afirma existem diversas estratégias de integração via BD. Procedures permitem uma maior flexibilidade na implementação, permitindo validação, enrichment, queuing, roteamento, etc. Views funcionam como uma API read-only, onde o sistema consumirdor não conhece nada da estrutura interna de tabelas.

## Um Banco de Dados por serviço

No artigo [Database per service](https://microservices.io/patterns/data/database-per-service.html) (RICHARDSON, 2018c), Chris Richardson argumenta em favor de um BD separado para cada serviço. O BD é um detalhe de implementação do serviço e não deve ser acessado diretamente por outros serviços.

> **Pattern: Database per service**
>
> Faça com que os dados de um Microservice sejam apenas acessíveis por sua API.

Podemos fazer um paralelo com o conceito de encapsulamento em Orientação a Objetos. Um objeto deve proteger seus detalhes internos e tornar os atributos privados é uma condição para isso. Qualquer manipulação dos atributos deve ser feita pelos métodos públicos.

No nível de serviços, os dados estarão em algum mecanismo de persistência, que devem ser privados. O acesso aos dados deve ser feito pela API do serviço, não diretamente.

A autonomia e o desacoplamento oferecidos por essa abordagem cumprem a promessa de uma Arquitetura de Microservices. As mudanças na implementação da persistência de um serviço não afetariam os demais. Cada serviço poderia usar o tipo de BD mais adequado às suas necessidades. Seria possível escalar um BD de um serviço independentemente, otimizando recursos computacionais.

Claro, não deixam de existir pontos negativos nessa abordagem. Temos que lidar com uma possível falta de consistência dos dados. Cenários de negócio transacionais passam a ser um desafio. Consultas que juntam dados de vários serviços são dificultadas. Há também a complexidade de operar e monitorar vários BDs distintos. Se forem usados múltiplos paradigmas de persistência, talvez seja difícil ter os especialistas necessários na organização.

### Um Servidor de Banco de Dados por serviço

Richardson cita algumas estratégias para tornar privados os dados persistidos de um serviço:

- Tabelas Privadas por Serviço: cada serviço tem um conjunto de tabelas que só deve ser acessada por esse serviço. Pode ser reforçado por um usuário para cada serviço e o uso de grants.
- Schema por Serviço: cada serviço tem seu próprio Schema no BD.
- Servidor de BD por Serviço: cada serviço tem seu próprio servidor de BD separado. Serviços com muitos acessos ou consultas pesadas trazem a necessidade de um servidor de BD separado.

Ter Tabelas Privadas ou um Schema Separado por serviço pode ser usado como um passo em direção à uma eventual migração para um servidor separado. 

Quando há a necessidade, para um serviço, de mecanismo de persistência com um paradigma diferente dos demais serviços, o servidor do BD deverá ser separado.

## Bancos de Dados separados no Caelum Eats

No Caelum Eats, vamos criar servidores de BD separados do MySQL do Monólito para os serviços de Pagamentos e de Distância.

O time de Pagamentos também usará um MySQL.

Já o time de Distância planeja explorar uma nova tecnologia de persistência, mais alinhada com as necessidades de geoprocessamento: o MongoDB. É um BD NoSQL, orientado a documentos, com um paradigma diferente do relacional.

![Servidores de BD separados para os serviços de Pagamentos e Distância {w=73}](imagens/04-migrando-dados/preparando-bds-separados-para-pagamentos-e-distancia.png)

Por enquanto, apenas criaremos os servidores de cada BD. Daria trabalho instalar e configurar os BDs manualmente. Então, para essas necessidades de infraestrutura, usaremos o Docker!

> O curso [Infraestrutura ágil com Docker e Docker Swarm](https://www.caelum.com.br/curso-infraestrutura-agil-com-docker-e-docker-swarm) (DO-26) da Caelum aprofunda nos conceitos do Docker e tecnologias relacionadas.

Vamos definir um MySQL 5.7 para o serviço de pagamentos com as seguintes configurações: `3308` como porta, `caelum123` como senha do `root` e `eats_pagamento` como um _database_ pré-configurado com o usuário `pagamento` e a senha `pagamento123`.

Para isso, adicione ao `docker-compose.yml`:

####### docker-compose.yml

```yml
mysql.pagamento:
  image: mysql:5.7
  ports:
    - "3308:3306"
  environment:
    MYSQL_ROOT_PASSWORD: caelum123
    MYSQL_DATABASE: eats_pagamento
    MYSQL_USER: pagamento
    MYSQL_PASSWORD: pagamento123
  volumes:
    - mysql.eats.pagamento:/var/lib/mysql
```

Adicione também o volume:

####### docker-compose.yml

```yml
volumes:
  mysql.eats.pagamento:
```

Além disso, vamos definir um MongoDB 3.6 para o serviço de distância , que será executado na porta `27018`, com o Docker Compose:

####### docker-compose.yml

```yml
mongo.distancia:
    image: mongo:3.6
    ports:
      - "27018:27017"
    volumes:
      - mongo.eats.distancia:/data/db
```

Não deixe de adicionar o volume:

```yml
volumes:
  mongo.eats.distancia:
```

## Exercício: Gerenciando containers de infraestrutura com Docker Compose

1. Para isso, baixe o arquivo `docker-compose.yml` completo, com o MySQL de pagamentos e o MongoDB de distância, para o seu Desktop, sobrescrevendo-o:

  ```sh
  cd ~/Desktop/
  curl https://gitlab.com/snippets/1859850/raw > docker-compose.yml
  ```

2. Ainda no Desktop, suba ambos os containers, do MySQL e do MongoDB, com o comando:

  ```sh
  cd ~/Desktop
  docker-compose up -d
  ```

  O MySQL do Monólito deve continuar no ar e os containers do MySQL de pagamentos e do MongoDB de distância devem ser criados. O resultado deve ser semelhante ao seguinte:

  ```txt
  Creating volume "eats-microservices_mysql.eats.pagamento" with default driver
  Creating volume "eats-microservices_mongo.eats.distancia" with default driver
  eats-microservices_mysql.monolito_1 is up-to-date
  Creating eats-microservices_mysql.pagamento_1 ... done
  Creating eats-microservices_mongo.distancia_1 ... done
  ```

  Observe quais os containers sendo executados com o comando:

  ```sh
  docker container ps
  ```

  Deverá ser impresso algo como:

  ```txt
  CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
  4890dcb9e898        mongo:3.6           "docker-entrypoint..."   26 minutes ago      Up 3 minutes        0.0.0.0:27018->27017/tcp            eats-microservices_mongo.distancia_1
  49bf0d3241ad        mysql:5.7           "docker-entrypoint..."   26 minutes ago      Up 3 minutes        33060/tcp, 0.0.0.0:3308->3306/tcp   eats-microservices_mysql.pagamento_1
  6b8c11246884        mysql:5.7           "docker-entrypoint..."   5 hours ago         Up 5 hours          33060/tcp, 0.0.0.0:3307->3306/tcp   eats-microservices_.mysql.monolito_1
  ```

3. Acesse o MySQL do serviço de pagamentos com o comando: 

  ```sh
  docker-compose exec mysql.pagamento mysql -upagamento -p
  ```

  Informe a senha `pagamento123`, registrada em passos anteriores.

  Informações sobre o MySQL, como _Server version: 5.7.29 MySQL Community Server (GPL)_ devem ser exibidas.

  Digite o comando:

  ```sql
  show databases;
  ```

  Deve ser exibido algo semelhante a:

  ```txt
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | eats_pagamento     |
  +--------------------+
  2 rows in set (0.00 sec)
  ```

  Para sair, digite `exit`.

4. Acesse o MongoDB do serviço de distância com o comando:

  ```sh
  docker-compose exec mongo.distancia mongo
  ```

  Devem aparecer informações sobre o MongoDB, como a versão, que deve ser algo como _MongoDB server version: 3.6.12_.

  Digite o seguinte comando:

  ```sh
  show dbs
  ```

  Deve ser impresso algo parecido com:

  ```txt
  admin   0.000GB
  config  0.000GB
  local   0.000GB
  ```

  Para sair, digite `exit`.

5. Observe os logs com o comando:

  ```sh
  docker-compose logs
  ```

## Separando Schemas

Agora temos um container com um MySQL específico para o serviço de Pagamentos. Vamos migrar os dados para esse servidor de BD. Mas o faremos de maneira progressiva e metódica.

No livro [Monolith to Microservices](https://learning.oreilly.com/library/view/monolith-to-microservices/9781492047834/) (NEWMAN, 2019), Sam Newman descreve, entre várias abordagens de migração, o uso de Views como um passo em direção a esconder informações entre serviços distintos que usam um Shared Database.

Um passo importante nessa progressão é usar, nos diferentes serviços, Schemas Separados dentro do Shared Database (ou, poderíamos dizer, um _database_ separado em um mesmo SGBD). Como comentado em capítulos anteriores, no livro [Building Microservices](https://learning.oreilly.com/library/view/building-microservices/9781491950340/) (NEWMAN, 2015), Sam Newman recomenda o uso de Schemas Separados mesmo mantendo o código no Monólito Modular.

> **Pattern: Schemas separados**
>
> Inicie a decomposição dos dados do Monólito usando Schemas Separados no mesmo servidor de BD, alinhados aos Bounded Contexts.

No livro [Monolith to Microservices](https://learning.oreilly.com/library/view/monolith-to-microservices/9781492047834/) (NEWMAN, 2019), Sam Newman argumenta que usar Schemas Separados seria uma _separação lógica_, enquanto usar um servidor de BD separado seria uma separação _física_. A separação lógica permite mudanças independentes e encapsulamento, enquanto que a separação física potencialmente melhor vazão, latência, uso de recursos e isolamento de falhas. Contudo, a separação lógica é uma condição para a separação física.

Mesmo com Schemas Separados, se for utilizado um mesmo servidor de BD, podemos ter usuários que tem acesso a mais de um Schema e, portanto, que conseguem fazer migração de dados.

Uma vez que decidimos por Schemas Separados, a integridade oferecida por _foreign keys_ (FKs) nos BDs relacionais tem que ser deixada de lado. Essa perda traz duas consequências:

- consultas que fazem join dos dados tem que ser feitas em memória, tornando a operação mais lenta
- perda de consistência dos dados, cujos efeitos discutiremos mais adiante

Tanto Sam Newman como Chris Richardson indicam como referência para a evolução de BDs relacionais o livro [Refactoring Databases](https://learning.oreilly.com/library/view/refactoring-databases-evolutionary/0321293533/) (SADALAGE; AMBER, 2006) de Pramod Sadalage e Scott Ambler.

## Separando schema do BD de pagamentos do monólito

Em capítulos anteriores, quebramos o Modelo de Domínio de `Pagamento` para que não dependesse de `FormaDePagamento` nem de `Pedido`, que são dos módulos Administrativo e de Pedido do Monólito, respectivamente. Porém, as FKs foram mantidas.

Nessa momento, criaremos um Schema Separado para o serviço de Pagamentos. Não existiram FKs às tabelas que representam `FormaDePagamento` e `Pedido`. Serão mantidos apenas os ids dessas tabelas.

![Schema separado para o BD de pagamentos {w=35}](imagens/04-migrando-dados/separando-schema-do-bd-de-pagamentos.png)

Mas como efetuar essa alteração? 

Criaremos scripts `.sql` com instruções DDL (Data Definition Language), como `CREATE TABLE` ou `ALTER TABLE`, para criar as estruturas das tabelas e, instruções DML (Data Manipulation Languagem), como `INSERT` ou `UPDATE`, para popular os dados.

Para executar esses scripts, usaremos uma ferramenta de Migration.

Entre as bibliotecas mais usadas para Migration em projetos Java estão Liquibase e Flyway. Ambas estão bem integradas com o Spring Boot. O Liquibase permite que as Migrations sejam definidas em XML, JSON, YAML e SQL. Já no Flyway, podem ser usados SQL e Java. Uma grande vantagem do Liquibase é a possibilidade de ter Migrations de rollback na versão _community_. 

Uma ferramenta de migração de dados tem uma maneira de definir a versão dos scripts e de controlar quais scripts já foram executados. Assim, é possível recriar um BD do zero, saber qual é a versão atual de um BD específico e evoluir para novas versões.

O Monólito já usa o Flyway para DDL e DML.

No caso do Flyway, há uma nomenclatura padrão para o nome dos arquivos `.sql`: 

- Um `V` como prefixo.
- Um número de versão incremental e único, como `0001` ou `0919`. Pode haver pontos para versões intermediárias, como `2.5`
- Dois underscores (`__`) como separador
- Uma descrição
- A extensão `.sql` como sufixo.

Um exemplo de nome de arquivo seria `V0001__cria-tabela-pagamento.sql`.

Para manter a versão atual do BD e saber quais scripts foram executados, o Flyway mantém uma tabela chamada `flyway_schema_history`. No livro [Refactoring Databases](https://learning.oreilly.com/library/view/refactoring-databases-evolutionary/0321293533/) (SADALAGE; AMBER, 2006), os autores já demonstram a necessidade de manter qual a última versão do Schema em uma tabela que chamam de _Database Configuration_.

Um novo Schema não teria essa tabela e, portanto, estaria vazia, significando que todos os scripts devem ser executados. Essa tabela tem colunas como: 

- `version`, que contém as versões executadas;
- `script`, que contém o nome do arquivo executado;
- `installed_on`, que contém a data/hora da execução;
- `checksum`, que contém um número calculado a partir do arquivo `.sql`;
- `success`, que indica se a execução foi bem sucedida.

_Observação: o `checksum` é checado para todos os scripts ao iniciar a aplicação. Não mude ou remova scripts porque a aplicação pode deixar de subir. Cuidado!_ 

Usaremos o Flyway também para o serviço de Pagamentos.

Para isso, deve ser adicionada uma dependência ao Flyway no `pom.xml` do `eats-pagamento-service`:

####### fj33-eats-pagamento-service/pom.xml

```xml
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-core</artifactId>
</dependency>
```

O database deve ser modificado para um novo database específico para o serviço de pagamentos. Podemos chamá-lo de `eats_pagamento`.

####### fj33-eats-pagamento-service/src/main/resources/application.properties

```properties
s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶u̶r̶l̶=̶j̶d̶b̶c̶:̶m̶y̶s̶q̶l̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶3̶3̶0̶7̶/̶e̶a̶t̶s̶?̶c̶r̶e̶a̶t̶e̶D̶a̶t̶a̶b̶a̶s̶e̶I̶f̶N̶o̶t̶E̶x̶i̶s̶t̶=̶t̶r̶u̶e̶
spring.datasource.url=jdbc:mysql://localhost:3307/eats_pagamento?createDatabaseIfNotExist=true
```

Estamos usando, no serviço de pagamentos, o usuário `root` do BD do Monólito. Esse usuário tem acesso a ambos os databases: `eats`, do monólito, e `eats_pagamento`, do serviço de pagamentos. Dessa maneira, é possível executar scripts que migram dados de um database para outro.

Numa nova pasta `db/migration` em  `src/main/resources` deve ser criada uma primeira migration, que cria a tabela de `pagamento`. O arquivo pode ter o nome `V0001__cria-tabela-pagamento.sql` e o seguinte conteúdo:

####### fj33-eats-pagamento-service/src/main/resources/db/migration/V0001__cria-tabela-pagamento.sql

```sql
CREATE TABLE pagamento (
  id bigint(20) NOT NULL AUTO_INCREMENT,
  valor decimal(19,2) NOT NULL,
  nome varchar(100) DEFAULT NULL,
  numero varchar(19) DEFAULT NULL,
  expiracao varchar(7) NOT NULL,
  codigo varchar(3) DEFAULT NULL,
  status varchar(255) NOT NULL,
  forma_de_pagamento_id bigint(20) NOT NULL,
  pedido_id bigint(20) NOT NULL,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

O conteúdo acima pode ser encontrado na seguinte URL: https://gitlab.com/snippets/1859564

Uma segunda migration, de nome `V0002__migra-dados-de-pagamento.sql`, obtem os dados do database `eats`, do monólito, e os insere no database `eats_pagamento`. Crie o arquivo  em `db/migration`, conforme a seguir:

####### fj33-eats-pagamento-service/src/main/resources/db/migration/V0002__migra-dados-de-pagamento.sql

```sql
insert into eats_pagamento.pagamento
  (id, valor, nome, numero, expiracao, codigo, status, forma_de_pagamento_id, pedido_id)
    select id, valor, nome, numero, expiracao, codigo, status, forma_de_pagamento_id, pedido_id
      from eats.pagamento;
```

O trecho de código acima pode ser encontrado em: https://gitlab.com/snippets/1859568

Essa migração só é possível porque o usuário tem acesso aos dois databases.

Após executar `EatsPagamentoServiceApplication`, nos logs, devem aparecer informações sobre a execução dos scripts `.sql`. Algo como:

```txt
2019-05-22 18:33:56.439  INFO 30484 --- [  restartedMain] o.f.c.internal.license.VersionPrinter    : Flyway Community Edition 5.2.4 by Boxfuse
2019-05-22 18:33:56.448  INFO 30484 --- [  restartedMain] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2019-05-22 18:33:56.632  INFO 30484 --- [  restartedMain] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2019-05-22 18:33:56.635  INFO 30484 --- [  restartedMain] o.f.c.internal.database.DatabaseFactory  : Database: jdbc:mysql://localhost:3307/eats_pagamento (MySQL 5.7)
2019-05-22 18:33:56.708  INFO 30484 --- [  restartedMain] o.f.core.internal.command.DbValidate     : Successfully validated 2 migrations (execution time 00:00.016s)
2019-05-22 18:33:56.840  INFO 30484 --- [  restartedMain] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table: `eats_pagamento`.`flyway_schema_history`
2019-05-22 18:33:57.346  INFO 30484 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Current version of schema `eats_pagamento`: << Empty Schema >>
2019-05-22 18:33:57.349  INFO 30484 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Migrating schema `eats_pagamento` to version 0001 - cria-tabela-pagamento
2019-05-22 18:33:57.596  INFO 30484 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Migrating schema `eats_pagamento` to version 0002 - migra-dados-de-pagamento
2019-05-22 18:33:57.650  INFO 30484 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Successfully applied 2 migrations to schema `eats_pagamento` (execution time 00:00.810s)
```

Para verificar se o conteúdo do database `eats_pagamento` condiz com o esperado, podemos acessar o MySQL em um Terminal:

```sh
mysql -u eats -p eats_pagamento
```

Dentro do MySQL, deve ser executada a seguinte query:

```sql
select * from pagamento;
```

Os pagamentos devem ter sido migrados.

Novos pagamentos serão armazenados apenas no schema `eats_pagamento`. Os dados do serviço de Pagamentos são suficientemente independentes para serem mantidos em um BD separado.

É importante lembrar que a mudança do status do pedido para _PAGO_, que perdemos ao extrair o serviço de Pagamentos do Monólito, ainda precisa ser resolvida. Faremos isso mais adiante.

## Exercício: migrando dados de pagamento para schema separado

1. Pare a execução de `EatsPagamentoServiceApplication`.

  Obtenha as configurações e scripts de migração para outro schema da branch `cap4-migrando-pagamentos-para-schema-separado` do serviço de pagamentos:

  ```sh
  cd ~/Desktop/fj33-eats-pagamento-service
  git checkout -f cap4-migrando-pagamentos-para-schema-separado
  ```

  Execute `EatsPagamentoServiceApplication`. Observe o resultado da execução das migrations nos logs.

2. No diretório do Docker Compose, o Desktop, acesse o MySQL do Monólito:

  ```sh
  cd ~/Desktop
  docker-compose exec mysql.monolito mysql -uroot -p eats_pagamento
  ```

  Quando solicitada, digite a senha `caelum123`.

  _Observação: no comando anterior, já informamos o database a ser acessado._

  Verifique se o conteúdo do database `eats_pagamento` condiz com o esperado com a query:

  ```sql
  select * from pagamento;
  ```

  Os pagamentos devem ter sido migrados. Note as colunas `forma_de_pagamento_id` e `pedido_id`.

  Se desejar, verifique dados sobre a  execução dos scripts SQL na tabela `flyway_schema_history`.

## Migrando dados de um servidor MySQL para outro

Nesse momento, o MySQL monolítico tem Schemas Separados para o Monólito e para o serviço de Pagamentos. Também temos um MySQL específico para Pagamentos já com um database `eats_pagamento`, mas ainda vazio.

Precisamos migrar as estruturas das tabelas e os dados. Uma maneira de fazer isso é gerar um dump com o comando `mysqldump` a partir do database `eats_pagamento` do MySQL do Monólito.

Como o MySQL monolítico está sendo executado pelo Docker Compose, devemos fazer algo como:

```sh
docker-compose exec -T mysql.monolito mysqldump -uroot -pcaelum123 --opt eats_pagamento
```

A opção `--opt` do `mysqldump` equivale às opções:

- `--add-drop-table`, que adiciona um `DROP TABLE` antes de cada `CREATE TABLE`
- `--add-locks`, que faz um `LOCK TABLES` e `UNLOCK TABLES` em volta de cada dump
- `--create-options`, que inclui opções específicas do MySQL nos `CREATE TABLE`
- `--disable-keys`, que desabilita e habilita PKs e FKs em volta de cada `INSERT`
- `--extended-insert`, que faz um `INSERT` de múltiplos registros de uma vez
- `--lock-tables`, que trava as tabelas antes de realizar o dump
- `--quick`, que lê os registros um a um
- `--set-charset`, que adiciona `default_character_set` ao dump

Será gerado um script com comandos de DDL e DML do Schema.

Já a opção `-T` do `docker-compose exec` desabilita o modo interativo. Dessa maneira, a saída desse comando será usada para alimentar o próximo comando.

O script com o dump pode ser carregado no outro MySQL, específico de Pagamentos, com o comando `mysql`.

Para usar a saída do `mysqldump` do Monólito como entrada do `mysql` de Pagamentos, podemos fazer:

```sh
docker-compose exec -T mysql.monolito mysqldump -uroot -pcaelum123 --opt eats_pagamento |
docker-compose exec -T mysql.pagamento mysql -upagamento -ppagamento123 eats_pagamento
```

![Dump do schema de Pagamentos importado para servidor de BD específico {w=65}](imagens/04-migrando-dados/dump-do-mysql-do-monolito-para-o-de-pagamentos.png)

## Exercício: migrando dados de pagamento para um servidor MySQL específico

1. Garanta que os containers de BDs estejam sendo executados:

  ```sh
  cd ~/Desktop
  docker-compose up -d 
  ```

2. Abra um Terminal e baixe o script de migração dos dados de pagamento para o seu Desktop. Dê as permissões necessárias, executando-o logo em seguida:

  ```sh
  cd ~/Desktop
  curl https://gitlab.com/snippets/1955294/raw > migra-dados-de-pagamentos.sh
  chmod +x migra-dados-de-pagamentos.sh
  ./migra-dados-de-pagamentos.sh
  ```

  _Observação: como colocamos as senhas do BD diretamente nos comandos, serão exibidos warnings de segurança._

3. Ainda no Desktop, acesse o MySQL de Pagamentos pelo Docker Compose:

  ```sh
  cd ~/Desktop
  docker-compose exec mysql.pagamento mysql -upagamento -p eats_pagamento
  ```

  Informe a senha `pagamento123`, quando requisitada.

  Digite o seguinte comando SQL e verifique o resultado:

  ```sql
  select * from pagamento;
  ```

  Devem ser exibidos todos os pagamentos já efetuados!

  Para sair, digite `exit`.

## Apontando serviço de pagamentos para o BD específico

O serviço de pagamentos deve deixar de usar o MySQL do monólito e passar a usar a sua própria instância do MySQL, que contém seu próprio schema e apenas os dados necessários.

Para isso, basta alterarmos a URL, usuário e senha de BD do serviço de pagamentos, para que apontem para o `mysql.pagamento` do Docker Compose:

####### fj33-eats-pagamento-service/src/main/resources/application.properties

```properties
s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶u̶r̶l̶=̶j̶d̶b̶c̶:̶m̶y̶s̶q̶l̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶3̶3̶0̶7̶/̶e̶a̶t̶s̶_̶p̶a̶g̶a̶m̶e̶n̶t̶o̶?̶c̶r̶e̶a̶t̶e̶D̶a̶t̶a̶b̶a̶s̶e̶I̶f̶N̶o̶t̶E̶x̶i̶s̶t̶=̶t̶r̶u̶e̶
spring.datasource.url=jdbc:mysql://localhost:3308/eats_pagamento?createDatabaseIfNotExist=true

s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶u̶s̶e̶r̶n̶a̶m̶e̶=̶e̶a̶t̶s̶
spring.datasource.username=pagamento

s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶p̶a̶s̶s̶w̶o̶r̶d̶=̶e̶a̶t̶s̶1̶2̶3̶
spring.datasource.password=pagamento123
```

Note que a porta `3308` foi incluída na URL, mas mantivemos ainda `localhost`.

## Exercício: fazendo serviço de pagamentos apontar para o BD específico

1. Obtenha as alterações no datasource do serviço de pagamentos da branch `cap4-apontando-pagamentos-para-BD-proprio`:

  ```sh
  cd ~/Desktop/fj33-eats-pagamento-service
  git checkout -f cap4-apontando-pagamentos-para-BD-proprio
  ```

  Reinicie o serviço de pagamentos, executando a classe `EatsPagamentoServiceApplication`.

2. Abra um Terminal e crie um novo pagamento:

  ```sh
  curl -X POST
    -i
    -H 'Content-Type: application/json'
    -d '{ "valor": 9.99, "nome": "MARIA DE SOUSA", "numero": "777 2222 8888 4444", "expiracao": "2025-04", "codigo": "777", "formaDePagamentoId": 1, "pedidoId": 2 }'
    http://localhost:8081/pagamentos
  ```

  Se desejar, baseie-se na seguinte URL, modificando os valores: https://gitlab.com/snippets/1859389

  A resposta deve ter sucesso, com status `200` e o um id e status `CRIADO` no corpo da resposta.

3. Pelo Eclipse, inicie o monólito e o serviço de distância. Suba também o front-end. Faça um novo pedido, até efetuar o pagamento. Deve funcionar!

4. (opcional) Apague a tabela `pagamento` do database `eats`, do monólito. Remova também o database `eats_pagamento` do MySQL do monólito. Atenção: muito cuidado para não remover dados indesejados!

## Migrando dados do MySQL para MongoDB

Provisionamos, pelo Docker Compose, um MongoDB específico para o serviço de Distância. Por enquanto, não há dados nesse BD.

O MongoDB não é um BD relacional, mas de um paradigma orientado a documentos.

Não existem tabelas no MongoDB, mas _collections_. As collections armazenam _documents_. Um document é _schemaless_, pois não tem colunas e tipos  definidos. Um document tem um _id_ como identificador, que deve ser único.

No MongoDB, um _database_ agrupa várias collections, de maneira semelhante ao MySQL.

Há um conflito entre os conceitos de um BD relacional como o MySQL e de um BD orientado a documentos, como o MongoDB. Por isso, as estratégias de migração devem ser diferentes.

Devemos exportar um subconjunto dos dados de um `Restaurante`, que são relevantes para o serviço de Distância: o `id`, o `cep`, o `tipoDeCozinhaId` e o atributo `aprovado`, que indica se o restaurante já foi revisado e aprovado pelo Administrativo do Caelum Eats.

Para isso, devemos executar o seguinte SQL:

```sql
select r.id, r.cep, r.tipo_de_cozinha_id from restaurante r where r.aprovado = true;
```

Não é possível fazer um dump para um script `.sql`. Porém, como a nossa migração é simples, podemos usar um arquivo CSV com os dados de restaurantes que são relevantes para o serviço de Distância.

Podemos executar o `select` no BD monolítico pelo Terminal com o comando:

```sh
docker-compose exec -T mysql.monolito mysql -uroot -pcaelum123 eats -N -B -e "select r.id, r.cep, r.tipo_de_cozinha_id from restaurante r where r.aprovado = true;"
```

A opção `-N`, ou `--skip-column-names`, remove do resultado a primeira linha de cabeçalho contendo o nome das colunas.

A opção `-B`, ou `--batch`, remove a formatação dos resultados, separando cada coluna por um TAB, com cada registro em uma nova linha.

A opção `-e`, ou `--execute`, permite que passemos um comando SQL.

O resultado será algo como:

```txt
1	70238500	1
2	71458-074	6
```
Podemos passar o resultado anterior por um `sed`, trocando os TABs por vírgulas:

```sh
docker-compose exec -T mysql.monolito mysql -uroot -pcaelum123 eats -N -B -e "select r.id, r.cep, r.tipo_de_cozinha_id from restaurante r where r.aprovado = true;" |
sed 's/\t/,/g'
```

Será impresso algo semelhante ao seguinte:

```csv
1,70238500,1
2,71458-074,6
```

A saída do `sed`, mostrada anteriormente, pode ser usada como entrada para o comando `mongoimport` do MongoDB de distância.

Algumas opções do comando:

- `--db`, o database de destino
- `--collection`, a collection de destino
- `--type`, o tipo do arquivo (no caso, um CSV)
- `--fields`, para definir os nomes das propriedades do document

Como  não há os nomes das propriedades na saída do `sed`, devemos definí-las usando a opção `--fields`. O campo de identificação do document deve se chamar `_id`. 

Para importar o conteúdo do CSV para a collection `restaurantes` do database `eats_distancia`, com os campos `_id`, `cep` e `tipoDeCozinhaId`, devemos executar o seguinte comando:

```sh
mongoimport --db eats_distancia --collection restaurantes --type csv  --fields=_id,cep,tipoDeCozinhaId
```

Juntando tudo, teremos o seguinte comando:

```sh
docker-compose exec -T mysql.monolito mysql -uroot -pcaelum123 eats -N -B -e "select r.id, r.cep, r.tipo_de_cozinha_id from restaurante r where r.aprovado = true;" |
sed 's/\t/,/g' |
docker-compose exec -T mongo.distancia mongoimport --db eats_distancia --collection restaurantes --type csv  --fields=_id,cep,tipoDeCozinhaId
```

![Dump para CSV para MongoDB de Distância {w=65}](imagens/04-migrando-dados/dump-dos-dados-de-restaurante-do-monolito-para-o-mongodb-de-distancia.png)


## Exercício: migrando dados de restaurantes do MySQL para o MongoDB

1. Garanta que todos os containers de BDs estejam sendo executados:

  ```sh
  cd ~/Desktop
  docker-compose up -d 
  ```

2. Abra um Terminal e baixe o script de migração dos dados de distância para o seu Desktop. Dê as permissões necessárias, executando-o logo em seguida:

  ```sh
  cd ~/Desktop
  curl https://gitlab.com/snippets/1955323/raw > migra-dados-de-distancia.sh
  chmod +x migra-dados-de-distancia.sh
  ./migra-dados-de-distancia.sh
  ```

  _Observação: como colocamos a senha do MySQL diretamente no comando, será exibidos um warning de segurança._

3. Ainda no Desktop, acesse o database de Distância do MongoDB pelo Docker Compose:

  ```sh
  cd ~/Desktop
  docker-compose exec mongo.distancia mongo eats_distancia
  ```

  _Observação: no comando anterior, já informamos o database a ser acessado._

  Dentro do Mongo Shell, verifique a collection de restaurantes foi criada:

  ```sh
  show collections;
  ```

  Deve ser retornado algo como:

  ```txt
  restaurantes
  ```

  Veja os documentos da collection `restaurantes` com o comando: 

  ```js
  db.restaurantes.find();
  ```

  O resultado será semelhante a:

  ```json
  { "_id" : 1, "cep" : 70238500, "tipoDeCozinhaId" : 1 }
  { "_id" : 2, "cep" : "71503-511", "tipoDeCozinhaId" : 7 }
  { "_id" : 3, "cep" : "70238-500", "tipoDeCozinhaId" : 9 }
  ```

  Pronto, os dados foram migrados para o MongoDB!

  _Observação: Apenas os restaurantes já aprovados terão seus dados migrados. Restaurantes ainda não aprovados ou novos restaurantes não aparecerão para o serviço de distância._

## Configurando MongoDB no serviço de distância

O _starter_ do Spring Data MongoDB deve ser adicionado ao `pom.xml` do `eats-distancia-service`.

Já as dependências ao Spring Data JPA e ao driver do MySQL devem ser removidas.

####### fj33-eats-distancia-service/pom.xml

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>

<̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
  <̶g̶r̶o̶u̶p̶I̶d̶>̶m̶y̶s̶q̶l̶<̶/̶g̶r̶o̶u̶p̶I̶d̶>̶
  <̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶m̶y̶s̶q̶l̶-̶c̶o̶n̶n̶e̶c̶t̶o̶r̶-̶j̶a̶v̶a̶<̶/̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶
  <̶s̶c̶o̶p̶e̶>̶r̶u̶n̶t̶i̶m̶e̶<̶/̶s̶c̶o̶p̶e̶>̶
<̶/̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶

<̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
  <̶g̶r̶o̶u̶p̶I̶d̶>̶o̶r̶g̶.̶s̶p̶r̶i̶n̶g̶f̶r̶a̶m̶e̶w̶o̶r̶k̶.̶b̶o̶o̶t̶<̶/̶g̶r̶o̶u̶p̶I̶d̶>̶
  <̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶s̶p̶r̶i̶n̶g̶-̶b̶o̶o̶t̶-̶s̶t̶a̶r̶t̶e̶r̶-̶d̶a̶t̶a̶-̶j̶p̶a̶<̶/̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶
<̶/̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
```

Devem ocorrer vários erros de compilação.

A classe `Restaurante` do serviço de distância deve ser modificada, removendo as anotações do JPA.

A anotação `@Document`, do Spring Data MongoDB, deve ser adicionada.

A anotação `@Id` deve ser mantida, porém o import será trocado para `org.springframework.data.annotation.Id`, uma anotação genérica do Spring Data.

> Perceba que, apesar do campo ser `_id` no document, o manteremos como `id` no código Java. A anotação `@Id` cuidará de informar qual dos atributos é o identificador do documento e está relacionado ao campo `_id`.

O atributo `aprovado` pode ser removido, já que a migração dos dados foi feita de maneira que o database de distância do MongoDB só contém restaurantes já aprovados.

####### fj33-eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/Restaurante.java

```java
@Document(collection = "restaurantes") // adicionado
@̶E̶n̶t̶i̶t̶y̶
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Restaurante {

  @Id
  @̶G̶e̶n̶e̶r̶a̶t̶e̶d̶V̶a̶l̶u̶e̶(̶s̶t̶r̶a̶t̶e̶g̶y̶ ̶=̶ ̶G̶e̶n̶e̶r̶a̶t̶i̶o̶n̶T̶y̶p̶e̶.̶I̶D̶E̶N̶T̶I̶T̶Y̶)̶
  private Long id;

  private String cep;

  p̶r̶i̶v̶a̶t̶e̶ ̶B̶o̶o̶l̶e̶a̶n̶ ̶a̶p̶r̶o̶v̶a̶d̶o̶;̶

  @̶C̶o̶l̶u̶m̶n̶(̶n̶u̶l̶l̶a̶b̶l̶e̶ ̶=̶ ̶f̶a̶l̶s̶e̶)̶
  private Long tipoDeCozinhaId;

}
```

Os seguinte imports devem ser removidos:

```java
i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶C̶o̶l̶u̶m̶n̶;̶
i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶E̶n̶t̶i̶t̶y̶;̶
i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶G̶e̶n̶e̶r̶a̶t̶e̶d̶V̶a̶l̶u̶e̶;̶
i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶G̶e̶n̶e̶r̶a̶t̶i̶o̶n̶T̶y̶p̶e̶;̶
i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶I̶d̶;̶
```

E os imports a seguir devem ser adicionados:

Os imports corretos são os seguintes:

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
```

Note que o `@Id` foi importado de `org.springframework.data.annotation` e **não** de `javax.persistence` (do JPA).

A interface `RestauranteRepository` deve ser modificada, para que passe a herdar de um `MongoRepository`.

Como removemos o atributo `aprovado`, as definições de métodos devem ser ajustadas.

####### fj33-eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/RestauranteRepository.java

```java
i̶n̶t̶e̶r̶f̶a̶c̶e̶ ̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶R̶e̶p̶o̶s̶i̶t̶o̶r̶y̶ ̶e̶x̶t̶e̶n̶d̶s̶ ̶J̶p̶a̶R̶e̶p̶o̶s̶i̶t̶o̶r̶y̶<̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶,̶ ̶L̶o̶n̶g̶>̶ ̶{̶
interface RestauranteRepository extends MongoRepository<Restaurante, Long> {

  P̶a̶g̶e̶<̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶>̶ ̶f̶i̶n̶d̶A̶l̶l̶B̶y̶A̶p̶r̶o̶v̶a̶d̶o̶A̶n̶d̶T̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶I̶d̶(̶b̶o̶o̶l̶e̶a̶n̶ ̶a̶p̶r̶o̶v̶a̶d̶o̶,̶ ̶L̶o̶n̶g̶ ̶t̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶I̶d̶,̶ ̶P̶a̶g̶e̶a̶b̶l̶e̶ ̶l̶i̶m̶i̶t̶)̶;̶
  Page<Restaurante> findAllByTipoDeCozinhaId(Long tipoDeCozinhaId, Pageable limit);

  P̶a̶g̶e̶<̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶>̶ ̶f̶i̶n̶d̶A̶l̶l̶B̶y̶A̶p̶r̶o̶v̶a̶d̶o̶(̶b̶o̶o̶l̶e̶a̶n̶ ̶a̶p̶r̶o̶v̶a̶d̶o̶,̶ ̶P̶a̶g̶e̶a̶b̶l̶e̶ ̶l̶i̶m̶i̶t̶)̶;̶
  Page<Restaurante> findAll(Pageable limit);

}
```

Os imports devem ser corrigidos:

```java
i̶m̶p̶o̶r̶t̶ ̶o̶r̶g̶.̶s̶p̶r̶i̶n̶g̶f̶r̶a̶m̶e̶w̶o̶r̶k̶.̶d̶a̶t̶a̶.̶j̶p̶a̶.̶r̶e̶p̶o̶s̶i̶t̶o̶r̶y̶.̶J̶p̶a̶R̶e̶p̶o̶s̶i̶t̶o̶r̶y̶;̶
import org.springframework.data.mongodb.repository.MongoRepository;
```

Como removemos o atributo `aprovado`, é necessário alterar a chamada ao `RestauranteRepository` em alguns métodos do `DistanciaService`:

####### fj33-eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/DistanciaService.java

```java
// anotações....
class DistanciaService {

  // atributos...

  public List<RestauranteComDistanciaDto> restaurantesMaisProximosAoCep(String cep) {
    L̶i̶s̶t̶<̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶>̶ ̶a̶p̶r̶o̶v̶a̶d̶o̶s̶ ̶=̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶.̶f̶i̶n̶d̶A̶l̶l̶B̶y̶A̶p̶r̶o̶v̶a̶d̶o̶(̶t̶r̶u̶e̶,̶ ̶L̶I̶M̶I̶T̶)̶.̶g̶e̶t̶C̶o̶n̶t̶e̶n̶t̶(̶)̶;̶
    List<Restaurante> aprovados = restaurantes.findAll(LIMIT).getContent(); // modificado
    return calculaDistanciaParaOsRestaurantes(aprovados, cep);
  }

  public List<RestauranteComDistanciaDto> restaurantesDoTipoDeCozinhaMaisProximosAoCep(Long tipoDeCozinhaId, String cep) {
    L̶i̶s̶t̶<̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶>̶ ̶a̶p̶r̶o̶v̶a̶d̶o̶s̶D̶o̶T̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶ ̶=̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶.̶f̶i̶n̶d̶A̶l̶l̶B̶y̶A̶p̶r̶o̶v̶a̶d̶o̶A̶n̶d̶T̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶I̶d̶(̶t̶r̶u̶e̶,̶ ̶t̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶I̶d̶,̶ ̶L̶I̶M̶I̶T̶)̶.̶g̶e̶t̶C̶o̶n̶t̶e̶n̶t̶(̶)̶;̶
    List<Restaurante> aprovadosDoTipoDeCozinha = restaurantes.findAllByTipoDeCozinhaId(tipoDeCozinhaId, LIMIT).getContent();
    return calculaDistanciaParaOsRestaurantes(aprovadosDoTipoDeCozinha, cep);
  }

  // restante do código...

}
```

No arquivo `application.properties` do `eats-distancia-service`, devem ser adicionadas as configurações do MongoDB. As configurações de datasource do MySQL e do JPA devem ser removidas.

####### fj33-eats-distancia-service/src/main/resources/application.properties

```properties
spring.data.mongodb.database=eats_distancia
spring.data.mongodb.port=27018

#̶D̶A̶T̶A̶S̶O̶U̶R̶C̶E̶ ̶C̶O̶N̶F̶I̶G̶S̶
s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶u̶r̶l̶=̶j̶d̶b̶c̶:̶m̶y̶s̶q̶l̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶/̶e̶a̶t̶s̶?̶c̶r̶e̶a̶t̶e̶D̶a̶t̶a̶b̶a̶s̶e̶I̶f̶N̶o̶t̶E̶x̶i̶s̶t̶=̶t̶r̶u̶e̶
s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶u̶s̶e̶r̶n̶a̶m̶e̶=̶<̶S̶E̶U̶ ̶U̶S̶U̶Á̶R̶I̶O̶>̶
s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶p̶a̶s̶s̶w̶o̶r̶d̶=̶<̶S̶U̶A̶ ̶S̶E̶N̶H̶A̶>̶

#̶J̶P̶A̶ ̶C̶O̶N̶F̶I̶G̶S̶
s̶p̶r̶i̶n̶g̶.̶j̶p̶a̶.̶h̶i̶b̶e̶r̶n̶a̶t̶e̶.̶d̶d̶l̶-̶a̶u̶t̶o̶=̶v̶a̶l̶i̶d̶a̶t̶e̶
s̶p̶r̶i̶n̶g̶.̶j̶p̶a̶.̶s̶h̶o̶w̶-̶s̶q̶l̶=̶t̶r̶u̶e̶
```

O database padrão do MongoDB é `test`. A porta padrão é `27017`.

Para saber sobre outras propriedades, consulte: https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html

## Exercício: Migrando dados de distância para o MongoDB

1. Interrompa o serviço de distância.

  Obtenha o código da branch `cap4-migrando-distancia-para-mongodb` do `fj33-eats-distancia-service`:

  ```sh
  cd ~/Desktop/fj33-eats-distancia-service
  git checkout -f cap4-migrando-distancia-para-mongodb
  ```

  Certifique-se que o MongoDB do serviço de distância esteja no ar com o comando:

  ```sh
  cd ~/Desktop
  docker-compose up -d mongo.distancia
  ```

  Execute novamente a classe `EatsDistanciaServiceApplication`.

  Use o cURL, ou o navegador, para testar algumas URLs do serviço de distância, como as seguir:

  ```sh
  curl -i http://localhost:8082/restaurantes/mais-proximos/71503510

  curl -i http://localhost:8082/restaurantes/mais-proximos/71503510/tipos-de-cozinha/1

  curl -i http://localhost:8082/restaurantes/71503510/restaurante/1
  ```

## Para saber mais: Change Data Capture

Mudar dados de um servidor de BD para outro sempre foi um desafio, mesmo para projetos com BDs monolíticos.

Nesse capítulo, fizemos um dump com um script SQL para que fosse importado no MySQL de Pagamentos e um dump com um arquivo CSV para que fosse importado no MongoDB de Distância.

Esse processo tem um uso limitado em uma aplicação real. O sistema que usa o BD original teria que ficar fora do ar durante o dump e import no BD de destino. Se o sistema for mantido no ar, os dados continuariam a ser alterados, inseridos e deletados.

Um dump, porém, é um passo útil para alimentar o novo BD com um snapshot dos dados em um momento inicial. Sam Newman descreve esse passo, no livro [Monolith to Microservices](https://learning.oreilly.com/library/view/monolith-to-microservices/9781492047834/) (NEWMAN, 2019), como _Bulk Synchronize Data_.

Logo em seguida, é necessário ter alguma forma de sincronização. Uma maneira é usar **Change Data Capture** (CDC): as modificações no BD original são detectadas e alimentadas no novo BD. A técnica é descrita no livro de Sam Newman e também por Mario Amaral e outros membros da equipe da Elo7, no episódio [Estratégias de migração de dados no Elo7](https://hipsters.tech/estrategias-de-migracao-de-dados-no-elo7-hipsters-on-the-road-07/) (AMARAL et al., 2019) do podcast Hipsters On The Road.

> **Pattern: Change Data Capture (CDC)**
>
> Capture as mudanças em um BD, para que ações sejam tomadas a partir dos dados modificados.

Uma das maneiras de implementar CDC é usando _triggers_. É algo que os BDs já fazem e não há a necessidade de introduzir nenhuma nova tecnologia. Porém, como Sam Newman diz em seu livro, as ferramentas e o controle de mudanças de triggers deixam a desejar e podem complicar a aplicação se forem usadas exageradamente. Além disso, na maioria dos BDs só é possível executar SQL. E o destino for um BD não relacional ou um outro sistema?

Sam Newman diz, em seu livro, que uma outra maneira de implementar é utilizar um _Batch Delta Copier_: um programa que roda de tempos em tempos, com um `cron` ou similares, e consulta o BD original, verificando os dados que foram alterados e copiando os dados para o BD de destino. Porém, a lógica de saber o que foi alterado pode ser complexa e requerer a adição de colunas de _timestamps_ nas tabelas. Além disso, as consultas podem ser pesadas, afetando a performance do BD original.

Uma outra maneira de implementar CDC, descrita por Renato Sardinha no post [Introdução ao Change Data Capture (CDC)](https://elo7.dev/cdc-parte-1/) (SARDINHA, 2019) do blog de desenvolvimento da Elo7, é publicar _eventos_ (que estudaremos mais adiante) junto ao código que faz as modificações no BD original. A vantagem é que os eventos poderiam ser consumidos por qualquer outro sistema, não só BDs. Sardinha levanta a complexidade dessa solução: a arquitetura requer um sistema de Mensageria, há a necessidade dos desenvolvedores emitirem esses eventos manualmente e, se alterações forem feitas diretamente no BD por SQL, os eventos não seriam disparados.

A conclusão que os livros, podcasts e posts mencionados chegam é a mesma: podem ser usados os **transaction logs** dos BDs para implementar CDC. A maioria dos BDs relacionais mantém um log de transações (no MySQL, o `binlog`), que contém um registro de todas as mudanças comitadas e é usado na replicação entre nós de um cluster de BDs.

Existem _transaction log miners_ como o [Debezium](https://debezium.io/), que lêem o transaction log de BDs como MySQL, PostgreSQL, MongoDB, Oracle e SQL Server e publicam eventos automaticamente em um Message Broker (especificamente o Kafka, no caso do Debezium). A solução é complexa e requer um sistema de Mensageria, mas consegue obter atualizações feitas diretamente no BD e permite que os eventos sejam consumidos por diferentes ferramentas. Além do Debezium, existem ferramentas semelhantes como o [LinkedIn Databus](https://github.com/linkedin/databus) para o Oracle, o [DynamoDB Streams](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html) para o DynamoDB da Amazon e o [Eventuate Tram](https://github.com/eventuate-tram/eventuate-tram-core), mantido por Chris Richardson.

Com o CDC funcionando com Debezium ou outra ferramenta parecida, podemos usar uma estratégia progressiva descrita por Sam Newman, em seu livro e, de maneira semelhante, pelo pessoal da Elo 7:

- Inicialmente, é feito um dump (ou Bulk Synchronize)
- Depois, leituras e escritas são mantidas no BD original e os dados são escritos no novo BD com CDC. Assim, é possível observar o comportamento do novo BD com o volume de dados real.
- Em passo seguinte, o BD original fica apenas para leitura e a leitura e escrita é feita no novo BD. No caso de problemas inesperados, o BD original fica como solução paliativa.
- Finalmente, com a estabilização das operações e da migração, o BD original é removido.

É importante salientar que uma Migração de Dados não acontece de uma hora pra outra. Jeff Barr, da Amazon, diz no post [Migration Complete – Amazon’s Consumer Business Just Turned off its Final Oracle Database](https://aws.amazon.com/pt/blogs/aws/migration-complete-amazons-consumer-business-just-turned-off-its-final-oracle-database/) (BARR, 2019), que a migração de BDs Oracle para BDs da AWS de diferentes paradigmas, como DynamoDB, Aurora e Redshift, foi feita ao longo de vários anos.
