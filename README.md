# itachi-reels

Pipeline automatizado para baixar os 10 últimos Reels de qualquer perfil do Instagram e transcrever o áudio de cada vídeo, gerando arquivos de texto com o roteiro completo.

> **Nota:** O `yt-dlp` tem suporte ao Instagram marcado como **broken** desde 2025. Este pipeline usa `gallery-dl` para download, que funciona corretamente com os cookies do Chrome.

---

## O que faz

1. Recebe um `@usuario` ou URL de perfil do Instagram
2. Baixa os 10 Reels mais recentes usando `gallery-dl` com cookies do Chrome
3. Transcreve cada vídeo localmente com OpenAI Whisper (sem API key, gratuito)
4. Detecta o idioma automaticamente
5. Apaga automaticamente os vídeos baixados
6. Claude Code lê cada transcrição e gera o roteiro adaptado ao posicionamento do Dr. Marcos Loiola — direto no chat, sem API externa

---

## Como usar

Abra um novo chat no Claude Code, cole o conteúdo do arquivo `PROMPT_IG_REELS_TRANSCRICAO.md` e informe o perfil alvo.

O Claude vai executar o pipeline completo e entregar os 10 roteiros adaptados.

---

## Dependências

| Ferramenta | Instalação |
|------------|------------|
| `gallery-dl` | `pip3 install --user gallery-dl` |
| `openai-whisper` | `pip3 install --user openai-whisper` |
| `imageio-ffmpeg` | `pip3 install --user imageio-ffmpeg` |

> Chrome deve estar aberto e logado no Instagram para o download funcionar.

> A adaptação é feita pelo próprio Claude Code no chat — sem API externa, sem chave adicional.

> **Por que não `yt-dlp`?** O extrator do Instagram no `yt-dlp` está marcado como broken e retorna erro `Unable to extract data` mesmo com cookies válidos. O `gallery-dl` é a alternativa funcional.

---

## Saída

Os arquivos são salvos em `~/Downloads/Instagram/roteiros/`:

```
01 - expert - shortcode_transcricao.txt
01 - expert - shortcode_roteiro_adaptado.txt
...
```

Dois arquivos por vídeo:
- `*_transcricao.txt` — texto bruto do Whisper (referência)
- `*_roteiro_adaptado.txt` — roteiro reescrito pelo Claude Code na voz do Dr. Marcos Loiola

Os vídeos são apagados automaticamente após a transcrição.

---

## Modelos do Whisper

| Modelo | Tamanho | Qualidade |
|--------|---------|----------|
| tiny | ~75MB | Baixa |
| base | ~145MB | Média |
| small | ~461MB | Boa ✅ |
| medium | ~1.5GB | Muito boa |
| large | ~3GB | Máxima |

O padrão é `small`. Para trocar, edite a linha no script:
```python
model = whisper.load_model("medium")
```

---

## Observações

- Funciona com perfis públicos e privados (logado no Chrome e seguindo o perfil)
- Transcrição roda 100% local com Whisper
- Adaptação feita pelo Claude Code — zero dependência de API externa
- O ffmpeg precisa estar no PATH — crie o symlink via `imageio-ffmpeg`
- Se o download falhar, certifique-se de estar logado no Instagram no Chrome
