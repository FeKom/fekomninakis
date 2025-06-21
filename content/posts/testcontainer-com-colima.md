---
title: "Testcontainer Com Colima"
date: 2025-06-20T17:10:08-03:00
draft: false
toc: false
images:
tags:
  - DevOps
  - SRE
  - Container
  - Testes
---

## Como corrigir falhas de conexão com o Ryuk ao usar Testcontainers com Colima no macOS

Recentemente, ao configurar testes de integração utilizando a biblioteca **Testcontainers Java** em um ambiente Docker gerenciado com **Colima** no macOS, me deparei com um problema que causava falhas na execução dos testes. O erro estava relacionado à falha no início do container **Ryuk**, responsável por gerenciar a remoção automática dos containers criados durante os testes.

Durante a execução dos testes, o log exibia as seguintes mensagens:

```bash
: Creating container for image: testcontainers/ryuk:0.5.1
2025-06-19T19:31:40.964-03:00 INFO 39602 --- [catalog] [ main] tc.testcontainers/ryuk:0.5.1 : Container testcontainers/ryuk:0.5.1 is starting: ...
2025-06-19T19:32:41.628-03:00 ERROR 39602 --- [catalog] [ main] tc.testcontainers/ryuk:0.5.1 : Could not start container

java.lang.IllegalStateException: Wait strategy failed. Container is removed
```

O container era criado, mas logo em seguida era removido antes de completar sua inicialização. Como consequência, os testes não conseguiam avançar, e os recursos temporários criados pelo Testcontainers não eram gerenciados corretamente.

## Diagnóstico

O ponto-chave do problema era que a variável de ambiente `TESTCONTAINERS_HOST_OVERRIDE` **não estava configurada**. Sem essa configuração, o Testcontainers tentava inferir automaticamente o endereço do host onde os containers estavam expostos, o que, no caso do Colima, não funcionava corretamente.

Através de pesquisas em fontes confiáveis, foi possível entender o comportamento e identificar a causa raiz. Os seguintes links foram fundamentais para a investigação:

- Stack Overflow: [https://stackoverflow.com/questions/78149093](https://stackoverflow.com/questions/78149093)
- GitHub: [https://github.com/testcontainers/testcontainers-java/issues/6450](https://github.com/testcontainers/testcontainers-java/issues/6450)
- Documentação oficial: [https://java.testcontainers.org/features/configuration/](https://java.testcontainers.org/features/configuration/)

Nesses materiais, é explicado que, quando o Docker é executado em uma máquina virtual (como é o caso do Colima), o Testcontainers não consegue descobrir automaticamente o endereço correto para acesso aos containers, e portanto a variável `TESTCONTAINERS_HOST_OVERRIDE` precisa ser definida manualmente.

## Solução

Configurar a variável de ambiente `TESTCONTAINERS_HOST_OVERRIDE` para `localhost`.

Isso porque o Colima redireciona automaticamente as portas expostas dos containers para o `localhost` do sistema host (macOS). Sendo assim, o acesso aos serviços dos containers deve ser feito por meio de `localhost`, e não diretamente pelo IP interno da VM.

### Exemplo de configuração no terminal

```bash
export TESTCONTAINERS_HOST_OVERRIDE=localhost
```

A partir do momento em que essa variável foi configurada corretamente, os testes passaram a funcionar normalmente, e o container Ryuk pôde ser iniciado com sucesso, cumprindo seu papel na limpeza dos recursos de teste.

## Considerações finais

Quando se utiliza Colima no macOS com Testcontainers, é essencial configurar corretamente o endereço de rede que será usado para acessar os containers. Como o Colima utiliza uma máquina virtual para executar o Docker, o mapeamento de portas para o host é feito via ```localhost```. Dessa forma, o Testcontainers não consegue inferir automaticamente o endereço correto, e a variável ```TESTCONTAINERS_HOST_OVERRIDE``` precisa ser definida explicitamente.

Essa simples configuração garante a compatibilidade entre o Testcontainers e o Colima, evitando falhas de inicialização de containers e garantindo a execução confiável dos testes


## Referências:

- Stack Overflow: https://stackoverflow.com/questions/78149093

- GitHub Issues: https://github.com/testcontainers/testcontainers-java/issues/6450

- Documentação oficial do Testcontainers Java: https://java.testcontainers.org/features/configuration/

