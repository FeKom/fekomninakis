---
title: "RAG - Retrieval Augmented Generation"
date: 2026-01-22T11:00:00-03:00
draft: false
toc: false
images:
tags:
  - ia
  - machine-learning
  - llm
---

Se você já usou ChatGPT ou qualquer LLM, provavelmente percebeu uma limitação frustrante: o modelo não sabe nada sobre seus documentos internos, sua base de conhecimento ou dados atualizados. LLMs só conhecem o que foi usado no treinamento, e ponto final. É aí que entra o RAG (Retrieval-Augmented Generation), uma arquitetura que resolve esse problema de forma elegante e escalável.

A ideia é simples: em vez de esperar que a LLM saiba tudo, você busca informações relevantes antes de fazer a pergunta. Pensa assim - é como se você desse uma "cola" para a LLM antes da prova. O fluxo funciona em cinco passos: recebe a pergunta do usuário, busca informações relevantes numa base local (PDFs, banco de dados, documentos), monta um contexto com o que encontrou, envia contexto + pergunta para a LLM, e ela gera a resposta considerando tudo isso.

```
Usuário → Pergunta → [Busca no Vector DB] → Contexto relevante
                                                    ↓
                              LLM ← Contexto + Pergunta → Resposta
```

## Como funciona na prática?

A maioria dos RAGs usa embeddings e banco de dados vetorial para fazer a busca de contexto. Embeddings transformam texto em vetores numéricos, e a busca por similaridade (cosine similarity) encontra os trechos mais relevantes. Esse é o caminho mais comum quando você tem documentos grandes que mudam frequentemente - PDFs, docs, wikis internas.

```python
# Fluxo típico de um RAG
from sentence_transformers import SentenceTransformer
from chromadb import Client

# 1. Indexar documentos (feito uma vez)
model = SentenceTransformer('all-MiniLM-L6-v2')
chunks = dividir_documento_em_chunks(documento)
embeddings = model.encode(chunks)
vector_db.add(embeddings, chunks)

# 2. Buscar contexto relevante (a cada query)
query_embedding = model.encode(pergunta_usuario)
contexto = vector_db.query(query_embedding, top_k=5)

# 3. Montar prompt e enviar para LLM
prompt = f"Contexto: {contexto}\n\nPergunta: {pergunta_usuario}"
resposta = llm.generate(prompt)
```

## Preciso sempre usar Vector Database?

Não necessariamente. Se sua fonte de verdade é um banco de dados relacional ou uma API, você pode buscar diretamente neles sem precisar de embeddings. O padrão de embeddings + vector database brilha quando você tem documentos não estruturados (PDFs, textos longos) que precisam ser indexados semanticamente. Mas se você só precisa consultar uma tabela de produtos ou buscar em uma API REST, um RAG mais simples resolve - basta fazer a query tradicional e injetar o resultado no prompt.

A escolha da arquitetura depende do seu caso de uso. Documentos vivos e grandes que mudam constantemente pedem vector database. Dados estruturados em banco relacional podem usar SQL direto. O importante é entender que RAG é o padrão, não a implementação específica - você pode adaptar conforme sua necessidade.
