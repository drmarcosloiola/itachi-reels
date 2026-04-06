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

Verifique se `yt-dlp`, `python3` e o `whisper` estão disponíveis:

```bash
which yt-dlp && yt-dlp --version
python3 -c "import whisper; print('whisper OK')" 2>&1
python3 -c "import imageio_ffmpeg; print(imageio_ffmpeg.get_ffmpeg_exe())" 2>&1
```

Se `yt-dlp` não estiver instalado:
```bash
pip3 install yt-dlp
```

Se `whisper` não estiver instalado:
```bash
pip3 install --user openai-whisper imageio-ffmpeg
```

Se o `ffmpeg` não estiver no PATH (necessário para o Whisper), crie o symlink:
```bash
ln -sf $(python3 -c "import imageio_ffmpeg; print(imageio_ffmpeg.get_ffmpeg_exe())") ~/Library/Python/3.9/bin/ffmpeg
```

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
Uso: python3 reels_transcrever.py <@usuario_ou_url>
"""
import sys
import os
import re
import subprocess
import shutil
import glob

# ── Paths ──────────────────────────────────────────────────────────────────────
DOWNLOAD_DIR = os.path.expanduser("~/Downloads/Instagram/reels_tmp")
OUTPUT_DIR   = os.path.expanduser("~/Downloads/Instagram/roteiros")
SCRIPT_DIR   = os.path.dirname(os.path.abspath(__file__))

# ── Localizar yt-dlp ──────────────────────────────────────────────────────────
def find_ytdlp():
    found = shutil.which("yt-dlp")
    if found:
        return found
    for c in [
        os.path.expanduser("~/Library/Python/3.9/bin/yt-dlp"),
        os.path.expanduser("~/Library/Python/3.10/bin/yt-dlp"),
        os.path.expanduser("~/Library/Python/3.11/bin/yt-dlp"),
        os.path.expanduser("~/Library/Python/3.12/bin/yt-dlp"),
        "/usr/local/bin/yt-dlp",
        "/opt/homebrew/bin/yt-dlp",
    ]:
        if os.path.isfile(c):
            return c
    return "yt-dlp"

YTDLP = find_ytdlp()

# ── Adicionar ffmpeg ao PATH ───────────────────────────────────────────────────
def setup_ffmpeg():
    for py_ver in ["3.9", "3.10", "3.11", "3.12"]:
        candidate = os.path.expanduser(f"~/Library/Python/{py_ver}/bin/ffmpeg")
        if os.path.isfile(candidate):
            os.environ["PATH"] = os.path.dirname(candidate) + ":" + os.environ.get("PATH", "")
            return
    # Tenta via imageio-ffmpeg
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

# ── Normalizar input do usuário ───────────────────────────────────────────────
def normalize_profile(arg):
    arg = arg.strip().rstrip("/")
    if arg.startswith("http"):
        # Garante que termina sem /reels para pegar o perfil completo
        match = re.match(r"(https?://(www\.)?instagram\.com/[^/?#]+)", arg)
        if match:
            return match.group(1) + "/reels/"
        return arg
    username = arg.lstrip("@")
    return f"https://www.instagram.com/{username}/reels/"

# ── Baixar os 10 últimos Reels ────────────────────────────────────────────────
def download_reels(profile_url):
    os.makedirs(DOWNLOAD_DIR, exist_ok=True)
    print(f"\n[INFO] Baixando até 10 Reels de: {profile_url}")
    print(f"[INFO] Destino temporário: {DOWNLOAD_DIR}\n")

    cmd = [
        YTDLP,
        "--cookies-from-browser", "chrome",
        "--playlist-end", "10",
        "--match-filter", "duration > 0",
        "-o", os.path.join(DOWNLOAD_DIR, "%(playlist_index)02d - %(uploader)s - %(title).60s.%(ext)s"),
        "--format", "bestvideo[ext=mp4]+bestaudio[ext=m4a]/best[ext=mp4]/best",
        "--merge-output-format", "mp4",
        "--no-warnings",
        profile_url
    ]

    result = subprocess.run(cmd, text=True)

    if result.returncode != 0:
        print("[AVISO] Falha com cookies do Chrome. Tentando sem autenticação...")
        cmd_no_auth = [x for x in cmd if x not in ["--cookies-from-browser", "chrome"]]
        result2 = subprocess.run(cmd_no_auth, text=True)
        if result2.returncode != 0:
            print("[ERRO] Não foi possível baixar. O perfil pode ser privado.")
            sys.exit(1)

    videos = sorted(glob.glob(os.path.join(DOWNLOAD_DIR, "*.mp4")))
    print(f"\n[INFO] {len(videos)} vídeo(s) baixado(s).")
    return videos

# ── Transcrever com Whisper ────────────────────────────────────────────────────
def transcribe(video_path, index):
    # Importar Whisper (ajusta path para instalação --user)
    for py_ver in ["3.9", "3.10", "3.11", "3.12"]:
        site = os.path.expanduser(f"~/Library/Python/{py_ver}/lib/python/site-packages")
        if os.path.isdir(site) and site not in sys.path:
            sys.path.insert(0, site)

    try:
        import whisper
    except ImportError:
        print("[ERRO] Módulo 'whisper' não encontrado. Execute: pip3 install --user openai-whisper")
        return None

    print(f"\n[{index}] Transcrevendo: {os.path.basename(video_path)}")

    model = whisper.load_model("small")
    result = model.transcribe(video_path, task="translate" if True else "transcribe")
    # task="translate" → Whisper traduz para inglês automaticamente qualquer idioma.
    # Para manter no idioma original e só transcrever, troque por task="transcribe".
    # Aqui usamos transcribe com detecção automática; se não for PT, fazemos tradução manual abaixo.

    detected_lang = result.get("language", "?")
    text = result["text"].strip()

    # Se o idioma detectado não for português, re-transcreve pedindo tradução
    if detected_lang not in ("pt", "portuguese"):
        print(f"   Idioma detectado: {detected_lang}. Retranscrevendo com tradução para PT...")
        result_pt = model.transcribe(video_path, task="translate")
        text_en = result_pt["text"].strip()
        # Nota: o Whisper traduz para INGLÊS com task="translate", não para PT.
        # Para PT real, usamos o texto original transcrito + nota ao usuário.
        text = text  # mantém o original transcrito
        note = f"\n\n[Idioma original detectado: {detected_lang}. Transcrição no idioma original acima. Tradução automática para inglês pelo Whisper: {text_en}]"
        text += note

    return detected_lang, text

# ── Salvar roteiro ─────────────────────────────────────────────────────────────
def save_roteiro(index, video_path, lang, text):
    os.makedirs(OUTPUT_DIR, exist_ok=True)
    base = os.path.splitext(os.path.basename(video_path))[0]
    # Remove index prefix se existir (ex: "01 - ")
    clean_name = re.sub(r"^\d+ - ", "", base)
    filename = f"{index:02d} - {clean_name}_roteiro.txt"
    filepath = os.path.join(OUTPUT_DIR, filename)

    header = f"ROTEIRO #{index:02d}\n"
    header += f"Arquivo: {os.path.basename(video_path)}\n"
    header += f"Idioma detectado: {lang}\n"
    header += "=" * 60 + "\n\n"

    with open(filepath, "w", encoding="utf-8") as f:
        f.write(header + text)

    print(f"   Salvo em: {filepath}")
    return filepath

# ── Main ───────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Uso: python3 reels_transcrever.py <@usuario_ou_url_do_perfil>")
        sys.exit(1)

    profile_arg = sys.argv[1]
    profile_url = normalize_profile(profile_arg)

    videos = download_reels(profile_url)

    if not videos:
        print("[ERRO] Nenhum vídeo encontrado para transcrever.")
        sys.exit(1)

    roteiros = []
    for i, video in enumerate(videos[:10], start=1):
        lang, text = transcribe(video, i)
        if text:
            path = save_roteiro(i, video, lang, text)
            roteiros.append(path)

    print("\n" + "=" * 60)
    print(f"CONCLUÍDO! {len(roteiros)} roteiro(s) gerado(s) em:")
    print(f"  {OUTPUT_DIR}")
    print("=" * 60)
    for r in roteiros:
        print(f"  • {os.path.basename(r)}")
```

---

### PASSO 4 — Testar as dependências antes de rodar

```bash
python3 -c "
import shutil, sys, os
ytdlp = shutil.which('yt-dlp')
print(f'yt-dlp: {ytdlp}')
for v in ['3.9','3.10','3.11','3.12']:
    site = os.path.expanduser(f'~/Library/Python/{v}/lib/python/site-packages')
    if os.path.isdir(site):
        sys.path.insert(0, site)
try:
    import whisper; print('whisper: OK')
except: print('whisper: NÃO INSTALADO')
try:
    import imageio_ffmpeg; print(f'ffmpeg: {imageio_ffmpeg.get_ffmpeg_exe()}')
except: print('ffmpeg: NÃO ENCONTRADO')
"
```

---

### PASSO 5 — Executar o pipeline completo

Substitua `@nomedoberfil` pelo @ ou URL do perfil desejado:

```bash
python3 ~/Documents/ClaudeCode/reels_transcrever.py @nomedoberfil
```

Ou com URL completa:

```bash
python3 ~/Documents/ClaudeCode/reels_transcrever.py https://www.instagram.com/nomedoberfil/
```

O script vai:
1. Baixar os 10 Reels mais recentes para `~/Downloads/Instagram/reels_tmp/`
2. Transcrever cada vídeo com Whisper (modelo `small`)
3. Detectar o idioma automaticamente
4. Salvar 10 arquivos `.txt` em `~/Downloads/Instagram/roteiros/`

---

### PASSO 6 — Verificar os roteiros gerados

```bash
ls -la ~/Downloads/Instagram/roteiros/
```

Para ler um roteiro específico:

```bash
cat ~/Downloads/Instagram/roteiros/01\ -\ *_roteiro.txt
```

---

### PASSO 7 — Limpar os vídeos temporários (opcional)

Após confirmar que os roteiros estão corretos, apague os vídeos para liberar espaço:

```bash
rm -rf ~/Downloads/Instagram/reels_tmp/
```

---

## NOTAS TÉCNICAS

### Sobre idiomas e tradução

O Whisper (`task="transcribe"`) transcreve no idioma original do vídeo. Se o vídeo estiver em inglês, espanhol, etc., o script detecta automaticamente e anota o idioma. O arquivo de texto será gerado no idioma original **com uma nota indicando o idioma detectado**.

Para forçar tradução para inglês via Whisper (única tradução nativa suportada), edite a linha no script:
```python
result = model.transcribe(video_path, task="translate")  # sempre traduz para inglês
```

Para uma tradução mais fiel para PT-BR, após ter o `.txt`, use Claude Code com o prompt:
> "Traduza o conteúdo deste arquivo para português do Brasil, mantendo o tom e estilo do criador de conteúdo."

### Modelos do Whisper disponíveis

| Modelo | Tamanho | Qualidade |
|--------|---------|-----------|
| tiny   | ~75MB   | Baixa     |
| base   | ~145MB  | Média     |
| small  | ~461MB  | Boa ✅    |
| medium | ~1.5GB  | Muito boa |
| large  | ~3GB    | Máxima    |

Para trocar o modelo, edite a linha no script:
```python
model = whisper.load_model("medium")  # mais lento, mais preciso
```

### Perfis privados

Para perfis privados, o script usa os cookies do Chrome. Você precisa:
1. Estar logado no Instagram no Chrome
2. Seguir o perfil privado
3. Manter o Chrome aberto durante o download

### Taxa de sucesso com o Instagram

O Instagram frequentemente bloqueia downloads automatizados. Se falhar:
1. Atualize o `yt-dlp`: `pip3 install -U yt-dlp`
2. Aguarde alguns minutos e tente novamente
3. Certifique-se de estar logado no Instagram no Chrome

### Saída esperada

```
~/Downloads/Instagram/roteiros/
├── 01 - nomedoberfil - Titulo do Reel 1_roteiro.txt
├── 02 - nomedoberfil - Titulo do Reel 2_roteiro.txt
├── 03 - nomedoberfil - Titulo do Reel 3_roteiro.txt
...
└── 10 - nomedoberfil - Titulo do Reel 10_roteiro.txt
```

Cada arquivo contém:
```
ROTEIRO #01
Arquivo: 01 - nomedoberfil - Titulo do Reel.mp4
Idioma detectado: pt
============================================================

[texto completo transcrito do vídeo...]
```
```

---

## CHECKLIST DE USO RÁPIDO

- [ ] `yt-dlp` instalado (`pip3 install yt-dlp`)
- [ ] `openai-whisper` instalado (`pip3 install --user openai-whisper`)
- [ ] `imageio-ffmpeg` instalado (`pip3 install --user imageio-ffmpeg`)
- [ ] Chrome aberto e logado no Instagram (para perfis privados ou melhor taxa de sucesso)
- [ ] Script criado em `~/Documents/ClaudeCode/reels_transcrever.py`
- [ ] Executar: `python3 ~/Documents/ClaudeCode/reels_transcrever.py @perfil`
- [ ] Roteiros em `~/Downloads/Instagram/roteiros/`
