# O robô que busca: sync.py

Este é o código que a skill gera. Ele é uma versão corrigida do que circula em
tutoriais: as quatro proteções da seção final não existem no original, e são elas
que fazem a diferença entre "funcionou quando eu rodei na mão" e "funciona sozinho
por seis meses".

**Gere o arquivo adaptando os nomes de coluna e o id da base para o que a pessoa
respondeu na entrevista. Não mude a lógica das quatro proteções.**

---

## O que ele faz, em ordem

1. Pega um lock de processo (impede duas cópias rodando juntas).
2. Lê o `config.json` e o `state.json`.
3. Monta uma sessão HTTP com os cookies do Instagram.
4. Valida a sessão: se os cookies expiraram, para aqui com mensagem clara.
5. Busca todos os salvos, de 50 em 50, com pausa de 1 segundo entre páginas.
6. Descobre quais são novos, comparando o `Media ID` com o `state.json`.
7. Empurra a `Ordem` das linhas existentes pra baixo.
8. Escreve os novos no Notion, gravando o state a cada 25.
9. Se entrou vídeo novo, chama a transcrição.

---

## O código

```python
#!/usr/bin/env python3
"""
Saves Engine - daemon de sincronizacao.

Puxa os posts salvos do Instagram usando os cookies de sessao do navegador
e grava cada salvo novo numa base do Notion com Status = New.

Roda sozinho pelo agendador, duas vezes por dia.
"""

import json
import logging
import os
import sys
import time
from datetime import datetime, timezone

import requests
from notion_client import Client
from notion_client.errors import APIResponseError

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
CONFIG_PATH = os.path.join(BASE_DIR, "config.json")
STATE_PATH = os.path.join(BASE_DIR, "state.json")
LOG_PATH = os.path.join(BASE_DIR, "sync.log")

# Id publico do app web do Instagram. E o mesmo para todo mundo, nao e segredo.
IG_APP_ID = "936619743392459"
USER_AGENT = (
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 "
    "(KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36"
)

CAPTION_LIMIT = 1900       # o Notion corta em 2000; 1900 e a margem
PAGE_SIZE = 50
SLEEP_BETWEEN_PAGES = 1.0  # educado com o Instagram, e o que mantem a conta discreta
STATE_EVERY = 25           # grava o progresso a cada 25 posts

# Quantos salvos trazer na PRIMEIRA rodada. 0 = todos.
# FICA AQUI, e nao no config.json, de proposito. O config e aberto num editor de
# texto para colar cookies, e um Cmd+S pode gravar uma versao antiga do arquivo,
# apagando o limite sem ninguem perceber. Aconteceu em 22/09/2026: vieram 2006
# salvos em vez de 20. Decisao de comportamento mora no codigo; o config guarda
# so o que muda por pessoa (cookies, token, ids das bases).
LIMITE_PRIMEIRA_RODADA = 30


def setup_logging():
    logger = logging.getLogger("saves-engine")
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


def load_config():
    if not os.path.exists(CONFIG_PATH):
        log.error("config.json nao encontrado. Copie config.example.json e preencha.")
        sys.exit(1)
    with open(CONFIG_PATH, encoding="utf-8") as f:
        cfg = json.load(f)

    required = ["ig_session_id", "ig_csrftoken", "ig_user_id",
                "notion_token", "notion_database_id"]
    missing = [k for k in required if not cfg.get(k) or str(cfg[k]).startswith("COLE_")]
    if missing:
        log.error("Faltam valores no config.json: %s", ", ".join(missing))
        sys.exit(1)
    return cfg


def load_state():
    if not os.path.exists(STATE_PATH):
        return {"synced_media_ids": []}
    try:
        with open(STATE_PATH, encoding="utf-8") as f:
            state = json.load(f)
        state.setdefault("synced_media_ids", [])
        return state
    except json.JSONDecodeError:
        log.warning("state.json corrompido. Comecando do zero.")
        return {"synced_media_ids": []}


def save_state(state):
    state["last_sync"] = datetime.now(timezone.utc).isoformat()
    # Escreve num arquivo temporario e so entao substitui o original: se a maquina
    # desligar no meio da escrita, o state antigo continua intacto.
    tmp = STATE_PATH + ".tmp"
    with open(tmp, "w", encoding="utf-8") as f:
        json.dump(state, f, indent=2)
    os.replace(tmp, STATE_PATH)


def make_session(cfg):
    s = requests.Session()
    s.headers.update({
        "User-Agent": USER_AGENT,
        "X-IG-App-ID": IG_APP_ID,
        "X-CSRFToken": cfg["ig_csrftoken"],
        "Accept": "*/*",
        "Accept-Language": "pt-BR,pt;q=0.9,en;q=0.8",
        "Referer": "https://www.instagram.com/",
    })
    s.cookies.set("sessionid", cfg["ig_session_id"], domain=".instagram.com")
    s.cookies.set("csrftoken", cfg["ig_csrftoken"], domain=".instagram.com")
    s.cookies.set("ds_user_id", str(cfg["ig_user_id"]), domain=".instagram.com")
    return s


def get_com_retry(s, url, params=None, tentativas=4, timeout=30):
    """
    PROTECAO 1 - GET que insiste quando a rede falha.

    O agendador roda as 9h e as 21h, e nesses horarios o computador pode estar
    acordando do sono, com o wi-fi ainda nao conectado. Sem retry, o sync do dia
    inteiro se perdia por causa de alguns segundos sem internet.
    """
    espera = 15
    ultimo_erro = None
    for tentativa in range(1, tentativas + 1):
        try:
            return s.get(url, params=params, timeout=timeout)
        except requests.RequestException as e:
            ultimo_erro = e
            if tentativa < tentativas:
                log.warning("Rede falhou (tentativa %d de %d). Nova tentativa em %ds.",
                            tentativa, tentativas, espera)
                time.sleep(espera)
                espera *= 2   # 15s, 30s, 60s
    raise ultimo_erro


def validate_session(s):
    """Confere os cookies antes de qualquer coisa, e da mensagem util se expiraram."""
    url = "https://www.instagram.com/api/v1/accounts/edit/web_form_data/"
    try:
        r = get_com_retry(s, url)
    except requests.RequestException as e:
        log.error("Erro de rede ao validar a sessao: %s", e)
        sys.exit(1)

    if r.status_code != 200:
        log.error("Sessao do Instagram invalida (HTTP %s). "
                  "Pegue cookies novos no navegador e atualize o config.json.",
                  r.status_code)
        sys.exit(1)
    try:
        username = r.json().get("form_data", {}).get("username", "?")
    except ValueError:
        log.error("Resposta inesperada do Instagram. Cookies provavelmente expiraram.")
        sys.exit(1)
    log.info("Sessao do Instagram valida para @%s", username)
    return username


def fetch_saved_posts(s, limite=0):
    """Busca os salvos, de 50 em 50. O mais recente vem primeiro.
       limite=0 traz todos; qualquer outro numero para ao chegar nele."""
    posts = []
    url = "https://www.instagram.com/api/v1/feed/saved/posts/"
    next_max_id = None
    page = 0

    while True:
        params = {"count": PAGE_SIZE}
        if next_max_id:
            params["max_id"] = next_max_id
        try:
            r = get_com_retry(s, url, params=params)
        except requests.RequestException as e:
            log.error("Erro de rede ao buscar salvos: %s", e)
            break

        if r.status_code != 200:
            log.error("Falha ao buscar salvos (HTTP %s).", r.status_code)
            break
        try:
            data = r.json()
        except ValueError:
            log.error("Resposta invalida ao buscar salvos.")
            break

        items = data.get("items", [])
        for it in items:
            posts.append(it.get("media", it))
        page += 1
        log.info("Pagina %d: %d salvos (total %d)", page, len(items), len(posts))

        if limite and len(posts) >= limite:
            posts = posts[:limite]
            log.info("Limite de %d salvos atingido. Parando por aqui.", limite)
            break

        if not data.get("more_available"):
            break
        next_max_id = data.get("next_max_id")
        if not next_max_id:
            break
        time.sleep(SLEEP_BETWEEN_PAGES)

    return posts


def classify(media):
    """Descobre o tipo do post a partir de media_type e product_type."""
    product = (media.get("product_type") or "").lower()
    mtype = media.get("media_type")
    if product in ("clips", "reels"):
        return "Reel"
    if product == "igtv":
        return "IGTV"
    if mtype == 8:
        return "Carousel"
    if mtype == 2:
        return "Reel"
    return "Post"


def build_url(media, kind):
    code = media.get("code") or ""
    if not code:
        return ""
    path = "reel" if kind == "Reel" else "p"
    return f"https://instagram.com/{path}/{code}/"


def extract_caption(media):
    cap = media.get("caption")
    text = (cap or {}).get("text", "") if isinstance(cap, dict) else ""
    return text[:CAPTION_LIMIT]


def resolve_data_source(notion, database_id):
    """
    Descobre o data source da base.

    A API nova do Notion consulta por data_source, nao mais por database. Versoes
    antigas da biblioteca ainda usam databases.query, entao damos suporte aos dois.
    Se a consulta falhar (permissao, base movida), devolvemos None em vez de
    derrubar o sync inteiro: a escrita por database_id continua funcionando.
    """
    if not hasattr(notion, "data_sources"):
        return None
    try:
        db = notion.databases.retrieve(database_id=database_id)
    except APIResponseError as e:
        log.warning("Nao consegui ler o data source da base: %s", e)
        log.warning("Confira se a integracao esta conectada NESTA base.")
        return None
    sources = db.get("data_sources") or []
    return sources[0]["id"] if sources else None


def query_base(notion, database_id, data_source_id, **kwargs):
    """Consulta a base pela API nova quando ela existe, pela antiga quando nao."""
    if data_source_id:
        return notion.data_sources.query(data_source_id=data_source_id, **kwargs)
    return notion.databases.query(database_id=database_id, **kwargs)


def shift_existing_order(notion, database_id, data_source_id, quantidade, teto=200):
    """
    PROTECAO 2 - Empurra a Ordem das linhas que ja existem, mas so ate o teto.

    A Ordem e a posicao na lista de salvos: 1 = o mais recente. Quando chegam
    salvos novos, o que ja estava desce essa quantidade de posicoes.

    O teto existe por um motivo pratico: reordenar 800 linhas sao 800 chamadas ao
    Notion, uns 5 minutos, e qualquer falha no meio derrubava a rodada inteira
    ANTES de escrever os salvos novos. Quem olha a lista olha o topo, entao
    reordenamos so as primeiras 200 linhas. As de baixo ficam com a Ordem antiga,
    o que nao muda nada na pratica: elas continuam depois das 200.

    Se a consulta falhar, avisamos e seguimos sem reordenar. Perder a ordem e
    chato; perder os salvos novos e pior.
    """
    if quantidade <= 0:
        return

    try:
        resp = query_base(notion, database_id, data_source_id,
                          page_size=min(teto, 100),
                          sorts=[{"property": "Ordem", "direction": "ascending"}])
    except APIResponseError as e:
        log.warning("Nao consegui reordenar (%s). Seguindo sem reordenar.", e)
        return

    linhas = resp.get("results", [])
    # Busca a segunda pagina se o teto for maior que 100
    while len(linhas) < teto and resp.get("has_more"):
        try:
            resp = query_base(notion, database_id, data_source_id,
                              page_size=min(teto - len(linhas), 100),
                              start_cursor=resp["next_cursor"],
                              sorts=[{"property": "Ordem", "direction": "ascending"}])
        except APIResponseError:
            break
        linhas.extend(resp.get("results", []))

    movidas = 0
    falhas = 0
    primeiro_erro = None
    for page in linhas[:teto]:
        atual = (page["properties"].get("Ordem") or {}).get("number")
        if atual is None:
            continue
        try:
            notion.pages.update(page_id=page["id"],
                                properties={"Ordem": {"number": atual + quantidade}})
            movidas += 1
        except APIResponseError as e:
            falhas += 1
            if primeiro_erro is None:
                primeiro_erro = e
            # Nao loga uma linha por falha: quando o problema e geral (permissao,
            # rede), seriam 200 linhas identicas escondendo o resto do log. Conta
            # e reporta uma vez so, no fim.
            if falhas >= 10:
                break

    if falhas:
        log.warning("Nao consegui reordenar %d linha(s). Primeiro erro: %s",
                    falhas, primeiro_erro)
        if falhas >= 10:
            log.warning("Parei de tentar reordenar. Os salvos novos entram do mesmo jeito.")

    log.info("Reordenadas %d linhas do topo (+%d)", movidas, quantidade)


def write_to_notion(notion, database_id, media, ordem=None):
    kind = classify(media)
    author = (media.get("user") or {}).get("username", "desconhecido")
    code = media.get("code") or media.get("pk")
    pk = str(media.get("pk"))

    props = {
        "Name": {"title": [{"text": {"content": f"@{author}/{code}"}}]},
        "Author": {"rich_text": [{"text": {"content": author}}]},
        "Type": {"select": {"name": kind}},
        "Status": {"select": {"name": "New"}},
        "Media ID": {"rich_text": [{"text": {"content": pk}}]},
        "Saved": {"date": {"start": datetime.now(timezone.utc).isoformat()}},
    }

    url = build_url(media, kind)
    if url:
        props["URL"] = {"url": url}

    caption = extract_caption(media)
    if caption:
        props["Caption"] = {"rich_text": [{"text": {"content": caption}}]}

    # Video entra na fila de transcricao. Foto nao tem audio para transcrever.
    if kind in ("Reel", "IGTV"):
        props["Transcricao Status"] = {"select": {"name": "Pendente"}}
    else:
        props["Transcricao Status"] = {"select": {"name": "Sem audio"}}

    if ordem is not None:
        props["Ordem"] = {"number": ordem}

    notion.pages.create(parent={"database_id": database_id}, properties=props)


def acquire_lock():
    """
    PROTECAO 3 - Impede duas copias do sync rodando juntas.

    Duas rodadas em paralelo brigam pelo state.json e acabam criando paginas
    duplicadas. O lock e liberado sozinho quando o processo termina, entao um
    arquivo sobrando de uma rodada anterior nao trava nada.
    """
    import fcntl
    lock_path = os.path.join(BASE_DIR, ".sync.lock")
    fh = open(lock_path, "w")
    try:
        fcntl.flock(fh, fcntl.LOCK_EX | fcntl.LOCK_NB)
    except OSError:
        log.error("Ja existe um sync rodando. Saindo para nao duplicar paginas.")
        sys.exit(1)
    fh.write(str(os.getpid()))
    fh.flush()
    return fh


def main():
    lock = acquire_lock()  # mantido aberto ate o processo terminar
    cfg = load_config()
    state = load_state()
    known = set(state["synced_media_ids"])

    s = make_session(cfg)
    validate_session(s)

    log.info("Buscando posts salvos...")
    # O limite vale so enquanto a base estiver vazia. Depois disso busca tudo:
    # a deduplicacao pelo Media ID segura o que ja existe.
    limite = 0
    if not state.get("primeira_rodada_feita"):
        limite = LIMITE_PRIMEIRA_RODADA
        if limite:
            log.info("Primeira rodada: pegando so os %d salvos mais recentes.", limite)

    posts = fetch_saved_posts(s, limite=limite)

    notion = Client(auth=cfg["notion_token"])
    database_id = cfg["notion_database_id"]
    data_source_id = resolve_data_source(notion, database_id)

    new = skipped = errors = 0

    # Quais sao realmente novos. O Instagram devolve a lista com o mais recente
    # primeiro, entao os novos estao no comeco.
    novos = [m for m in posts
             if str(m.get("pk", "")) and str(m["pk"]) not in known]

    if novos:
        shift_existing_order(notion, database_id, data_source_id, len(novos))

    # Ordem 1 = o mais recente de todos.
    posicao = {str(m["pk"]): i for i, m in enumerate(novos, 1)}

    for media in posts:
        pk = str(media.get("pk", ""))
        if not pk:
            continue
        if pk in known:
            skipped += 1
            continue

        try:
            write_to_notion(notion, database_id, media, ordem=posicao.get(pk))
        except APIResponseError as e:
            errors += 1
            log.error("Erro do Notion no post %s: %s", pk, e)
            continue
        except Exception as e:
            errors += 1
            log.error("Erro inesperado no post %s: %s", pk, e)
            continue

        # So entra no state DEPOIS que o Notion confirmou a escrita.
        known.add(pk)
        new += 1

        # PROTECAO 4 - Grava o state a cada 25 posts.
        # O primeiro sync costuma ser longo. Se o script for interrompido no meio,
        # o trabalho feito nao se perde e a proxima rodada nao recria as paginas
        # ja escritas. Sem isso, uma interrupcao no meio de 800 posts vira 1.600
        # linhas duplicadas na rodada seguinte.
        if new % STATE_EVERY == 0:
            state["synced_media_ids"] = sorted(known)
            save_state(state)
            log.info("  ... %d gravados no Notion", new)

    state["synced_media_ids"] = sorted(known)
    state["primeira_rodada_feita"] = True
    save_state(state)

    log.info("Sync completo: %d novos | %d ja existiam | %d total | %d erros",
             new, skipped, len(posts), errors)

    # Video novo ja sai transcrito: a transcricao roda em seguida, sozinha.
    if new > 0:
        transcrever_novos()


def transcrever_novos():
    """Chama o transcribe.py logo apos o sync, para os videos que entraram."""
    import subprocess
    script = os.path.join(BASE_DIR, "transcribe.py")
    python = os.path.join(BASE_DIR, ".venv", "bin", "python3")
    if not os.path.exists(script):
        return
    log.info("Transcrevendo os videos que acabaram de entrar...")
    try:
        r = subprocess.run(
            [python if os.path.exists(python) else sys.executable, script, "--all"],
            capture_output=True, text=True, timeout=3600, cwd=BASE_DIR,
        )
        for linha in (r.stdout or "").strip().splitlines()[-3:]:
            log.info("  %s", linha.split("  ", 1)[-1])
    except subprocess.TimeoutExpired:
        log.warning("A transcricao passou do tempo limite; roda na proxima.")
    except Exception as e:
        log.warning("Nao consegui rodar a transcricao: %s", e)


if __name__ == "__main__":
    main()
```

---

## As quatro proteções, e por que elas existem

Estas quatro coisas não estão nos tutoriais que ensinam esse sistema. Cada uma nasceu
de um problema real, em produção.

**1. Retry de rede com espera crescente.** O agendador acorda o script às 9h, quando o
computador pode estar saindo do sono com o wi-fi ainda subindo. Sem retry, o sync do dia
inteiro se perdia por três segundos sem internet. Espera 15s, 30s, 60s.

**2. Reordenamento com teto de 200 linhas.** A versão sem teto percorre a base inteira,
uma chamada por linha. Com 800 salvos isso são uns 5 minutos de chamadas, e **essa etapa
acontece antes de escrever os salvos novos**: qualquer falha no meio derrubava a rodada
sem gravar nada. Quem olha a lista olha o topo, então 200 linhas resolvem. Se o
reordenamento falhar, o script avisa e segue: perder a ordem é chato, perder os salvos
novos é pior.

**3. Lock de processo.** Duas cópias rodando juntas brigam pelo `state.json` e duplicam
páginas. Acontece quando a pessoa roda na mão enquanto o agendador dispara.

**4. State incremental a cada 25.** É a proteção mais importante. O código original só
grava o controle de duplicatas no fim. Num primeiro carregamento de 800 posts, uma
interrupção no meio deixa 600 linhas escritas e zero registradas, e a próxima rodada
recria todas as 600. Gravar a cada 25 custa nada e evita isso.

---

## A conexão com o Notion, e o erro que confunde

`resolve_data_source` pode falhar por dois motivos que dão a **mesma mensagem**:

```
Could not find database with ID: xxxx. Make sure the relevant pages and
databases are shared with your integration.
```

1. A integração não está conectada nessa base (o mais comum).
2. O id no `config.json` está errado ou a base foi movida.

A mensagem fala em "could not find", o que faz a pessoa procurar defeito no id. Na
maioria das vezes é permissão. **Confira a conexão antes de mexer no id.**

Nesta versão, essa falha não derruba mais o sync: `resolve_data_source` devolve `None`,
o código cai na API antiga e a escrita continua funcionando.
