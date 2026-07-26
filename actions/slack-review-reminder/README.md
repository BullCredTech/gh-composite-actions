# slack-review-reminder

Roda em **schedule (cron)** e avisa no Slack, num **canal de aviso** (diferente do canal de PR review), sobre os PRs abertos parados **sem review**.

- **1º aviso** (aos `remind-after-minutes`): mensagem **nova no canal**, que **já é o encaminhamento do card original** do [`slack-pr-notify`](../slack-pr-notify) — nota (`⏰ PR aguardando review há 15m @time`) + o **permalink** da mensagem original, e o Slack **desdobra** o card embutido como citação.
- **Re-aviso** (a cada `reping-every-minutes`, enquanto sem review): **resposta na thread** do 1º aviso, **marcando o time** (notifica os membros) mas **sem broadcast** — não repete o card nem polui o canal.
- **Para** assim que houver **qualquer review**, ou se o PR virar draft/fechar.

```
⏰ PR aguardando review há 15m @time                ← 1º aviso (no canal)
┌─────────────────────────────────────────────┐   ← card original do pr-notify (unfurl)
│ 🔵 Pull Request · Aguardando review · Author…│
│ • <descrição> · Repositório · PR · Branch …  │
└─────────────────────────────────────────────┘
  └─ ⏰ PR aguardando review há 45m @time          ← re-avisos (na thread, marcam o time)
```

> ⚠️ **Cross-channel:** como o card original vive no canal de PR review, o preview embutido só renderiza pra quem é **membro daquele canal**. Se a audiência do canal de aviso não estiver no canal de review, ela vê só o link. Requer também que o `slack-pr-notify` (modo bot token) esteja **ativo** no repo — senão não há card pra encaminhar. **Fallback:** sem card original guardado, o lembrete posta um card próprio (barra âmbar, autor via `user-map.json`) pra não deixar de avisar.

## Quando age

Só em **horário comercial BRT**, seg–sex, dentro de `business-hours-start`..`business-hours-end` (default 09h–18h). **Pula** sábado, domingo e **feriado nacional** (BrasilAPI). Diferente do `weekend-holiday-merge-gate`, **sexta conta** como dia útil aqui (não tem risco de desembolso). Se a API de feriados cair, segue em frente (fail-open — um lembrete a mais é inofensivo).

## Filtros (não cutuca)

- PR **draft**
- base fora de `target-branches` (default `main`)
- autor em `ignore-authors` (default `dependabot[bot]`)
- PR de sync `main → staging` (bull-sync-bot)
- PR que **já tem qualquer review** (aprovado / mudanças pedidas / comentado)
- PR aberto há menos de `remind-after-minutes`
- PR aberto **antes de 26/07/2026** — corte de adoção, chumbado na action (ver abaixo)

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

Guarda última cutucada + contador num **comentário oculto do PR** (`<!-- slack-review-reminder: ... -->`) — por isso o caller precisa de `pull-requests: write`. O `ts`/canal do card a encaminhar vêm do comentário do `slack-pr-notify`. Usa o `GITHUB_TOKEN` do próprio repo (varre só os PRs dele), então **não precisa de token cross-repo**: é um workflow por repo.

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

> **Corte de adoção — `2026-07-26`, chumbado na action.** PR aberto antes dessa data
> nunca é cutucado. Não é input: a constante `ONLY_AFTER` está no script e vale igual
> pra todos os repos que consomem a action. Pra mudar, edita aqui.
>
> **Por que existe:** PR antigo sem review não tem o comentário de estado, e o gate de
> re-cutucada só se aplica quando esse comentário existe. Sem o corte, a primeira
> execução num repo com backlog manda o 1º aviso de **todos** de uma vez.
>
> **Limitação:** é corte fixo, não "anda" com o tempo. Ele resolve a adoção, mas daqui
> uns meses não protege de PR novo que ficar esquecido — esse vai ser cutucado
> indefinidamente. Pra isso o filtro certo seria teto de idade, que a action não tem.

> O cron do GitHub Actions pode **atrasar** sob carga e **desliga após 60 dias** sem commit no repo. Pra um lembrete é tolerável; se precisar de precisão, migrar pra EventBridge + Lambda.
