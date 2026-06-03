---
name: video-demonstrativo
description: Cria vídeos de DEMONSTRAÇÃO (walkthrough/tutorial) de uma aplicação web em PT-BR, a partir do link do app — navega o app de verdade com um navegador automatizado, captura as telas reais passo a passo, e monta um vídeo narrado com moldura de navegador, cursor animado que clica exatamente nos controles, destaque/zoom e CTA do INEMA.CLUB (16:9 via HyperFrames). Use SEMPRE que o usuário pedir "vídeo de demonstração", "vídeo demonstrativo", "demo do app/sistema", "walkthrough", "tutorial em vídeo de uma ferramenta", "mostrar passo a passo usando o app", "gravar a tela do sistema", "vídeo mostrando como usar X", ou der um link/localhost de uma aplicação e quiser um vídeo mostrando o uso. Diferente da skill video-explicativo (que explica um conceito com motion graphics): esta MOSTRA um app real sendo usado. Cobre captura, narração TTS local, animação de cursor/zoom, render e a CTA final.
---

# Vídeo Demonstrativo (HyperFrames + captura de tela)

Gera um **walkthrough narrado** de uma aplicação web: a partir do **link do app**, navega
de verdade pelos passos, captura as telas reais e monta um vídeo com **moldura de
navegador + cursor animado + destaque/zoom + narração**. Identidade dark premium âmbar
(ver [references/house-style.md](references/house-style.md)) e termina na **CTA do INEMA.CLUB**.

Stack: `agent-browser` (Playwright) pra captura · HyperFrames (HTML→MP4) · TTS local
Kokoro. Tudo na máquina, **sem chave de API**.

## Princípio que rege tudo: capturar antes, animar depois
O render do HyperFrames é **determinístico** (sem rede/`fetch` na renderização). Então
**nunca** se carrega o site ao vivo dentro do vídeo: a gente **captura screenshots reais
antes** e anima por cima. O viewport fixo da captura vira o **espaço de coordenadas** das
caixas (bounding boxes) que o cursor vai mirar.

## Pré-requisitos (já nesta máquina)
- Node 22+ e FFmpeg; Chrome do HyperFrames (`npx hyperframes browser ensure`).
- TTS Kokoro: `pip install kokoro-onnx soundfile` (voz `pf_dora`, PT-BR).
- `agent-browser` disponível no PATH (skill de navegação). O app-alvo precisa estar no ar
  (ex.: `localhost:8000`).

## Fluxo (sempre nesta ordem)

1. **Roteiro de passos** — escreva `STEPS.md`: a lista de ações a demonstrar (ex.:
   "escrever prompt → escolher 512² → ajustar altura → Gerar → salvar"), com 1 frase de
   narração por passo. 5–8 passos + CTA ≈ 35–50s de vídeo. Veja
   [references/pipeline.md](references/pipeline.md).
2. **Captura** — com `agent-browser`: abra a URL num **viewport fixo**, faça `snapshot -i`,
   execute cada ação, tire 1 **screenshot por estado** e pegue a **bounding box real** de
   cada elemento-alvo (`getBoundingClientRect`). É o que faz o cursor cair *exato* no
   controle. Use [scripts/capture.mjs](scripts/capture.mjs) (lê `actions.json`) ou dirija o
   `agent-browser` na mão (passo a passo) — o resultado é `assets/shots/*.png` + `steps.json`.
   Detalhes em [references/pipeline.md](references/pipeline.md).
3. **Projeto** — `npx hyperframes init <nome> --example blank --non-interactive`. Copie as
   fontes embutidas nesta skill (`assets/fonts/` → `assets/fonts/` do projeto), OU rode
   `node scripts/fetch-fonts.mjs` na raiz do projeto pra baixá-las. A house style (paleta,
   tipografia) está em [references/house-style.md](references/house-style.md). Esta skill é
   auto-contida — não depende de nenhum outro projeto.
4. **Narração** — escreva `assets/txt/sN.txt` (1 por passo + CTA) e gere os WAVs com Kokoro
   voz `pf_dora`, `--speed 0.98`. Expanda números/siglas pra fala ("512" → "quinhentos e
   doze"; URLs → "inema ponto club"). Template: [scripts/narration-template.sh](scripts/narration-template.sh).
5. **Composição** — copie [scripts/composition-template.mjs](scripts/composition-template.mjs)
   como `build-demo.mjs`. Ele **lê `steps.json`** (telas + bounding boxes + captions) e
   **mede as durações dos WAVs com ffprobe** automaticamente — timing único, áudio e
   animação sempre batidos. A moldura de navegador, o cursor global animado, o destaque, o
   zoom no resultado e a **CTA do INEMA.CLUB** já vêm prontos. Rode `node build-demo.mjs`.
   Como o cursor e o zoom funcionam: [references/cursor-and-chrome.md](references/cursor-and-chrome.md).
6. **Validar** — `npx hyperframes lint` (0 erros) e `npx hyperframes inspect --samples 14`
   (0 problemas). Armadilhas em [references/gotchas.md](references/gotchas.md).
7. **Render** — `--quality draft` pra conferir (extraia 1 frame por passo e mostre ao
   usuário), depois `--quality high --fps 30 --output renders/<nome>-16x9.mp4`.

## Regras de ouro (não-negociáveis)
- **Capturar antes, animar depois** — render determinístico; nada de site ao vivo no render.
- **Viewport fixo na captura = espaço das coordenadas.** Mantenha o mesmo viewport na hora
  de pegar as bounding boxes e de tirar os screenshots. Largura ≤ ~1280 pra caber no 16:9.
- **Cursor mira a bounding box REAL** (`getBoundingClientRect`), não no olho. É isso que faz
  parecer profissional.
- **Cursor é global** (animado na timeline principal, não por cena), com o *hotspot* na
  ponta — a ponta cai no centro do elemento; clique = pulse + ripple.
- **Screenshots reais dentro de uma moldura de navegador** (barra + URL) pra manter o look
  premium e disfarçar que é print.
- **Animar o `.scene-inner`**, nunca o wrapper `.clip`; cenas e captions em tracks
  alternados (1/3 e 2/4); decorativos e moldura com `data-layout-ignore`; fontes locais
  (sem CDN — use as de `assets/fonts/`). Detalhes em [references/gotchas.md](references/gotchas.md).
- **Timing é fonte única**: o gerador mede os WAVs com ffprobe → não há `AUDIO[]` manual.
- Sempre **conferir frames com o usuário** antes do render final (não dá pra ouvir o áudio
  — peça pra ele validar a locução).

## Limites conhecidos (seja honesto com o usuário)
- Estado dinâmico do app (animações, vídeo, dados ao vivo) vira **print estático**. Pra
  movimento real, é o modo **v3** (gravar vídeo da tela com `agent-browser record`) — outro
  caminho, com sincronia de narração mais difícil.
- App com login precisa de **credenciais de teste** (ou capture só telas públicas).
- Formato natural é **16:9** (telas de app são landscape). 9:16 exigiria recortar/reenquadrar.
- Mesma limitação de voz do Kokoro (boa, sem atuação) e você não escuta — o usuário valida.

## CTA INEMA.CLUB (cena final padrão)
Última cena: "CONTINUA EM" + **INEMA.CLUB** (INEMA creme, .CLUB âmbar com glow) + `🌐 inema.club`.
Narração: "Isso é conteúdo do INEMA ponto CLUB. Acesse: inema ponto club." Já vem pronta no
`composition-template.mjs`.
