# Tutorial: provisionando bancos de dados na AWS com CloudFormation

Este passo a passo mostra como Instanciar, **um único template**, quatro serviços de dados da AWS e uma EC2 para testar o cache:

| Serviço | Tipo | O que é |
|---|---|---|
| **RDS PostgreSQL** | Relacional | Uma instância PostgreSQL gerenciada, com disco próprio |
| **Aurora PostgreSQL** | Relacional | Cluster compatível com PostgreSQL, com armazenamento distribuído |
| **DynamoDB** | NoSQL (chave-valor) | Tabela serverless, aqui com capacidade provisionada de 5 RCU / 5 WCU |
| **ElastiCache (Redis)** | Cache em memória | Grupo de replicação Redis com 1 nó, para dados temporários e de acesso rápido. Com mais nós, ganha réplicas e failover automático |
| **EC2** | Máquina virtual | Cliente de testes dentro da mesma VPC do Redis |

O arquivo do template é o `database-stack-simples.yaml`. O código completo está no [Anexo](#anexo-código-completo-do-template), no final deste documento: basta copiar e salvar com esse nome.

---

## 1. Como o template está organizado

Um template CloudFormation tem seções. Este usa três:

1. **Parameters**: valores que você pode mudar sem editar os recursos.
2. **Resources**: os serviços que serão criados.
3. **Outputs**: informações úteis ao final, como os endereços (endpoints) para conectar.

### Parâmetros

| Parâmetro | Padrão | Para que serve |
|---|---|---|
| `DBUsername` | `postgres` | Usuário administrador do RDS e do Aurora |
| `DBPassword` | `Senha12345` | Senha do administrador (mínimo 8 caracteres) |
| `DBName` | `appdb` | Nome do banco criado dentro do RDS e do Aurora |
| `AllowedCidr` | `0.0.0.0/0` | Faixa de IPs com acesso às portas dos bancos, do Redis e do SSH |
| `TableName` | `demo-tabela` | Nome da tabela DynamoDB |
| `ReadCapacity` | `5` | Unidades de leitura (RCU) do DynamoDB |
| `WriteCapacity` | `5` | Unidades de escrita (WCU) do DynamoDB |
| `KeyName` | `vockey` | Key pair para acessar a EC2 por SSH |
| `LatestAmiId` | (automático) | Imagem do Amazon Linux 2023 mais recente |

### Dependências entre recursos

O CloudFormation descobre a ordem de criação sozinho:

- **Implícita:** quando um recurso usa `!Ref` ou `!GetAtt` de outro, ele espera o outro ficar pronto. Exemplo: os bancos usam o ID do security group, então o security group é criado primeiro.
- **Explícita:** `DependsOn` força uma ordem. Exemplo: a instância do Aurora declara `DependsOn: AuroraCluster`.
- A EC2 lê o endpoint do Redis no `UserData`, então só é criada depois que o cache existe.

---

## 2. Criando a stack pelo console

2. Abra o CloudFormation.
3. Clique em **Create stack** e depois em **With new resources (standard)**.
4. Em *Prerequisite - Prepare template*, deixe **Choose an existing template**.
5. Em *Specify template*, escolha **Upload a template file**, clique em **Choose file** e selecione `database-stack-simples.yaml`. Clique em **Next**.
6. Em *Stack name*, digite `demo-bancos`.
7. A tela mostra os **parâmetros** do template. Você pode manter os padrões ou alterar, por exemplo, a senha ou a capacidade do DynamoDB. Clique em **Next**.
8. Na tela *Configure stack options*, não precisa mudar nada. Clique em **Next**.
9. Revise e clique em **Submit**.
10. Acompanhe a criação:
    - **Events:** mostra cada recurso sendo criado, na ordem das dependências.
    - **Resources:** lista os recursos e seus status.
    - **Outputs:** aparece quando a stack termina, com os endpoints.

A stack estará pronta quando o status for **`CREATE_COMPLETE`**. O Aurora e o RDS são os mais demorados.

---

## 3. Criando a stack pelo CloudShell

O CloudShell é um terminal dentro do console da AWS, já com a AWS CLI e as credenciais configuradas.

1. No console, clique no ícone **`>_`** da barra superior (ou procure por **CloudShell**). Aguarde o terminal iniciar.
2. Envie o template: clique em **Actions → Upload file**, selecione `database-stack-simples.yaml` e confirme.
3. Confira se o arquivo chegou:

```bash
ls
```

4. Crie a stack:

```bash
aws cloudformation deploy \
  --template-file database-stack-simples.yaml \
  --stack-name demo-bancos
```

O comando só retorna quando a stack terminar de ser criada. Para alterar parâmetros, acrescente `--parameter-overrides`:

```bash
aws cloudformation deploy \
  --template-file database-stack-simples.yaml \
  --stack-name demo-bancos \
  --parameter-overrides ReadCapacity=10 WriteCapacity=10 DBPassword=OutraSenha123
```

5. Veja os outputs (endpoints):

```bash
aws cloudformation describe-stacks \
  --stack-name demo-bancos \
  --query "Stacks[0].Outputs" \
  --output table
```

6. Para acompanhar os eventos em outra aba do terminal enquanto cria:

```bash
aws cloudformation describe-stack-events \
  --stack-name demo-bancos \
  --query "StackEvents[].[Timestamp,LogicalResourceId,ResourceStatus]" \
  --output table
```

---

## 5. Testando cada serviço

Os endpoints aparecem na aba **Outputs** da stack (ou no comando `describe-stacks`).

### Redis (pela EC2)

O Redis não aceita acesso público, por isso o teste é feito de dentro da EC2.

1. No console, vá em **EC2 → Instances**, selecione `demo-redis-client` e clique em **Connect**.
2. Na aba **EC2 Instance Connect**, clique em **Connect**. Abre um terminal no navegador.
3. Aguarde cerca de 1 a 2 minutos após a criação para o cliente terminar de instalar e rode:

```bash
redis6-cli -h $REDIS_HOST
```

4. Dentro do cliente Redis:

```
SET usuario:1 "Ana"
GET usuario:1
SET sessao:abc "token123" EX 30
TTL sessao:abc
INCR contador
INCR contador
```

O `EX 30` faz a chave expirar em 30 segundos. Rode `GET sessao:abc` depois desse tempo e o resultado será vazio.

> Se o comando `redis6-cli` não for encontrado, espere mais um pouco. O log da instalação fica em `/var/log/cloud-init-output.log`.

> Se `$REDIS_HOST` estiver vazia (confira com `echo $REDIS_HOST`), rode `source /etc/profile.d/demo.sh`. Se continuar vazia, copie o endpoint do output `RedisEndpoint` da stack (ou o **Endpoint primário** na página do cache, em ElastiCache) e use `redis6-cli -h ENDPOINT`.

### RDS e Aurora (pela EC2)

Na mesma EC2, o cliente `psql` já está instalado. Troque `ENDPOINT` pelo valor de `RdsEndpoint` ou `AuroraWriterEndpoint`:

```bash
psql -h ENDPOINT -U postgres -d appdb
```

A senha padrão é `Senha12345`. Dentro do `psql`:

```sql
CREATE TABLE produtos (id serial PRIMARY KEY, nome text);
INSERT INTO produtos (nome) VALUES ('Teclado'), ('Mouse');
SELECT * FROM produtos;
\q
```

O `AuroraReaderEndpoint` aponta para o endpoint de leitura do cluster.

### DynamoDB (pelo CloudShell)

```bash
aws dynamodb put-item \
  --table-name demo-tabela \
  --item '{"id": {"S": "1"}, "nome": {"S": "Ana"}}'

aws dynamodb get-item \
  --table-name demo-tabela \
  --key '{"id": {"S": "1"}}'
```

A tabela usa só a chave de partição `id` (tipo string). Todos os outros atributos são livres, pois o DynamoDB não tem esquema fixo para eles.

---

## 6. Limpeza: apagando tudo

Apagar a stack remove **todos** os recursos criados por ela, na ordem inversa das dependências. Como o template usa `DeletionPolicy: Delete`, nenhum snapshot final é criado.

**Pelo console:** em **CloudFormation → Stacks**, selecione `demo-bancos`, clique em **Delete** e confirme. Acompanhe na aba **Events** até aparecer `DELETE_COMPLETE`.

**Pelo CloudShell:**

```bash
aws cloudformation delete-stack --stack-name demo-bancos

aws cloudformation wait stack-delete-complete --stack-name demo-bancos
```

Espere o `DELETE_COMPLETE` antes de criar a stack de novo, porque o nome da tabela DynamoDB é fixo.

---

## 7. Problemas comuns

| Sintoma | O que fazer |
|---|---|
| Stack em `ROLLBACK_COMPLETE` | Abra a aba **Events**, procure a linha com `CREATE_FAILED` e leia o motivo. Apague a stack e crie de novo após corrigir. |
| Erro de `vockey` não encontrado | Troque o parâmetro `KeyName` por um key pair que exista na sua região. |
| Erro de classe de instância não permitida | O Learner Lab limita alguns tamanhos. A mensagem aparece em **Events**. |
| `DELETE_FAILED` | Tente **Delete** de novo. O security group às vezes demora a ser liberado. |

---

## 8. Resumo do que foi demonstrado

- **Provisionamento e configurações:** cinco serviços criados com infraestrutura como código.
- **Dependências:** implícitas (`!Ref`, `!GetAtt`) e explícitas (`DependsOn`).
- **Parâmetros:** o mesmo template serve para ambientes diferentes.
- **Outputs:** endpoints disponíveis ao final da criação.
- **Limpeza:** um único `delete-stack` remove tudo.

---

## Anexo: código completo do template

Salve o conteúdo abaixo em um arquivo chamado `database-stack-simples.yaml` (a indentação com espaços precisa ser mantida).

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Demo de computacao em nuvem - RDS PostgreSQL, Aurora PostgreSQL, DynamoDB e ElastiCache (Redis)

Parameters:
  DBUsername:
    Type: String
    Default: postgres
    Description: Usuario administrador do RDS e do Aurora

  DBPassword:
    Type: String
    Default: Senha12345        
    NoEcho: true               # esconde o valor no console e nos logs
    MinLength: 8
    Description: Senha do usuario administrador

  DBName:
    Type: String
    Default: appdb
    Description: Nome do banco criado dentro do RDS e do Aurora

  AllowedCidr:
    Type: String
    Default: 0.0.0.0/0        
    Description: Faixa de IPs com acesso aos bancos

  TableName:
    Type: String
    Default: demo-tabela-dynamo
    Description: Nome da tabela DynamoDB

  ReadCapacity:
    Type: Number
    Default: 5
    MinValue: 1
    Description: Unidades de leitura (RCU) provisionadas na tabela

  WriteCapacity:
    Type: Number
    Default: 5
    MinValue: 1
    Description: Unidades de escrita (WCU) provisionadas na tabela

  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Default: vockey            # key pair que o Learner Lab ja cria
    Description: Key pair para acessar a EC2 por SSH

  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64
    Description: AMI do Amazon Linux 2023 mais recente (resolvida automaticamente)

Resources:
  # Security group: firewall compartilhado pelos bancos e pelo cache.
  # E a primeira dependencia: os outros recursos o referenciam com !Ref/!GetAtt,
  # entao o CloudFormation cria ele ANTES deles (dependencia implicita).
  SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Acesso PostgreSQL (5432) e Redis (6379)
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          CidrIp: !Ref AllowedCidr
        - IpProtocol: tcp
          FromPort: 6379
          ToPort: 6379
          CidrIp: !Ref AllowedCidr
        - IpProtocol: tcp      # SSH para acessar a EC2 da demo
          FromPort: 22
          ToPort: 22
          CidrIp: !Ref AllowedCidr

  # RDS PostgreSQL:
  RdsPostgres:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Delete             # LIMPEZA: apaga sem criar snapshot final
    Properties:
      Engine: postgres
      DBInstanceClass: db.t3.micro
      AllocatedStorage: "20"           # GiB
      DBName: !Ref DBName
      MasterUsername: !Ref DBUsername
      MasterUserPassword: !Ref DBPassword
      VPCSecurityGroups:
        - !GetAtt SecurityGroup.GroupId   # dependencia implicita
      PubliclyAccessible: true
      DeletionProtection: false

  # Aurora PostgreSQL:
  AuroraCluster:
    Type: AWS::RDS::DBCluster
    DeletionPolicy: Delete
    Properties:
      Engine: aurora-postgresql
      DatabaseName: !Ref DBName
      MasterUsername: !Ref DBUsername
      MasterUserPassword: !Ref DBPassword
      VpcSecurityGroupIds:
        - !GetAtt SecurityGroup.GroupId
      DeletionProtection: false

  # a instancia fornece a CPU/memoria. Sem ela o cluster nao tem endpoint util.
  AuroraInstance:
    Type: AWS::RDS::DBInstance
    DependsOn: AuroraCluster           # dependencia EXPLICITA (a Ref abaixo ja a tornaria implicita)
    Properties:
      Engine: aurora-postgresql
      DBClusterIdentifier: !Ref AuroraCluster
      DBInstanceClass: db.t3.medium    # menor classe suportada pelo Aurora PostgreSQL
      PubliclyAccessible: true

  # DynamoDB:
  DynamoTable:
    Type: AWS::DynamoDB::Table
    DeletionPolicy: Delete
    Properties:
      TableName: !Ref TableName
      BillingMode: PROVISIONED         # capacidade fixa; sem Auto Scaling configurado
      ProvisionedThroughput:
        ReadCapacityUnits: !Ref ReadCapacity
        WriteCapacityUnits: !Ref WriteCapacity
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH                # chave de particao

  # --- ElastiCache (Redis): grupo de replicacao com 1 no.
  #     Ele expoe o "Endpoint primario". Com NumCacheClusters >= 2 passa a ter
  #     replicas (Endpoint leitor) e failover automatico. ---
  RedisCache:
    Type: AWS::ElastiCache::ReplicationGroup
    Properties:
      ReplicationGroupDescription: Redis da demo
      Engine: redis
      CacheNodeType: cache.t3.micro
      NumCacheClusters: 1
      SecurityGroupIds:
        - !GetAtt SecurityGroup.GroupId

  # --- EC2 da demo: o Redis nao tem acesso publico, entao precisamos de uma
  #     maquina DENTRO da mesma VPC para conversar com ele ---
  DemoInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: !Ref LatestAmiId
      KeyName: !Ref KeyName
      SecurityGroupIds:
        - !GetAtt SecurityGroup.GroupId
      Tags:
        - Key: Name
          Value: demo-redis-client
      # O !Sub abaixo le o endpoint do Redis: isso cria uma dependencia implicita,
      # entao a EC2 so e criada depois que o cache existir.
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          dnf install -y redis6 postgresql15
          echo "export REDIS_HOST=${RedisCache.PrimaryEndPoint.Address}" > /etc/profile.d/demo.sh
          echo "source /etc/profile.d/demo.sh" >> /home/ec2-user/.bashrc

Outputs:
  RdsEndpoint:
    Description: Endpoint do RDS PostgreSQL (porta 5432)
    Value: !GetAtt RdsPostgres.Endpoint.Address

  AuroraWriterEndpoint:
    Description: Endpoint de escrita do Aurora (porta 5432)
    Value: !GetAtt AuroraCluster.Endpoint.Address

  AuroraReaderEndpoint:
    Description: Endpoint de leitura do Aurora
    Value: !GetAtt AuroraCluster.ReadEndpoint.Address

  DynamoTableName:
    Description: Nome da tabela DynamoDB
    Value: !Ref DynamoTable

  DynamoTableArn:
    Description: ARN da tabela DynamoDB
    Value: !GetAtt DynamoTable.Arn

  RedisEndpoint:
    Description: Endpoint primario do Redis (porta 6379)
    Value: !GetAtt RedisCache.PrimaryEndPoint.Address

  DemoInstancePublicIp:
    Description: IP publico da EC2 da demo
    Value: !GetAtt DemoInstance.PublicIp

  SshCommand:
    Description: Comando para acessar a EC2 (use o .pem do vockey)
    Value: !Sub "ssh -i labsuser.pem ec2-user@${DemoInstance.PublicIp}"
```