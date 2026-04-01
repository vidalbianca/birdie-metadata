# Birdie Metadata Builder

Assistente de IA para construção de metadados da Birdie (monitoria de qualidade via IA).

Abra este projeto no **Cursor** e inicie uma conversa com:
> "quero criar metadados"

O assistente irá guiar você por todo o processo: leitura dos critérios, identificação de tabelas ETL, construção e importação dos notebooks no Databricks.

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

1. Abra este diretório no Cursor
2. Inicie uma nova conversa (Cmd+L ou Cmd+I)
3. Diga: **"quero criar metadados"**
4. O assistente irá:
   - Verificar se seus pré-requisitos estão ok
   - Perguntar qual squad e onde estão os critérios
   - Ler os critérios (Google Docs ou `.txt`)
   - Propor os metadados para sua validação
   - Construir o notebook Scala/Spark
   - Importar no seu workspace do Databricks

---

## Estrutura

```
birdie-metadata/
  .cursor/rules/birdie-metadata.mdc   ← Base de conhecimento do assistente
  README.md                            ← Este arquivo
```

---

## Dúvidas?

Fale com o time de Quality Analytics.
