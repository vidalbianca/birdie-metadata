# Birdie Metadata Builder

## Início da conversa — Onboarding

Quando o usuário iniciar uma conversa mencionando metadados, Birdie ou critérios, **ANTES de construir qualquer coisa**, siga este fluxo:

### Passo 0 — Verificação de pré-requisitos
Antes de tudo, verifique se o ambiente está configurado. Execute os checks em paralelo:

1. **Databricks CLI**: execute `databricks --version` — se falhar, informe:
   > "Você precisa do Databricks CLI instalado e autenticado. Instale com `brew install databricks` (macOS) e configure com `databricks configure --profile ops_analytics` usando o host `https://nubank-e2-general.cloud.databricks.com`."

2. **MCP Google Workspace**: verifique se existe `.mcp.json` na raiz do projeto com o server `google-workspace` configurado — se não existir, informe:
   > "Para ler critérios direto do Google Docs, você precisa configurar o MCP Google Workspace. Peça as instruções de setup para quem já configurou ou use arquivos `.txt` locais como alternativa."

3. **Este arquivo (CLAUDE.md)**: se o usuário diz que não tem ou quer configurar em outra máquina, instrua:
   > "A base de conhecimento fica no repositório. Para ter acesso:"
   > 1. Clone o repositório: `git clone git@github.com:vidalbianca/birdie-metadata.git`
   > 2. Entre no diretório: `cd birdie-metadata`
   > 3. Inicie o Claude Code: `claude`
   > 4. O CLAUDE.md será carregado automaticamente

Se algum pré-requisito falhar, ajude o usuário a resolver antes de prosseguir. Se o MCP Google não estiver disponível, ofereça o fluxo com arquivos `.txt` locais como alternativa.

### Passo 1 — Apresentação
Explique brevemente o que você faz:
> "Eu sou o assistente de construção de metadados para a Birdie. Consigo ler os critérios das squads (direto do Google Docs ou de arquivos locais), identificar quais dados do sistema a IA precisa, buscar as tabelas ETL no Databricks e construir os notebooks em Scala/Spark."

### Passo 2 — Coleta de informações
Pergunte ao usuário:

1. **Squad**: Qual squad estamos trabalhando? (ex: NuCel, Crypto, PJ, Collections, Growth)
2. **Fonte dos critérios**: Os critérios estão em Google Docs (compartilhe o link da planilha ou dos Docs) ou em arquivos `.txt` locais?
3. **Notebook existente**: É para criar um notebook novo ou adicionar critérios a um notebook existente?
4. **Workspace Databricks**: Qual o caminho do seu workspace? (ex: `/Workspace/Users/nome@nubank.com.br/birdie/`)
5. **Profile CLI**: Qual profile do Databricks CLI você usa? Se não souber, posso verificar os profiles configurados na sua máquina com `databricks auth profiles`.

### Passo 3 — Leitura dos critérios
Após receber as respostas:
- Se Google Docs: use o MCP `google-workspace` — `sheets_getText` para listar critérios e `docs_getText` para ler cada Doc
- Se arquivos locais: leia os `.txt` da pasta indicada pelo usuário

### Passo 4 — Proposta de metadados
Para cada critério, apresente ao usuário:
- Nome do critério
- Metadados propostos (nome, tipo, lógica)
- Tabela ETL fonte
- Metadados indisponíveis no ETL (se houver)

**Aguarde validação do usuário antes de construir o notebook.**

### Passo 5 — Construção
Só após validação, construa/atualize o notebook e importe no Databricks.

---

## O que é a Birdie

A **Birdie** é a ferramenta de monitoria de qualidade de atendimento (chat/voz) via IA. Metadados são dados objetivos do sistema que a IA da Birdie cruza com o **transcrito do atendimento** para avaliar se o agente seguiu as regras corretamente.

**Exemplo:** A IA lê o metadado `subscription_renewal_date = 2025-04-15` e compara com o que o agente disse no chat. Se o agente informou uma data diferente, é um erro.

## Como ler critérios

Cada critério tem um Google Doc com 5 seções. A leitura deve priorizar:

1. **Regras de avaliação** (seção 4) — MAIS IMPORTANTE. Define YES/NO (erro/sem erro). Aqui está o que a IA precisa comparar.
2. **Escopo da avaliação** (seção 3) — Descreve a ação esperada do agente e o contexto do atendimento.
3. **Objetivo do critério** (seção 1) — Contexto geral.

**IGNORAR como fonte primária:** A seção "Variáveis (metadados)" (seção 2) frequentemente não está alinhada com as regras. Use-a apenas como referência inicial, nunca como definição final.

**Pergunta-chave:** "Que dado objetivo do sistema a IA precisa para comparar com o que o agente disse no transcrito?"

## Fontes de input

### Via Google Workspace MCP (preferencial)
- `sheets_getText` com o ID da planilha da squad → lista critérios e links dos Docs
- `docs_getText` com o documentId extraído do link → conteúdo completo do critério

### Via arquivos locais (fallback)
- Critérios em `.txt` numa pasta local

## Estrutura do notebook Scala/Spark

Cada notebook segue esta estrutura fixa:

```
1. Imports          — %run "/data-analysts/utils" + imports Scala padrão
2. Start Dates      — val start_date = "YYYY-MM-DD"
3. Tickets Data     — OKR → fraudsters → segments → stop times → interaction_with_stop_time
4. Metadata         — Uma seção por critério com DataFrames de origem
5. Final Join       — LEFT JOINs encadeados em interaction_with_stop_time
6. Validação        — Row counts, samples, distribuições
```

### Regras críticas do join

- **Output: 1 linha por `source_id`** — o join NUNCA deve duplicar linhas
- Todos os DataFrames de metadado devem ter **1 linha por `customer__id`** antes do join
- Usar `groupBy` + `collect_list` para históricos, depois `filter`/`aggregate` no join
- Usar `row_number()` + `filter(_rn === 1)` quando a tabela fonte pode ter múltiplas linhas por cliente
- Sempre validar com `println(s"interaction_with_stop_time: ${interaction_with_stop_time.count}")` vs `interaction_with_metadata`

### Janela temporal

Metadados devem respeitar a data do ticket. Se o cliente não era elegível no momento do atendimento, o campo deve ser `null`:

```scala
// Exemplo: só popula se era cliente NuCel no momento do ticket
.withColumn("nucel_plan_status",
  when(!$"is_nucel_customer", lit(null))
  .when($"nucel_account_status" === "active", lit("ACTIVE"))
  .otherwise(lit(null))
)
```

### Separadores de células Databricks

```scala
// COMMAND ----------
// DBTITLE 1,titulo_da_celula
// MAGIC %md
// MAGIC ## Título markdown
```

## Convenções

- Máximo **5 metadados por critério**
- Tipos: `String`, `Boolean`, `Timestamp`, `Integer`, `Float`
- Nomenclatura: `snake_case` (ex: `nucel_portability_status`, `crypto_first_purchase_date`)
- Prefixos: `is_` (boolean), `has_` (boolean), `_date`/`_at` (timestamps)
- **AJIS já foi criado na Onda 1** — não pesquisar nem criar metadados AJIS
- Metadados indisponíveis no ETL: documentar com `⚠️ NÃO DISPONÍVEL NO ETL:` no markdown do notebook
- Metadados da Onda 1: documentar com `✅ Onda 1:` no markdown

## Catálogo de tabelas ETL

### Base de tickets (todas as squads)
| Tabela | Uso |
|--------|-----|
| `etl.br__dataset.ops_canonical_next_contact_okr` | Base de tickets (source_id, subject_id, start_time) |
| `etl.br__contract.big_mama__reports` | Filtro de fraudsters |
| `etl.br__dataset.br_segments_v5` | Segmentos de clientes |
| `etl.br__dataset.ops_canonical_chats/calls/emails` | Stop times |

### NuCel
| Tabela | Campos-chave | Metadados |
|--------|-------------|-----------|
| `etl.br__dataset.nucel_accounts` | `status`, `last_payment`, `last_suspension_reason`, `current_port_in_finished_at` | `subscription_renewal_date`, `is_nucel_customer`, `nucel_plan_status`, `nucel_portability_completed_at` |
| `etl.br__dataset.nucel_current_portability_in` | `status`, `scheduled_to`, `cancellation_reason`, `finished_at` | `nucel_portability_status`, `nucel_scheduled_portability_date`, `nucel_portability_rejection_reason` |
| `etl.br__dataset.nucel_psim_delivery_details_dashboard` | `delivery__status`, `delivery__created_at`, `delivered_at` | `psim_delivery_status`, `psim_estimated_delivery_date` |

### Crypto
| Tabela | Campos-chave | Metadados |
|--------|-------------|-----------|
| `etl.br__contract.crypto_asset_vaults__asset_vaults` | `asset_vault__asset`, `asset_vault__amount` | `has_nucoin_only`, `usdc_current_balance` |
| `etl.br__contract.crypto_broker__order_requests` | `order_request__status`, `order_request__created_at`, `order_request__gross_value` | `crypto_first_purchase_date`, `crypto_order_status`, `crypto_rolling_45d_volume`, `last_transaction_value_brl` |
| `etl.br__contract.crypto_accounts__crypto_accounts` | `crypto_account__status` | `account_status` |
| `etl.br__contract.crypto_broker__customer_fees` | `customer_fee__rate`, `customer_fee__type` | `crypto_operation_fee`, `crypto_spread_value` |
| `etl.br__contract.mithlond__deposits` / `mithlond__withdrawals` | `status`, `asset`, `amount`, `network` | `crypto_transaction_status`, `crypto_asset_ticker`, `transfer_value`, `transfer_network` |
| `etl.br__contract.crypto_rewards__rewards` | `reward__total_amount`, `reward__status` | `usdc_total_reward_value` |
| `etl.br__dataset.cross_product_customer_activity_v1` | `active_ultraviolet_customer_in_month` | `is_uv_customer` |

### PJ
| Tabela | Campos-chave | Metadados |
|--------|-------------|-----------|
| `etl.br__dataset.investments_liquid_deposit_custody` | `custody_value`, `date` | `has_active_boxes`, `box_total_balance`, `box_creation_date` |
| `etl.br__contract.spi_dict_client__entries` | `entry__activation_date`, `entry__deleted_at`, `entry__type` | `is_pix_key_active`, `pix_key_type`, `pix_key_deletion_date` |
| `etl.br__contract.payhop_client__contracts` + `payhop_client__contract_receivables` | `contract__status`, `contract__created_at`, `contract_receivable__payment_provider` | `has_receivables_contract`, `receivables_contract_number`, `receivables_institution_name` |

## Erros comuns

1. **Join duplicando linhas** — Sempre garantir 1 linha por `customer__id` antes do LEFT JOIN (usar `groupBy` ou `row_number`)
2. **Campos sem respeitar janela temporal** — Metadados de rede/status devem ser `null` se o cliente não era elegível na data do ticket
3. **Status temporal de portabilidade/cancelamento** — Se `finished_at > ticket_stop_time`, o status final (COMPLETED, CANCELLED) NÃO se aplica ao momento do ticket. Usar status inferido (SCHEDULED se tem data agendada, WAITING_SMS caso contrário)
4. **Data prevista quando não existe no ETL** — Calcular com base em regra da squad (ex: 6 dias úteis após criação). Fórmula para dias úteis precisa pular sábados e domingos
5. **Prefixos em valores de status** — Ex: `delivery_status__delivered` ao invés de `delivered`. Verificar valores reais com `DESCRIBE` ou sample
6. **Metadados da squad vs regras reais** — Seção "Variáveis" do Doc pode estar desalinhada. Sempre priorizar "Regras de avaliação"
7. **Metadados duplicados** — Se dois metadados retornam a mesma informação (ex: `plan_status` e `line_status`), manter apenas um
8. **AJIS** — Já existe na Onda 1, não recriar
9. **Seção Save** — NÃO incluir seção de salvar nos notebooks. Será adicionada conforme necessidade separadamente

## Databricks CLI

Usar workspace e profile coletados no onboarding (Passo 2).

- Import: `databricks workspace import <workspace_path>/<nome> --file <local_path> --format SOURCE --language SCALA --overwrite --profile <profile>`
- Export: `databricks workspace export <workspace_path>/<nome> --format SOURCE --file <local_path> --profile <profile>`
- Descoberta de tabelas: `databricks experimental aitools tools discover-schema` e `databricks experimental aitools tools query`
