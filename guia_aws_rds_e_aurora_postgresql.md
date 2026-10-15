# ☁️ Guia Prático: AWS RDS e Amazon Aurora (PostgreSQL)

Este guia documenta o passo a passo para provisionar, configurar o acesso público e conectar bancos de dados PostgreSQL gerenciados na AWS usando o **Amazon RDS** e o **Amazon Aurora**. 

> **Aviso:** Este laboratório foi desenhado para o ambiente acadêmico (AWS Academy / Learner Lab). As configurações de **Acesso Público** demonstradas aqui não são recomendadas para ambientes corporativos de produção.

---

## 🛠️ Parte 1: Criando um RDS PostgreSQL Tradicional

### Passo 1: Iniciar a Criação da Instância
1. Acesse o AWS Console e busque pelo serviço **RDS**.
2. Clique no botão laranja **Create database**.
3. Escolha o método **Standard create**.
4. Em *Engine options*, selecione **PostgreSQL**.
5. Em *Templates*, é crucial selecionar **Free tier** (Camada gratuita) para não consumir seus créditos.
6. Em *Settings*, defina:
   * **DB instance identifier:** `meubanco-demo`
   * **Master username:** `postgres`
   * **Master password:** Crie uma senha forte (e não a perca!).

### Passo 2: Configurar Conectividade e Acesso Público
Descendo a página de criação até a seção **Connectivity**:
1. **Virtual private cloud (VPC):** Deixe a `Default VPC` selecionada.
2. **Public access:** Mude para **Yes**.
3. **VPC security group (firewall):** Escolha **Create new** e digite o nome `rds-public-sg`.
4. Vá até o final da página e clique em **Create database**. Aguarde o status mudar de *Creating* para *Available* (5 a 10 minutos).

### Passo 3: Liberar a porta 5432 no Security Group
1. Com o banco *Available*, clique no nome dele (`meubanco-demo`).
2. Na aba **Connectivity & security**, clique no link abaixo de **VPC security groups** (`rds-public-sg`).
3. No painel de EC2, selecione o Security Group, vá na aba inferior **Inbound rules** e clique em **Edit inbound rules**.
4. Clique em **Add rule**:
   * **Type:** PostgreSQL
   * **Port range:** 5432
   * **Source:** Mude para `Anywhere-IPv4` (0.0.0.0/0). 
5. Salve as regras.

---

## 🚀 Parte 2: Criando um Cluster Amazon Aurora PostgreSQL

*Nota: Se aparecer um erro vermelho sobre `ram:GetResourceShares` no topo da tela, ignore. É uma restrição visual do AWS Academy e não impede a criação.*

### Passo 1: Iniciar a Criação do Cluster
1. No painel do RDS, clique em **Create database**.
2. Escolha **Standard create**.
3. Em *Engine options*, selecione **Amazon Aurora**.
4. Em *Edition*, selecione **Amazon Aurora PostgreSQL-Compatible Edition**.
5. Em *Capacity type*, selecione **Provisioned**.
6. Em *Templates*, escolha **Dev/Test** (Não existe Free Tier aqui).

### Passo 2: Configurações e Tamanho da Máquina
1. Em *Settings*, defina o **DB cluster identifier** (ex: `meuaurora-cluster`).
2. Defina o **Master username** (`postgres`) e crie uma **Master password** manualmente (evite a opção de autogerar para não perder a senha).
3. Em **Instance configuration**, mude para **Burstable classes (includes t classes)** e selecione a menor possível (ex: `db.t3.medium`).
4. Em **Availability & durability**, selecione **Don't create an Aurora Replica** (para economizar créditos).

### Passo 3: Rede e Proteção
1. Em **Connectivity**, mantenha a VPC padrão.
2. **Public access:** Marque **Yes**.
3. **VPC security group:** Selecione **Choose existing** e escolha o `rds-public-sg` (que já tem a porta 5432 aberta).
4. Abra a aba **Additional configuration** (no final da página) e **DESMARQUE** a opção **Enable deletion protection**.
5. Clique em **Create database**.

---

## 🔌 Parte 3: Conectando no pgAdmin

Para conectar no pgAdmin, DBeaver ou sua aplicação, você precisará do **Endpoint**.
* **No RDS Padrão:** Clique na instância e copie o valor de **Endpoint** na aba *Connectivity & security*.
* **No Aurora:** Clique no **Cluster**, vá na aba *Connectivity & security* e copie o **Writer instance endpoint** (não copie o de leitura/reader).

### Dados de Conexão:
* **Host/Server:** [Cole o Endpoint copiado]
* **Port:** 5432
* **Maintenance database:** postgres
* **Username:** postgres
* **Password:** [Sua senha]

### Script de Teste (SQL)
```sql
-- 1. Criação da estrutura
CREATE TABLE desenvolvedores (
    id SERIAL PRIMARY KEY,
    nome VARCHAR(100) NOT NULL,
    tecnologia VARCHAR(50)
);

-- 2. Inserção de dados
INSERT INTO desenvolvedores (nome, tecnologia) 
VALUES ('Seu Nome', 'PostgreSQL na AWS');

-- 3. Leitura para validação
SELECT * FROM desenvolvedores;
```

---

## 🧹 Parte 4: Limpeza de Recursos (IMPORTANTE)
Para evitar que sua conta do AWS Academy esgote os créditos, exclua os recursos após o uso.

1. **RDS Padrão:** Selecione a instância -> Actions -> Delete. Desmarque a criação de snapshot final.
2. **Aurora:** Selecione **apenas a Instância** (role *Writer*, que fica abaixo do cluster) -> Actions -> Delete. Desmarque o snapshot final. O cluster será excluído automaticamente após a exclusão da instância.