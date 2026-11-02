---
lang: pt
page_id: career_artsvp
permalink: /posts/career-artsvp
title: "ARTSVP: Construindo a Plataforma de Eventos por Trás das Inaugurações do Mundo da Arte"
date: 2024-07-01
categories: [Career, Rails]
tags: [Career, ARTSVP, Ruby on Rails, JavaScript, PostgreSQL, DevOps, Security]
author: Angel
description: Engenharia full stack na ARTSVP, a plataforma de eventos e reservas para galerias, feiras de arte e instituições culturais, de funcionalidades de produto a infraestrutura, desempenho e segurança.
image:
  path: /assets/img/career-artsvp.png
  alt: Logotipo da ARTSVP
---

## ARTSVP: Construindo a Plataforma de Eventos por Trás das Inaugurações do Mundo da Arte

**Julho de 2024 até o presente · ARTSVP · Full Stack Engineer · Remoto Reino Unido/UE**

A [ARTSVP](https://artsvp.com/) cuida do lado dos convidados no mundo da arte: convites, confirmações de presença, reservas, venda de ingressos e check-in para galerias, feiras de arte e instituições culturais. Quando uma galeria inaugura uma exposição ou uma feira de arte abre sua pré-estreia VIP, a ARTSVP decide quem entra, quando e com que fluidez.

Entrei em julho de 2024 como Full Stack Engineer. Antes disso, passei alguns meses na Emma Sleep mantendo microsserviços em Go e componentes SaaS em Java da plataforma de e-commerce deles, depois de [dois anos e meio na Exodus](/posts/career-exodus).

| | |
|---|---|
| **Cargo** | Full Stack Engineer |
| **Desde** | 07/2024 |
| **Stack** | Ruby on Rails, Hotwire e Stimulus, JavaScript, PostgreSQL, Sidekiq, Stripe |
| **Áreas** | Funcionalidades de produto, desempenho de banco de dados, DevOps, monitoramento, code review, segurança |

---

## O que a plataforma faz

O produto parece simples por fora e não é por baixo:

- **Eventos e reservas**, incluindo horários marcados, capacidade e listas de espera.
- **Convites e listas de convidados** que as instituições curam com cuidado e que precisam permanecer privadas.
- **Portais de expositores** que permitem às galerias de uma feira de arte gerenciar seus próprios convidados dentro do evento da feira.
- **Pagamentos** de eventos com ingresso, via Stripe.
- **Check-in por QR code** na porta, que precisa funcionar na noite de abertura com fila lá fora.

É multi-tenant, então cada organização vê apenas os próprios dados, e as organizações se conectam entre si (uma feira e seus expositores, uma instituição e seus parceiros).

---

## Meu trabalho

### Trabalho de produto full stack

Construo funcionalidades de ponta a ponta em Rails e JavaScript, do modelo de dados até a interface. Minha ideia central é que o melhor software parece atencioso: funcionalidades pequenas e contextuais que percebem o que a pessoa está tentando fazer, usam o contexto que a plataforma já tem e oferecem o próximo passo em vez de deixá-la procurando.

### Desempenho e infraestrutura

Lidero grande parte do nosso trabalho de infraestrutura e otimização: ajuste do banco de dados, trabalho em queries e índices nas páginas mais movimentadas, jobs em segundo plano, deploys e monitoramento. Para uma plataforma de eventos, os momentos difíceis são previsíveis (convites sendo enviados, portas abrindo), então o trabalho de desempenho mira esses picos.

### Segurança

A segurança me acompanha desde [meu tempo na Exodus](/posts/career-exodus). Na ARTSVP, conduzo a revisão e a auditoria de SecOps, e esse trabalho já barrou várias ameaças ativas, **incluindo um ataque à cadeia de suprimentos**. Também fico de olho nas fronteiras de privacidade: listas de convidados são sensíveis, e um expositor nunca deve ver os convidados de outra galeria.

### Trabalhando com ferramentas de IA

Boa parte do meu dia roda em um setup local de programação com IA e memória: agentes em paralelo em workspaces isolados, um grafo de conhecimento do código e notas que levam decisões de uma sessão para outra. Escrevi sobre toda a stack em [A Pilha Definitiva de Programação e Memória por IA Local](/posts/developer-setup-ai-tools).

---

## Por que gosto deste trabalho

A ARTSVP combina coisas com as quais me importo: arte, design de produto cuidadoso e a engenharia por baixo. As pessoas que a usam estão realizando eventos ao vivo, então confiabilidade e privacidade não são abstratas. Quando o check-in funciona numa noite de abertura lotada, ninguém percebe o software, e esse é o objetivo.
