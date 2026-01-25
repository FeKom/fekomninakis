---
title: "Embeddings e Similaridade Semântica"
date: 2026-01-22T10:00:00-03:00
draft: false
toc: false
images:
tags:
  - ia
  - machine-learning
  - python
---

Imagina que você pudesse transformar qualquer texto em um ponto no espaço, onde textos com significados parecidos ficam próximos uns dos outros. É exatamente isso que embeddings fazem! Em vez de buscar por palavras-chave exatas (que todo mundo sabe que falha miseravelmente), usamos vetores numéricos que capturam o significado semântico do texto. É como dar coordenadas GPS para as ideias.

O fluxo é bem direto: você pega um texto, passa por um modelo de embedding, e ele cospe um vetor com centenas de dimensões. A mágica acontece quando você compara esses vetores usando similaridade de cosseno - quanto mais próximo de 1, mais parecidos são os textos. Mas nem todo modelo é igual, e escolher o certo faz toda a diferença no seu projeto.

## Modelos de Embedding: Qual Escolher?

Aqui é onde muita gente se perde. Sentence-BERT não é um modelo específico, é uma arquitetura - um método para gerar embeddings de sentenças inteiras. Os modelos específicos são implementações dessa arquitetura com pesos diferentes, treinados para casos de uso distintos. Pensa assim: Sentence-BERT é a receita, os modelos são os pratos prontos.

| Modelo | Dimensões | Quando Usar |
|--------|-----------|-------------|
| `all-MiniLM-L6-v2` | 384 | O coringa - rápido e preciso o suficiente para a maioria dos casos |
| `all-mpnet-base-v2` | 768 | Quando precisa de mais precisão e tem recursos sobrando |
| `paraphrase-multilingual-MiniLM-L12-v2` | 384 | Textos em português ou múltiplos idiomas |
| OpenAI `text-embedding-ada-002` | 1536 | Projetos que já usam a API da OpenAI |

O `all-MiniLM-L6-v2` é o queridinho da comunidade por um bom motivo: ele tem apenas 6 camadas (o L6 no nome), o que significa inferência rápida, mas ainda mantém uma qualidade muito boa. É perfeito para prototipagem e produção com volume moderado. Já o `all-mpnet-base-v2` usa a arquitetura MPNet que é mais robusta - ele captura melhor nuances semânticas, mas vai consumir mais CPU/GPU e memória.

Para quem trabalha com português, o `paraphrase-multilingual-MiniLM-L12-v2` é essencial. Ele foi treinado em mais de 50 idiomas e consegue entender que "cachorro" e "dog" são semanticamente próximos. As 12 camadas (L12) dão mais capacidade de representação comparado ao L6, mantendo um tamanho gerenciável.

```python
from sentence_transformers import SentenceTransformer
from sklearn.metrics.pairwise import cosine_similarity

# Para português e multilíngue
model = SentenceTransformer('paraphrase-multilingual-MiniLM-L12-v2')

textos = ["O gato preto corre", "Um felino escuro corre rapidamente"]
embeddings = model.encode(textos)

similaridade = cosine_similarity([embeddings[0]], [embeddings[1]])[0][0]
print(f"Similaridade: {similaridade:.3f}")  # ~0.85
```

Um detalhe importante: as dimensões do vetor são fixas por modelo. O MiniLM sempre vai gerar 384 dimensões, o MPNet sempre 768. Não dá para mudar isso sem retreinar o modelo do zero. Então se você começou um projeto com MiniLM e quer migrar para MPNet, vai precisar regerar todos os embeddings do seu banco.

## Índices Vetoriais: HNSW vs IVFFlat

Quando você precisa fazer busca semântica em escala (milhões de registros), entra o pgvector no PostgreSQL com índices especializados. Sem índice, o banco precisa calcular a distância para cada registro - O(n) que fica inviável rápido.

O **HNSW** (Hierarchical Navigable Small World) é baseado em grafos. Imagina uma rede social onde cada vetor é conectado aos seus "amigos" mais próximos, organizados em camadas hierárquicas. A busca navega por essas conexões até encontrar os vizinhos mais similares. É muito rápido (O(log n)), mas consome mais memória porque precisa manter toda essa estrutura de grafo.

O **IVFFlat** (Inverted File with Flat) divide o espaço vetorial em clusters usando k-means. Na busca, primeiro identifica quais clusters são candidatos, depois faz busca exata só dentro deles. É mais leve em memória e a construção é mais rápida, mas a busca é um pouco mais lenta e precisa de reconstrução quando os dados mudam muito.

| Característica | HNSW | IVFFlat |
|----------------|------|---------|
| Velocidade de busca | Muito rápida | Rápida |
| Tempo de construção | Lento | Rápido |
| Uso de memória | Alto | Moderado |
| Dados dinâmicos | Suporta bem | Precisa reconstruir |

```sql
-- HNSW para dados que mudam frequentemente
CREATE INDEX ON documentos USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- IVFFlat para dados mais estáticos
CREATE INDEX ON documentos USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Busca por similaridade
SELECT nome, 1 - (embedding <=> query_embedding) as similaridade
FROM documentos
ORDER BY embedding <=> query_embedding
LIMIT 10;
```

A regra prática: se seus dados são atualizados frequentemente (como um sistema de busca de produtos), vai de HNSW. Se são mais estáticos (como um corpus de documentos legais), IVFFlat vai te dar resultados similares com menos recursos.
