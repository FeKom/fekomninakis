---
title: "Containers E Linux"
date: 2025-05-22T15:47:11-03:00
draft: false
toc: false
images:
tags:
  - Docker
  - Linux
  - OS
---
## O que são containers e Porque não se pode criar um no macOS?

A criação de contêineres como conhecemos, popularizados pelo Docker, depende fundamentalmente de características específicas do kernel Linux. Duas tecnologias centrais, cgroups e namespaces, são os pilares que permitem o isolamento eficiente e leve de processos, rede e sistema de arquivos.

O cgroups (control groups) possibilita o gerenciamento e a limitação dos recursos do sistema, como CPU, memória e I/O de disco, para grupos de processos. Já os namespaces fornecem o isolamento, fazendo com que um processo dentro de um contêiner enxergue seu próprio ambiente, separado do sistema operacional hospedeiro e de outros contêineres (Containers não possuem kernel).

No entanto, o macOS opera sobre um kernel diferente, o XNU (ou Darwin). Este kernel possui uma arquitetura distinta e não implementa as primitivas de cgroups e namespaces da mesma forma que o kernel Linux. As ferramentas de isolamento nativas do macOS, como o sandboxing de aplicativos, servem a um propósito diferente, focando na segurança de aplicativos e na restrição de acesso a recursos, como a utilização de camera, microfone e de protocolos HTTP, mas não replicam a capacidade de contêineres em nível de sistema operacional que o Linux oferece.

Portanto, quando o Docker e outras plataformas de contêineres são utilizados no macOS, eles empregam uma máquina virtual Linux subjacente. Esta VM roda um sistema operacional Linux leve em segundo plano, e é dentro dela que o Docker Engine opera, utilizando os cgroups e namespaces do kernel Linux da VM para criar e gerenciar os contêineres. A interface do Docker Desktop no macOS serve como uma ponte, direcionando os comandos do usuário para o ambiente Linux virtualizado.

Conclusão: a natureza do kernel do sistema operacional define a possibilidade de operar contêineres de forma nativa. Para criar e manipular contêineres utilizando cgroups e namespaces diretamente.
