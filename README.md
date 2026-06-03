# skill-video-demonstrativo

> 🌐 **Curso navegável (GitHub Pages):** https://inematds.github.io/skill-video-demonstrativo/
> — landing + 3 trilhas / 10 módulos no formato INEMA.CLUB, com a Skill em `.zip` pra baixar.
> Também listado no portal [inema.club](https://inema.club).

Projeto de desenvolvimento da skill **`video-demonstrativo`** — gera vídeos de
demonstração (walkthrough) de uma aplicação web a partir do link do app: navega o app de
verdade, captura as telas reais passo a passo e monta um vídeo narrado com moldura de
navegador, cursor animado, destaque/zoom e CTA do INEMA.CLUB.

É a irmã da skill `video-explicativo`: aquela **explica um conceito** (motion graphics);
esta **mostra um app real sendo usado**.

## Estrutura
```
skill-video-demonstrativo/
├── README.md                      (este arquivo)
└── video-demonstrativo/           (a skill instalável — AUTO-CONTIDA)
    ├── SKILL.md
    ├── references/
    │   ├── pipeline.md            captura → narração → composição → render
    │   ├── cursor-and-chrome.md   mapeamento de coordenadas, cursor, moldura, zoom
    │   ├── gotchas.md             armadilhas de captura/cursor/layout
    │   └── house-style.md         paleta dark premium (embutida)
    ├── assets/
    │   └── fonts/                 Sora / Inter / JetBrains Mono (.woff2 + fonts.css)
    └── scripts/
        ├── capture.mjs            dirige o agent-browser → shots + steps.json
        ├── actions.example.json   entrada do capture.mjs (URL, viewport, passos)
        ├── steps.example.json     saída de exemplo
        ├── composition-template.mjs  gerador (lê steps.json, mede WAVs, monta o vídeo)
        ├── narration-template.sh  gera os WAVs Kokoro a partir do steps.json
        └── fetch-fonts.mjs        (opcional) baixa as fontes, caso prefira não copiar
```

> **Auto-contida:** a skill traz as próprias fontes e house style. Não depende de nenhum
> outro projeto, repositório ou skill pra funcionar.

## Como instalar a skill
Três opções. Em todas, reinicie a sessão depois — a skill dispara com pedidos como
"vídeo de demonstração", "demo do app", "walkthrough", "tutorial em vídeo da
ferramenta", ou ao dar um link/localhost.

**A) A partir do repositório (clone):**
```bash
git clone git@github.com:inematds/skill-video-demonstrativo.git
cp -r skill-video-demonstrativo/video-demonstrativo ~/.claude/skills/   # global
# ou cp -r .../video-demonstrativo <repo>/.claude/skills/               # só num projeto
```

**B) A partir do pacote `.zip`** (versionado na raiz do repo):
```bash
unzip video-demonstrativo.zip -d ~/.claude/skills/    # extrai a pasta video-demonstrativo/
```

**C) Symlink** (pra desenvolver — edição no repo reflete na hora):
```bash
ln -s "$(pwd)/video-demonstrativo" ~/.claude/skills/video-demonstrativo
```

## Fluxo de uso (resumo)
1. Num projeto novo: `npx hyperframes init <nome> --example blank --non-interactive`.
2. Copie pro projeto: `capture.mjs`, `composition-template.mjs` (como `build-demo.mjs`),
   `narration-template.sh`, e `assets/fonts/` (ou rode `node fetch-fonts.mjs`).
3. `actions.json` com URL + viewport + passos (seletores dos alvos).
4. `node capture.mjs actions.json` → `assets/shots/*.png` + `steps.json`.
5. `bash narration-template.sh` → WAVs (voz `pf_dora`).
6. `node build-demo.mjs` → `index.html`.
7. `npx hyperframes lint` + `inspect` → render `--quality high`.

## Modos no roadmap
- **v1** (esta skill, default): screenshots estáticos + cursor/zoom animados.
- **v2**: mais automação via `actions.json`/`steps.json` (já embutido no capture.mjs).
- **v3**: gravação de tela real (`agent-browser record`) com narração/zoom por cima —
  ainda não implementado nesta skill.

## Validado em apps reais
- **App simples** (geração de imagem): fluxo de 7 passos, ~42s — primeiro POC.
- **App complexo** (suíte de dublagem com IA): tutorial ponta a ponta de **14 passos, 2:08**
  (configurar → analisar → aprovar → dublar → resultado). Provou a skill em página longa,
  input controlado por React e fluxo assíncrono multi-estado.
- O `composition-template.mjs` roda end-to-end a partir de um `steps.json` com **0 erros de
  lint e 0 problemas de layout**, medindo os WAVs sozinho.

> Os vídeos de exemplo foram gerados fora desta pasta e **não fazem parte da skill** — ela
> é independente e gera tudo do zero em qualquer projeto.

## Backlog de melhorias (descobertas nas demos reais — pra trabalhar)
Prioridade alta primeiro. Cada item é uma evolução concreta do `capture.mjs`/template:

1. **Scroll na captura (páginas longas).** O `capture.mjs` hoje não rola a página; nas
   telas do inemaVOX tive que dirigir o agent-browser na mão (scrollIntoView por seção).
   → Adicionar ação `scroll`/`scrollTo` e, por passo, `scrollIntoView({block})` antes do
   screenshot. As bounding boxes já são viewport-relative, então batem com o shot daquele
   scroll.
2. **Inputs controlados por framework (React/Vue).** `eval .value` mostra o texto mas NÃO
   dispara o estado (botões continuam disabled). O `capture.mjs` deveria usar o `fill`
   nativo do Playwright (via `agent-browser fill @ref`) em vez de setar `.value`.
3. **Esperar condição em vez de tempo fixo.** Trocar `do:{type:"wait",ms}` por um
   `waitFor` (texto/seletor/“status mudou”). Fluxos assíncronos (ex.: `waiting_approval`,
   etapas de pipeline, geração que leva minutos) precisam disso — usei um poll externo.
4. **Capturar fluxos multi-estado** (analisar → aprovar → dublar → concluído) como
   sub-passos nomeados, com poll de conclusão embutido (hoje foi script `poll-job*.sh` à mão).
5. **9:16 / Shorts.** Telas de app são landscape; o template só faz 16:9. Avaliar
   reenquadre (zoom/pan na região ativa) pra gerar vertical.
6. **v3 — gravação de tela real** (`agent-browser record`) com narração/zoom por cima,
   pra apps com muito movimento.
7. **Polimento de cursor**: caminho em curva (não linear) e “digitação” (typewriter) ao
   preencher campos, pra ficar mais natural.

> Próxima sessão: abrir aqui e atacar o item 1 (scroll no capture.mjs) — é o que mais
> economiza trabalho manual. Detalhes do que já funciona em `video-demonstrativo/references/`.
