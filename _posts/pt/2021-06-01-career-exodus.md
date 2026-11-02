---
lang: pt
page_id: career_exodus
permalink: /posts/career-exodus
title: "Exodus: Da Fila de Suporte à Engenharia de Segurança"
date: 2021-06-01
categories: [Career, Security]
tags: [Career, Exodus, Security, JavaScript, Ruby on Rails, AWS, Chrome Extension]
author: Angel
description: Dois anos e meio na Exodus, de responder tickets de suporte a construir as ferramentas de segurança que protegiam os usuários da carteira, criadas resolvendo os problemas que eu via de perto.
image:
  path: /assets/img/career-exodus.png
  alt: Logotipo da Exodus
---

## Exodus: Da Fila de Suporte à Engenharia de Segurança

**Junho de 2021 a fevereiro de 2024 · Exodus Movement Inc. · 100% remoto**

Entrei na [Exodus](https://www.exodus.com/), a carteira cripto de autocustódia, como Technical Support Engineer e saí como Security Engineer. Essa mudança não veio de uma candidatura a vaga. Veio de construir ferramentas para os problemas que eu via todos os dias na fila de suporte, até que construir ferramentas virou o trabalho.

| Cargo | Datas | Foco |
|---|---|---|
| Technical Support Engineer | 06/2021 a 01/2023 | Suporte ao cliente, ferramentas internas, automação de QA |
| Security Engineer | 01/2023 a 02/2024 | Ferramentas de resposta a golpes e ameaças, automação de SecOps |

---

## Começando no suporte

Suporte em uma empresa de carteiras é mais parecido com perícia forense do que parece. Um usuário escreve sobre uma transação que sumiu ou um endereço suspeito, e antes de poder ajudar, você precisa descobrir em qual de dezenas de blockchains ela está, encontrá-la no explorador de blocos certo e verificar se o endereço já é conhecido como malicioso. Numa equipe de mais de 100 pessoas, esses minutos se acumulavam a cada ticket.

Então construí algo para eliminá-los.

### O projeto do hackathon

A Exodus fez seu primeiro hackathon de suporte ao cliente, o primeiro desse departamento a envolver código. Liderei uma equipe e vencemos com uma **extensão de Chrome em ReactJS** que recebe qualquer endereço ou hash de transação, identifica a blockchain correta (61 blockchains para endereços, 31 para transações) e consulta um banco de dados de endereços maliciosos conhecidos.

Não ficou só na demonstração. Continuei programando e mantendo a extensão, e o departamento inteiro, com mais de 100 pessoas, a adotou. Ela levou o tempo médio de resposta para **menos de 10 minutos**, um indicador-chave de desempenho da equipe.

O artigo e o código: [Extensão Exploradora de Blockchain de Criptomoedas](/posts/chrome-extension).

### Mais automação

O mesmo instinto se estendeu a outros trabalhos repetitivos:

- **Automação de QA.** Automatizei o teste de regressão de uma extensão de navegador com Puppeteer, reduzindo em **20%** os scripts manuais de regressão.
- **Comunidade.** Projetei, construí e containerizei em Kubernetes um bot de Discord para a comunidade, o que aumentou o engajamento dos usuários em **20%**.

No caminho, também estudei de forma estruturada: o programa Blockchain Developer da Consensys Academy em 2022 e o bootcamp de desenvolvimento web da Le Wagon em 2023.

---

## Migrando para segurança

Em 2023, a maior parte do meu tempo ia para combater golpes que miram usuários de carteiras: sites de phishing, aplicativos falsos, endereços de drainers. O acompanhamento ficava em planilhas, que não escalam quando várias pessoas trabalham no mesmo caso e é preciso anexar evidências e fazer o acompanhamento.

Como Security Engineer, liderei a engenharia, a containerização e o deploy de um app em **Ruby on Rails, JavaScript e PostgreSQL** para reportar e acompanhar indicadores de comprometimento (IoCs), rodando em **EKS na AWS**. Ele permitia que várias pessoas trabalhassem no mesmo caso, se conectava a diversas APIs externas para reportes e aceitava envios de pessoas de fora da equipe.

- As ameaças que chegavam aos usuários da Exodus caíram **50%** em seis meses.
- O tempo de resolução ficou **20%** mais rápido, graças às integrações com APIs.

Mais tarde, migrei o app em Ruby para Node com um front end em React e TypeScript, e continuei construindo extensões de Chrome e bibliotecas NodeJS para automatizar o trabalho de QA e SecOps.

Como o app funciona e como fazer o deploy do seu próprio: [Implantando Defesa SecOps com Capacidade de Banco de Dados IoC](/posts/secops-data).

---

## O que levei comigo

- **As melhores ferramentas internas vêm de quem as usa.** Eu sabia o que construir porque eu mesmo tinha respondido àqueles tickets.
- **Meça o que a equipe já valoriza.** Tempo de resposta e ameaças que chegavam aos usuários eram números que todos acompanhavam, então as ferramentas eram avaliadas por eles.
- **Segurança é um produto, não um relatório.** Um banco de dados de ameaças só ajuda se as pessoas conseguirem usá-lo sob pressão.

Essa combinação de empatia do suporte, automação e segurança é o que levei para meus próximos cargos, incluindo [meu trabalho atual na ARTSVP](/posts/career-artsvp).
