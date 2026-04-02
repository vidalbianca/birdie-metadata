# Birdie Metadata Builder

Assistente de IA para construção de metadados da Birdie (monitoria de qualidade via IA). Guia você por todo o processo: leitura dos critérios, identificação de tabelas ETL, construção e importação dos notebooks no Databricks.

---

## Primeiro passo: clonar o repositório

```bash
git clone git@github.com:vidalbianca/birdie-metadata.git
cd birdie-metadata
```

Abra o diretório no **Cursor** ou **Claude Code** e inicie uma conversa com:
> "quero criar metadados"

---

## Pré-requisitos

### 1. Cursor IDE
- Instale em [cursor.com](https://www.cursor.com/)

### 2. Databricks CLI
```bash
# Instalar
brew install databricks

# Configurar (usar o host do Nubank)
databricks configure --profile ops_analytics
# Host: https://nubank-e2-general.cloud.databricks.com
# Token: seu token pessoal do Databricks
```

Verifique com:
```bash
databricks --version
```

### 3. MCP Google Workspace (opcional, mas recomendado)

Permite ler critérios direto do Google Docs, sem precisar baixar arquivos.

**Setup:**
1. Obtenha o servidor MCP de Google Workspace (peça ao time de Quality)
2. Adicione ao seu `~/.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "google-workspace": {
      "command": "node",
      "args": [
        "/caminho/para/workspace-server/dist/index.js"
      ],
      "env": {}
    }
  }
}
```
3. Autentique com sua conta Google quando solicitado

**Sem o MCP Google?** Sem problemas — você pode salvar os critérios como `.txt` e o assistente lê normalmente.

---

## Como usar

### No Cursor

1. Abra este diretório no Cursor
2. Inicie uma nova conversa (Cmd+L ou Cmd+I)
3. Diga: **"quero criar metadados"**

### No Claude Code

1. Abra o terminal neste diretório
2. Edite o `.mcp.json` com o caminho correto do seu workspace-server (se for usar Google Docs)
3. Execute: `claude`
4. Diga: **"quero criar metadados"**

### O que o assistente faz

- Verifica se seus pré-requisitos estão ok
- Pergunta qual squad e onde estão os critérios
- Lê os critérios (Google Docs ou `.txt`)
- Propõe os metadados para sua validação
- Constrói o notebook Scala/Spark
- Importa no seu workspace do Databricks

---

## Atualizando a base de conhecimento

O arquivo `.cursor/rules/birdie-metadata.mdc` é a base de conhecimento do assistente. Ele contém: como ler critérios, padrão dos notebooks, catálogo de tabelas ETL, erros comuns, etc.

**Quando atualizar?**
- Nova squad validada (adicionar tabelas e metadados ao catálogo)
- Erro novo descoberto durante validação
- Mudança de padrão ou convenção
- Nova tabela ETL descoberta

**Como atualizar?**

Opção 1 — Pedir ao assistente durante a conversa:
> "adiciona no Rule que a tabela X serve para Y"

Opção 2 — Editar o arquivo `.cursor/rules/birdie-metadata.mdc` diretamente no Cursor.

**Importante: sempre compartilhe a atualização com o time depois de editar.**

```bash
git add . && git commit -m "atualiza Rule: <descreva o que mudou>" && git push
```

Ou peça ao assistente: *"faz commit e push das alterações no Rule"*.

**Antes de começar a trabalhar**, garanta que você tem a versão mais recente:

```bash
git pull
```

---

## Estrutura

```
birdie-metadata/
  .cursor/rules/birdie-metadata.mdc   ← Base de conhecimento (Cursor)
  CLAUDE.md                            ← Base de conhecimento (Claude Code)
  .mcp.json                            ← Config MCP Google Workspace (Claude Code)
  README.md                            ← Este arquivo
```

---

## Dúvidas?

Fale com o time de Quality Analytics.
