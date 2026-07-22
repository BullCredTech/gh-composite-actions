# slack-pr-notify

Ao **abrir um PR**, posta uma notificação no Slack (via Incoming Webhook) num card com **barra colorida lateral**. O título "Pull Request" fica **fora** do card (negrito), e o conteúdo dentro:

```
🔔 Pull Request                         ← fora do card (negrito) + preview da notificação
┃ • <descrição de 1 linha em PT (Gemini; fallback = título do PR)>
┃ ──────────────
┃ 📌 PR: <link>
┃ 📦 Repositório: <repo>
┃ 🙋 Aberto por: <nome do perfil GitHub; fallback = login>
┃ 🌿 Branch: <head> → <base>
┃ 👥 Time responsável: @time   ← menção que notifica (quando team-id é setado)
```
(`┃` = a barra colorida)

Lê os dados do PR do contexto `github.event.pull_request`, então o workflow que chama **precisa rodar em `on: pull_request`**.

## Inputs

| Input | Obrigatório | Default | Descrição |
|-------|-------------|---------|-----------|
| `slack-webhook-url` | não* | `""` | Incoming Webhook do canal. Vazio → o job sai sem falhar (falha fechada). |
| `team-id` | não | `""` | Slack User Group ID (`S0XXXXXXX`). Setado → menção `<!subteam^ID>` que **notifica** o time. Vazio → linha omitida. |
| `bar-color` | não | `#4A90D9` | Cor (hex) da barra lateral do card. |
| `gemini-api-key` | não | `""` | API key do Google AI Studio para a descrição de 1 linha em PT. Vazio → usa o título do PR. |
| `gemini-model` | não | `gemini-2.5-flash` | Modelo do Google AI Studio. |

\* Sem `slack-webhook-url` a action não posta nada (não quebra).

## Uso

```yaml
name: Notify Slack on PR open
on:
  pull_request:
    types: [opened]
permissions:
  contents: read
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
```

## Notas

- A descrição é **1 linha em PT gerada pelo Gemini** (título + descrição do PR); **fallback = título** se a IA falhar ou faltar a chave.
- **Aberto por** usa o **nome do perfil GitHub** (`.name` via API pública `/users/{login}`), com **fallback pro login**. Usa o `github.token` só pra evitar rate limit.
- A **barra colorida** usa `attachments`; por causa disso o Slack mostra um rótulo automático "Adicionado por \<app\>" no rodapé (atribuição do app do webhook — não removível pelo webhook).
- **Falha fechada**: sem `slack-webhook-url` sai sem erro.
- `permissions: contents: read` no caller já basta. Não requer `actions/checkout`.
