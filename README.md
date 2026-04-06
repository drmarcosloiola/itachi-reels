# itachi-reels

Pipeline automatizado para baixar os 10 últimos Reels de qualquer perfil do Instagram e transcrever o áudio de cada vídeo, gerando arquivos de texto com o roteiro completo.

---

## O que faz

1. Recebe um `@usuario` ou URL de perfil do Instagram
2. Baixa os 10 Reels mais recentes usando `yt-dlp`
3. Transcreve cada vídeo localmente com OpenAI Whisper (sem API key, gratuito)
4. Detecta o idioma automaticamente
5. Salva 10 arquivos `.txt` com o roteiro de cada vídeo

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
| `yt-dlp` | `pip3 install yt-dlp` |
| `openai-whisper` | `pip3 install --user openai-whisper` |
| `imageio-ffmpeg` | `pip3 install --user imageio-ffmpeg` |

> Chrome deve estar aberto e logado no Instagram para melhor taxa de sucesso no download.

---

## Saída

Os arquivos são salvos em `~/Downloads/Instagram/roteiros/`:

```
01 - nomedoberfil - Titulo do Reel_roteiro.txt
02 - nomedoberfil - Titulo do Reel_roteiro.txt
...
10 - nomedoberfil - Titulo do Reel_roteiro.txt
```

Cada arquivo contém o idioma detectado e o texto transcrito completo.

---

## Modelos do Whisper

| Modelo | Tamanho | Qualidade |
|--------|---------|-----------|
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
- Roda 100% local, sem custos de API
- Se o download falhar, atualize o yt-dlp: `pip3 install -U yt-dlp`
- O Whisper traduz nativamente para inglês (`task="translate"`); para PT-BR, use o Claude Code para traduzir os `.txt` gerados
