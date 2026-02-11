---
title: "GraalVM, Native Image e como o Quarkus se beneficia disso"
date: 2026-02-11T10:00:00-03:00
draft: false
toc: false
images:
tags:
  - java
  - graalvm
  - quarkus
  - cloud-native
---

Se você trabalha com Java e nunca ouviu falar de GraalVM, vale parar pra entender. Criada pela Oracle em 2018, a GraalVM é uma distribuição da máquina virtual Java com uma premissa clara: ser cloud-native, rápida no startup e ainda por cima poliglota. Ela permite rodar Python, Ruby, JavaScript, R e outras linguagens baseadas na LLVM na mesma JVM, graças ao **Truffle Language Implementation Framework** — que faz interpretadores de diferentes linguagens compartilharem dados sem custo adicional de chamadas entre linguagens.

## JIT vs AOT: o que muda na prática?

A maioria das JVMs usa compilação **JIT (Just-In-Time)**. O fluxo é: o bytecode Java (ou Kotlin, Scala, Clojure) é interpretado inicialmente com otimizações mínimas. O compilador JIT monitora a execução, identifica os métodos mais usados ("hot spots") e compila esses trechos em código de máquina nativo otimizado. O problema? Toda vez que a aplicação inicia, ela precisa carregar dependências, bibliotecas e classes novamente — aquele startup lento que todo dev Java conhece.

A GraalVM resolve isso com compilação **AOT (Ahead-Of-Time)** através do **Native Image Builder**. Em vez de compilar em runtime, todo o código — classes, dependências, bibliotecas — é transformado em um binário nativo antes da execução. Quando você inicia a aplicação, não há interpretação de bytecode nem compilação JIT acontecendo. É como a diferença entre traduzir um livro inteiro antes de ler (AOT) versus traduzir cada página conforme vai lendo (JIT).

```bash
# Compilando uma aplicação Java para binário nativo
native-image -jar minha-app.jar -o minha-app

# O resultado é um executável nativo
./minha-app  # Startup em milissegundos
```

O trade-off: o tempo de build com AOT é maior, mas o startup e o consumo de memória caem drasticamente — estamos falando de milissegundos de startup contra segundos (ou minutos) no JIT.

## Como o Quarkus se beneficia do Native Image

O Quarkus foi desenhado desde o início para tirar proveito máximo do GraalVM Native Image. Enquanto frameworks tradicionais como Spring fazem muito trabalho em runtime — scanning de classpath, criação de proxies, injeção de dependência via reflection — o Quarkus move tudo isso para **build time**. Ele resolve injeção de dependência, configuração e scanning de anotações durante a compilação, não durante a execução.

```bash
# Criando um projeto Quarkus com suporte a native
mvn io.quarkus:quarkus-maven-plugin:create \
    -DprojectGroupId=com.exemplo \
    -DprojectArtifactId=minha-api

# Build nativo (requer GraalVM instalado)
./mvnw package -Dnative

# Resultado: binário nativo com startup ~0.02s
./target/minha-api-1.0-runner
```

Na prática, uma API REST em Quarkus compilada com Native Image sobe em **~20ms** e consome **~30MB de RAM**. A mesma aplicação rodando em JVM tradicional leva segundos e consome centenas de MB. Para ambientes como Kubernetes e serverless (AWS Lambda, Cloud Run), onde cada milissegundo de cold start e cada MB de memória contam, essa diferença é enorme.

O Quarkus ainda oferece um modo dev com hot reload (`quarkus dev`) que roda em JVM normal para desenvolvimento, e o build nativo fica só para produção — o melhor dos dois mundos.
