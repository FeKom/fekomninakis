---
title: "Como construí um assistente nutricional com IA generativa do zero"
date: 2026-04-09T10:00:00-03:00
draft: false
toc: true
images:
tags:
  - ia
  - machine-learning
  - typescript
  - python
---

Passei 4 meses construindo sozinho o **Nutria**, um assistente nutricional com IA generativa. O projeto combina um agente conversacional, busca semântica por alimentos, cálculo de macros, geração de planos alimentares e um pipeline de avaliação para medir a qualidade das respostas. Nesse post vou detalhar cada decisão técnica — o que funcionou, o que errei e o que faria diferente.

## A arquitetura geral

O sistema tem três serviços que se comunicam:

```
Frontend (Next.js)
    └── Backend — agente Mastra (Node.js + TypeScript)
            └── Catalog API — dados nutricionais + busca semântica (FastAPI + Python)
                        └── PostgreSQL (pgvector + pg_trgm)
```

O **backend** é a camada de IA: gerencia o agente, memória, e decide quais ferramentas usar em cada turno. O **catalog** é onde vivem os dados de alimentos, os embeddings e toda a lógica de ML — basicamente um serviço Python especializado que o agente consome via HTTP. Toda a parte de **Machine Learning, NLP e LLMOps** está no catalog, feita em Python. O backend em TypeScript foca em orquestração.

## O Catalog: Python, embeddings e busca híbrida

### O modelo de embeddings

O catalog usa `intfloat/multilingual-e5-small` via `SentenceTransformer` para gerar os embeddings dos alimentos. O E5 usa prefixos assimétricos — uma decisão de design importante que eu não conhecia antes desse projeto:

```python
# Alimentos indexados com "passage:" — representa um documento
embedding_alimento = model.encode("passage: frango grelhado proteína")

# Buscas usam "query:" — representa uma pergunta/consulta
embedding_busca = model.encode("query: frango")
```

Sem isso, a busca semântica perde qualidade. Os vetores têm 384 dimensões e são armazenados no PostgreSQL via **pgvector**.

### Normalização da query antes do embedding

Antes de gerar o embedding de uma busca, o sistema limpa a query removendo quantidades e métodos de preparo. Isso foi um dos primeiros erros que cometi: sem normalização, "frango grelhado 100g" gerava um embedding diferente de "frango", e a busca não encontrava o alimento certo.

```python
_COOKING_METHODS = re.compile(
    r"\b(grelhad[oa]s?|cozid[oa]s?|fritad[oa]s?|cru[a]?s?|grilled|boiled|raw)\b",
    re.IGNORECASE,
)
_QUANTITIES = re.compile(
    r"\b\d+[\.,]?\d*\s*(g|kg|ml|kcal|cal|colher[es]*)\b",
    re.IGNORECASE,
)

def _normalize_food_query(query: str) -> str:
    normalized = _QUANTITIES.sub("", query)
    normalized = _COOKING_METHODS.sub("", normalized)
    return " ".join(normalized.split()).strip() or query
```

Resultado: `"frango grelhado 100g"` → `"frango"`. Simples, mas fez diferença nos scores de retrieval.

### Busca híbrida: pgvector + pg_trgm

Só busca vetorial não é suficiente para nomes de alimentos. O vetor captura semântica, mas pode errar em nomes curtos ou muito específicos. A solução foi combinar **pgvector** (similaridade de cosseno) com **pg_trgm** (similaridade por trigrama) num score híbrido:

```python
# Busca vetorial via pgvector
statement = select(
    Food,
    Food.embedding.cosine_distance(query_embedding).label("distance")
).order_by("distance").limit(fetch_limit)

candidates = session.exec(statement).all()

# Busca léxica via pg_trgm para os candidatos
trgm_rows = session.execute(
    sa_text("SELECT id, similarity(name, :q) AS trgm FROM foods WHERE id = ANY(:ids)"),
    {"q": query_text, "ids": food_ids},
).fetchall()

# Score híbrido: 85% vetor + 15% texto
for food, distance in candidates:
    vec_score  = round(1 - float(distance), 4)
    text_score = trgm_by_id.get(food.id, 0.0)
    hybrid     = round(0.85 * vec_score + 0.15 * text_score, 4)
```

O pg_trgm divide palavras em trigramas (ex: "fran" → `fra`, `ran`, `ang`) e calcula a sobreposição entre query e nome do alimento. Não requer embedding — é puramente textual e muito rápido.

### Por que os alimentos não precisam de chunking

Um conceito importante em RAG é o **chunking** — dividir documentos longos em pedaços menores antes de indexar. No Nutria, isso não é necessário: cada alimento já é um registro atômico no banco, com nome e categoria. É o "chunk perfeito" por natureza. Chunking só faz sentido para documentos longos como PDFs.

## O Backend: Mastra e o agente

### Por que Mastra

O **Mastra** é um framework TypeScript para construção de agentes. Ele cuida do loop do agente (decidir qual ferramenta usar, executar, voltar para o LLM), gerencia a memória entre conversas e oferece uma interface de desenvolvimento via Mastra Studio em `localhost:4111`. Isso elimina muito boilerplate.

```typescript
export const nutritionAnalystAgent = new Agent({
  id: "nutrition-analyst",
  model: env.MODEL,                          // GitHub Models
  instructions: loadNutritionAnalystInstructions(),
  memory: createNutritionMemory(),
  inputProcessors: [toolInjectorProcessor],  // injeção de ferramentas por intenção
  tools: {
    add_goal: addGoalTool,
    add_activity: addActivityTool,
    save_recipe: suggestRecipeTool,
  },
});
```

### O problema de dar ferramentas demais para um agente

Esse foi o aprendizado mais caro do projeto. Com 18+ ferramentas disponíveis, o agente ficava confuso: chamava ferramentas irrelevantes, usava mais tokens do que o necessário e às vezes entrava em loop. A solução foi **não passar todas as ferramentas de uma vez** — injetar apenas as relevantes para a intenção do usuário em cada turno.

### ToolInjectorProcessor: intenção antes do LLM

Criei um `ToolInjectorProcessor` que classifica a intenção do usuário **antes** do primeiro call para o LLM. Assim o agente já recebe só as ferramentas certas:

```typescript
// 1. Regex fast-path — cobre ~90% dos casos em português
const INTENT_PATTERNS = [
  { intent: "log_meal",    pattern: /\b(comi|registrei|almoç[ao]|jantar|lanche[i]?)\b/i },
  { intent: "meal_plan",   pattern: /\b(plano alimentar|cardápio|criar plano)\b/i },
  { intent: "search_food", pattern: /\b(calorias d[oe]|valor nutricional|macros)\b/i },
  // ...
];

// 2. Fallback com modelo pequeno para casos ambíguos
async function detectIntentByModel(text: string): Promise<Intent> {
  const { text: raw } = await generateText({
    model: github("Phi-4-mini"),
    system: "Reply with ONLY ONE WORD: search_food, log_meal, meal_plan, ...",
    prompt: text,
    maxOutputTokens: 10,
  });
  return VALID_INTENTS.includes(raw.trim()) ? raw.trim() : "general";
}
```

O Phi-4-mini classifica a intenção em menos de 200ms e custa praticamente nada em tokens. O mapa de intenção → ferramentas garante que o agente de busca de comida nunca vê as ferramentas de plano alimentar, e vice-versa.

### Memória híbrida

O agente precisa lembrar de conversas anteriores, mas o limite de tokens é real. A memória tem três camadas:

| Camada | O que faz | Por quê |
|---|---|---|
| **Message history** | Últimas 5 mensagens | Contexto imediato da conversa |
| **Semantic recall** | Busca no histórico antigo via pgvector (HNSW) | Lembra de conversas de semanas atrás |
| **Working memory** | Template markdown preenchido pelo agente | Preferências, ajustes temporários, insights |

O working memory usa markdown em vez de JSON por um motivo prático: o LLM atualiza esse campo em linguagem natural — um JSON com vírgula faltando silenciosamente corrompe o objeto inteiro, enquanto markdown é tolerante a escrita parcial.

```typescript
template: `# Session Context
- Current focus:
- Last meal plan discussed:

# Discovered Preferences
- Likes:
- Dislikes:

# Temporary Adjustments
- Active adjustment:
- Valid until:

# Nutrition Insights
- Observation:
`
```

Sem template, o agente ficava reescrevendo a working memory com conteúdo vazio a cada turno — sem estrutura, ele não sabia o que preencher.

## O sistema de evals

Medir qualidade de agente é difícil. A resposta parece certa, mas será que o retrieval trouxe o contexto certo? O agente está alucinando? Escrevi um sistema de evals do zero para responder isso de forma sistemática.

### Como funciona

O fluxo é orquestrado pelo catalog (Python) e o agente roda no Mastra (TypeScript):

```
Catalog lê golden_dataset.json
  └── Para cada pergunta:
        ├── Chama Mastra /eval/run com { prompt, question, retrieval_source }
        ├── Mastra busca chunks, monta contexto, chama o LLM
        ├── Devolve { answer, context_chunks, latency_ms }
        └── Catalog calcula os scores e salva no banco
```

### Golden dataset

Um arquivo fixo com perguntas e respostas esperadas escritas à mão. É a referência que nunca muda entre experimentos — só o pipeline muda.

```json
[
  {
    "question": "Quantas calorias tem 100g de peito de frango?",
    "ground_truth": "Peito de frango sem pele tem aproximadamente 165 calorias por 100g."
  }
]
```

### As métricas: similaridade de cosseno, sem LLM como juiz

Todos os scores são calculados matematicamente via **similaridade de cosseno entre embeddings** — sem LLM-as-judge, sem custo extra de tokens.

```
Q = embedding(pergunta)
A = embedding(resposta do agente)
C = embedding(contexto completo — todos os chunks concatenados)
E = embedding(resposta esperada)
```

| Métrica | Fórmula | O que mede |
|---|---|---|
| `faithfulness` | `cos(A, C)` | O agente usou o contexto ou alucionou? |
| `answer_relevancy` | `cos(Q, A)` | A resposta respondeu a pergunta? |
| `context_relevancy` | `cos(Q, C)` | O contexto recuperado era sobre o assunto? |
| `context_recall` | `cos(E, C)` | O contexto tinha o que era necessário? |
| `context_precision` | `média(cos(Q, Cᵢ))` por chunk | Os chunks mais úteis estavam no topo? |

O overall score pondera as métricas por importância:

```python
overall = (faithfulness      * 0.30 +
           answer_relevancy  * 0.25 +
           context_relevancy * 0.20 +
           context_recall    * 0.15 +
           context_precision * 0.10)
```

### Comparando experimentos no Jupyter

Cada variação de prompt, modelo ou estratégia de retrieval vira um experimento nomeado. Os resultados ficam salvos no banco e o Jupyter Notebook plota os scores lado a lado — fácil de ver se uma mudança melhorou ou piorou a qualidade do agente.

| Sintoma | Causa provável |
|---|---|
| `context_recall` alto + `faithfulness` baixo | Recupera bem mas alucina na resposta |
| `answer_relevancy` baixo em todos | Problema no prompt, não no RAG |
| `context_precision` ≈ 0 | Dataset não ingerido ou retrieval quebrado |

## GitHub Models: tudo de graça

O LLM principal e o Phi-4-mini para classificação de intenção rodam via **GitHub Models** — gratuito, sem cartão de crédito. Basta gerar um token com permissão de leitura em `Models` e configurar duas variáveis de ambiente:

```bash
# apps/backend/.env
GITHUB_TOKEN=seu-token-aqui
MODEL=github-models/openai/gpt-4.1-mini
```

Em desenvolvimento, use sempre um modelo `mini` ou `nano` para preservar a cota gratuita.

## O que faria diferente

**Começar os evals mais cedo.** Passei semanas ajustando prompts no feeling, sem métrica nenhuma. Quando montei o sistema de evals, descobri que mudanças que pareciam melhorar na conversa às vezes pioravam o faithfulness.

**Definir a intenção antes de escolher ferramentas.** O `ToolInjectorProcessor` deveria ter sido a primeira coisa que eu implementei — não a última. Dar todas as ferramentas ao agente desde o início gerou muito ruído nos primeiros experimentos.

**Testar o pipeline de retrieval no Jupyter antes de integrar.** Prototipei a busca híbrida diretamente no notebook, vendo os scores de similaridade em tempo real. Só depois portei para o serviço Python. Isso economizou horas de debug.

---

O código é aberto e você já consegue rodar localmente com mais de 600 alimentos na base. Nas próximas semanas vou detalhar cada parte separadamente — embeddings, evals e a arquitetura do agente com Mastra.
