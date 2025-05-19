---
title: "[ELB] Elastic Load Balancing"
date: 2025-05-19T11:00:31-03:00
draft: false
toc: false
images:
tags:
  - infra
  - aws
---

O Elastic Load Balancing (ELB) é um serviço da AWS (Amazon Web Services) que distribui automaticamente o tráfego de entrada de aplicativos por vários destinos, como instâncias EC2, contêineres, endereços IP e funções Lambda. Ele atua como um único ponto de contato para os clientes, aumentando a disponibilidade e a tolerância a falhas de suas aplicações.

Imagine o ELB como um porteiro de um prédio com muitos apartamentos (seus servidores). Em vez de todos os visitantes tentarem entrar pela mesma porta e sobrecarregarem o sistema, o porteiro direciona cada pessoa para um apartamento diferente que esteja disponível. Se um apartamento estiver com problemas (servidor não saudável), o porteiro para de enviar pessoas para lá até que o problema seja resolvido.

Principais benefícios do ELB:

Alta disponibilidade: Distribui o tráfego para evitar que um único servidor fique sobrecarregado e redireciona o tráfego de instâncias não saudáveis para as saudáveis.
Escalabilidade: Lida automaticamente com o aumento ou diminuição do tráfego de aplicativos. Você pode adicionar ou remover instâncias sem interromper o fluxo de solicitações.
Segurança: Oferece recursos como terminação SSL/TLS e integração com o AWS Certificate Manager (ACM) e o AWS WAF (Web Application Firewall).
Flexibilidade: Suporta vários protocolos (HTTP, HTTPS, TCP, UDP, TLS) e diferentes tipos de balanceadores de carga para atender a diversas necessidades.
Monitoramento: Integra-se com o Amazon CloudWatch para fornecer métricas de desempenho em tempo real e logs de acesso.
Existem diferentes tipos de Elastic Load Balancing, otimizados para diferentes tipos de tráfego e necessidades:

Application Load Balancer (ALB): Ideal para tráfego HTTP e HTTPS. Oferece roteamento avançado baseado no conteúdo da solicitação (por exemplo, caminho da URL, cabeçalhos do host). É adequado para microsserviços e aplicações baseadas em contêineres.
Network Load Balancer (NLB): Ideal para tráfego TCP, UDP e TLS onde o desempenho e a baixa latência são cruciais. Opera na camada 4 (Transporte) do modelo OSI e é capaz de lidar com milhões de solicitações por segundo.
Gateway Load Balancer (GWLB): Permite implantar, dimensionar e gerenciar appliances virtuais de terceiros (como firewalls, sistemas de detecção de intrusão e sistemas de inspeção profunda de pacotes) na AWS.
Classic Load Balancer (CLB): É a geração anterior de balanceadores de carga da AWS e está sendo substituído pelos ALBs e NLBs, que oferecem mais recursos e melhor desempenho. A AWS recomenda migrar para os tipos mais recentes.
Em resumo, o Elastic Load Balancing é um componente fundamental para construir aplicações escaláveis, altamente disponíveis e tolerantes a falhas na AWS, garantindo que o tráfego seja distribuído de forma eficiente entre seus servidores.