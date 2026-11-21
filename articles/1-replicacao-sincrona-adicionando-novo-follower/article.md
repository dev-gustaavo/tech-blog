# Replicação Síncrona - Adicionando Um Novo Follower Em Produção

# Introdução

No [artigo anterior](https://medium.com/@gtbarbosa/replica%C3%A7%C3%A3o-s%C3%ADncrona-o-que-ela-garante-e-quanto-custa-single-leader-replication-a25833391423), nós destrinchamos conceitualmente como funciona a **Replicação Síncrona de Dados** e algumas das principais preocupações que precisamos ter ao usar este tipo de arquitetura.

Um tópico importante, que vale um artigo a parte e que vamos destrinchar neste artigo é, como adicionamos uma nova réplica do banco, sem causar *downtime* na nossa aplicação e como esta réplica recebe os dados históricos das demais réplicas.

Na medida em que nossa aplicação cresce, é natural que tenhamos que adotar estratégias que melhorem sua performance e uma das estratégias pode ser adicionar uma nova réplica do banco de dados, pensando em:

- Melhorar throughput de leitura; ou
- Aproximar os dados em uma região geográfica que estejam mais próximas dos usuários.

# Adicionando Um Novo Follower

Em arquiteturas mais modernas, usando algum Cloud Provider, adicionar um novo follower pode ser, aparentemente, uma tarefa simples. No entanto, o que realmente acontece, é uma tarefa mais complexa e que exige alguns cuidados, a fim de garantir a consistência dos dados entre todas as réplicas.

## Snapshot e Posição do WAL

Vamos relembrar alguns pontos. Primeiro, o *snapshot* nada mais é do que uma foto do estado atual de um banco de dados. Segundo, o WAL é um acrônimo para *Write Ahead Log* e é a partir dele que o banco sabe o que ele precisa escrever de fato nas tabelas (para maiores detalhes, [leia o primeiro artigo desta série de replicação](https://medium.com/@gtbarbosa/replica%C3%A7%C3%A3o-s%C3%ADncrona-o-que-ela-garante-e-quanto-custa-single-leader-replication-a25833391423)).

Quando adicionamos uma nova réplica em nossa arquitetura, precisamos garantir que ela contenha os dados históricos, a fim de garantir a consistência dos dados entre todas as réplicas. Para isso, no processo de adicionar uma nova réplica, o primeiro passo é fazer o *snapshot* de uma das réplicas já presentes na arquitetura, geralmente do *leader*. Neste *snapshot* a gente também vai capturar a posição em que o WAL está e isso é importante, por que novas escritas continuam acontecendo e também queremos garantir que elas cheguem na nova réplica.

<p align="center"><img src="img/snapshot-wal.png"></p>

## Garantindo Sequencialidade Nos Dados

Uma vez que a gente tenha o *snapshot* na nova réplica e ela esteja pronta para se conectar ao *leader*, precisamos então sincronizá-la com os dados que foram escritos durante o processo de captura, transporte e restauração do *snapshot* e é por isso que temos a posição do WAL no *snapshot*.

Quando a nova réplica se conecta ao *leader* pelo *replication stream* em modo *standby*, inicia-se o processo de sincronização dos dados. A réplica pede ao *leader*, através da posição do WAL, os dados que ela precisa receber. É algo como: *me dê tudo a partir da posição 123 do WAL*.

A partir da posição indicada, o *leader* lê o WAL e envia os dados de forma ordenada, garantindo sequencialidade dos dados.

> O modo `standby` indica que é um banco de dados réplica, que recebe dados vindos de um leader e que não aceita escritas diretamente.

> A posição do WAL é um valor crescente, representado em hexadecimal. O 123 mencionado acima é apenas um exemplo.

Todas as operações citadas acontecem enquanto o *leader* continua recebendo escritas, ou seja, a réplica está correndo atrás de algo que continua se movendo. Quando a réplica chega na mesma posição do WAL que o *leader* está, chamamos isso de *catch-up* e então a nova réplica passa a receber os dados praticamente em tempo real, conforme vimos no [artigo anterior](https://medium.com/@gtbarbosa/replica%C3%A7%C3%A3o-s%C3%ADncrona-o-que-ela-garante-e-quanto-custa-single-leader-replication-a25833391423). Tudo isso pressupõe que o *leader* ainda tem o WAL desde a posição do *snapshot*. Mas será que ele tem?

# Lidando Com Limpeza do WAL

Como vimos no [primeiro artigo](https://medium.com/@gtbarbosa/replica%C3%A7%C3%A3o-s%C3%ADncrona-o-que-ela-garante-e-quanto-custa-single-leader-replication-a25833391423), o WAL é um arquivo de *logging* que armazena sequencialmente as operações de escrita no banco e isso tem um custo. Quanto mais dados retidos, maior o tamanho de ocupação em disco, aumentando a chance de degradar o banco em operação. Por isso, é comum que os bancos de dados, em determinado momento, façam a reciclagem do WAL, a fim de garantir uma operação saudável ao banco de dados. É como se adicionar uma nova réplica fosse um plano secundário ao banco e manter a saúde da operação do banco fosse o plano primário e isso traz um risco associado ao *setup* de uma nova réplica.

Para ambientes que lidam com baixo volume de escrita, este risco diminui, pois o processo de *setup* tende a ser mais rápido para alcançar a posição do WAL, enquanto ambientes que lidam com alto volume de escrita, tendem a ser mais críticos e suscetíveis a reciclagem do WAL.

Mas afinal, o que fazer se o WAL for reciclado durante o *setup* de uma nova réplica?

## Garantindo o Mínimo de Retenção do WAL

A solução mais trivial e mais simples é garantir um mínimo de retenção dos *logs*. A retenção funciona de maneiras diferentes a depender do banco de dados que você esteja trabalhando, por isso, é extremamente importante conferir na documentação oficial. Para efeitos didáticos, abaixo vou deixar dois exemplos (PostgreSQL e MySQL).

- **PostgreSQL**: ele trabalha na retenção por tamanho. Na configuração do banco, você consegue determinar o tamanho mínimo de retenção de *logs*;

- **MySQL**: já o MySQL, lida com a retenção por tempo. Na configuração do banco, Você consegue determinar o tempo de retenção dos *logs*.

Desta forma, conseguimos lidar com uma nova réplica, garantindo que ela seja adicionada antes da reciclagem do WAL.

## Reserva do WAL Ancorada à Réplica

Outra forma de minimizar que as réplicas sofram com a reciclagem do WAL, é ancorar a reciclagem ao término da cópia pela nova réplica. Ou seja, determinar que o *log* só pode ser reciclado quando a réplica confirma a leitura.

Cada *append only* no WAL recebe um identificador que garante a sequencialidade das operações, então quando ancoramos a reciclagem do WAL na conclusão da cópia pela réplica, basicamente o que acontece é: *"pode reciclar do log XPTO para trás"*.

É importante saber que para bancos que se enquadrem no conceito do *ACID*, a disponibilidade será sempre uma prioridade, portanto:

1. Atente-se ao tamanho do WAL e espaço em disco, pois caso ele seja saturado, o WAL poderá ser reciclado;
2. Se houver alguma indisponibilidade ou falha no processo de *setup* da nova réplica, o ponto de reciclagem fica preso para sempre, podendo também ocasionar a reciclagem forçada do WAL;
3. Ter uma válvula de escape é importante, evitando gargálos no estado atual do banco. Prefira perder a nova réplica, do que saturar toda sua arquitetura.

Essa prática também funciona diferente a depender do banco que você esteja usando, portanto, consulte a documentação oficial para referências.

## Arquivamento do WAL

Já que o ponto importante para garantir a consistência entre todas as réplicas é o WAL, outra estratégia pode ser, arquivar os *logs*. Isso garante que tenhamos uma cópia dos *logs* fora do *node* do banco e caso sejam reciclados, ainda tenhamos da onde restaurar.

Na prática, essa abordagem envolve configurar o banco para que cada *append only* que ocorra, seja armazenado fisicamente em um storage, antes que a reciclagem aconteça e somente depois da confirmação que o arquivamento aconteceu efetivamente. No processo de *setup* da nova réplica, fazemos a restauração deste storage, garantindo a consistência entre todas as reṕlicas e o novo *follower*.

Esse modelo precisa que, o banco esteja configurado para armazenar os *logs* em um *object storage*. No PostgreSQL por exemplo, você define um `archive_command`, que envia o arquivo para um local seguro. Para que a réplica consiga ler o WAL, você pode definir o `restore_command`.


# Fontes

- Backups (PostgreSQL): https://www.postgresql.org/docs/18/app-pgbasebackup.html
- Adicionando nova réplica (PostgreSQL): https://www.postgresql.org/docs/18/warm-standby.html
- Retenção por tamanho (PostgreSQL): https://www.postgresql.org/docs/18/runtime-config-replication.html#RUNTIME-CONFIG-REPLICATION
- Retenção por tempo (MySQL): https://dev.mysql.com/doc/refman/26.10/en/purge-binary-logs.html
- Replication Slots (PostgreSQL): https://www.postgresql.org/docs/18/warm-standby.html#STREAMING-REPLICATION-SLOTS
- WAL Archiving (PostgreSQL): https://www.postgresql.org/docs/current/runtime-config-wal.html#RUNTIME-CONFIG-WAL-ARCHIVING