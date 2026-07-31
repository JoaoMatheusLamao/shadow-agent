# Contrato — Base de Conhecimento (Excel em `docs/`)

Fonte da base de conhecimento no MVP. Um ou mais arquivos `.xlsx` na pasta `docs/`,
carregados em memória no startup e indexados com BM25 (D5). **Sem pgvector.**

## Formato esperado da planilha

Cada **linha** = um `KnowledgeEntry`. Colunas (nomes case-insensitive; aceitar
sinônimos PT):

| Coluna | Obrigatória | Sinônimos aceitos | Uso |
|---|---|---|---|
| pergunta | sim* | `titulo`, `title`, `question` | texto indexado no BM25 |
| resposta | sim | `conteudo`, `content`, `answer` | fundamentação da resposta |
| fonte | não | `source`, `tag`, `categoria` | citação/auditoria (`source_ref`) |

\* pelo menos uma coluna de texto pesquisável deve existir; se só houver `resposta`,
indexa-se `resposta`.

### Exemplo

| pergunta | resposta | fonte |
|---|---|---|
| Qual o prazo de entrega? | O prazo padrão é de 5 a 7 dias úteis. | logistica |
| Como cancelo o plano anual? | Acesse Conta → Assinatura → Cancelar. Sem multa após 12 meses. | faturamento |

## Regras de carga

- Ignora linhas totalmente vazias; faz `strip` nas células.
- Múltiplas abas: todas as abas são lidas (a aba vira parte de `source_ref`).
- Erro de leitura (arquivo ausente/corrompido) = **erro de infra** → a consulta falha,
  dispara retry/registro em `failed_messages` (não é “sem conhecimento”).
- Recarga: no boot; opcionalmente por mtime do arquivo (config).

## Recuperação

- Tokeniza a mensagem do cliente e ranqueia entries por BM25 sobre a coluna de texto.
- Retorna top-K (`KNOWLEDGE_TOP_K`, default 4) com score ≥ `KNOWLEDGE_MIN_SCORE`.
- Nenhum entry acima do limiar → `retrieve()` retorna `[]` → escalonamento (D7).

## Configuração (env)

| Var | Default | Descrição |
|---|---|---|
| `KNOWLEDGE_DIR` | `docs/` | pasta dos `.xlsx` |
| `KNOWLEDGE_TOP_K` | `4` | nº de trechos retornados |
| `KNOWLEDGE_MIN_SCORE` | (calibrar) | limiar de relevância p/ não escalar |
