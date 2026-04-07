# itachi-reels

Pipeline automatizado para baixar os 10 últimos Reels de qualquer perfil do Instagram e transcrever o áudio de cada vídeo, gerando arquivos de texto com o roteiro completo.

> **Nota:** O `yt-dlp` tem suporte ao Instagram marcado como **broken** desde 2025. Este pipeline usa `gallery-dl` para download, que funciona corretamente com os cookies do Chrome.

---

## O que faz

1. Recebe um `@usuario` ou URL de perfil do Instagram
2. Baixa os 10 Reels mais recentes usando `gallery-dl` com cookies do Chrome
3. Transcreve cada vídeo localmente com OpenAI Whisper (sem API key, gratuito)
4. Detecta o idioma automaticamente
5. Traduz para português do Brasil e adapta ao posicionamento do Dr. Marcos Loiola via Claude API
6. Salva 10 arquivos `.txt` com o roteiro adaptado + transcrição original
7. Apaga automaticamente os vídeos baixados

---

## Como usar

Abra um novo chat no Claude Code, cole o conteúdo do arquivo `PROMPT_IG_REELS_TRANSCRICAO.md` e substitua o perfil alvo:

```
[COLE AQUI O @ OU URL DO PERFIL — ex: @nomedoberfil]
```

O Claude vai executar o pipeline passo a passo e entregar os 10 roteiros prontos.

---

## Dependências

| Ferramenta | Instalação |
|------------|------------|
| `gallery-dl` | `pip3 install --user gallery-dl` |
| `openai-whisper` | `pip3 install --user openai-whisper` |
| `imageio-ffmpeg` | `pip3 install --user imageio-ffmpeg` |
| `anthropic` | `pip3 install --user anthropic` |

> Chrome deve estar aberto e logado no Instagram para o download funcionar.

> A etapa de adaptação usa o Claude via API. Configure a variável `ANTHROPIC_API_KEY` antes de rodar.

> **Por que não `yt-dlp`?** O extrator do Instagram no `yt-dlp` está marcado como broken e retorna erro `Unable to extract data` mesmo com cookies válidos. O `gallery-dl` é a alternativa funcional.

---

## Saída

Os arquivos são salvos em `~/Downloads/Instagram/roteiros/`:

```
01 - nomedoberfil - shortcode_roteiro_adaptado.txt
02 - nomedoberfil - shortcode_roteiro_adaptado.txt
...
10 - nomedoberfil - shortcode_roteiro_adaptado.txt
```

Cada arquivo contém duas seções:
1. **Roteiro adaptado** — reescrito na voz do Dr. Marcos Loiola para médicos brasileiros
2. **Transcrição original** — texto bruto do Whisper no idioma original

Os vídeos são apagados automaticamente ao final do processo.

---

## Modelos do Whisper

| Modelo | Tamanho | Qualidade |
|--------|---------|----------|
| tiny | ~75MB | Baixa |
| base | ~145MB | Média |
| small | ~461MB | Boa ✅ |
| medium | ~1.5GB | Muito boa |
| large | ~3GB | Máxima |

O padrão é `small`. Para trocar, edite a linha no script gerado:
```python
model = whisper.load_model("medium")
```

---

## Observações

- Funciona com perfis públicos e privados (desde que você siga o perfil e esteja logado no Chrome)
- Transcrição roda 100% local com Whisper
- Adaptação requer `ANTHROPIC_API_KEY` configurada
- O ffmpeg precisa estar no PATH — o script cria o symlink automaticamente via `imageio-ffmpeg`
- Se o download falhar, certifique-se de estar logado no Instagram no Chrome e tente novamente
