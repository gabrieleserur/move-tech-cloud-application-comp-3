Markdown
# Modelagem de Dados

## Entidades

### Pedido (`orders`)
Representa a entidade principal da compra efetuada por um cliente.

| Coluna | Tipo | Chave | Nulo? | Padrão / Comportamento | Descrição |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | `VARCHAR` | PK | Não | UUID v4 (`str(uuid4())`) | Identificador único do pedido |
| `customer` | `VARCHAR` | - | Não | - | Nome/Identificação do cliente |
| `status` | `VARCHAR` | - | Sim | `"open"` | Status do pedido (ex: `open`, `closed`) |
| `created_at` | `TIMESTAMPTZ` | - | Sim | `datetime.now(timezone.utc)` | Data e hora de criação do registro (UTC) |

---

### Item (`items`)
Representa os itens que compõem um pedido específico.

| Coluna | Tipo | Chave | Nulo? | Padrão / Comportamento | Descrição |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | `VARCHAR` | PK | Não | UUID v4 (`str(uuid4())`) | Identificador único do item |
| `order_id` | `VARCHAR` | FK | Não | Referência a `orders.id` | ID do pedido associado |
| `sku` | `VARCHAR` | - | Não | - | Código de identificação do produto (Stock Keeping Unit) |
| `description` | `VARCHAR` | - | Não | - | Descrição detalhada do produto |
| `quantity` | `INTEGER` | - | Não | - | Quantidade solicitada do produto |

---

## Relacionamento

* **Tipo:** **1:N (Um para Muitos)** entre `orders` e `items`.
* **Integridade Referencial & Cascata:**
  * A tabela `items` referencia a tabela `orders` através da chave estrangeira `order_id`.
  * Configuração do ORM: `cascade="all, delete-orphan"`. Se um registro em `orders` for removido, todos os itens (`items`) associados a ele serão deletados automaticamente.

text
+------------------+         1 : N         +-------------------+
|      orders      |---------------------->|       items       |
+------------------+                       +-------------------+
| id (PK)          |                       | id (PK)           |
| customer         |                       | order_id (FK)     |
| status           |                       | sku               |
| created_at       |                       | description       |
+------------------+                       | quantity          |
+-------------------+

---

## Como as tabelas são criadas

A criação das tabelas é gerenciada pelo **SQLAlchemy ORM** através da classe base comum de metadados (`Base.metadata.create_all(bind=engine)`).

* **Em desenvolvimento/testes (SQLite):** As tabelas são criadas localmente no arquivo `orders.db` na inicialização do serviço, caso ainda não existam.
* **Em produção (PostgreSQL):** Ao definir a variável de ambiente `DATABASE_URL` apontando para o banco relacional, o ORM executa a DDL (Data Definition Language) de criação das tabelas no esquema do banco configurado no boot da aplicação (ou via migrações com Alembic).
