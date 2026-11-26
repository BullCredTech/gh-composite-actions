# slack-pr-notify

Ao **abrir um PR**, posta uma notificação no Slack (via Incoming Webhook) num card com **barra colorida lateral**. O título "Pull Request" fica **fora** do card (negrito), e o conteúdo dentro:

```
✅ Pull Request — checks OK              ← fora do card (negrito) + preview da notificação
┃ • <título do PR>
┃ ──────────────
┃ 📌 PR: <link>
┃ 📦 Repositório: <repo>
┃ 🙋 Aberto por: <nome do perfil GitHub; fallback = login>
┃ 🌿 Branch: <head> → <base>
┃ 👥 Time responsável: @time   ← menção que notifica (quando team-id é setado)
```
(`┃` = a barra colorida)

Só notifica quando **três condições** são satisfeitas:

1. **Branch de destino**: a base do PR está em `target-branches` (default: `main`). PR para outra branch → sai sem notificar.
2. **PR não está em draft**: draft é rascunho — sai sem notificar. A notificação começa quando o PR é marcado como pronto (evento `ready_for_review`), que posta o card do zero.
3. **Checks concluídos** (quando `wait-for-checks: true`, o default): a action faz *poll* de todos os check-runs e commit statuses do commit (head SHA) — **ignorando o próprio check run** do workflow de notificação — e só posta quando **todos passaram**. Se algum falhar, fica em silêncio (a menos que `notify-on-failure: true`).

O cabeçalho reflete o resultado: `✅ Pull Request — checks OK`, `❌ Pull Request — checks falharam` ou `🔔 Pull Request` (quando `wait-for-checks: false`).

Lê os dados do PR do contexto `github.event.pull_request`, então o workflow que chama **precisa rodar em `on: pull_request`**.

## Inputs

| Input | Obrigatório | Default | Descrição |
|-------|-------------|---------|-----------|
| `slack-webhook-url` | não* | `""` | Incoming Webhook do canal. Vazio → o job sai sem falhar (falha fechada). |
| `slack-bot-token` | não* | `""` | Bot token (`xoxb-...`). Presente → posta via `chat.postMessage`, guarda o `ts` num comentário oculto do PR e **edita a mensagem** conforme o PR evolui (Aguardando checks 🟠 → Aguardando review 🟡 → Aprovado 🔵 → Mergeado 🟢 / Fechado 🔴). Requer `channel-id` + eventos `closed`/`pull_request_review` no caller. Ausente → usa o webhook (sem edição). |
| `channel-id` | não* | `""` | ID do canal (`C0XXXX`) para o `chat.postMessage`. Obrigatório com `slack-bot-token`. |
| `ignore-checks` | não | `gate,weekend-holiday-merge-gate` | Nomes de check-runs a **ignorar** na espera/decisão (vírgula/nova-linha). Não contam como pendente nem falha — útil pra "merge gates" que falham de propósito (não viram ❌ na notificação). **Use o nome do check-run = nome do JOB** (não do workflow): o gate org-wide tem workflow `weekend-holiday-merge-gate` mas job `gate`, então o check-run é `gate`. Não afeta o branch protection. Default cobre ambos. |
| `team-id` | não | `""` | Slack User Group ID (`S0XXXXXXX`). Setado → menção `<!subteam^ID>` que **notifica** o time. Vazio → linha omitida. |
| `bar-color` | não | `#d4a72c` | Cor (hex) da barra lateral do card. Em falha de checks, usa vermelho automaticamente. |
| `target-branches` | não | `main` | Branches de destino (base do PR) que disparam notificação, separadas por vírgula e/ou nova-linha. Match exato. Base fora da lista → não notifica. |
| `wait-for-checks` | não | `true` | `true` → espera **todos** os checks do commit concluírem e só notifica se todos passarem. `false` → notifica na hora que o PR abre (comportamento antigo). |
| `checks-timeout` | não | `1800` | Tempo máximo (s) aguardando os checks. Estourou → sai sem notificar (falha fechada). |
| `checks-poll-interval` | não | `30` | Intervalo (s) entre as checagens. |
| `checks-initial-grace` | não | `60` | Janela (s) para os checks aparecerem. Se após esse tempo **nenhum** check existir, notifica (não há o que aguardar). |
| `notify-on-failure` | não | `false` | `false` → PR ainda não anunciado só aparece no canal depois que os checks passam; com checks falhando fica em silêncio (o próximo push re-tenta). `true` → modo bot posta logo em 🟠 e edita pra ❌ na falha; modo webhook posta um card vermelho. |
| `trigger-mode` | não | `poll` | `poll` → o run fica vivo consultando os checks até eles terminarem (cobra minutos durante a espera). `event` → olha os checks uma vez e sai; quem acorda a action quando um CI termina é o `workflow_run` do caller. Exige `slack-bot-token`. Ver [Modo event](#modo-event--não-espera-os-checks). |

\* Precisa de **`slack-webhook-url` OU `slack-bot-token`**. Sem nenhum, a action não posta nada (não quebra).

### Modo bot token — edita a mensagem conforme o PR evolui

Com `slack-bot-token` + `channel-id`, a action posta via `chat.postMessage` e guarda o `ts` (+ autor + descrição) num comentário oculto do PR. Depois **edita a mesma mensagem** em cada transição:

| Estado | Card | Barra |
|--------|------|-------|
| Aguardando checks | 🟠 `Pull Request · Aguardando checks · Author: …` | laranja |
| Aguardando review (checks ok) | 🟡 `Pull Request · Aguardando review · Author: …` | amarela |
| Checks falharam | ❌ `Pull Request · Checks falharam · Author: …` | vermelha |
| Aprovado (review) | 🔵 `Pull Request · Aprovado · Aguardando merge · Author: …` | azul |
| Mergeado | 🟢 `Pull Request · Mergeado · Author: …` | verde |
| Fechado (sem merge) | 🔴 `Pull Request · Fechado · Author: …` | vermelha |

Com `wait-for-checks: true` e `notify-on-failure: false` (os defaults), o PR **só é anunciado quando a CI fica verde**: a action espera os checks e só então posta o card, já em 🟡 Aguardando review. Se os checks falham, nada é postado — o PR não aparece no canal nem menciona o time — e o próximo push (`synchronize`) tenta de novo. PR **já anunciado** que recebe um push volta pra 🟠 enquanto a CI re-roda e vira 🟡 ou ❌ conforme o resultado (o time já viu o card; apagar confundiria).

Com `notify-on-failure: true`, o card abre em **🟠 Aguardando checks** e é editado pra 🟡 ou ❌ quando a CI conclui. Com `wait-for-checks: false`, abre direto em 🟡.

Antes de postar/editar com o resultado dos checks, a action re-consulta o PR: se ele foi mergeado/fechado (🟢/🔴 é terminal) ou se o head mudou (push novo — o run mais novo decide), o run sai sem mexer no card. PR que nunca teve CI verde e é mergeado mesmo assim não aparece no canal (review/merge só editam card existente).

Pra isso o caller precisa:

- disparar em **`pull_request`** (`types: [opened, reopened, synchronize, closed, ready_for_review]`) **e** em **`pull_request_review`** (`types: [submitted]`) — **`ready_for_review` é obrigatório**: é ele que anuncia o PR que nasceu draft; sem ele, marcar o PR como pronto não dispara run nenhum e o PR nunca aparece no canal. `converted_to_draft` virou opcional — o run dispara e sai sem fazer nada. Um PR aberto pronto e devolvido a draft mantém o card no último estado vivo até fechar;
- permissão **`pull-requests: write`** (pra gravar/ler o `ts` no comentário oculto);
- passar `slack-bot-token` + `channel-id`.

```yaml
name: Notify Slack on PR
on:
  pull_request:
    types: [opened, reopened, synchronize, closed, ready_for_review]
  pull_request_review:
    types: [submitted]         # edita pra "aprovado" quando aprovarem
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
          team-id: S0AT0TS4QTD
```

> Sem `slack-bot-token`, tudo segue igual ao webhook (sem edição no merge) — retrocompatível.

### Modo event — não espera os checks

Com `trigger-mode: event`, nenhum run fica parado esperando a CI. Cada run olha os checks do commit **uma vez**:

- **Algum check ainda rodando** (ou nenhum check criado ainda): sai na hora. O próximo workflow que terminar acorda a action de novo.
- **Todos terminaram**: posta o card em 🟡 (ou edita pra 🟡/❌, se o PR já foi anunciado), com as mesmas regras do modo `poll`.

Quem acorda a action quando um check termina é o evento `workflow_run` no caller, com **os workflows de CI do repo listados pelo nome**. O evento só dispara com o workflow já na `main`.

```yaml
name: Notify Slack on PR
on:
  pull_request:
    types: [opened, reopened, synchronize, closed, ready_for_review]
    branches: [main]
  pull_request_review:
    types: [submitted]
  workflow_run:
    workflows: ["CI", "Security scans"]   # todos os workflows de checks do repo
    types: [completed]

# Avaliação dos checks (push no PR + CI terminando) roda uma por vez por commit:
# dois CIs terminando juntos não postam o card duas vezes. Review e closed só
# editam o card e ficam fora da fila (grupo próprio).
concurrency:
  group: slack-pr-notify-${{ (github.event_name == 'workflow_run' && github.event.workflow_run.head_sha) || (github.event_name == 'pull_request' && github.event.action != 'closed' && github.event.pull_request.head.sha) || github.run_id }}
  cancel-in-progress: false

permissions:
  contents: read
  checks: read
  statuses: read
  pull-requests: write

jobs:
  notify:
    # workflow_run de push na main, de CI cancelado ou de PR de fork (roda com os secrets): pula antes de subir o runner.
    if: >-
      github.actor != 'dependabot[bot]' &&
      (github.event_name != 'workflow_run' ||
       (github.event.workflow_run.event == 'pull_request' && github.event.workflow_run.conclusion != 'cancelled' &&
        github.event.workflow_run.head_repository.full_name == github.repository))
    runs-on: ubuntu-latest
    steps:
      - uses: BullCredTech/gh-composite-actions/actions/slack-pr-notify@<sha>
        with:
          slack-bot-token: ${{ secrets.SLACK_BOT_TOKEN }}
          channel-id: C0XXXXXXX
          trigger-mode: event
```

Limites do modo event:

- **Workflow fora da lista não acorda a action.** Se ele (ou um check de fora do Actions, como o `org-security` da org) for o último a terminar, o card só é decidido no próximo evento do PR. Liste todos os workflows de checks do repo e revise a lista ao criar um novo.
- **Repo sem nenhum workflow de checks** nunca anuncia o PR: use `trigger-mode: poll`.
- **Só funciona com bot token**: no modo webhook não há card editável pra saber se outro run já postou.

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
          team-id: S0AT0TS4QTD   # frontend-team (backend-team = S0AT79KD1PC)
          # target-branches: main        # default; use "main,develop" p/ mais de uma
          # wait-for-checks: "true"      # default; "false" volta ao comportamento antigo
```

## Notas

- **Filtro de branch**: só notifica PRs cuja **base** (`github.event.pull_request.base.ref`) esteja em `target-branches` (default `main`). Isso evita ruído de PRs entre branches de trabalho.
- **Espera de checks** (`wait-for-checks: true`): o job faz *poll* dos endpoints `commits/{sha}/check-runs` e `commits/{sha}/status`. Ignora os check runs do **próprio** workflow run (identificados pelo `details_url` que contém `/actions/runs/<RUN_ID>/`), evitando esperar por si mesmo. Só posta quando não há nenhum check pendente **nem** com falha.
  - **Enquanto espera, o runner fica ativo** consumindo minutos de Actions (até `checks-timeout`). Para não pagar a espera, use `trigger-mode: event`.
  - **Timeout**: se estourar `checks-timeout` com checks ainda pendentes, **não notifica** (falha fechada) — melhor não anunciar "verde" sem confirmar.
  - **Sem checks**: se após `checks-initial-grace` não existir nenhum check para o commit, notifica assim mesmo (não há o que aguardar).
  - **Permissões**: requer `checks: read` e `statuses: read` no caller (além de `contents: read`).
- A descrição é o **título do PR**.
- **Aberto por** usa o **nome do perfil GitHub** (`.name` via API pública `/users/{login}`), com **fallback pro login**. Usa o `github.token` só pra evitar rate limit.
- A **barra colorida** usa `attachments`; por causa disso o Slack mostra um rótulo automático "Adicionado por \<app\>" no rodapé (atribuição do app do webhook — não removível pelo webhook).
- **Falha fechada**: sem `slack-webhook-url` sai sem erro.
- Não requer `actions/checkout`.
