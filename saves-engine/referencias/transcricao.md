# O robô que escuta: transcribe.py

A transcrição é a peça que faz o sistema valer a pena. Num reel, a legenda quase
nunca diz do que o vídeo trata: é "comenta EU QUERO" ou "link na bio". A ideia de
verdade está na fala, e a fala não é pesquisável, não é filtrável, não existe em
lugar nenhum além do vídeo.

**Por que é um arquivo separado do sync:** o sync precisa ser rápido, a transcrição
é lenta. Se a transcrição travar, a captura dos salvos continua funcionando.

---

## Os três caminhos

A skill pergunta o sistema operacional e gera o arquivo certo. Não pergunte duas vezes.

| Caminho | Sistema | Custo | Velocidade | Áudio sai da máquina? |
|---|---|---|---|---|
| `mlx-whisper` | Mac (chip M1 ou superior) | R$ 0 | 7 a 8 segundos por reel | Não |
| `faster-whisper` | Windows e Mac Intel | R$ 0 | Mais lento; depende do processador | Não |
| API da OpenAI | Qualquer um | US$ 0,006 por minuto de áudio | Rápido, depende da internet | **Sim** |

**Padrão por sistema:**
- Mac com chip M: `mlx-whisper`. É o mais rápido e é grátis.
- Windows: perguntar (pergunta 8 da entrevista). `faster-whisper` é grátis e privado;
  a API é rápida e custa pouco, mas o áudio sobe pra OpenAI.

**Sobre o custo da API:** US$ 0,006 por minuto sai em torno de US$ 0,36 por hora de
áudio. Um reel de 1 minuto custa menos de um centavo de dólar. Cem reels por mês, com
média de 1 minuto, dão uns US$ 0,60. É barato, mas **não é zero**, e a pessoa precisa
saber disso antes de escolher. Fonte: tabela de preços da OpenAI, consultada em
setembro de 2026.

---

## Caminho 1 — Mac com chip M (mlx-whisper)

### Instalação

```bash
brew install yt-dlp ffmpeg
pip install mlx-whisper
```

O modelo usado é `mlx-community/whisper-large-v3-turbo`. Ele baixa sozinho na primeira
execução (uns 1,5 GB) e fica guardado. Depois disso roda offline.

### O código

```python
#!/usr/bin/env python3
"""
Saves Engine - transcricao dos videos salvos.

Procura no Notion os salvos com Transcricao Status = "Pendente", baixa o audio
de cada video, transcreve com o whisper rodando na propria maquina e grava o
texto de volta na base.

Uso:
    .venv/bin/python3 transcribe.py            # transcreve ate 10 pendentes
    .venv/bin/python3 transcribe.py --limit 3  # transcreve so 3
    .venv/bin/python3 transcribe.py --all      # transcreve todos os pendentes
"""

import argparse
import json
import logging
import os
import shutil
import subprocess
import sys
import tempfile

from notion_client import Client
from notion_client.errors import APIResponseError

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
CONFIG_PATH = os.path.join(BASE_DIR, "config.json")
LOG_PATH = os.path.join(BASE_DIR, "transcribe.log")

WHISPER_MODEL = "mlx-community/whisper-large-v3-turbo"
NOTION_TEXT_LIMIT = 1900
DEFAULT_LIMIT = 10
DOWNLOAD_TIMEOUT = 180
WHISPER_TIMEOUT = 900


def setup_logging():
    logger = logging.getLogger("transcribe")
    logger.setLevel(logging.INFO)
    logger.handlers.clear()
    fmt = logging.Formatter("%(asctime)s  %(message)s", datefmt="%Y-%m-%d %H:%M:%S")
    stream = logging.StreamHandler(sys.stdout)
    stream.setFormatter(fmt)
    logger.addHandler(stream)
    fileh = logging.FileHandler(LOG_PATH, encoding="utf-8")
    fileh.setFormatter(fmt)
    logger.addHandler(fileh)
    return logger


log = setup_logging()


def check_tools():
    """Confere se yt-dlp, ffmpeg e mlx_whisper estao instalados."""
    missing = [t for t in ("yt-dlp", "ffmpeg", "mlx_whisper") if not shutil.which(t)]
    if missing:
        log.error("Faltam ferramentas: %s. Instale com: brew install %s",
                  ", ".join(missing), " ".join(missing))
        sys.exit(1)


def load_config():
    if not os.path.exists(CONFIG_PATH):
        log.error("config.json nao encontrado.")
        sys.exit(1)
    with open(CONFIG_PATH, encoding="utf-8") as f:
        cfg = json.load(f)
    for k in ("notion_token", "notion_database_id"):
        if not cfg.get(k) or str(cfg[k]).startswith("COLE_"):
            log.error("Falta %s no config.json", k)
            sys.exit(1)
    return cfg


def resolve_data_source(notion, database_id):
    """API nova do Notion consulta por data_source. Suporta as duas versoes."""
    if not hasattr(notion, "data_sources"):
        return None
    try:
        db = notion.databases.retrieve(database_id=database_id)
    except APIResponseError as e:
        log.warning("Nao consegui ler o data source: %s", e)
        log.warning("Confira se a integracao esta conectada NESTA base.")
        return None
    sources = db.get("data_sources") or []
    return sources[0]["id"] if sources else None


def fetch_pending(notion, database_id, limit):
    """Busca os salvos que ainda esperam transcricao."""
    pending = []
    cursor = None
    data_source_id = resolve_data_source(notion, database_id)

    while True:
        kwargs = {
            "filter": {"property": "Transcricao Status",
                       "select": {"equals": "Pendente"}},
            "page_size": 100,
        }
        if cursor:
            kwargs["start_cursor"] = cursor

        if data_source_id:
            resp = notion.data_sources.query(data_source_id=data_source_id, **kwargs)
        else:
            resp = notion.databases.query(database_id=database_id, **kwargs)
        pending.extend(resp.get("results", []))
        if limit and len(pending) >= limit:
            return pending[:limit]
        if not resp.get("has_more"):
            break
        cursor = resp.get("next_cursor")
    return pending


def get_url(page):
    return (page.get("properties", {}).get("URL", {}) or {}).get("url") or ""


def get_title(page):
    title = (page.get("properties", {}).get("Name", {}) or {}).get("title") or []
    return "".join(t.get("plain_text", "") for t in title) or "(sem titulo)"


def cookies_file(cfg, tmpdir):
    """Escreve os cookies do Instagram no formato Netscape para o yt-dlp."""
    path = os.path.join(tmpdir, "cookies.txt")
    rows = [
        ("sessionid", cfg.get("ig_session_id")),
        ("csrftoken", cfg.get("ig_csrftoken")),
        ("ds_user_id", str(cfg.get("ig_user_id", ""))),
    ]
    with open(path, "w", encoding="utf-8") as f:
        f.write("# Netscape HTTP Cookie File\n")
        for name, value in rows:
            if value and not str(value).startswith("COLE_"):
                f.write(f".instagram.com\tTRUE\t/\tTRUE\t0\t{name}\t{value}\n")
    return path


def download_audio(url, tmpdir, cookies):
    """Baixa so o audio do post. Devolve (caminho, duracao) ou (None, None)."""
    out = os.path.join(tmpdir, "audio.%(ext)s")
    cmd = [
        "yt-dlp", "--quiet", "--no-warnings",
        "--cookies", cookies,
        "-f", "bestaudio/best",
        "-x", "--audio-format", "mp3",
        "--print-json", "--no-simulate",
        "-o", out, url,
    ]
    try:
        r = subprocess.run(cmd, capture_output=True, text=True,
                           timeout=DOWNLOAD_TIMEOUT)
    except subprocess.TimeoutExpired:
        log.warning("  download passou do tempo limite")
        return None, None

    if r.returncode != 0:
        err = (r.stderr or "").strip().splitlines()
        log.warning("  yt-dlp falhou: %s", err[-1] if err else "erro desconhecido")
        return None, None

    duration = None
    for line in (r.stdout or "").splitlines():
        try:
            duration = json.loads(line).get("duration")
            break
        except json.JSONDecodeError:
            continue

    for f in os.listdir(tmpdir):
        if f.startswith("audio.") and not f.endswith(".txt"):
            return os.path.join(tmpdir, f), duration
    return None, None


def transcribe(audio_path, tmpdir):
    """Roda o whisper local. Devolve o texto ou None."""
    # Saida vai para uma subpasta propria: o yt-dlp escreve o cookies.txt no
    # tmpdir, e procurar ".txt" solto acabava lendo o arquivo de cookies.
    outdir = os.path.join(tmpdir, "whisper")
    os.makedirs(outdir, exist_ok=True)

    cmd = [
        "mlx_whisper", audio_path,
        "--model", WHISPER_MODEL,
        "--language", "pt",
        "--output-format", "txt",
        "--output-dir", outdir,
        "--verbose", "False",
    ]
    try:
        r = subprocess.run(cmd, capture_output=True, text=True,
                           timeout=WHISPER_TIMEOUT)
    except subprocess.TimeoutExpired:
        log.warning("  whisper passou do tempo limite")
        return None

    if r.returncode != 0:
        err = (r.stderr or "").strip().splitlines()
        log.warning("  whisper falhou: %s", err[-1] if err else "erro desconhecido")
        return None

    for f in os.listdir(outdir):
        if f.endswith(".txt"):
            with open(os.path.join(outdir, f), encoding="utf-8") as fh:
                text = fh.read().strip()
            if text.startswith("# Netscape HTTP Cookie File"):
                log.warning("  saida invalida (arquivo de cookies), ignorando")
                return None
            return text
    return None


def fmt_duration(seconds):
    if not seconds:
        return None
    total = int(seconds)
    return f"{total // 60}:{total % 60:02d}"


def update_page(notion, page_id, status, text=None, duration=None):
    props = {"Transcricao Status": {"select": {"name": status}}}
    if text:
        props["Transcricao"] = {
            "rich_text": [{"text": {"content": text[:NOTION_TEXT_LIMIT]}}]
        }
    if duration:
        props["Duracao"] = {"rich_text": [{"text": {"content": duration}}]}
    notion.pages.update(page_id=page_id, properties=props)


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--limit", type=int, default=DEFAULT_LIMIT,
                    help=f"quantos transcrever nesta rodada (padrao {DEFAULT_LIMIT})")
    ap.add_argument("--all", action="store_true", help="transcreve todos os pendentes")
    args = ap.parse_args()

    check_tools()
    cfg = load_config()
    notion = Client(auth=cfg["notion_token"])

    limit = None if args.all else args.limit
    pending = fetch_pending(notion, cfg["notion_database_id"], limit)

    if not pending:
        log.info("Nenhum video pendente de transcricao.")
        return

    log.info("%d video(s) para transcrever", len(pending))
    done = failed = 0

    for i, page in enumerate(pending, 1):
        title = get_title(page)
        url = get_url(page)
        log.info("[%d/%d] %s", i, len(pending), title)

        if not url:
            log.warning("  sem URL, pulando")
            update_page(notion, page["id"], "Falhou")
            failed += 1
            continue

        with tempfile.TemporaryDirectory() as tmpdir:
            cookies = cookies_file(cfg, tmpdir)
            audio, seconds = download_audio(url, tmpdir, cookies)

            if not audio:
                update_page(notion, page["id"], "Falhou")
                failed += 1
                continue

            duration = fmt_duration(seconds)
            log.info("  audio baixado%s, transcrevendo...",
                     f" ({duration})" if duration else "")
            text = transcribe(audio, tmpdir)

        if not text:
            update_page(notion, page["id"], "Falhou", duration=duration)
            failed += 1
            continue

        try:
            update_page(notion, page["id"], "Transcrito", text=text, duration=duration)
        except APIResponseError as e:
            log.error("  erro do Notion: %s", e)
            failed += 1
            continue

        preview = text[:70].replace("\n", " ")
        log.info("  ok (%d chars): %s...", len(text), preview)
        done += 1

    log.info("Transcricao completa: %d transcritos | %d falharam", done, failed)


if __name__ == "__main__":
    main()
```

---

## Caminho 2 — Windows ou Mac Intel (faster-whisper)

Mesmo arquivo, com três diferenças. **Gere o arquivo completo, não um patch.**

### Instalação

```
pip install faster-whisper yt-dlp
```

O `ffmpeg` no Windows: baixar em `ffmpeg.org/download.html`, descompactar, e adicionar
a pasta `bin` ao PATH do sistema. Ou, se a pessoa tiver o winget:

```
winget install ffmpeg
```

### Diferença 1: `check_tools` não procura o mlx_whisper

```python
def check_tools():
    """No Windows o whisper roda como biblioteca, nao como programa."""
    missing = [t for t in ("yt-dlp", "ffmpeg") if not shutil.which(t)]
    if missing:
        log.error("Faltam ferramentas: %s", ", ".join(missing))
        log.error("Instale com: pip install yt-dlp  |  ffmpeg: ffmpeg.org/download.html")
        sys.exit(1)
    try:
        import faster_whisper  # noqa: F401
    except ImportError:
        log.error("Falta o faster-whisper. Instale com: pip install faster-whisper")
        sys.exit(1)
```

### Diferença 2: o modelo carrega uma vez só, fora do laço

Carregar o modelo leva alguns segundos. Fazer isso por vídeo desperdiça tempo.

```python
from faster_whisper import WhisperModel

WHISPER_MODEL = "large-v3-turbo"
_modelo = None


def get_modelo():
    """Carrega o modelo uma vez e reaproveita nas transcricoes seguintes."""
    global _modelo
    if _modelo is None:
        log.info("Carregando o modelo (so na primeira vez)...")
        # int8 e a versao compacta: cabe em uns 1,5 GB de memoria e roda em CPU.
        _modelo = WhisperModel(WHISPER_MODEL, device="cpu", compute_type="int8")
    return _modelo


def transcribe(audio_path, tmpdir):
    """Roda o faster-whisper local. Devolve o texto ou None."""
    try:
        modelo = get_modelo()
        segmentos, _ = modelo.transcribe(audio_path, language="pt")
        texto = " ".join(s.text.strip() for s in segmentos).strip()
        return texto or None
    except Exception as e:
        log.warning("  whisper falhou: %s", e)
        return None
```

### Diferença 3: avisar que a primeira rodada demora

Na primeira execução o modelo baixa (uns 1,5 GB). Avise antes, senão a pessoa acha
que travou.

**Sobre a velocidade:** em CPU, o `large-v3-turbo` com `int8` transcreve um reel de
1 minuto em algo entre 15 e 40 segundos, dependendo do processador. É mais lento que
os 7 a 8 segundos do Mac com chip M, mas roda sozinho no agendador, então a lentidão
não incomoda: ninguém está esperando na frente da tela.

---

## Caminho 3 — API da OpenAI (qualquer sistema)

Para quem prefere velocidade e não se importa que o áudio suba pra nuvem.

### Instalação

```
pip install openai yt-dlp
```

A pessoa precisa de uma chave da OpenAI (`platform.openai.com`), que vai no
`config.json` como `openai_api_key`. **Essa chave é senha: mesmo cuidado do
sessionid.**

### A função

```python
from openai import OpenAI

def transcribe(audio_path, tmpdir):
    """Transcreve pela API da OpenAI. Custa US$ 0,006 por minuto de audio."""
    cfg = load_config()
    chave = cfg.get("openai_api_key", "")
    if not chave or chave.startswith("COLE_"):
        log.error("Falta a openai_api_key no config.json")
        return None
    try:
        client = OpenAI(api_key=chave)
        with open(audio_path, "rb") as f:
            r = client.audio.transcriptions.create(
                model="whisper-1", file=f, language="pt",
            )
        return (r.text or "").strip() or None
    except Exception as e:
        log.warning("  API falhou: %s", e)
        return None
```

**Avise sobre o custo na hora de escolher, não depois.** E avise que o áudio sai da
máquina: para quem salva conteúdo sensível, o caminho local é o certo.

---

## A decisão que vale mais que o código

Quando a pessoa monta o sistema, ela já tem um acervo de salvos antigos. O instinto é
transcrever tudo.

**Não transcreva.** A Amanda tinha 488 vídeos antigos e marcou todos como
`Nao transcrever`. O raciocínio: ela não ia ler 488 transcrições de posts salvos meses
atrás. O sistema passou a transcrever só o que entra daqui pra frente, e os antigos
continuam lá: quando ela quer um, muda o status pra `Pendente` e ele entra na fila.

**Sistema bom não é o que processa tudo. É o que processa o que você vai usar.**

Na pergunta 6 da entrevista, o padrão é "só o que entrar daqui pra frente", e a skill
explica esse porquê. Se a pessoa insistir em transcrever o acervo, deixe, mas mostre a
conta primeiro: 488 vídeos × 8 segundos são uns 65 minutos de máquina no Mac, e várias
horas no Windows.

---

## Fontes dos números

- Preço da API: `platform.openai.com/docs/pricing`, consultado em setembro de 2026
  (US$ 0,006 por minuto para `whisper-1`).
- Velocidade em CPU do `large-v3-turbo` com `int8`: documentação do projeto
  faster-whisper e benchmarks públicos. Varia muito por processador, por isso a faixa.
- Os 7 a 8 segundos por reel: medidos na máquina da Amanda (Mac com chip M,
  `mlx-community/whisper-large-v3-turbo`), em setembro de 2026.
