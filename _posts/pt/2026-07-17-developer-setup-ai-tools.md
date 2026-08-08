---
lang: pt
page_id: dev_setup_ai_tools
permalink: /posts/developer-setup-ai-tools
title: "A Pilha Definitiva de Programação e Memória por IA Local: Conductor, Claude Code, RTK, Headroom e Obsidian"
date: 2026-07-17
categories: [Developer Tools, AI, Workflow]
tags: [Tutorial, Conductor, Claude, Codex, Obsidian, Git, AI Tools, RTK, Graphify, Headroom]
author: Angel
description: Um guia instrucional abrangente sobre como construir uma pilha de desenvolvimento de IA local, com backup de memória e otimização de tokens, usando Claude Code, Conductor, Graphify, Headroom, Obsidian e RTK.
image: /assets/img/deepseek.png
---

## Construindo uma Pilha Local de Programação e Memória por IA

Na engenharia de software moderna, assistentes de IA como Claude e Codex deixaram de ser utilitários básicos de copiar e colar para se tornarem agentes totalmente ativos que operam em nossos ambientes de desenvolvimento. No entanto, escalar esses agentes em fluxos de trabalho complexos e multiprojetos introduz fricção: faturas exorbitantes de tokens de API, falta de memória persistente entre sessões e "sobrecarga de contexto", onde os modelos se perdem no ruído da base de código.

Para superar esses obstáculos, interliguei uma **pilha de programação e memória por IA local de sete camadas**. Ela transforma o Claude Code em um agente consciente do projeto e com backup de memória, capaz de executar tarefas de código complexas de forma segura, eficiente e com preservação de contexto.

Todas as camadas são executadas localmente na sua máquina, sem interação na nuvem além das chamadas diretas e seguras de API para os modelos. Abaixo está um blueprint detalhado e instrucional sobre como configurar e executar essa mesma estrutura.

---

## Arquitetura de Alto Nível

Veja como as sete camadas cooperam para otimizar o raciocínio, controlar custos e capturar contexto:

```text
                  ┌──────────────────────────────────────────────┐
                  │       Claude Code (Agente Principal)         │
                  │  instruções · hooks · skills · plugins       │
                  └──┬──────────┬──────────┬──────────┬──────────┘
                     │          │          │          │
            (Hooks)  │  (MCP)   │  (Sub-   │  (Skill) │
                     │          │  agente) │          │
                     ▼          ▼          ▼          ▼
  ┌─────────────┐ ┌──────────┐ ┌────────┐ ┌─────────────────────┐
  │ RTK         │ │ Obsidian │ │ Codex  │ │ Skills              │
  │ (PreToolUse │ │ Vault    │ │ CLI    │ │ graphify, ponytail, │
  │  hook)      │ │ (MCP +   │ │ plugin │ │ rails-*, pr-review  │
  └──────┬──────┘ │ hooks)   │ └────┬───┘ └─────────────────────┘
         │        └────┬─────┘      │
         ▼             │            ▼
  ┌─────────────┐      │     ┌───────────┐
  │ Headroom    │      ▼     │  OpenAI   │
  │ Proxy :8787 │ ┌────────┐ │  API      │
  └──────┬──────┘ │Obsidian│ └───────────┘
         │        │Git     │
         ▼        └────────┘
  ┌─────────────┐
  │ Anthropic   │
  │ API         │
  └─────────────┘

         ┌──────────────────────────────────────┐
         │  Conductor                           │
         │  Orquestra worktrees paralelas,      │
         │  cada uma com sua própria sessão     │
         └──────────────────────────────────────┘
```

Vamos detalhar cada camada e construir o ambiente passo a passo.

---

## Camada 1: Claude Code com Instruções em Camadas, Hooks e Skills

O **Claude Code** é o agente principal. A configuração não é monolítica — ela é carregada em uma cascata estrita, da mais global à mais local, e um sistema de hooks permite interceptar e transformar o comportamento do agente em tempo de execução.

### Passo 1: Estabeleça sua Cascata de Instruções

O Claude Code carrega instruções nesta ordem, cada camada sobrescrevendo ou complementando a anterior:

1. **`~/.claude/CLAUDE.md`** — Regras globais da máquina. Aplicam-se em todos os lugares, todos os projetos.
2. **`<REPO>/CLAUDE.md`** — Convenções de projeto visíveis pela equipe, versionadas no controle de código.
3. **`<REPO>/AGENTS.md`** — Estrutura do repositório, convenções de estilo e armadilhas (referenciado via `@AGENTS.md` pelo CLAUDE.md do projeto).
4. **`AGENTS.md` de subdiretório** (ex: `<REPO>/app/javascript/AGENTS.md`) — Ativa apenas ao trabalhar dentro desse diretório. Use para escopo de regras específicas do frontend.

Use importações `@filename` para compor instruções a partir de arquivos externos:

```markdown
# ~/.claude/CLAUDE.md

@RTK.md

# Diretrizes Globais
- Sempre execute `/graphify query` para qualquer questão arquitetural ou
  comportamental antes de abrir arquivos de código.
- Prefira usar comandos `rtk` em vez de utilitários de terminal nativos
  para manter o tamanho do payload otimizado.
```

A importação `@RTK.md` puxa um arquivo separado (`~/.claude/RTK.md`) com regras específicas do RTK, mantendo o arquivo principal enxuto.

### Passo 2: Configure o Sistema de Hooks

Os hooks são o mecanismo chave que transforma o Claude Code de um assistente passivo em um agente auto-otimizante. Eles são configurados em `~/.claude/settings.json` e disparam em eventos específicos do ciclo de vida:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/rtk-rewrite.sh"
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/graphify-nudge.sh"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.claude/skills/obsidian-second-brain/hooks/load_vault_context.py"
          }
        ]
      }
    ],
    "PostCompact": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/skills/obsidian-second-brain/hooks/obsidian-bg-agent.sh",
            "timeout": 10,
            "async": true
          }
        ]
      }
    ]
  }
}
```

Cada hook tem um propósito distinto:

| Evento do Hook | O que Faz |
|---|---|
| **PreToolUse** (Bash) | Intercepta cada comando shell e o reescreve via RTK para otimização de tokens — completamente transparente |
| **UserPromptSubmit** | Analisa seu prompt para questões de comportamento/arquitetura e injeta um lembrete para consultar o Graphify primeiro |
| **SessionStart** | Carrega o contexto do cofre Obsidian (`_CLAUDE.md`) na sessão quando se trabalha dentro do diretório do cofre |
| **PostCompact** | Quando a janela de contexto é compactada, gera um agente em segundo plano que extrai itens relevantes do transcript para o cofre |

### Passo 3: Instale Plugins

O Claude Code suporta um sistema de plugins para integrar ferramentas externas. Ative os plugins em `settings.json`:

```json
{
  "enabledPlugins": {
    "headroom@headroom-marketplace": true,
    "codex@openai-codex": true,
    "obsidian@obsidian-skills": true,
    "discord@claude-plugins-official": true
  }
}
```

Plugins adicionam ferramentas MCP, hooks, skills e tipos de subagente. O plugin Codex, por exemplo, registra o subagente `codex:codex-rescue` e vários comandos slash (`/codex:rescue`, `/codex:review`, `/codex:status`).

### Passo 4: Configure as Skills

Skills são conjuntos empacotados de instruções que ativam com comandos slash. Eles ficam em `~/.claude/skills/` como diretórios contendo um arquivo `SKILL.md`:

```
~/.claude/skills/
├── graphify/SKILL.md        # /graphify — código-para-grafo-de-conhecimento
├── obsidian-second-brain/   # /obsidian-* — gerenciamento do cofre
│   ├── SKILL.md
│   └── hooks/               # Hooks SessionStart + PostCompact
├── pr-review/SKILL.md       # /pr-review — revisão de PR multi-revisor paralelo
├── pull-request/SKILL.md    # /pull-request — criar PR com issue no Linear
├── resolve-fault/SKILL.md   # /resolve-fault — erro Honeybadger → PR
├── review-fix/SKILL.md      # /review-fix — auto-revisão e correção
├── ponytail -> ~/code/agent-skills/skills/ponytail  # symlink
├── rails-best-practices-core -> ...                 # symlink
└── ...
```

Skills com links simbólicos apontam para um repositório compartilhado `~/code/agent-skills/`, então atualizações no repositório de skills se propagam para todos os projetos sem cópia manual.

---

## Camada 2: Headroom (Proxy de Compressão de Contexto)

O **Headroom** é o maior economizador de tokens de toda a pilha. Ele roda como um proxy HTTP local que comprime tudo que flui entre seu agente e a API do modelo — saídas de ferramentas, conteúdo de arquivos, histórico de conversas — antes de chegar ao LLM.

### Como Funciona

O Headroom intercepta chamadas de API ficando entre o Claude Code e a API da Anthropic. Você aponta o Claude Code para o proxy em vez da API real:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:8787"
  }
}
```

Essa única variável de ambiente roteia todas as chamadas do modelo através do pipeline de compressão do Headroom. Sem mudanças no código, sem scripts wrapper — o Claude Code pensa que está conversando diretamente com a Anthropic.

### Passo 1: Instale e Inicie

```bash
pip install "headroom-ai[all]"
headroom proxy --port 8787
```

Ou envolva seu agente diretamente:

```bash
headroom wrap claude
```

### Passo 2: Entenda o Pipeline de Compressão

O Headroom aplica múltiplas estratégias de compressão baseadas no tipo de conteúdo:

```text
 Prompt do agente / saída de ferramenta
        │
    ┌───┴──────────────────────────────────────────┐
    │  Headroom (roda localmente)                  │
    │  CacheAligner  →  ContentRouter  →  CCR      │
    │                    ├─ SmartCrusher   (JSON)   │
    │                    ├─ CodeCompressor (AST)    │
    │                    └─ Kompress-base  (prosa)  │
    └───┬──────────────────────────────────────────┘
        │  prompt comprimido + ferramenta de recuperação
        ▼
    Provedor de LLM
```

- **SmartCrusher** — Comprime JSON (saídas de ferramentas, respostas de API) removendo ruído estrutural
- **CodeCompressor** — Usa análise em nível de AST para remover código não essencial preservando o significado semântico
- **Kompress-base** — Um modelo HuggingFace que comprime linguagem natural preservando afirmações chave
- **CacheAligner** — Estabiliza prefixos de prompt para que os caches KV do provedor realmente sejam aproveitados, reduzindo ainda mais os custos
- **CCR (Compressed Context Retrieval)** — Armazena originais localmente; o LLM pode chamar `headroom_retrieve` se precisar da versão completa de algo que foi comprimido

### Passo 3: Verifique a Economia

```bash
headroom perf            # relatório de economia instantâneo
headroom dashboard       # dashboard ao vivo (proxy deve estar rodando)
```

Em cargas de trabalho reais de agente, o Headroom tipicamente entrega **redução de 47–92% nos tokens** dependendo do tipo de conteúdo:

| Carga de Trabalho | Antes | Depois | Economia |
|---|---:|---:|---:|
| Busca de código (100 resultados) | 17.765 | 1.408 | **92%** |
| Depuração de incidentes SRE | 65.694 | 5.118 | **92%** |
| Triagem de issues do GitHub | 54.174 | 14.761 | **73%** |
| Exploração de codebase | 78.502 | 41.254 | **47%** |

O Headroom também reduz **tokens de saída** — ele corta cerimônia e código replicado do que o modelo escreve de volta, não apenas do que você envia.

### Passo 4: Implantação Persistente com launchd

Para um proxy "configure e esqueça", implante-o como um LaunchAgent do macOS:

```xml
<!-- ~/Library/LaunchAgents/com.user.headroom-proxy.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.headroom-proxy</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/headroom</string>
        <string>proxy</string>
        <string>--port</string>
        <string>8787</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/headroom-proxy.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/headroom-proxy.err</string>
</dict>
</plist>
```

Carregue uma vez:

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.headroom-proxy.plist
```

Agora o Headroom inicia automaticamente no login e reinicia se travar.

---

## Camada 3: Conductor (Agentes de Worktree em Paralelo)

O **Conductor** é um aplicativo de orquestração que cria worktrees git paralelas sob um caminho canônico, como `~/conductor/workspaces/<project>/<workspace-name>/`. Cada espaço de trabalho hospeda uma sessão isolada do Claude Code. É assim que se escala de "um agente em uma tarefa" para "muitos agentes em muitas tarefas" sem conflitos de branch ou poluição de contexto.

### Passo 1: Configuração Global do Conductor

As configurações globais do Conductor ficam em `~/.conductor/settings.toml`. Isso configura qual modelo cada agente usa, convenções de git e caminhos para executáveis dos agentes:

```toml
"$schema" = "https://conductor.build/schemas/settings.schema.json"
claude_code_executable_path = "~/.local/bin/claude"
codex_executable_path = "/opt/homebrew/bin/codex"

[git]
branch_prefix_type = "github_username"
delete_branch_on_archive = true

[models]
default = "opus-4-6-1m"
review = "opus-4-6-1m"

[models.codex]
default_thinking_level = "high"
review_thinking_level = "high"
```

Configurações chave:
- **`branch_prefix_type = "github_username"`** — Branches recebem automaticamente o prefixo do seu handle do GitHub para fácil identificação
- **`delete_branch_on_archive`** — Branches limpos não poluem seu remoto
- **Seleção de modelo** — Diferentes modelos para diferentes tarefas. A variante de contexto 1M é essencial para bases de código grandes

### Passo 2: Scripts de Setup por Projeto

Cada projeto tem um arquivo `.conductor/settings.local.toml` (gitignored, privado da sua máquina) que define um **script de setup**. Este script roda automaticamente quando o Conductor cria uma nova worktree:

```toml
# <REPO>/.conductor/settings.local.toml
"$schema" = "https://conductor.build/schemas/settings.repo.schema.json"
file_include_globs = ".env*\n.gitignore\n"

[scripts]
run = "dev"
run_mode = "concurrent"
setup = """#!/usr/bin/env bash
set -euo pipefail

ROOT="$(git rev-parse --show-toplevel)"
MAIN_REPO="${CONDUCTOR_ROOT_PATH:-$HOME/code/my-project}"

# Garantir que o PATH inclui gerenciadores de versão de linguagem
export RBENV_ROOT="${RBENV_ROOT:-$HOME/.rbenv}"
export PATH="$RBENV_ROOT/shims:$RBENV_ROOT/bin:$PATH"

if [[ -s "$HOME/.nvm/nvm.sh" ]]; then
  export NVM_DIR="${NVM_DIR:-$HOME/.nvm}"
  source "$NVM_DIR/nvm.sh"
  nvm use 22 --silent || nvm install 22 --no-progress
fi

echo "Root: $ROOT"

# Criar link simbólico do graphify-out do repo principal nesta worktree
if [[ "$ROOT" != "$MAIN_REPO" ]]; then
  GRAPHIFY_BIN="$HOME/.local/bin/graphify"
  if [[ ! -d "$MAIN_REPO/graphify-out" && -x "$GRAPHIFY_BIN" ]]; then
    echo "graphify-out ausente - reconstruindo (somente AST, sem LLM)"
    ( cd "$MAIN_REPO" && "$GRAPHIFY_BIN" update . ) \\
      || echo "reconstrução do graphify falhou - worktree ficará sem grafo"
  fi
  if [[ -d "$MAIN_REPO/graphify-out" && ! -e "$ROOT/graphify-out" ]]; then
    ln -s "$MAIN_REPO/graphify-out" "$ROOT/graphify-out"
    echo "graphify-out vinculado de $MAIN_REPO"
  fi
fi

# Instalar dependências
bundle install
yarn install

# Copiar credenciais do repo principal (se aplicável)
SRC_KEY="$MAIN_REPO/config/credentials/development.key"
DEST_KEY="$ROOT/config/credentials/development.key"
if [[ -f "$SRC_KEY" ]]; then
  mkdir -p "$(dirname "$DEST_KEY")"
  cp "$SRC_KEY" "$DEST_KEY"
  echo "development.key copiada"
fi
"""
```

Decisões de design chave:
- **Script inline no TOML** — O script de setup é embutido diretamente no arquivo de configurações. Toda a config do workspace é autocontida.
- **Symlink do Graphify** — Cada worktree compartilha um único `graphify-out/` canônico do repo principal. Se o grafo estiver ausente, ele reconstrói automaticamente (somente AST, sem LLM).
- **`file_include_globs`** — O Conductor copia arquivos `.env*` e `.gitignore` para novas worktrees, garantindo que a configuração do ambiente esteja sempre presente.

### Passo 3: Trabalhando com Espaços de Trabalho

O Conductor nomeia cada espaço de trabalho com o nome de uma cidade. Quando você abre o app Conductor e cria um novo workspace para um projeto, você recebe algo como:

```
~/conductor/workspaces/my-project/
├── tokyo/       ← feature branch A
├── berlin/      ← bug fix B
├── nairobi/     ← refatoração C
└── amsterdam/   ← tarefa de revisão D
```

Cada um tem seu próprio branch, sua própria sessão Claude e seu próprio servidor de desenvolvimento rodando. O script de setup roda uma vez na criação, garantindo que cada workspace esteja imediatamente pronto para trabalho sem setup manual.

---

## Camada 4: Graphify (Código para Grafo de Conhecimento)

Em vez de enviar centenas de linhas de código bruto para o modelo procurar um bug, usamos o **Graphify** — uma ferramenta de CLI que constrói um grafo de conhecimento local da sua base de código com detecção de comunidades, trilha de auditoria (arestas EXTRACTED/INFERRED/AMBIGUOUS) e múltiplos modos de consulta.

### Passo 1: Construa o Grafo

```bash
graphify .                              # pipeline completo no diretório atual
graphify . --mode deep                  # extração profunda, arestas mais ricas
graphify . --update                     # incremental - apenas arquivos novos/alterados
graphify . --obsidian --obsidian-dir ~/vaults/my-project-graph  # exportar para Obsidian
```

O Graphify produz dois artefatos chave:
1. **Grafo de Máquina** (`graphify-out/graph.json` + `GRAPH_REPORT.md`) — Consultado diretamente pelo Claude Code
2. **Exportação Visual** — Um cofre Obsidian com arquivos markdown interligados e um `graph.canvas` mostrando relações entre controllers, models, jobs e serviços

### Passo 2: Consulte o Grafo pelo Claude

O Claude pode consultar o grafo diretamente via a skill `/graphify query`:

```bash
/graphify query "Como funciona a autenticação neste app?"       # BFS - contexto amplo
/graphify query "Trace o fluxo de pagamento" --dfs              # DFS - segue um caminho profundo
/graphify query "O que chama o UserMailer?" --budget 1500       # limitar resposta a N tokens
/graphify path "AuthModule" "Database"                          # caminho mais curto entre conceitos
/graphify explain "BackgroundWorker"                            # explicação de um nó em linguagem natural
```

### Passo 3: Auto-Imposição Graphify-First com um Hook

O verdadeiro poder é fazer o Claude usar o Graphify automaticamente. O hook `graphify-nudge.sh` (registrado como hook `UserPromptSubmit`) analisa cada prompt que você submete em busca de questões de comportamento/arquitetura:

```bash
#!/bin/bash
# ~/.claude/hooks/graphify-nudge.sh
set -u

INPUT=$(cat)
PROMPT=$(echo "$INPUT" | jq -r '.prompt // empty')
CWD=$(echo "$INPUT" | jq -r '.cwd // .working_directory // empty')

# Dispara apenas quando graphify-out existe no projeto atual
if [ ! -f "$CWD/graphify-out/graph.json" ]; then
  exit 0
fi

# Pattern-match para questões de comportamento/fluxo/arquitetura
if echo "$PROMPT" | grep -qiE \
  'how (does|do|is|are|can)|what calls|where is .+ (used|called)|why does|walk me through|explain (the|how)|architecture|data ?flow|control ?flow|trace the|life ?cycle|end-to-end'; then

  jq -n '{
    "hookSpecificOutput": {
      "hookEventName": "UserPromptSubmit",
      "additionalContext": "GRAPHIFY REMINDER: Uma questão de comportamento/fluxo foi detectada e graphify-out/graph.json está presente. Sua PRIMEIRA chamada de ferramenta DEVE ser /graphify query. Só use Read/Grep nos arquivos que o grafo indicar."
    }
  }'
fi
```

Quando você pergunta "Como funciona a autenticação?", o hook injeta um lembrete contextual no prompt que força o Claude a consultar o grafo primeiro e depois ler apenas os arquivos específicos identificados pelo grafo.

### Passo 4: Automatize com um Agente launchd Semanal

Para garantir que o grafo nunca fique desatualizado, configure uma tarefa de segundo plano do launchd no macOS:

```xml
<!-- ~/Library/LaunchAgents/com.user.graphify-update.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.graphify-update</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>~/.local/bin/graphify-update.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Weekday</key>
        <integer>1</integer>
        <key>Hour</key>
        <integer>9</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
</dict>
</plist>
```

O script complementar:

```bash
#!/bin/bash
# ~/.local/bin/graphify-update.sh

REPO_PATH="$HOME/code/my-project"
VAULT_PATH="$HOME/vaults/my-project-graph"

cd "$REPO_PATH"
graphify update .
graphify export obsidian --dir "$VAULT_PATH"

echo "Atualização do Graphify concluída em $(date)"
```

---

## Camada 5: Obsidian como um "Segundo Cérebro"

Uma decisão de design chave: **nada durável é escrito na memória embutida do Claude**. A memória embutida do modelo é transitória e difícil de versionar. Em vez disso, um cofre dedicado do Obsidian hospeda toda a memória durável entre projetos, e um par de hooks mantém o cofre sincronizado com suas sessões de agente automaticamente.

### Passo 1: Imponha o Manual de Operação AI-First

Crie um arquivo `_CLAUDE.md` na raiz do seu cofre. Este arquivo é automaticamente carregado nas sessões Claude via o hook `SessionStart` quando se trabalha dentro do cofre:

```markdown
# _CLAUDE.md (raiz do cofre)

## Regras de Escrita de Notas AI-First:
1. **Auto-Contido**: Cada nota deve ser compreensível sem requerer outro contexto.
2. **Preâmbulo de Contexto**: Cada nota começa com um resumo de 2-3 frases para futuras sessões de IA.
3. **Frontmatter Rico**:
   ```yaml
   type: logic-explainer
   date: 2026-07-17
   topic: Authentication
   confidence: high
   ai-first: true
   ```
4. **Marcadores de Tempo**: Referencie eventos com datas precisas: "A partir de 2026-07...".
5. **Links Wiki**: Conecte entidades com `[[wikilinks]]` para o Conductor/Claude analisar o grafo.
```

### Passo 2: O Hook SessionStart (Carregador de Contexto do Cofre)

O script `load_vault_context.py` dispara no início da sessão e verifica se o diretório de trabalho atual está dentro do cofre Obsidian. Se sim, injeta o `_CLAUDE.md` do cofre no contexto do agente:

```python
# ~/.claude/skills/obsidian-second-brain/hooks/load_vault_context.py

import json, os, sys
from pathlib import Path

def main():
    vault = os.environ.get("OBSIDIAN_VAULT_PATH", "")
    if not vault:
        return 0

    payload = json.load(sys.stdin)
    cwd = payload.get("cwd", "")

    # Injeta contexto apenas quando trabalhando dentro do cofre
    if not cwd.startswith(vault):
        return 0

    claude_md = Path(vault) / "_CLAUDE.md"
    if not claude_md.is_file():
        return 0

    content = claude_md.read_text(encoding="utf-8")
    output = {
        "hookSpecificOutput": {
            "hookEventName": "SessionStart",
            "additionalContext": f"Manual operacional do cofre:\n\n{content}"
        }
    }
    json.dump(output, sys.stdout)
    return 0

if __name__ == "__main__":
    raise SystemExit(main())
```

Configure o caminho do cofre em `settings.json`:

```json
{
  "env": {
    "OBSIDIAN_VAULT_PATH": "/caminho/para/seu/cofre",
    "OBSIDIAN_BG_AGENT_ENABLED": "1"
  }
}
```

### Passo 3: O Hook PostCompact (Agente de Cofre em Segundo Plano)

Este é o hook mais poderoso do sistema. Quando a janela de contexto do Claude enche e é compactada (resumida), o hook `PostCompact` gera uma **sessão Claude em segundo plano** que lê o transcript compactado e propaga tudo que vale preservar para o cofre:

```bash
#!/bin/bash
# ~/.claude/skills/obsidian-second-brain/hooks/obsidian-bg-agent.sh

VAULT="${OBSIDIAN_VAULT_PATH:-}"
[[ -z "$VAULT" ]] && exit 0
[[ "${OBSIDIAN_BG_AGENT_ENABLED:-0}" != "1" ]] && exit 0

INPUT=$(cat)
TRANSCRIPT=$(printf '%s' "$INPUT" | jq -r '.transcript_path // ""')
[[ -z "$TRANSCRIPT" || ! -f "$TRANSCRIPT" ]] && exit 0

# Extrair o resumo de compactação do transcript
SUMMARY=$(jq -rc 'select(.isCompactSummary == true) | .message.content' \
  "$TRANSCRIPT" | tail -n 1)
[[ -z "$SUMMARY" ]] && exit 0

TODAY=$(date +%Y-%m-%d)

# Construir o prompt para o agente em segundo plano
PROMPT="Você é um agente autônomo do cofre Obsidian. A sessão Claude acabou de
ser compactada. Propague tudo que vale preservar do resumo para o cofre.

VAULT: $VAULT
TODAY: $TODAY

RESUMO DA SESSÃO:
$SUMMARY

INSTRUÇÕES:
1. Leia _CLAUDE.md na raiz do cofre - siga suas regras exatamente.
2. Identifique itens dignos do cofre: decisões, tarefas, pessoas, projetos,
   trabalho de dev, ideias, aprendizados.
3. Antes de criar qualquer nota, procure uma existente. Nunca duplique.
4. Atualize a nota diária de hoje com links para tudo que você tocou.
5. Propague: nada é salvo isoladamente - cada escrita repercute na nota
   diária, boards e notas vinculadas."

# Gerar o agente em segundo plano (fire-and-forget)
(cd "$VAULT" && claude --dangerously-skip-permissions -p "$PROMPT" \
  >> /tmp/obsidian-bg-agent.log 2>&1) &

exit 0
```

Isso significa que toda sessão longa do Claude automaticamente captura decisões, mudanças no código e contexto do projeto no cofre — sem você fazer nada.

### Passo 4: O Catálogo de Skills do Obsidian

A skill `obsidian-second-brain` expõe um rico conjunto de comandos slash para interação manual com o cofre:

| Comando | Propósito |
|---|---|
| `/obsidian-save` | Salvar destaques da conversa no cofre |
| `/obsidian-find <tópico>` | Busca inteligente no cofre com contexto |
| `/obsidian-log` | Registrar uma sessão de dev no cofre |
| `/obsidian-daily` | Criar/atualizar a nota diária de hoje |
| `/obsidian-project` | Criar/atualizar uma nota de projeto |
| `/obsidian-person` | Criar/atualizar uma nota de pessoa |
| `/obsidian-task` | Adicionar uma tarefa ao board kanban correto |
| `/obsidian-decide` | Registrar uma decisão (leve ou ADR formal) |
| `/obsidian-recap` | Resumir um período do histórico do cofre |
| `/obsidian-health` | Verificação de saúde do cofre — contradições, lacunas, obsolescência |
| `/obsidian-synthesize` | Auto-scan para padrões não nomeados, escrever páginas de síntese |

### Passo 5: Configure o Plugin Obsidian Git para Controle de Versão

1. No Obsidian, vá em **Plugins de Comunidade**, pesquise por **Obsidian Git** e instale.
2. Configure os intervalos de backup para serem acionados a cada 5 a 10 minutos. Isso fornece um registro de memória com carimbo de data/hora de como suas notas e raciocínio evoluem ao longo do tempo.

---

## Camada 6: Codex CLI como Subagente de Segunda Opinião

Às vezes o Claude fica preso em um loop recursivo ou precisa de uma segunda perspectiva. Para isso, o **Codex CLI** (OpenAI) roda como um plugin do Claude Code que registra um subagente chamado `codex:codex-rescue`.

### Passo 1: Instale o Plugin

O plugin Codex é instalado pelo sistema de marketplace de plugins do Claude Code. Uma vez habilitado, ele registra:

- **`codex:codex-rescue`** — Um subagente que encaminha tarefas ao Codex CLI para investigação independente
- **`/codex:rescue`** — Comando slash para delegação manual
- **`/codex:review`** — Solicitar ao Codex uma revisão de código adversarial
- **`/codex:status`** — Verificar o status de uma tarefa Codex em segundo plano

### Passo 2: Configure o Codex no Conductor

As configurações globais do Conductor especificam o caminho do executável Codex e níveis de pensamento:

```toml
# ~/.conductor/settings.toml
codex_executable_path = "/opt/homebrew/bin/codex"

[models.codex]
default_thinking_level = "high"
review_thinking_level = "high"
```

### Passo 3: Como o Agente de Resgate Funciona

O agente `codex-rescue` é um wrapper fino de encaminhamento. Quando o Claude encontra um bloqueio, ele gera uma tarefa Codex através do runtime compartilhado:

```
Claude (preso em um bug)
  → gera subagente codex:codex-rescue
    → subagente chama: node codex-companion.mjs task "<prompt>" --write
      → Codex CLI roda independentemente, lê/escreve código
        → resultado retorna ao contexto do Claude
```

Orientação de seleção embutida no agente:
- **Proativo** — Não espera o usuário pedir; dispara quando Claude está preso
- **Delimitado** — Não captura tarefas simples que a thread principal pode finalizar rapidamente
- **Com escrita** — Modo `--write` por padrão para que o Codex possa aplicar sua correção diretamente
- **Ciente de retomada** — Se continuando trabalho anterior, adiciona `--resume-last` automaticamente

---

## Camada 7: RTK (Rust Token Killer) — O Proxy Transparente

O **RTK** é um wrapper de CLI de alto desempenho escrito em Rust. Diferente do Headroom (que comprime no nível da API), o RTK opera no **nível de comando shell** — ele intercepta `git`, `cat`, `grep`, `find` e outras ferramentas CLI, remove ruído da saída e retorna um resultado otimizado em tokens.

### Passo 1: O Hook PreToolUse (Não Aliases)

O RTK é conectado ao Claude Code via um hook `PreToolUse` — não aliases do shell. Isso é crucial: o hook intercepta a ferramenta Bash no nível do Claude Code, reescreve o comando via `rtk rewrite`, e retorna a versão otimizada com uma concessão automática de permissão:

```bash
#!/bin/bash
# ~/.claude/hooks/rtk-rewrite.sh

# Ler a entrada da ferramenta do Claude Code
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
[ -z "$CMD" ] && exit 0

# Pedir ao RTK para reescrever o comando
REWRITTEN=$(rtk rewrite "$CMD" 2>/dev/null) || exit 0

# Se inalterado, deixar passar
[ "$CMD" = "$REWRITTEN" ] && exit 0

# Retornar o comando reescrito com auto-allow
ORIGINAL_INPUT=$(echo "$INPUT" | jq -c '.tool_input')
UPDATED_INPUT=$(echo "$ORIGINAL_INPUT" | jq --arg cmd "$REWRITTEN" '.command = $cmd')

jq -n \
  --argjson updated "$UPDATED_INPUT" \
  '{
    "hookSpecificOutput": {
      "hookEventName": "PreToolUse",
      "permissionDecision": "allow",
      "permissionDecisionReason": "RTK auto-rewrite",
      "updatedInput": $updated
    }
  }'
```

Na prática: quando o Claude roda `git status`, o hook reescreve para `rtk git status`. Quando o Claude roda `cat schema.rb`, o hook reescreve para `rtk read schema.rb`. O agente nunca sabe que aconteceu — ele só recebe uma saída menor e mais limpa.

### Passo 2: Use as Regras do RTK para Economizar Tokens

O arquivo `~/.claude/RTK.md` (importado via `@RTK.md`) ensina o Claude quando preferir comandos RTK:

- Para arquivos acima de **50KB**, use `rtk read <path>` para obter uma visualização filtrada estruturalmente
- Para busca de padrões: `rtk grep <pattern> <path>` em vez de `cat | grep`
- Para descoberta de arquivos: `rtk find` em vez do `find` nativo

### Passo 3: Monitore sua Economia

```bash
rtk gain              # Mostrar economia acumulada de tokens
rtk gain --history    # Mostrar histórico de economia por comando
rtk discover          # Analisar transcripts do Claude Code para oportunidades RTK perdidas
```

Resultados típicos: **redução de 40–90% nos tokens** em operações comuns como listagem de diretórios, diffs do git e leitura de arquivos.

---

## Juntando Tudo: O Fluxo de Desenvolvimento por IA

Quando você faz ao Claude uma pergunta complexa como *"Como funciona a autenticação dentro deste microsserviço?"*, veja como toda a pilha opera:

1. O **Conductor** criou este workspace como uma worktree isolada com seu próprio branch. O script de setup rodou automaticamente, criando o symlink de `graphify-out/` e instalando dependências.

2. O **Claude lê a cascata de instruções**: global `~/.claude/CLAUDE.md` → projeto `CLAUDE.md` → `@AGENTS.md` → quaisquer arquivos `AGENTS.md` de subdiretório.

3. O **hook UserPromptSubmit dispara.** `graphify-nudge.sh` detecta que sua pergunta é uma questão de comportamento/arquitetura e injeta um lembrete: "Sua PRIMEIRA chamada de ferramenta DEVE ser `/graphify query`."

4. O **Claude consulta o Graphify.** `/graphify query "Como funciona a autenticação?"` faz uma travessia BFS do `graphify-out/graph.json` e retorna os arquivos e relações específicos envolvidos.

5. O **Claude lê apenas os arquivos indicados.** O hook `PreToolUse` intercepta cada comando `cat` e `grep`, reescrevendo-os transparentemente via **RTK** para saída otimizada em tokens.

6. O **Headroom comprime toda a troca.** Cada chamada de API flui pelo proxy local na porta 8787, onde SmartCrusher comprime saídas de ferramentas e CodeCompressor remove código não essencial. O modelo vê 50–90% menos tokens sem perder conteúdo semântico.

7. Se o Claude encontra um bloqueio inesperado, ele **aciona `codex:codex-rescue`** — gerando uma tarefa Codex CLI para uma segunda opinião, que roda independentemente e retorna suas descobertas.

8. Uma vez que a solução é encontrada, o Claude usa **`/obsidian-save`** para escrever um registro permanente no cofre.

9. Quando a janela de contexto eventualmente enche e é compactada, o **hook PostCompact** dispara — gerando um agente em segundo plano que extrai decisões, mudanças no código e aprendizados do transcript para o cofre automaticamente.

10. O **Obsidian Git** auto-comita as mudanças do cofre a cada poucos minutos, preservando seu progresso entre sessões.

Ao combinar **otimização transparente de tokens em dois níveis** (Headroom na camada da API, RTK na camada do shell) com um **grafo de conhecimento local estruturado** (Graphify) e **memória persistente orientada por hooks** (Obsidian), essa configuração cria um espaço de trabalho de desenvolvimento que é rápido, consciente do contexto e eficiente em custos — com conhecimento se acumulando entre sessões em vez de evaporar.
