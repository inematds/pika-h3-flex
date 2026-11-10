# Pika H3 Flex no DGX

[![Pika H3 Flex no DGX](guia/assets/banner.jpg)](https://inematds.github.io/pika-h3-flex/guia/)

**🇧🇷 [Português](README.md) · 🇺🇸 [English](README.en.md) · 🇪🇸 [Español](README.es.md)**

## O que é

Um kit para rodar o **Pika H3 Flex** no seu próprio computador, sem nuvem e sem pagar crédito. O Flex é um modelo aberto da Pika que gera **vídeo com som sincronizado** a partir de texto, em só 6 passos. Ele não é um modelo inteiro: são adaptadores (LoRA) que encaixam no MiniMax H3. Aqui está a receita que funcionou num **NVIDIA DGX Spark (GB10, 128 GB de memória unificada)**, com o ComfyUI: o que baixar, como subir sem estourar a memória e um script que gera vários clipes de uma vez.

## 📖 Guia de uso

Guia completo (landing + passo a passo): **https://inematds.github.io/pika-h3-flex/guia/**

## O processo, do início ao fim

Quem faz: 🧠 sessão com o Nei · ⚙️ script · 🤖 agente (`codex exec` / `claude -p`).

1. 🧠 **Conferir se cabe.** O Flex são 2 LoRAs de ~2 GB sobre o MiniMax H3. Se o H3 já roda, o Flex roda.
2. 🤖 **Baixar os LoRAs no formato ComfyUI** (feito pelo Astra do Codex, com hash conferido):
   `hf download Pika-Labs/Pika-H3-Flex --include 'comfy/*' --local-dir pika-flex && (cd pika-flex/comfy && sha256sum -c SHA256SUMS)`
3. ⚙️ **Mover para `models/loras/`** do ComfyUI os dois arquivos `pika_h3_flex_*_comfy_bf16.safetensors`.
4. ⚙️ **Subir o ComfyUI com as opções de memória:** `./scripts/comfyui-start.sh` (já passa `--disable-dynamic-vram --disable-pinned-memory --cache-none`).
5. ⚙️ **Ligar o guarda de memória:** `./scripts/guarda.sh guarda.log 8 &`.
6. ⚙️ **Gerar:** `python3 scripts/gera.py --arquivo exemplos/prompts.txt --largura 1344 --altura 768 --segundos 5`.
7. 🧠 **Assistir e conferir** imagem e som; ajustar o prompt e repetir.
8. ⚙️ **Desligar:** `systemctl --user stop comfyui-h3.scope`.

## O que você precisa

- **Uma GPU com ~60 GB livres** (o DGX Spark serve; num PC comum, use versões quantizadas menores).
- **ComfyUI** recente, com os nós nativos do MiniMax H3 (`MiniMaxH3ImageToVideo`, `ManualSigmas`, `SamplerCustomAdvanced`).
- **Modelos do MiniMax H3** em `models/`:

| Pasta | Arquivo | Tamanho |
|---|---|---|
| `diffusion_models/` | `minimax_h3_fl2va_pruned_fp8_scaled.safetensors` | 21 GB |
| `text_encoders/` | `qwen3vl_32b_minimax_h3_int8_convrot.safetensors` | 27 GB |
| `vae/` | `minimax_h3_video_vae_fp16.safetensors` + `minimax_h3_audio_vae_fp32.safetensors` | 5,8 GB |
| `loras/` | `pika_h3_flex_fl2va_comfy_bf16.safetensors` (+ `ref2va`) | 2 GB cada |

Os LoRAs do Flex **não estão neste repositório**: baixe da Pika no Hugging Face (`Pika-Labs/Pika-H3-Flex`). Eles seguem a licença comunitária do MiniMax H3.

## Como o Flex gera em 6 passos

Não basta escolher "6 passos" num sampler qualquer. O Flex usa um agendamento próprio, que o `gera.py` já monta:

- **Sigmas manuais:** `1.0, 0.972973, 0.923077, 0.8, 0.571429, 0.333333, 0.0`
- **Sampler `lcm`** (ruído novo a cada passo), **LoRA com força 1.0**, sem CFG.
- **Quadros:** o H3 quer `17k + 5` (73 = 3 s, 124 = 5 s a 24 fps). O `--segundos` faz a conta.

## A armadilha da memória (GB10)

No DGX Spark a GPU e a CPU **dividem a mesma memória**. Por isso:

- Sem as opções `--disable-dynamic-vram --disable-pinned-memory --cache-none`, o ComfyUI guardava os pesos **duas vezes** (RAM + GPU) e passou de 80 GB só no carregamento. Com elas, sobrou folga mesmo com outro serviço de imagem ligado.
- **O teto do systemd (`MemoryMax`) não enxerga a memória da GPU** no GB10. No primeiro teste o teto marcava 41 GB e a máquina chegou a 0 GB livre. O `guarda.sh` ajuda: olha a memória livre da máquina e derruba o ComfyUI antes do travamento.
- **Rode sozinho na GPU.** Num lote de clipes de 1344×768, rodando junto com outros renders de vídeo, o driver da GPU ficou sem memória e o servidor travou. O `guarda.sh` não segura esse caso. Feche os outros trabalhos pesados de GPU e comece por 832×480.

## Exemplos

Todos gerados no DGX com o `gera.py`:

| Clipe | Tamanho | Tempo |
|---|---|---|
| `exemplos/pescador-832x480.mp4` — pescador jogando a rede ao amanhecer | 832×480, 3 s | 341 s (com carga dos modelos; os 6 passos ≈ 30 s) |

Mais prompts prontos (gaúcho no pampa, robô humanoide, temporal no litoral, estúdio em casa) estão em `exemplos/prompts.txt`, ainda não gerados aqui.

## Arquivos

| Arquivo | Para quê |
|---|---|
| `scripts/gera.py` | texto → vídeo + áudio pelo ComfyUI (vários prompts numa chamada) |
| `scripts/comfyui-start.sh` | sobe o ComfyUI com teto de memória e as opções do GB10 |
| `scripts/guarda.sh` | derruba o ComfyUI se a memória livre da máquina cair demais |
| `workflows/pika_flex_*_ui.json` | workflows oficiais (t2v, fl2va, ref2va) com os nomes trocados para os modelos acima — abra na tela do ComfyUI |
| `exemplos/` | prompts e vídeos gerados |

## Limites

- A versão aberta gera até **768p**. Upscale para 1080p/2K/4K e separação do áudio em faixas existem só no serviço pago da Pika.
- O prompt segue o formato dos exemplos oficiais, em inglês: `integrated_multimodal_description:` (a cena) e `overall_soundscape:` (o som). Prompt em português não foi testado.
- Testado em 09/11/2026 no DGX Spark com ComfyUI 0.33.0.

---
**INEMA.CLUB** — https://inema.club
