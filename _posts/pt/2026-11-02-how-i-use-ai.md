---
lang: pt
page_id: how_i_use_ai
permalink: /posts/how-i-use-ai
title: "Como Eu Realmente Uso IA: Regras, Memória e Salvaguardas"
date: 2026-11-02
categories: [AI, Workflow]
tags: [AI, Claude, Codex, Obsidian, Ollama, Agents, Productivity]
author: Angel
description: Os princípios de trabalho por trás de como uso agentes de IA todos os dias, de um cofre de segundo cérebro e salvaguardas impostas por código até um modelo auto-hospedado rodando em casa.
image: /assets/img/how-i-use-ai.png
---

## A versão curta

Uso agentes de IA na maior parte do meu trabalho de engenharia: escrever e revisar código, fazer triagem de bugs, redigir pull requests, responder perguntas de suporte e manter um pequeno assistente em casa. Depois de muito uso diário, a lição mais útil que aprendi é que **o modelo é a parte fácil**. O trabalho está no sistema ao redor dele: o que ele lembra, o que tem permissão para fazer, o que precisa verificar antes de falar e como ele se comunica comigo.

Este post é o "porquê" por trás desse sistema. Se você quer o "como" (comandos de instalação, arquivos de configuração, hooks), escrevi um guia separado passo a passo: [A Pilha Definitiva de Programação e Memória por IA Local](/posts/developer-setup-ai-tools).

---

## 1. O modelo decide, o código garante

No começo, toda vez que um agente fazia algo errado, eu adicionava uma frase às instruções dele. "Sempre leia o link antes de responder." "Nunca escreva notas naquele arquivo." "Traduza antes de falar." A lista crescia, e o agente continuava ignorando partes dela.

A solução foi parar de tratar instruções como a única alavanca. Existem seis, e elas não são igualmente confiáveis:

| Camada | O que é | Confiabilidade |
|---|---|---|
| **Ferramentas** | Pequenos programas que o agente chama e que fazem o trabalho de fato, com verificações embutidas | Máxima: o código garante |
| **Permissões** | Quais ferramentas rodam livremente e quais precisam de um toque humano antes | Alta: imposta fora do modelo |
| **Hooks / plugins** | Código que roda em volta de cada chamada ao modelo, independente do que o modelo lembra | Alta |
| **Configuração** | Escolha do modelo, nível de raciocínio, ferramentas desligadas | Alta |
| **Regras sempre carregadas** | Regras curtas enviadas com cada mensagem | Média: o modelo pode se desviar |
| **Skills** | Instruções de tarefa carregadas sob demanda ("como fazer X") | Mínima sozinha: orientação, não imposição |

Modelos são bons em **decidir o que fazer** e pouco confiáveis em **lembrar formatos, caminhos e regras exatos**. Então minha regra prática é:

1. Tudo que precisa estar certo sempre vai para uma **ferramenta ou hook**, onde o código garante.
2. Tudo que é arriscado fica atrás de uma **permissão**, e a ferramenta confere o que foi aprovado (por exemplo, um comando de push que recusa se o commit for diferente daquele que eu vi).
3. Skills e regras então só precisam apontar para a ferramenta certa, algo que um modelo segue com muito mais confiabilidade do que um procedimento de dez passos.
4. **Quando uma regra continua sendo ignorada, não a escreva mais alto. Mova-a uma camada para baixo.**

Alguns exemplos reais de mover uma correção uma camada para baixo:

| Problema | Primeira tentativa | O que realmente resolveu |
|---|---|---|
| O agente respondia sobre links sem abri-los | "Sempre leia os links" | Um plugin busca toda URL antes de o modelo rodar |
| O agente lia PDFs como imagens, gastando tokens | "Converta documentos primeiro" | Um hook bloqueia a leitura bruta e mostra o comando certo |
| O agente riscava itens de uma lista em vez de removê-los | Uma skill descrevendo os passos de remoção | Uma ferramenta de listas que remove por número, com mudanças destrutivas exigindo um toque |
| Lembretes criados num formato que nunca disparava | Uma skill com o formato exato do job | Um comando `remind` monta o job; a ferramenta embutida que confundia o modelo foi desligada |

Fine-tuning quase nunca entra na história. Para um assistente que usa ferramentas, essas seis camadas cobrem quase tudo, são baratas de mudar e podem ser inspecionadas e revertidas em minutos.

---

## 2. A memória vive fora do modelo

Toda sessão de agente começa com amnésia. Resolvo isso com três lugares para guardar as coisas, cada um com uma função:

```text
 ┌───────────────────────────┐   carregadas em TODA sessão
 │  Regras sempre carregadas │   ordens permanentes, poucas linhas cada
 └─────────────┬─────────────┘
               │ apontam para
 ┌─────────────▼─────────────┐   pesquisado quando relevante
 │  Cofre de segundo cérebro │   decisões, armadilhas, o "porquê"
 │  (Obsidian, markdown)     │
 └─────────────┬─────────────┘
               │ ligado a
 ┌─────────────▼─────────────┐   a fila de trabalho
 │  Gerenciador de tarefas   │   o que ainda falta, em checkboxes
 └───────────────────────────┘
```

**O cofre é escrito para a IA, não para mim.** Raramente o leio diretamente; peço a um agente que o pesquise. Por isso toda nota segue algumas regras:

- Começa com um resumo de duas ou três frases **"Para o Claude do futuro"**, para que um agente decida em segundos se a nota é relevante.
- Tem **frontmatter** consistente (tipo, data, tópico, tags, projetos relacionados, fontes, confiança).
- Fatos levam **datas** ("em 2026-09"), para que o agente saiba o que verificar de novo.
- Afirmações levam um **nível de confiança**: declarado, alto, médio ou especulação.
- Tudo é **interligado**, então uma nota leva à próxima.

A captura acontece de duas formas: deliberadamente, com comandos como `/obsidian-log` e `/obsidian-save`, e automaticamente quando uma sessão longa é compactada, quando um agente em segundo plano colhe o que vale a pena guardar. Esse agente roda com uma lista restrita de ferramentas e sem acesso ao shell, porque lê resumos de sessão e não quero que texto injetado execute comandos.

**A lição que mais demorei a aprender:** uma regra que vive só no cofre se perde, porque sessões de programação não carregam o cofre a menos que vão procurá-lo. Repeti "responda de forma concisa" quatro vezes antes de perceber que a cópia sempre carregada tinha sumido. Agora toda ordem permanente existe como uma versão curta nas instruções sempre carregadas (tanto para o Claude quanto para o Codex) e uma versão completa com histórico no cofre, e cada cópia cita as outras para que fiquem sincronizadas.

**Trabalho pendente vai para o ticket.** Todo TODO, acompanhamento ou pergunta em aberto de uma sessão é escrito na issue relacionada, nunca deixado no histórico do chat, numa mensagem de commit ou num comentário no código. O histórico morre com a sessão. O ticket guarda o que falta fazer; o cofre guarda o raciocínio e o que já foi descartado.

---

## 3. Verifique o estado real antes de agir

A maioria das respostas ruins de IA que vejo não são alucinações do nada. São raciocínios confiantes a partir de contexto desatualizado: uma branch antiga, um nome de coluna lembrado, um palpite sobre como são os dados de um cliente. Por isso os agentes seguem uma regra permanente: **leia a coisa real primeiro.**

- Leia o schema antes de declarar um bug. Os nomes reais dos campos vivem no código, não na memória.
- Busque e leia a branch que está em produção, não uma cópia local que pode estar atrasada.
- Busque datas e taxas ao vivo; nunca as deduza dos dados de treino.
- Leia um payload real antes de chamar uma integração de "pronta". Testes com mocks não pegam divergências.
- Quando um fato de produção resolveria a questão, consulte-o por um wrapper de console **somente leitura** antes de me pedir para fazer isso. Selecione o mínimo, nunca imprima segredos ou registros inteiros e separe fatos de produção de inferências na resposta.

A regra que mais valorizo veio de um bug que precisou de dois pull requests. Um cliente relatou dados faltando num download. O agente encontrou uma causa raiz real na camada de dados, corrigiu, publicou um backfill e os testes passaram. Mas a exportação que o cliente realmente usava nem tinha uma coluna para aqueles dados. A primeira correção estava certa e não mudou nada do que ele via.

Desde então, **corrigir um bug relatado significa verificar o artefato de quem relatou**: nomeie a tela ou o arquivo exato que a pessoa abriu, reproduza a saída errada nesse caminho e reproduza a saída certa no mesmo caminho depois da correção. Um sintoma parcial muitas vezes tem uma causa por camada, então continue procurando depois da primeira causa que o explica.

---

## 4. Gaste tokens pensando, não lendo

Agentes desperdiçam a maior parte do orçamento lendo coisas: arquivos inteiros, HTML bruto, saídas de comando verbosas. Ataco cada fonte de desperdício separadamente:

| Desperdício | O que uso no lugar |
|---|---|
| Ler seis arquivos para entender um fluxo | Um **grafo de conhecimento do código** consultado primeiro para qualquer pergunta do tipo "como X funciona"; busca simples só para identificadores exatos |
| Páginas HTML brutas | Um **leitor web de linha de comando** compacto que renderiza uma página em cerca de 500 tokens |
| PDFs e arquivos Office lidos como imagens | O **MarkItDown** os converte para Markdown antes, imposto por um hook |
| Saída de shell barulhenta | Um **filtro de saída de CLI** que resume a saída de `git`, `grep`, `ls` e testes |
| Contexto inchado | Um **proxy de compressão** local entre o agente e a API |

Uma regra sutil: um stack trace te dá nomes de arquivos e métodos, o que faz uma investigação *parecer* uma busca por identificador. Normalmente não é. "Que caminho de código leva a esse erro?" é uma pergunta de fluxo, e o grafo revela os callbacks, jobs e outros chamadores que o backtrace esconde.

---

## 5. Humanos aprovam tudo que sai para o mundo

Os agentes trabalham em paralelo em worktrees git isoladas (uso o [Conductor](https://conductor.build) para isso), então várias tarefas rodam ao mesmo tempo sem que uma atropele a outra. Mas tudo que sai da minha máquina passa por mim:

- **Mensagens para canais compartilhados são rascunhadas antes.** Um agente uma vez adivinhou o canal errado da equipe e postou uma atualização direto. Agora ele mostra o rascunho, nomeia o canal e espera.
- **Ferramentas de escrita precisam de um toque.** No meu assistente de casa, a leitura é pré-aprovada e toda escrita pede confirmação no meu celular.
- **Pull requests carregam suas próprias evidências.** Mudanças visuais incluem capturas de tela reais de cada tela alterada. Mudanças entre vários serviços incluem um diagrama de fluxo com as partes novas destacadas. Todo PR recebe uma faixa de risco (vermelho, amarelo, verde) no título e nas labels para que revisores façam a triagem num relance.
- **Sair do rascunho dispara uma barreira.** Um hook dispara quando um PR é marcado como pronto para revisão e injeta um checklist pré-merge: limpar código de debug esquecido, procurar partes não conectadas, polir o texto, rodar de verdade, procurar regressões em funcionalidades existentes e fazer uma revisão adversarial final.
- **Diagnosticar não é permissão para mudar.** Se pergunto por que algo acontece, o agente explica; ele não faz um commit silencioso de uma correção dentro da resposta.

---

## 6. Ajuste a voz, depois deixe quieto

Como um agente fala importa tanto quanto o que ele faz, porque leio centenas de respostas dele por semana. Minhas preferências permanentes:

- **Comece pela resposta.** Sem preâmbulo, sem recapitular o que acabou de fazer, sem repetir uma saída que já estou vendo.
- **Conciso não é o mesmo que omitir.** Testes falhando, afirmações não verificadas e escopo pulado continuam sendo relatados, só que em menos palavras.
- **Próximos passos só quando tenho algo a fazer.** Testei uma lista obrigatória de "Próximos passos" em toda resposta; ela enchia tudo, então virou condicional.
- **Nada de travessões, em lugar nenhum.** São o sinal mais rápido de texto escrito por máquina. Quando essa regra colidiu com alguns rótulos da interface de um produto que um bot de suporte precisava citar literalmente, a solução foi mudar os rótulos, não abrir uma exceção no bot. Exceções se espalham por todos os sistemas seguintes e nunca são removidas.
- **Experimentos podem falhar.** Por duas semanas toda resposta passou por uma skill de "explique como se eu tivesse cinco anos". As respostas ficaram mais longas, não mais simples, então agora é opcional por mensagem.

---

## 7. Um modelo só meu

Nem tudo precisa de um modelo de ponta. Também mantenho um assistente auto-hospedado em casa, que começou como [um bot de Telegram em Ruby num Raspberry Pi](/posts/telegram-ollama-bot-raspberry-pi) e agora vive num mini PC de IA com 128 GB de memória compartilhada.

- **Modelos:** modelos abertos via Ollama. Um grande modelo mixture-of-experts (`gpt-oss:120b`) é o padrão, um menor cuida de chamadas rápidas de ferramentas, um modelo de código cuida de conversas sobre código e um modelo de embeddings alimenta a busca na memória.
- **Interface:** um app de chat no celular, inclusive por voz sem usar as mãos. Fala para texto e texto para fala rodam localmente.
- **Tarefas:** lembretes, listas, notas rápidas, leitura de links e pequenas tarefas de código que terminam num pull request que eu reviso.
- **Segurança:** o bot roda como um usuário sem privilégios, só é acessível por uma VPN privada, guarda segredos numa conta de serviço de um gerenciador de senhas em vez da configuração e precisa do meu toque no celular para qualquer escrita. Atualizar o próprio modelo também é um pedido de um toque.

A lição das seis camadas da seção 1 veio quase toda dessa máquina. Um modelo menor se desvia mais do que um de ponta, então ele me forçou a mover regras para ferramentas muito antes, e esses hábitos depois melhoraram meus agentes de programação também.

---

## O que eu diria a quem está começando

1. **Anote toda correção.** Se você já disse a mesma coisa duas vezes a um agente, ela pertence a um arquivo que ele sempre carrega.
2. **Mantenha esse arquivo curto.** Modelos também passam o olho em arquivos de instrução longos. Coloque o histórico e o raciocínio em algum lugar pesquisável.
3. **Mova os reincidentes uma camada para baixo.** Um hook vence uma regra; uma ferramenta vence uma skill.
4. **Faça-o checar a realidade.** A maioria das respostas erradas vem de contexto desatualizado, não de falta de inteligência.
5. **Mantenha um humano em tudo que sai para o mundo.** Rascunhos são baratos; retratações não.
6. **Meça seu desperdício de tokens.** É na leitura que o orçamento vai embora.

Nada disso está pronto. Notas são reescritas conforme os fatos mudam, regras são promovidas e aposentadas, e as ferramentas mudam por baixo de tudo. Mas os princípios se mantiveram, e valem para qualquer modelo que esteja no topo neste mês.
