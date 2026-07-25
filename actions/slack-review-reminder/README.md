# slack-review-reminder

Roda em **schedule (cron)** e cutuca no Slack os PRs abertos parados **sem review**. Posta um card no mesmo padrão do [`slack-pr-notify`](../slack-pr-notify) (barra âmbar, autor **mencionado de verdade** via `user-map.json`) e, enquanto o PR seguir sem review, **re-cutuca a cada X min** como resposta na thread (mencionando o time de novo). Para assim que houver **qualquer review**, ou se o PR virar draft/fechar.

```
⏰ Pull Request · Aguardando review há 15m           ← 1º ping (mensagem nova)
┃ • <descrição de 1 linha em PT (Gemini; fallback = título)>
┃ ──────────────
┃ Repositório: <repo>
┃ PR: <link>
┃ Author: @autor
┃ Time responsável: @time, dá uma olhada 🙏
```
Re-ping (a cada `reping-every-minutes`): posta uma **nota + o permalink** da mensagem original (`⏰ Ainda sem review — PR aguardando há 45m. @time` + link), e o Slack **desdobra** a original embutida como citação — jeito "encaminhado". Ressurge no fim do canal, não fica escondido em thread.

## Quando age

Só em **horário comercial BRT**, seg–sex, dentro de `business-hours-start`..`business-hours-end` (default 09h–18h). **Pula** sábado, domingo e **feriado nacional** (BrasilAPI). Diferente do `weekend-holiday-merge-gate`, **sexta conta** como dia útil aqui (não tem risco de desembolso). Se a API de feriados cair, segue em frente (fail-open — um lembrete a mais é inofensivo).

## Filtros (não cutuca)

- PR **draft**
- base fora de `target-branches` (default `main`)
- autor em `ignore-authors` (default `dependabot[bot]`)
- PR de sync `main → staging` (bull-sync-bot)
- PR que **já tem qualquer review** (aprovado / mudanças pedidas / comentado)
- PR aberto há menos de `remind-after-minutes`

## Inputs

| Input | Obrigatório | Default | Descrição |
|-------|-------------|---------|-----------|
| `slack-bot-token` | sim* | `""` | Bot token (`xoxb-...`). Vazio → sai sem falhar. |
| `channel-id` | sim* | `""` | Canal (`C0XXXX`) onde postar. |
| `team-id` | não | `""` | User Group (`S0XXXXXXX`) mencionado no card e nas re-cutucadas. Vazio → omite. |
| `bar-color` | não | `#E8A317` | Cor da barra (âmbar). |
| `target-branches` | não | `main` | Bases de PR a considerar (vírgula/nova-linha, match exato). |
| `remind-after-minutes` | não | `15` | Minutos sem review antes do 1º lembrete. |
| `reping-every-minutes` | não | `30` | Intervalo entre re-cutucadas. |
| `ignore-authors` | não | `dependabot[bot]` | Logins a ignorar (vírgula/nova-linha). |
| `business-hours-start` | não | `9` | Hora inicial (BRT, inclusiva). |
| `business-hours-end` | não | `18` | Hora final (BRT, **exclusiva** — cutuca até 17h59). |
| `gemini-api-key` | não | `""` | Descrição de 1 linha em PT. Vazio → usa o título. |
| `gemini-model` | não | `gemini-2.5-flash` | Modelo do Google AI Studio. |

\* Sem `slack-bot-token`/`channel-id` a action não faz nada (não quebra).

## Estado

Guarda `ts` + última cutucada + contador num **comentário oculto do PR** (`<!-- slack-review-reminder: ... -->`) — por isso o caller precisa de `pull-requests: write`. Usa o `GITHUB_TOKEN` do próprio repo (varre só os PRs dele), então **não precisa de token cross-repo**: é um workflow por repo.

## Uso (caller)

```yaml
name: Lembrete de PR sem review
on:
  schedule:
    - cron: '*/15 12-21 * * 1-5'   # a cada 15 min, ~09h–18h BRT (UTC-3), seg–sex
  workflow_dispatch:               # roda manual pra testar
permissions:
  contents: read
  pull-requests: write             # guardar/atualizar o estado no comentário oculto
jobs:
  remind:
    runs-on: ubuntu-latest
    steps:
      - uses: BullCredTech/gh-composite-actions/actions/slack-review-reminder@main
        with:
          slack-bot-token: ${{ secrets.SLACK_BOT_TOKEN }}
          channel-id: C0XXXXXXXXX
          team-id: S0XXXXXXXXX
          gemini-api-key: ${{ secrets.GEMINI_API_KEY }}
```

> O cron do GitHub Actions pode **atrasar** sob carga e **desliga após 60 dias** sem commit no repo. Pra um lembrete é tolerável; se precisar de precisão, migrar pra EventBridge + Lambda.
