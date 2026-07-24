# slack-pr-notify

Ao **abrir um PR**, posta uma notificação no Slack (via Incoming Webhook) num card com **barra colorida lateral**. O título "Pull Request" fica **fora** do card (negrito), e o conteúdo dentro:

```
✅ Pull Request — checks OK              ← fora do card (negrito) + preview da notificação
┃ • <descrição de 1 linha em PT (Gemini; fallback = título do PR)>
┃ ──────────────
┃ 📌 PR: <link>
┃ 📦 Repositório: <repo>
┃ 🙋 Aberto por: <nome do perfil GitHub; fallback = login>
┃ 🌿 Branch: <head> → <base>
┃ 👥 Time responsável: @time   ← menção que notifica (quando team-id é setado)
```
(`┃` = a barra colorida)

Só notifica quando **duas condições** são satisfeitas:

1. **Branch de destino**: a base do PR está em `target-branches` (default: `main`). PR para outra branch → sai sem notificar.
2. **Checks concluídos** (quando `wait-for-checks: true`, o default): a action faz *poll* de todos os check-runs e commit statuses do commit (head SHA) — **ignorando o próprio check run** do workflow de notificação — e só posta quando **todos passaram**. Se algum falhar, fica em silêncio (a menos que `notify-on-failure: true`).

O cabeçalho reflete o resultado: `✅ Pull Request — checks OK`, `❌ Pull Request — checks falharam`, `⏰ Pull Request (Draft)` ou `🔔 Pull Request` (quando `wait-for-checks: false`).

Lê os dados do PR do contexto `github.event.pull_request`, então o workflow que chama **precisa rodar em `on: pull_request`**.

## Inputs

| Input | Obrigatório | Default | Descrição |
|-------|-------------|---------|-----------|
| `slack-webhook-url` | não* | `""` | Incoming Webhook do canal. Vazio → o job sai sem falhar (falha fechada). |
| `slack-bot-token` | não* | `""` | Bot token (`xoxb-...`). Presente → posta via `chat.postMessage`, guarda o `ts` num comentário oculto do PR e **edita a mensagem no merge** (🔔 → ✅). Requer `channel-id` + evento `closed` no caller. Ausente → usa o webhook (sem edição no merge). |
| `channel-id` | não* | `""` | ID do canal (`C0XXXX`) para o `chat.postMessage`. Obrigatório com `slack-bot-token`. |
| `team-id` | não | `""` | Slack User Group ID (`S0XXXXXXX`). Setado → menção `<!subteam^ID>` que **notifica** o time. Vazio → linha omitida. |
| `bar-color` | não | `#4A90D9` | Cor (hex) da barra lateral do card. Em falha de checks, usa vermelho automaticamente. |
| `gemini-api-key` | não | `""` | API key do Google AI Studio para a descrição de 1 linha em PT. Vazio → usa o título do PR. |
| `gemini-model` | não | `gemini-2.5-flash` | Modelo do Google AI Studio. |
| `target-branches` | não | `main` | Branches de destino (base do PR) que disparam notificação, separadas por vírgula e/ou nova-linha. Match exato. Base fora da lista → não notifica. |
| `wait-for-checks` | não | `true` | `true` → espera **todos** os checks do commit concluírem e só notifica se todos passarem. `false` → notifica na hora que o PR abre (comportamento antigo). |
| `checks-timeout` | não | `1800` | Tempo máximo (s) aguardando os checks. Estourou → sai sem notificar (falha fechada). |
| `checks-poll-interval` | não | `30` | Intervalo (s) entre as checagens. |
| `checks-initial-grace` | não | `60` | Janela (s) para os checks aparecerem. Se após esse tempo **nenhum** check existir, notifica (não há o que aguardar). |
| `notify-on-failure` | não | `false` | `true` → posta um card ❌ quando os checks falham. `false` → silêncio na falha. |

\* Precisa de **`slack-webhook-url` OU `slack-bot-token`**. Sem nenhum, a action não posta nada (não quebra).

### Modo bot token — edita a mensagem no merge (🔔 → ✅)

Com `slack-bot-token` + `channel-id`, a action posta via `chat.postMessage` e guarda o `ts` da mensagem num comentário oculto do PR. Quando o PR é **mergeado**, a mesma mensagem é **editada** para ✅. Pra isso o caller precisa:

- disparar também no evento **`closed`** (`types: [opened, ..., closed]`);
- permissão **`pull-requests: write`** (pra gravar/ler o `ts` no comentário oculto);
- passar `slack-bot-token` + `channel-id`.

```yaml
name: Notify Slack on PR
on:
  pull_request:
    types: [opened, reopened, ready_for_review, closed]
permissions:
  contents: read
  checks: read
  statuses: read
  pull-requests: write         # guardar/ler o ts da mensagem
jobs:
  notify:
    if: github.actor != 'dependabot[bot]'
    runs-on: ubuntu-latest
    steps:
      - uses: BullCredTech/gh-composite-actions/actions/slack-pr-notify@v1.6.0
        with:
          slack-bot-token: ${{ secrets.SLACK_BOT_TOKEN }}
          channel-id: ${{ vars.SLACK_PR_CHANNEL_ID }}
          gemini-api-key: ${{ secrets.GEMINI_API_KEY }}
          team-id: S0AT0TS4QTD
```

> Sem `slack-bot-token`, tudo segue igual ao webhook (sem edição no merge) — retrocompatível.

## Uso

```yaml
name: Notify Slack on PR
on:
  pull_request:
    types: [opened]          # com wait-for-checks, notifica quando a CI do PR fica verde
permissions:
  contents: read
  checks: read               # ler check-runs do commit
  statuses: read             # ler commit statuses
jobs:
  notify:
    if: github.actor != 'dependabot[bot]'   # ignora PRs de bots
    runs-on: ubuntu-latest
    steps:
      - uses: BullCredTech/gh-composite-actions/actions/slack-pr-notify@v1.5.0
        with:
          slack-webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
          gemini-api-key: ${{ secrets.GEMINI_API_KEY }}
          team-id: S0AT0TS4QTD   # frontend-team (backend-team = S0AT79KD1PC)
          # target-branches: main        # default; use "main,develop" p/ mais de uma
          # wait-for-checks: "true"      # default; "false" volta ao comportamento antigo
```

## Notas

- **Filtro de branch**: só notifica PRs cuja **base** (`github.event.pull_request.base.ref`) esteja em `target-branches` (default `main`). Isso evita ruído de PRs entre branches de trabalho.
- **Espera de checks** (`wait-for-checks: true`): o job faz *poll* dos endpoints `commits/{sha}/check-runs` e `commits/{sha}/status`. Ignora os check runs do **próprio** workflow run (identificados pelo `details_url` que contém `/actions/runs/<RUN_ID>/`), evitando esperar por si mesmo. Só posta quando não há nenhum check pendente **nem** com falha.
  - **Enquanto espera, o runner fica ativo** consumindo minutos de Actions (até `checks-timeout`). Ajuste `checks-poll-interval`/`checks-timeout` conforme a duração típica da sua CI.
  - **Timeout**: se estourar `checks-timeout` com checks ainda pendentes, **não notifica** (falha fechada) — melhor não anunciar "verde" sem confirmar.
  - **Sem checks**: se após `checks-initial-grace` não existir nenhum check para o commit, notifica assim mesmo (não há o que aguardar).
  - **Permissões**: requer `checks: read` e `statuses: read` no caller (além de `contents: read`).
- A descrição é **1 linha em PT gerada pelo Gemini** (título + descrição do PR); **fallback = título** se a IA falhar ou faltar a chave.
- **Aberto por** usa o **nome do perfil GitHub** (`.name` via API pública `/users/{login}`), com **fallback pro login**. Usa o `github.token` só pra evitar rate limit.
- A **barra colorida** usa `attachments`; por causa disso o Slack mostra um rótulo automático "Adicionado por \<app\>" no rodapé (atribuição do app do webhook — não removível pelo webhook).
- **Falha fechada**: sem `slack-webhook-url` sai sem erro.
- Não requer `actions/checkout`.
