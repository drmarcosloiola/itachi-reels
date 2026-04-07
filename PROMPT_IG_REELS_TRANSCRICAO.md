# Prompt: Baixar, Transcrever, Traduzir e Adaptar os 10 Últimos Reels de um Perfil do Instagram

Cole o conteúdo abaixo em um novo chat do Claude Code para executar o pipeline completo.

---

## PROMPT PARA COLAR NO CLAUDE CODE

```
Quero que você execute um pipeline completo: baixar os 10 últimos Reels de um perfil do Instagram, transcrever cada vídeo, traduzir para português do Brasil e adaptar ao meu posicionamento como Dr. Marcos Loiola.

O perfil alvo é: [COLE AQUI O @ OU URL DO PERFIL — ex: @nomedobperfil ou https://www.instagram.com/nomedoberfil/]

Execute todos os passos abaixo em sequência, confirmando cada um antes de passar para o próximo.

---

### PASSO 1 — Verificar dependências

```bash
python3 -c "
import shutil, sys, os
gdl = shutil.which('gallery-dl')
if not gdl:
    for v in ['3.9','3.10','3.11','3.12']:
        c = os.path.expanduser(f'~/Library/Python/{v}/bin/gallery-dl')
        if os.path.isfile(c):
            gdl = c
            break
print(f'gallery-dl: {gdl or \"NÃO ENCONTRADO\"}')
for v in ['3.9','3.10','3.11','3.12']:
    site = os.path.expanduser(f'~/Library/Python/{v}/lib/python/site-packages')
    if os.path.isdir(site): sys.path.insert(0, site)
try:
    import whisper; print('whisper: OK')
except: print('whisper: NÃO INSTALADO')
try:
    import imageio_ffmpeg; print(f'ffmpeg: {imageio_ffmpeg.get_ffmpeg_exe()}')
except: print('ffmpeg: NÃO ENCONTRADO')
try:
    import anthropic; print('anthropic SDK: OK')
except: print('anthropic: NÃO INSTALADO')
"
```

Se alguma dependência estiver faltando:
```bash
pip3 install --user gallery-dl openai-whisper imageio-ffmpeg anthropic
```

Criar symlink do ffmpeg:
```bash
ln -sf $(python3 -c "import imageio_ffmpeg; print(imageio_ffmpeg.get_ffmpeg_exe())") ~/Library/Python/3.9/bin/ffmpeg
```

> IMPORTANTE: NÃO use `yt-dlp` para baixar do Instagram. Use `gallery-dl`.

---

### PASSO 2 — Criar estrutura de pastas

```bash
mkdir -p ~/Downloads/Instagram/reels_tmp
mkdir -p ~/Downloads/Instagram/roteiros
```

---

### PASSO 3 — Criar o script principal

Crie o arquivo `~/Documents/ClaudeCode/reels_transcrever.py` com o conteúdo abaixo:

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
            os.environ["PATH"] = os.path.dirname(c) + ":" + os.environ.get("PATH","")
            return
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
    print(f"\n[{index}] Transcrevendo: {os.path.basename(video_path)}")
    model = whisper.load_model("small")
    result = model.transcribe(video_path, task="transcribe")
    lang, text = result.get("language","?"), result["text"].strip()
    print(f"   Idioma: {lang} | {len(text)} chars")
    return lang, text

POSICIONAMENTO = """
Você é o Dr. Marcos Loiola, clínico geral e especialista em marketing para médicos que querem usar o digital para crescer, conseguir pacientes ou criar produtos digitais.

POSICIONAMENTO: Ensine médicos a pensar melhor e crescer com inteligência no digital. Misture autoridade clínica + mentalidade + estratégia. Evite superficialidade.
TOM: Assertivo, direto, seguro. Levemente provocador. Sem motivacional clichê.
LINGUAGEM: Clara, objetiva, sem enrolação. Termos técnicos quando agregam. Sem jargão acadêmico excessivo.
DIFERENCIAL: Não agrade. Não seja neutro. Gere desconforto produtivo quando necessário.
EVITAR: Frases tipo "não é sobre isso, é sobre aquilo". Conteúdo genérico. Tom de coach. Opiniões óbvias. Falta de ponto de vista.
PÚBLICO: Médicos jovens e residentes que querem crescer no digital.
OBJETIVO: Fazer pensar melhor, gerar clareza prática, aumentar percepção de autoridade.
"""

def traduzir_e_adaptar(texto, idioma, index, expert):
    try: import anthropic
    except ImportError: print("[ERRO] anthropic não encontrado."); return None
    print(f"\n[{index}] Adaptando via Claude...")
    prompt = f"""Transcrição de Reel do @{expert} (idioma: {idioma}):\n\n{texto}\n\n---\nTAREFA:\n1. Traduza para PT-BR se não estiver em português.\n2. Reescreva o roteiro adaptado para o Dr. Marcos Loiola, mantendo a ideia central mas na voz e posicionamento abaixo.\n3. Retorne apenas o roteiro final, sem comentários ou cabeçalhos.\n\n{POSICIONAMENTO}"""
    msg = anthropic.Anthropic().messages.create(model="claude-sonnet-4-6", max_tokens=2048, messages=[{"role":"user","content":prompt}])
    adapted = msg.content[0].text.strip()
    print(f"   Concluído | {len(adapted)} chars")
    return adapted

def save_roteiro(index, video_path, username, idioma, original, adaptado):
    os.makedirs(OUTPUT_DIR, exist_ok=True)
    shortcode = os.path.splitext(os.path.basename(video_path))[0]
    filepath = os.path.join(OUTPUT_DIR, f"{index:02d} - {username} - {shortcode}_roteiro_adaptado.txt")
    with open(filepath, "w", encoding="utf-8") as f:
        f.write(f"ROTEIRO ADAPTADO #{index:02d}\nExpert original: @{username}\nShortcode: {shortcode}\nIdioma original: {idioma}\n" + "="*60 + "\n\n── ROTEIRO ADAPTADO (Dr. Marcos Loiola) ──\n\n" + adaptado + "\n\n" + "─"*60 + "\n── TRANSCRIÇÃO ORIGINAL ──\n\n" + original)
    print(f"   Salvo: {os.path.basename(filepath)}")
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
    if not videos: print("[ERRO] Nenhum vídeo."); sys.exit(1)
    roteiros = []
    for i, video in enumerate(videos[:10], start=1):
        result = transcribe(video, i)
        if not result: continue
        idioma, original = result
        adaptado = traduzir_e_adaptar(original, idioma, i, username) or original
        roteiros.append(save_roteiro(i, video, username, idioma, original, adaptado))
    limpar_videos()
    print("\n" + "="*60)
    print(f"CONCLUÍDO! {len(roteiros)} roteiro(s) em {OUTPUT_DIR}")
    for r in roteiros: print(f"  • {os.path.basename(r)}")
```

---

### PASSO 4 — Testar dependências

```bash
export PATH="$HOME/Library/Python/3.9/bin:$PATH" && python3 -c "
import shutil, sys, os
for v in ['3.9','3.10','3.11','3.12']:
    s = os.path.expanduser(f'~/Library/Python/{v}/lib/python/site-packages')
    if os.path.isdir(s): sys.path.insert(0, s)
print('gallery-dl:', shutil.which('gallery-dl'))
try: import whisper; print('whisper: OK')
except: print('whisper: FALTANDO')
try: import imageio_ffmpeg; print('ffmpeg:', imageio_ffmpeg.get_ffmpeg_exe())
except: print('ffmpeg: FALTANDO')
try: import anthropic; print('anthropic: OK')
except: print('anthropic: FALTANDO')
"
```

---

### PASSO 5 — Configurar a API Key da Anthropic

A etapa de adaptação usa o Claude via API. Configure a chave:

```bash
export ANTHROPIC_API_KEY="sua-chave-aqui"
```

Para persistir entre sessões:
```bash
echo 'export ANTHROPIC_API_KEY="sua-chave-aqui"' >> ~/.zshrc
```

---

### PASSO 6 — Executar o pipeline completo

```bash
export PATH="$HOME/Library/Python/3.9/bin:$PATH" && python3 ~/Documents/ClaudeCode/reels_transcrever.py @nomedoberfil
```

O script vai:
1. Baixar os 10 Reels mais recentes
2. Transcrever com Whisper (`small`)
3. Traduzir + adaptar ao posicionamento do Dr. Marcos Loiola via Claude
4. Salvar roteiros em `~/Downloads/Instagram/roteiros/`
5. Apagar automaticamente os vídeos

---

### PASSO 7 — Verificar os roteiros

```bash
ls -la ~/Downloads/Instagram/roteiros/
cat ~/Downloads/Instagram/roteiros/01\ -\ *_roteiro_adaptado.txt
```

---

## FORMATO DE SAÍDA

```
ROTEIRO ADAPTADO #01
Expert original: @nomedoberfil
Shortcode: 3820992786792885884
Idioma original: en
============================================================

── ROTEIRO ADAPTADO (Dr. Marcos Loiola) ──

[roteiro reescrito na voz do Dr. Marcos, para médicos brasileiros]

────────────────────────────────────────────────────────────
── TRANSCRIÇÃO ORIGINAL ──

[transcrição bruta do Whisper no idioma original]
```

---

## NOTAS TÉCNICAS

### Por que gallery-dl e não yt-dlp?
O `yt-dlp` tem extrator Instagram **broken** desde 2025. O `gallery-dl` funciona com cookies do Chrome.

### ffmpeg no PATH
Sem ffmpeg, Whisper falha com `FileNotFoundError`:
```bash
ln -sf $(python3 -c "import imageio_ffmpeg; print(imageio_ffmpeg.get_ffmpeg_exe())") ~/Library/Python/3.9/bin/ffmpeg
export PATH="$HOME/Library/Python/3.9/bin:$PATH"
```

### Formato DASH
Sem ffmpeg, `gallery-dl` baixa apenas vídeo (`.fdash-*v.mp4`). Com ffmpeg, mescla automático.

### Adaptação via Claude
Usa `claude-sonnet-4-6`. Requer `ANTHROPIC_API_KEY`. ~500–2000 tokens por roteiro.

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

- [ ] `gallery-dl` instalado
- [ ] `openai-whisper` instalado
- [ ] `imageio-ffmpeg` instalado
- [ ] `anthropic` instalado
- [ ] ffmpeg symlink criado
- [ ] `ANTHROPIC_API_KEY` configurada
- [ ] Chrome logado no Instagram
- [ ] `export PATH="$HOME/Library/Python/3.9/bin:$PATH" && python3 ~/Documents/ClaudeCode/reels_transcrever.py @perfil`
```
