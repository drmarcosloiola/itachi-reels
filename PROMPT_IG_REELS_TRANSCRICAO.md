# Prompt: Pipeline Completo de Reels — Download, Transcrição e Adaptação

Cole o conteúdo abaixo em um novo chat do Claude Code para executar o pipeline completo.
Tudo roda localmente ou dentro do próprio Claude Code — sem dependência de API de IA externa.

---

## PROMPT PARA COLAR NO CLAUDE CODE

```
Quero que você execute um pipeline completo para o expert: @[NOME_DO_EXPERT]

Passos:
1. Baixar os 10 últimos Reels do perfil
2. Transcrever cada vídeo localmente com Whisper
3. Apagar os vídeos após a transcrição
4. Ler cada transcrição e gerar o roteiro adaptado ao meu posicionamento
5. Salvar os roteiros adaptados em ~/Downloads/Instagram/roteiros/

Execute cada passo em sequência e me mostre o progresso.
```

---

## INSTRUÇÕES PARA O CLAUDE CODE EXECUTAR

### PASSO 1 — Verificar dependências

```bash
export PATH="$HOME/Library/Python/3.9/bin:$PATH" && python3 -c "
import shutil, sys, os
for v in ['3.9','3.10','3.11','3.12']:
    s = os.path.expanduser(f'~/Library/Python/{v}/lib/python/site-packages')
    if os.path.isdir(s): sys.path.insert(0, s)
gdl = shutil.which('gallery-dl') or next((c for c in [os.path.expanduser(f'~/Library/Python/{v}/bin/gallery-dl') for v in ['3.9','3.10','3.11','3.12']] if os.path.isfile(c)), None)
print(f'gallery-dl: {gdl or \"NÃO ENCONTRADO\"}')
try: import whisper; print('whisper: OK')
except: print('whisper: NÃO INSTALADO')
try: import imageio_ffmpeg; print(f'ffmpeg: {imageio_ffmpeg.get_ffmpeg_exe()}')
except: print('ffmpeg: NÃO ENCONTRADO')
"
```

Se alguma dependência estiver faltando:
```bash
pip3 install --user gallery-dl openai-whisper imageio-ffmpeg
ln -sf $(python3 -c "import imageio_ffmpeg; print(imageio_ffmpeg.get_ffmpeg_exe())") ~/Library/Python/3.9/bin/ffmpeg
```

> NÃO use `yt-dlp` para Instagram — extrator está broken desde 2025. Use `gallery-dl`.

---

### PASSO 2 — Criar pastas

```bash
mkdir -p ~/Downloads/Instagram/reels_tmp ~/Downloads/Instagram/roteiros
```

---

### PASSO 3 — Criar o script de download + transcrição

Crie `~/Documents/ClaudeCode/reels_transcrever.py`:

```python
#!/usr/bin/env python3
import sys, os, re, subprocess, shutil, glob

DOWNLOAD_DIR = os.path.expanduser("~/Downloads/Instagram/reels_tmp")
OUTPUT_DIR   = os.path.expanduser("~/Downloads/Instagram/roteiros")

for _v in ["3.9","3.10","3.11","3.12"]:
    _s = os.path.expanduser(f"~/Library/Python/{_v}/lib/python/site-packages")
    if os.path.isdir(_s) and _s not in sys.path: sys.path.insert(0, _s)

def find_gallery_dl():
    found = shutil.which("gallery-dl")
    if found: return found
    for c in [os.path.expanduser(f"~/Library/Python/{v}/bin/gallery-dl") for v in ["3.9","3.10","3.11","3.12"]] + ["/usr/local/bin/gallery-dl","/opt/homebrew/bin/gallery-dl"]:
        if os.path.isfile(c): return c
    return "gallery-dl"
GALLERY_DL = find_gallery_dl()

def setup_ffmpeg():
    for v in ["3.9","3.10","3.11","3.12"]:
        c = os.path.expanduser(f"~/Library/Python/{v}/bin/ffmpeg")
        if os.path.isfile(c):
            os.environ["PATH"] = os.path.dirname(c) + ":" + os.environ.get("PATH",""); return
    try:
        import imageio_ffmpeg
        os.environ["PATH"] = os.path.dirname(imageio_ffmpeg.get_ffmpeg_exe()) + ":" + os.environ.get("PATH","")
    except ImportError: pass
setup_ffmpeg()

def normalize_profile(arg):
    arg = arg.strip().rstrip("/")
    if arg.startswith("http"):
        m = re.match(r"(https?://(www\.)?instagram\.com/[^/?#]+)", arg)
        return (m.group(1) if m else arg) + "/reels/"
    return f"https://www.instagram.com/{arg.lstrip('@')}/reels/"

def get_username(arg):
    arg = arg.strip().lstrip("@").rstrip("/")
    if arg.startswith("http"):
        m = re.search(r"instagram\.com/([^/?#]+)", arg)
        return m.group(1) if m else "instagram"
    return arg.split("/")[-1]

def download_reels(url):
    os.makedirs(DOWNLOAD_DIR, exist_ok=True)
    print(f"\n[INFO] Baixando Reels de: {url}")
    cmd = [GALLERY_DL, "--cookies-from-browser", "chrome", "--directory", DOWNLOAD_DIR, "--range", "1-10", url]
    if subprocess.run(cmd, text=True).returncode != 0:
        cmd2 = [x for x in cmd if x not in ["--cookies-from-browser","chrome"]]
        if subprocess.run(cmd2, text=True).returncode != 0:
            print("[ERRO] Download falhou."); sys.exit(1)
    videos = sorted(glob.glob(os.path.join(DOWNLOAD_DIR, "*.mp4")))
    print(f"[INFO] {len(videos)} vídeo(s) baixado(s).")
    return videos

def transcribe(video_path, index):
    try: import whisper
    except ImportError: print("[ERRO] whisper não encontrado."); return None
    print(f"\n[{index:02d}] Transcrevendo: {os.path.basename(video_path)}")
    model = whisper.load_model("small")
    result = model.transcribe(video_path, task="transcribe")
    lang, text = result.get("language","?"), result["text"].strip()
    print(f"      Idioma: {lang} | {len(text)} chars")
    return lang, text

def save_transcricao(index, video_path, username, lang, text):
    os.makedirs(OUTPUT_DIR, exist_ok=True)
    shortcode = os.path.splitext(os.path.basename(video_path))[0]
    filepath = os.path.join(OUTPUT_DIR, f"{index:02d} - {username} - {shortcode}_transcricao.txt")
    header = f"TRANSCRIÇÃO #{index:02d}\nExpert: @{username}\nShortcode: {shortcode}\nIdioma detectado: {lang}\n" + "="*60 + "\n\n"
    with open(filepath, "w", encoding="utf-8") as f:
        f.write(header + text)
    print(f"      Salvo: {os.path.basename(filepath)}")
    return filepath

def limpar_videos():
    videos = glob.glob(os.path.join(DOWNLOAD_DIR, "*.mp4"))
    for v in videos: os.remove(v)
    if os.path.isdir(DOWNLOAD_DIR) and not os.listdir(DOWNLOAD_DIR): shutil.rmtree(DOWNLOAD_DIR)
    print(f"\n[INFO] {len(videos)} vídeo(s) removido(s).")

if __name__ == "__main__":
    if len(sys.argv) < 2: print("Uso: python3 reels_transcrever.py <@usuario>"); sys.exit(1)
    username = get_username(sys.argv[1])
    videos = download_reels(normalize_profile(sys.argv[1]))
    if not videos: print("[ERRO] Nenhum vídeo encontrado."); sys.exit(1)
    transcritos = []
    for i, video in enumerate(videos[:10], start=1):
        result = transcribe(video, i)
        if result:
            lang, text = result
            transcritos.append(save_transcricao(i, video, username, lang, text))
    limpar_videos()
    print("\n" + "="*60)
    print(f"TRANSCRIÇÃO CONCLUÍDA! {len(transcritos)} arquivo(s) em: {OUTPUT_DIR}")
    for t in transcritos: print(f"  • {os.path.basename(t)}")
    print("\n[PRÓXIMO PASSO] Claude Code lê cada arquivo e gera os roteiros adaptados.")
```

---

### PASSO 4 — Executar download + transcrição

```bash
export PATH="$HOME/Library/Python/3.9/bin:$PATH" && python3 ~/Documents/ClaudeCode/reels_transcrever.py @[NOME_DO_EXPERT]
```

---

### PASSO 5 — Claude Code adapta cada roteiro

Após o script terminar, o Claude Code executa automaticamente:

1. Lê cada arquivo `*_transcricao.txt` em `~/Downloads/Instagram/roteiros/`
2. Traduz para português do Brasil se necessário
3. Reescreve o roteiro adaptado ao posicionamento do Dr. Marcos Loiola (abaixo)
4. Salva como `*_roteiro_adaptado.txt` no mesmo diretório

**Posicionamento do Dr. Marcos Loiola:**

- **Quem é:** Clínico geral, especialista em marketing para médicos que querem usar o digital para crescer, conseguir pacientes ou criar produtos digitais
- **Tom:** Assertivo, direto, seguro — levemente provocador quando necessário. Sem motivacional clichê
- **Linguagem:** Clara, objetiva, sem enrolação. Termos técnicos quando agregam. Sem jargão acadêmico
- **Diferencial:** Não agrada, não é neutro, tem posicionamento claro, gera desconforto produtivo
- **Evitar:** "não é sobre isso, é sobre aquilo" · conteúdo genérico · tom de coach · opiniões óbvias
- **Público:** Médicos jovens e residentes que querem crescer no digital
- **Objetivo:** Fazer pensar melhor · gerar clareza prática · aumentar percepção de autoridade

---

### PASSO 6 — Verificar os roteiros adaptados

```bash
ls -la ~/Downloads/Instagram/roteiros/
```

---

## FORMATO DE SAÍDA

```
01 - expert - shortcode_transcricao.txt       ← transcrição bruta (referência)
01 - expert - shortcode_roteiro_adaptado.txt  ← roteiro final na voz do Dr. Marcos
```

Cada `_roteiro_adaptado.txt` contém:
```
ROTEIRO ADAPTADO #01
Expert original: @expert
Idioma original: en
============================================================

── ROTEIRO ADAPTADO (Dr. Marcos Loiola) ──

[roteiro reescrito para médicos brasileiros]

────────────────────────────────────────────────────────────
── TRANSCRIÇÃO ORIGINAL ──

[texto bruto do Whisper]
```

---

## NOTAS TÉCNICAS

### Por que gallery-dl e não yt-dlp?
O `yt-dlp` tem extrator Instagram **broken** desde 2025. O `gallery-dl` funciona com cookies do Chrome.

### ffmpeg no PATH
Sem ffmpeg, Whisper falha com `FileNotFoundError`:
```bash
ln -sf $(python3 -c "import imageio_ffmpeg; print(imageio_ffmpeg.get_ffmpeg_exe())") ~/Library/Python/3.9/bin/ffmpeg
```

### Formato DASH
Sem ffmpeg no PATH, gallery-dl baixa só vídeo (`.fdash-*v.mp4`). Com ffmpeg, mescla automático.

### Sem dependência de API externa
A adaptação é feita pelo próprio Claude Code, diretamente no chat. Nenhuma chave de API adicional necessária.

### Modelos Whisper

| Modelo | Tamanho | Qualidade |
|--------|---------|-----------|
| tiny | ~75MB | Baixa |
| base | ~145MB | Média |
| small | ~461MB | Boa ✅ |
| medium | ~1.5GB | Muito boa |
| large | ~3GB | Máxima |

---

## CHECKLIST

- [ ] `gallery-dl` instalado (`pip3 install --user gallery-dl`)
- [ ] `openai-whisper` instalado (`pip3 install --user openai-whisper`)
- [ ] `imageio-ffmpeg` instalado (`pip3 install --user imageio-ffmpeg`)
- [ ] ffmpeg symlink criado
- [ ] Chrome aberto e logado no Instagram
- [ ] Rodar: `export PATH="$HOME/Library/Python/3.9/bin:$PATH" && python3 ~/Documents/ClaudeCode/reels_transcrever.py @expert`
- [ ] Claude Code lê e adapta cada transcrição automaticamente
