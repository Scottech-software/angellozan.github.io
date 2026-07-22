---
lang: pt
page_id: dev_setup_ai_tools
permalink: /posts/developer-setup-ai-tools
title: "A Pilha Definitiva de Programação e Memória por IA Local: Conductor, Claude Code, RTK e Obsidian"
date: 2026-07-17
categories: [Developer Tools, AI, Workflow]
tags: [Tutorial, Conductor, Claude, Codex, Obsidian, Git, AI Tools, RTK, Graphify]
author: Angel
description: Um guia instrucional abrangente sobre como construir uma pilha de desenvolvimento de IA local, com backup de memória e otimização de tokens, usando Claude Code, Conductor, Graphify, Obsidian e RTK.
image: /assets/img/deepseek.png
---

## Construindo uma Pilha Local de Programação e Memória por IA

Na engenharia de software moderna, assistentes de IA como Claude e Codex deixaram de ser utilitários básicos de copiar e colar para se tornarem agentes totalmente ativos que operam em nossos ambientes de desenvolvimento. No entanto, escalar esses agentes em fluxos de trabalho complexos e multiprojetos introduz fricção: faturas exorbitantes de tokens de API, falta de memória persistente entre sessões e "sobrecarga de contexto", onde os modelos se perdem no ruído da base de código.

Para superar esses obstáculos, interliguei uma **pilha de programação e memória por IA local de seis camadas**. Ela transforma o Claude Code em um agente consciente do projeto e com backup de memória, capaz de executar tarefas de código complexas de forma segura, eficiente e com preservação de contexto.

Todas as camadas são executadas localmente na sua máquina, sem interação na nuvem além das chamadas diretas e seguras de API para os modelos. Abaixo está um blueprint detalhado e instrucional sobre como configurar e executar essa mesma estrutura.

---

## Arquitetura de Alto Nível

Veja como as seis camadas cooperam para otimizar o raciocínio, controlar custos e capturar contexto:

```text
                  ┌──────────────────────────────────────────────┐
                  │          Claude Code (Agente Principal)      │
                  └──────┬────────────────────┬───────────┬──────┘
                         │                    │           │
                         │ (Proxy de Hooks)   │ (MCP)     │ (Subagente)
                         ▼                    ▼           ▼
  ┌──────────────────────────────┐     ┌───────────┐     ┌───────────────────────┐
  │  RTK (Rust Token Killer)     │     │ Obsidian  │     │   Codex CLI           │
  │  Proxy Transparente          │     │ Vault     │     │   (codex:codex-rescue)│
  └──────────────┬───────────────┘     └─────┬─────┘     └───────────────────────┘
                 │                           │
                 ▼                           ▼
  ┌──────────────────────────────┐     ┌───────────┐
  │  APIs de Modelos (Claude/GP) │     │ Obsidian  │
  └──────────────────────────────┘     │ Git       │
                                       └───────────┘
```

Vamos detalhar cada camada e construir o ambiente passo a passo.

---

## Camada 1: Claude Code com Instruções em Camadas

O **Claude Code** é o agente principal. Para fazer com que as regras de configuração "simplesmente se apliquem" dependendo de onde o agente está operando, ele carrega instruções em uma cascata estrita de camadas, da mais global para a mais local.

### Passo 1: Estabeleça suas Regras de Composição
Ao usar importações `@filename`, podemos compor instruções a partir de arquivos externos em vez de manter configurações gigantescas e duplicadas. Crie suas instruções globais da máquina:

```markdown
# ~/.claude/CLAUDE.md

@import "~/.claude/RTK.md"

# Diretrizes Globais
- Sempre execute `/graphify query` para qualquer questão arquitetural ou comportamental antes de abrir arquivos de código.
- Prefira usar comandos `rtk` em vez de utilitários de terminal nativos para manter o tamanho do payload otimizado.
```

### Passo 2: Configure Agentes Escopados
No repositório do seu projeto, defina diretrizes específicas que serão salvas no controle de versão. Isso garante que os membros da equipe também se beneficiem das convenções de IA padrão:

- `<REPO>/CLAUDE.md`: Configurado como o ponto de entrada visível para a equipe, referenciando `@AGENTS.md`.
- `<REPO>/AGENTS.md`: Descreve a estrutura do repositório, convenções de estilo e armadilhas.
- `AGENTS.md` de subdiretório (ex: `<REPO>/app/javascript/AGENTS.md`): Ativa apenas ao trabalhar dentro desse diretório. Por exemplo, especifique regras específicas de React/Vite aqui para evitar poluir tarefas de backend em Rails com ruídos do frontend.

---

## Camada 2: Conductor (Agentes de Worktree em Paralelo)

O **Conductor** é um aplicativo que cria worktrees git paralelas sob um caminho canônico, como `~/conductor/workspaces/<project>/<workspace-name>/`. Cada espaço de trabalho hospeda uma sessão isolada do Claude Code.

### Passo 1: Configure a Automação de Espaço de Trabalho Privada
Cada projeto possui um arquivo `.conductor/settings.local.toml` (ignorado, privado para a sua máquina). Defina um **script de configuração** que automatize a construção de espaços de trabalho auto-regenerativos quando uma nova worktree for criada:

```toml
# .conductor/settings.local.toml
[workspace]
setup_script = "~/.conductor/scripts/setup-worktree.sh"
```

### Passo 2: Escreva o Script de Auto-Regeneração
Crie o script setup em bash para rodar dentro do seu espaço de trabalho. Esse script resolve a raiz da nova worktree, verifica o grafo de código e cria um link simbólico para que cada worktree paralela compartilhe uma referência canônica única:

```bash
#!/bin/bash
# ~/.conductor/scripts/setup-worktree.sh

set -e

# Resolver caminhos da worktree atual e do repositório principal
WORKTREE_ROOT=$(pwd)
MAIN_REPO_ROOT="$ARTSVP_MAIN_REPO" # Variável de ambiente fornecida pelo Conductor

echo "Inicializando worktree paralela em: $WORKTREE_ROOT"

# Garantir que a pasta compartilhada do graphify existe no repositório principal
if [ ! -d "$MAIN_REPO_ROOT/graphify-out" ]; then
  mkdir -p "$MAIN_REPO_ROOT/graphify-out"
fi

# Criar link simbólico do grafo canônico na nova worktree
ln -sfn "$MAIN_REPO_ROOT/graphify-out" "$WORKTREE_ROOT/graphify-out"

# Executar passos de configuração do projeto (ex: instalar dependências)
bundle config set --local path 'vendor/bundle'
bundle install
yarn install
```

---

## Camada 3: Graphify (Código para Grafo de Conhecimento)

Em vez de enviar centenas de linhas de código bruto para o modelo procurar um bug, usamos o **Graphify**—uma ferramenta de CLI instalada em `~/.local/bin/graphify` que constrói um grafo de dependências local de nossa base de código.

### Passo 1: Entenda os Artefatos do Graphify
O Graphify analisa os arquivos do seu projeto e exporta dois ativos:
1. **Grafo de Máquina** (`<REPO>/graphify-out/graph.json` + `GRAPH_REPORT.md`): Consultado diretamente pelo Claude Code usando `/graphify query "..."`.
2. **Exportação Visual** (`<VAULT_STORAGE>/<project>-graph/`): Um cofre do Obsidian que consiste em arquivos markdown interligados e um `graph.canvas` mostrando a relação entre controllers, models e background jobs.

### Passo 2: Automatize com um Agente launchd Semanal
Para garantir que o grafo nunca fique desatualizado, configure uma tarefa de segundo plano do launchd no macOS. Crie um arquivo plist:

```xml
<!-- ~/Library/LaunchAgents/com.user.graphify-project-update.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.graphify-project-update</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/admin/.local/bin/graphify-project-update.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Weekday</key>
        <integer>1</integer> <!-- Segunda-feira -->
        <key>Hour</key>
        <integer>9</integer>
        <key>Minute</key>
        <integer>37</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/admin/.local/state/graphify-project-update.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/admin/.local/state/graphify-project-update.err</string>
</dict>
</plist>
```

E escreva o script complementar `~/.local/bin/graphify-project-update.sh`:

```bash
#!/bin/bash
# ~/.local/bin/graphify-project-update.sh

REPO_PATH="/Users/admin/code/meu-projeto"
VAULT_PATH="/Users/admin/vaults/meu-projeto-graph"

cd "$REPO_PATH"
/Users/admin/.local/bin/graphify update .
/Users/admin/.local/bin/graphify export obsidian --dir "$VAULT_PATH"

echo "Atualização do Graphify concluída em $(date)"
```

Carregue e inicie o agente em segundo plano:
```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.graphify-project-update.plist
```

---

## Camada 4: Obsidian como um "Segundo Cérebro"

Uma decisão de design importante nesta pilha é que **nada durável é escrito diretamente na memória embutida do Claude**. A memória embutida do modelo é transitória e difícil de versionar. Em vez disso, um cofre dedicado do Obsidian localizado em `<VAULT_STORAGE>/second-brain/` hospeda toda a memória durável entre projetos.

### Passo 1: Imponha o Manual de Operação AI-First
Otimizamos a estrutura do cofre para analisadores de LLM. Crie um arquivo de manual raiz dentro de seu cofre do Obsidian:

```markdown
# second-brain/_CLAUDE.md

## Regras de Escrita de Notas AI-First:
1. **Auto-Contido**: Garanta que cada nota seja totalmente compreensível sem exigir outro contexto.
2. **Preâmbulo de Contexto**: Cada nota deve começar com um resumo de 2 a 3 frases destinado a uma futura sessão de IA.
3. **Frontmatter Rico**: Inclua metadados em cada nota:
   ```yaml
   type: logic-explainer
   date: 2026-07-17
   topic: Authentication
   confidence: high
   ai-first: true
   ```
4. **Marcadores de Tempo**: Referencie eventos externos com datas precisas: "A partir de 2026-07...".
5. **Links Wiki**: Conecte entidades com `[[wikilinks]]` para que o Conductor/Claude possa analisar o grafo de notas facilmente.
```

### Passo 2: Configure o Plugin Obsidian Git para Controle de Versão
1. No Obsidian, vá em **Plugins de Comunidade**, pesquise por **Obsidian Git** e clique em instalar.
2. Configure os intervalos de backup para serem acionados a cada 5 a 10 minutos em segundo plano. Isso fornece um registro de memória transparente e com carimbo de data/hora de como suas notas e raciocínio de arquitetura evoluem ao longo do tempo.

---

## Camada 5: Codex CLI como Subagente de Segunda Opinião

Às vezes, o Claude fica preso em um loop recursivo ou precisa de uma segunda perspectiva sobre um desafio algorítmico complexo. Para isso, conectamos um **Codex CLI** separado como um subagente do Claude Code chamado `codex:codex-rescue`.

1. Configure habilidades personalizadas dentro de `~/.claude/skills/codex/SKILL.md`.
2. Defina como o Claude se comunica com o subagente usando especificações de ferramentas especializadas (`codex:rescue`, `codex:setup`).
3. O Claude delega tarefas complexas com:
   ```bash
   /subagent run codex:codex-rescue "Explique este erro de segmentação (segmentation fault)"
   ```
4. Isso mantém seus fluxos de trabalho fluindo sem forçá-lo a copiar e colar erros manualmente entre diferentes abas do navegador.

---

## Camada 6: RTK (Rust Token Killer) — O Proxy Transparente

O **RTK** é um wrapper de CLI de alto desempenho escrito em Rust. Ele intercepta utilitários de shell comuns e remove logs inúteis, metadados e arquivos de código excessivamente detalhados antes de passá-los para o agente de IA.

### Passo 1: Configure o Hook do Claude Code
O RTK é conectado ao Claude Code por meio de um hook Bash `PreToolUse` em `~/.claude/CLAUDE.md` ou como comandos de wrapper:

```bash
# Interceptar comandos do git e leitura de arquivos de forma transparente
alias git="rtk git"
alias cat="rtk read"
alias grep="rtk grep"
```

### Passo 2: Use as Regras do RTK para Economizar Tokens Drasticamente
- Para arquivos que excedem **50KB**, use `rtk read <path>` para resumir as definições estruturais em vez de ler o arquivo de código inteiro.
- Verifique as estatísticas de economia de tokens usando:
  ```bash
  rtk gain
  ```
  *Normalmente, os desenvolvedores veem uma **redução de 40 a 90% nos tokens** em operações comuns como listagem de diretórios e diffs de sistema.*

---

## Juntando Tudo: O Fluxo de Desenvolvimento por IA

Quando você faz ao Claude uma pergunta complexa como *"Como funciona a autenticação dentro deste microsserviço?"*, veja como sua pilha opera de forma integrada:

1. O Claude lê as instruções da cascata hierárquica `CLAUDE.md` -> `AGENTS.md`.
2. As regras forçam o Claude a executar `/graphify query` como sua **primeira** operação.
3. O Graphify lê o arquivo compartilhado `graphify-out/graph.json` (vinculado simbolicamente de forma automática pelo Conductor quando o espaço de trabalho foi criado) e indica os arquivos de destino.
4. O Claude lê apenas os arquivos indicados, contando com o `rtk read` para manter o payload de contexto limpo e barato.
5. Se o Claude encontrar um bloqueio inesperado, ele aciona o `codex:codex-rescue` para obter uma segunda opinião.
6. Uma vez encontrada a solução, o Claude usa automaticamente a habilidade `/obsidian-save` para gravar um registro altamente formatado e permanente no cofre do Obsidian `second-brain/`.
7. O Obsidian Git salva automaticamente o novo registro, preservando seu progresso para futuras sessões.

Ao combinar **otimização de tokens transparente** com um **grafo de conhecimento local e memória estruturados**, essa configuração cria um espaço de trabalho de desenvolvimento que é incrivelmente rápido, consciente do contexto e altamente eficiente em termos de custos.
