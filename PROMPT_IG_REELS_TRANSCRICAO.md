# Prompt: Baixar e Transcrever os 10 Últimos Reels de um Perfil do Instagram

Cole o conteúdo abaixo em um novo chat do Claude Code para executar o pipeline completo.

---

## PROMPT PARA COLAR NO CLAUDE CODE

```
Quero que você execute um pipeline completo: baixar os 10 últimos Reels de um perfil do Instagram e transcrever cada vídeo, entregando 10 arquivos de texto com o roteiro de cada um.

O perfil alvo é: [COLE AQUI O @ OU URL DO PERFIL — ex: @nomedobperfil ou https://www.instagram.com/nomedoberfil/]

Execute todos os passos abaixo em sequência, confirmando cada um antes de passar para o próximo.

---

### PASSO 1 — Verificar dependências

Verifique se `gallery-dl`, `python3` e o `whisper` estão disponíveis:

```bash
python3 -c "
import shutil, sys, os
# gallery-dl
gdl = shutil.which('gallery-dl')
if not gdl:
    for v in ['3.9','3.10','3.11','3.12']:
        c = os.path.expanduser(f'~/Library/Python/{v}/bin/gallery-dl')
        if os.path.isfile(c):
            gdl = c
            break
print(f'gallery-dl: {gdl or \"NÃO ENCONTRADO\"}')

# whisper
for v in ['3.9','3.10','3.11','3.12']:
    site = os.path.expanduser(f'~/Library/Python/{v}/lib/python/site-packages')
    if os.path.isdir(site):
        sys.path.insert(0, site)
try:
    import whisper; print('whisper: OK')
except: print('whisper: NÃO INSTALADO')

# ffmpeg via imageio
try:
    import imageio_ffmpeg; print(f'ffmpeg: {imageio_ffmpeg.get_ffmpeg_exe()}')
except: print('ffmpeg: NÃO ENCONTRADO')
"
```

Se `gallery-dl` não estiver instalado:
```bash
pip3 install --user gallery-dl
```

Se `whisper` não estiver instalado:
```bash
pip3 install --user openai-whisper imageio-ffmpeg
```

Criar o symlink do ffmpeg para que o Whisper o encontre no PATH:
```bash
ln -sf $(python3 -c "import imageio_ffmpeg; print(imageio_ffmpeg.get_ffmpeg_exe())") ~/Library/Python/3.9/bin/ffmpeg
```

> IMPORTANTE: NÃO use `yt-dlp` para baixar do Instagram. O extrator está marcado como broken e não funciona mesmo com cookies válidos. Use `gallery-dl`.

---

### PASSO 2 — Criar estrutura de pastas

```bash
mkdir -p ~/Downloads/Instagram/reels_tmp
mkdir -p ~/Downloads/Instagram/roteiros
```

- `reels_tmp/` → vídeos baixados (podem ser apagados depois)
- `roteiros/` → arquivos `.txt` finais com as transcrições

---

### PASSO 3 — Criar o script principal

Crie o arquivo `~/Documents/ClaudeCode/reels_transcrever.py` com o conteúdo abaixo:

```python
#!/usr/bin/env python3
"""
Pipeline: baixa os 10 últimos Reels de um perfil do Instagram e transcreve cada um.
Usa gallery-dl para download (yt-dlp tem extrator do Instagram quebrado).
Uso: python3 reels_transcrever.py <@usuario_ou_url>
"""
import sys
import os
import re
import subprocess
import shutil
import glob

DOWNLOAD_DIR = os.path.expanduser("~/Downloads/Instagram/reels_tmp")
OUTPUT_DIR   = os.path.expanduser("~/Downloads/Instagram/roteiros")

def find_gallery_dl():
    found = shutil.which("gallery-dl")
    if found:
        return found
    for c in [
        os.path.expanduser("~/Library/Python/3.9/bin/gallery-dl"),
        os.path.expanduser("~/Library/Python/3.10/bin/gallery-dl"),
        os.path.expanduser("~/Library/Python/3.11/bin/gallery-dl"),
        os.path.expanduser("~/Library/Python/3.12/bin/gallery-dl"),
        "/usr/local/bin/gallery-dl",
        "/opt/homebrew/bin/gallery-dl",
    ]:
        if os.path.isfile(c):
            return c
    return "gallery-dl"

GALLERY_DL = find_gallery_dl()

def setup_ffmpeg():
    for py_ver in ["3.9", "3.10", "3.11", "3.12"]:
        candidate = os.path.expanduser(f"~/Library/Python/{py_ver}/bin/ffmpeg")
        if os.path.isfile(candidate):
            os.environ["PATH"] = os.path.dirname(candidate) + ":" + os.environ.get("PATH", "")
            return
    try:
        for py_ver in ["3.9", "3.10", "3.11", "3.12"]:
            site = os.path.expanduser(f"~/Library/Python/{py_ver}/lib/python/site-packages")
            if os.path.isdir(site):
                sys.path.insert(0, site)
        import imageio_ffmpeg
        ffmpeg_bin = imageio_ffmpeg.get_ffmpeg_exe()
        os.environ["PATH"] = os.path.dirname(ffmpeg_bin) + ":" + os.environ.get("PATH", "")
    except ImportError:
        pass

setup_ffmpeg()

def normalize_profile(arg):
    arg = arg.strip().rstrip("/")
    if arg.startswith("http"):
        match = re.match(r"(https?://(www\.)?instagram\.com/[^/?#]+)", arg)
        if match:
            return match.group(1) + "/reels/"
        return arg
    username = arg.lstrip("@")
    return f"https://www.instagram.com/{username}/reels/"

def get_username(arg):
    arg = arg.strip().lstrip("@").rstrip("/")
    if arg.startswith("http"):
        match = re.search(r"instagram\.com/([^/?#]+)", arg)
        return match.group(1) if match else "instagram"
    return arg.split("/")[-1]

def download_reels(profile_url):
    os.makedirs(DOWNLOAD_DIR, exist_ok=True)
    print(f"\n[INFO] Baixando até 10 Reels de: {profile_url}")
    print(f"[INFO] Destino temporário: {DOWNLOAD_DIR}\n")
    cmd = [
        GALLERY_DL,
        "--cookies-from-browser", "chrome",
        "--directory", DOWNLOAD_DIR,
        "--range", "1-10",
        profile_url
    ]
    result = subprocess.run(cmd, text=True)
    if result.returncode != 0:
        print("[AVISO] Falha com cookies do Chrome. Tentando sem autenticação...")
        cmd_no_auth = [x for x in cmd if x not in ["--cookies-from-browser", "chrome"]]
        result2 = subprocess.run(cmd_no_auth, text=True)
        if result2.returncode != 0:
            print("[ERRO] Não foi possível baixar.")
            sys.exit(1)
    videos = sorted(glob.glob(os.path.join(DOWNLOAD_DIR, "*.mp4")))
    print(f"\n[INFO] {len(videos)} vídeo(s) baixado(s).")
    return videos

def transcribe(video_path, index):
    for py_ver in ["3.9", "3.10", "3.11", "3.12"]:
        site = os.path.expanduser(f"~/Library/Python/{py_ver}/lib/python/site-packages")
        if os.path.isdir(site) and site not in sys.path:
            sys.path.insert(0, site)
    try:
        import whisper
    except ImportError:
        print("[ERRO] Módulo 'whisper' não encontrado.")
        return None
    print(f"\n[{index}] Transcrevendo: {os.path.basename(video_path)}")
    model = whisper.load_model("small")
    result = model.transcribe(video_path, task="transcribe")
    detected_lang = result.get("language", "?")
    text = result["text"].strip()
    print(f"   Idioma detectado: {detected_lang} | {len(text)} chars")
    return detected_lang, text

def save_roteiro(index, video_path, username, lang, text):
    os.makedirs(OUTPUT_DIR, exist_ok=True)
    shortcode = os.path.splitext(os.path.basename(video_path))[0]
    filename = f"{index:02d} - {username} - {shortcode}_roteiro.txt"
    filepath = os.path.join(OUTPUT_DIR, filename)
    header = f"ROTEIRO #{index:02d}\nPerfil: @{username}\nShortcode: {shortcode}\nIdioma detectado: {lang}\n" + "=" * 60 + "\n\n"
    with open(filepath, "w", encoding="utf-8") as f:
        f.write(header + text)
    print(f"   Salvo em: {filepath}")
    return filepath

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Uso: python3 reels_transcrever.py <@usuario_ou_url_do_perfil>")
        sys.exit(1)
    profile_arg = sys.argv[1]
    username = get_username(profile_arg)
    profile_url = normalize_profile(profile_arg)
    videos = download_reels(profile_url)
    if not videos:
        print("[ERRO] Nenhum vídeo encontrado.")
        sys.exit(1)
    roteiros = []
    for i, video in enumerate(videos[:10], start=1):
        result = transcribe(video, i)
        if result:
            lang, text = result
            path = save_roteiro(i, video, username, lang, text)
            roteiros.append(path)
    print("\n" + "=" * 60)
    print(f"CONCLUÍDO! {len(roteiros)} roteiro(s) gerado(s) em: {OUTPUT_DIR}")
    for r in roteiros:
        print(f"  • {os.path.basename(r)}")
```

---

### PASSO 4 — Testar dependências

```bash
export PATH="$HOME/Library/Python/3.9/bin:$PATH" && python3 -c "
import shutil, sys, os
gdl = shutil.which('gallery-dl')
print(f'gallery-dl: {gdl}')
for v in ['3.9','3.10','3.11','3.12']:
    site = os.path.expanduser(f'~/Library/Python/{v}/lib/python/site-packages')
    if os.path.isdir(site): sys.path.insert(0, site)
try:
    import whisper; print('whisper: OK')
except: print('whisper: NÃO INSTALADO')
try:
    import imageio_ffmpeg; print(f'ffmpeg: {imageio_ffmpeg.get_ffmpeg_exe()}')
except: print('ffmpeg: NÃO ENCONTRADO')
"
```

---

### PASSO 5 — Executar o pipeline

```bash
export PATH="$HOME/Library/Python/3.9/bin:$PATH" && python3 ~/Documents/ClaudeCode/reels_transcrever.py @nomedoberfil
```

---

### PASSO 6 — Verificar roteiros

```bash
ls -la ~/Downloads/Instagram/roteiros/
```

---

### PASSO 7 — Limpar temporários

```bash
rm -rf ~/Downloads/Instagram/reels_tmp/
```

---

## NOTAS TÉCNICAS

### Por que gallery-dl e não yt-dlp?

O `yt-dlp` tem o extrator do Instagram marcado como **broken** desde meados de 2025. Mesmo com cookies válidos retorna `Unable to extract data`. O `gallery-dl` usa abordagem diferente e funciona com cookies do Chrome.

### ffmpeg no PATH

O Whisper chama o `ffmpeg` como subprocess. Se não estiver no PATH, falha com `FileNotFoundError`:

```bash
ln -sf $(python3 -c "import imageio_ffmpeg; print(imageio_ffmpeg.get_ffmpeg_exe())") ~/Library/Python/3.9/bin/ffmpeg
export PATH="$HOME/Library/Python/3.9/bin:$PATH"
```

### Formato DASH (vídeo sem áudio)

Sem ffmpeg no PATH, o `gallery-dl` baixa apenas vídeo (arquivos `.fdash-*v.mp4`). Com ffmpeg, mescla automático.

### Tradução

Whisper transcreve no idioma original. Para tradução para PT-BR, use Claude Code após ter os `.txt`:
> "Traduza o conteúdo deste arquivo para português do Brasil, mantendo o tom do criador."

### Modelos Whisper

| Modelo | Tamanho | Qualidade |
|--------|---------|-----------|
| tiny | ~75MB | Baixa |
| base | ~145MB | Média |
| small | ~461MB | Boa ✅ |
| medium | ~1.5GB | Muito boa |
| large | ~3GB | Máxima |

### Perfis privados

Precisa estar logado no Instagram no Chrome e seguir o perfil.

### Se falhar

1. Certifique-se de estar logado no Instagram no Chrome
2. Aguarde alguns minutos
3. `pip3 install --user -U gallery-dl`

---

## CHECKLIST

- [ ] `gallery-dl` instalado
- [ ] `openai-whisper` instalado
- [ ] `imageio-ffmpeg` instalado
- [ ] ffmpeg symlink criado
- [ ] Chrome logado no Instagram
- [ ] `export PATH="$HOME/Library/Python/3.9/bin:$PATH" && python3 ~/Documents/ClaudeCode/reels_transcrever.py @perfil`
```
