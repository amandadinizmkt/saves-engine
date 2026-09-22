# Problemas conhecidos

Cada item aqui nasceu de um problema real, em produção. Quando a pessoa trouxer um
sintoma, procure aqui antes de investigar do zero.

---

## As quatro pedras do caminho

Este sistema é adaptado do tutorial "The Instagram Saves Engine", de Justyn Berk.
Quatro coisas não funcionaram como descrito, e é justamente aí que está o valor de
ensinar isso direito.

### Pedra 1 — A biblioteca do Notion mudou

**Sintoma:** `AttributeError: 'Client' object has no attribute 'databases'` ou erro no
`databases.query`.

**O que houve:** o `notion-client` 3.x removeu `databases.query`. A consulta agora é
por **data source**, não por database. O id do data source vem de:

```python
db = notion.databases.retrieve(database_id=database_id)
data_source_id = db["data_sources"][0]["id"]
```

**Importante:** a **escrita** com `parent={"database_id": ...}` continua funcionando
normalmente. Só a leitura mudou. Quem segue o tutorial original toma erro na primeira
tentativa de consulta.

**Como o código resolve:** `resolve_data_source()` testa se a biblioteca tem
`data_sources` e usa a API nova quando tem, a antiga quando não tem. Funciona nas duas
versões.

---

### Pedra 2 — O Instagram fechou a porta das coleções

**Sintoma:** `Nao consegui listar colecoes (HTTP 404)` no log, toda rodada.

**O que houve:** o endereço `/api/v1/collections/list/` foi descontinuado. Foram
testadas quatro variações, todas mortas.

**A consequência:** **não existe mais mapeamento automático de pasta do Instagram para
pilar editorial.** Os posts trazem um campo `saved_collection_ids`, mas ele vem vazio.

**O que fazer:** não tente consertar, e **não gere o código que chama esse endereço**.
O pilar é decidido pelo cérebro, na ideação, com base no conteúdo. Funciona melhor que
o mapeamento por pasta funcionava, porque lê a transcrição em vez do nome da pasta.

Se o sistema da pessoa já tem esse aviso no log, é cosmético: pode ignorar ou remover
a função.

---

### Pedra 3 — O controle de duplicatas só no fim

**Esta é a que mais dói.**

**Sintoma:** a base do Notion com o dobro das linhas, tudo duplicado.

**O que houve:** o código original grava o `state.json` (o controle de duplicatas) só
no final da execução. O primeiro carregamento da Amanda levou mais de 10 minutos e foi
interrompido no meio, com 616 linhas já escritas no Notion e **zero** registradas no
state. Rodar de novo do jeito original teria criado 1.232 linhas.

**Como o código resolve:** grava o state a cada 25 posts, e só registra um id **depois**
que o Notion confirmou a escrita daquela linha. Custa nada e resolve de vez.

**Se já aconteceu com a pessoa:** a saída é apagar as linhas duplicadas no Notion
(dá pra filtrar por `Media ID` repetido) e reconstruir o `state.json` a partir dos
`Media ID` que sobraram. Não rode o sync antes de fazer isso, senão duplica de novo.

---

### Pedra 4 — O reordenamento derrubando a rodada inteira

Esta não está no tutorial original: apareceu em produção, no sistema da Amanda, em
setembro de 2026.

**Sintoma:** o `sync.log` termina em `Reordenando as linhas ja existentes (+N)` e nunca
chega ao `Sync completo`. Os salvos novos não entram no Notion. O `state.json` fica
com `last_sync: None`.

**O que houve:** duas coisas ao mesmo tempo.

1. A função de reordenamento percorria **a base inteira**, uma chamada de update por
   linha. Com 825 salvos são uns 5 minutos de chamadas ao Notion.
2. Essa etapa acontece **antes** de escrever os salvos novos. Qualquer falha no meio
   derruba a rodada sem gravar nada.

No caso real, a falha foi um erro de permissão do Notion no meio do reordenamento, e o
efeito foi o sistema parar de capturar por dias sem ninguém perceber, porque o log não
gritava.

**Como o código resolve:** três mudanças.

- O reordenamento tem teto de 200 linhas. Quem olha a lista olha o topo.
- Se o reordenamento falhar, o script **avisa e segue**. Perder a ordem é chato;
  perder os salvos novos é pior.
- `resolve_data_source` não derruba mais o sync quando falha: devolve `None` e o código
  cai na API antiga.

**A lição, que vale pra qualquer sistema:** uma etapa cosmética nunca pode bloquear a
etapa essencial. Se o bonito falhar, o importante continua.

---

## Sessão do Instagram expirada

**Sintoma:** `Sessao do Instagram invalida (HTTP 401)` ou `(HTTP 403)` no log, e o sync
para logo no começo.

**O que houve:** os cookies expiraram. **Isso é normal e vai acontecer.** Não é sinal
de que a conta foi bloqueada.

**Como resolver:** ação 5 do comando (`refresh session`). Pegar `sessionid`,
`csrftoken` e `ds_user_id` de novo no DevTools e atualizar o `config.json`.

**Com que frequência:** varia. Costuma durar semanas ou meses. Trocar de senha, sair
da conta ou o Instagram invalidar a sessão por conta própria derrubam antes.

---

## "Could not find database with ID"

**Sintoma:**

```
Could not find database with ID: xxxxx. Make sure the relevant pages and
databases are shared with your integration.
```

**Duas causas possíveis, mesma mensagem:**

1. **A integração não está conectada nessa base.** É o mais comum, de longe. A pessoa
   conectou na Instagram Saves e esqueceu da Content Ideas, ou vice-versa.
2. O id no `config.json` está errado, ou a base foi movida/duplicada no Notion.

**A mensagem engana:** ela fala em "could not find", o que faz a pessoa procurar defeito
no id. **Confira a conexão primeiro:** abra a base, menu `...` no canto superior
direito, `Conexões`, e veja se a integração está lá.

---

## A transcrição falha em todos os vídeos

**Sintoma:** `yt-dlp falhou` em todos, `Transcricao Status` virando `Falhou` em série.

**Causas, em ordem de probabilidade:**

1. **`yt-dlp` desatualizado.** O Instagram muda as coisas e o `yt-dlp` corre atrás. É a
   causa mais comum. Resolve com:
   ```bash
   pip install -U yt-dlp
   ```
   Vale tentar isso antes de qualquer outra coisa.

2. **Cookies expirados.** O download também usa os cookies. Se o sync está reclamando
   de sessão, é isso.

3. **`ffmpeg` faltando.** O `yt-dlp` precisa dele para extrair o áudio.

**Se falha em alguns só:** posts de contas privadas que a pessoa deixou de seguir, ou
posts apagados. Normal. Deixe como `Falhou` e siga.

---

## A saída da transcrição vem com o texto dos cookies

**Sintoma:** a coluna `Transcricao` com `# Netscape HTTP Cookie File`.

**O que houve:** o `yt-dlp` escreve o `cookies.txt` na mesma pasta temporária, e o
código procurava por qualquer arquivo `.txt` ali.

**Como o código resolve:** a saída do whisper vai para uma subpasta própria, e existe
uma checagem extra que descarta o texto se ele começar com o cabeçalho de cookies.

Se a pessoa gerou o código sem essa proteção, é isso que está acontecendo.

---

## O agendador não dispara

**Mac:**
```bash
launchctl list | grep saves-engine
```
Se não retorna nada, não está carregado. Carregue com `launchctl load`.
Se retorna com a segunda coluna diferente de 0, a última rodada deu erro: veja o
`launchd-stderr.log`.

**Causa comum no Mac:** caminho relativo no `.plist`. O launchd não usa o ambiente do
terminal, então `python3` sozinho não resolve. Tem que ser o caminho completo do
`.venv`.

**Windows:**
```
schtasks /query /tn "SavesEngine-Manha"
```
**Causa comum no Windows:** espaço no caminho da pasta. Ver `agendador.md`.

---

## Páginas duplicadas mesmo com o state certo

**Sintoma:** linhas repetidas, e o `state.json` parece correto.

**Causa:** duas cópias do sync rodando ao mesmo tempo. Acontece quando a pessoa roda na
mão enquanto o agendador dispara.

**Como o código resolve:** o lock de processo (`acquire_lock`). A segunda cópia sai com
`Ja existe um sync rodando`.

**Se aparecer essa mensagem e não houver sync rodando:** não é o arquivo de lock
sobrando. O lock é do sistema operacional e morre junto com o processo, então um lock
ocupado significa que existe um sync vivo de verdade. Confira com:

```bash
ps aux | grep sync.py | grep -v grep
```

---

## O `Media ID` parando de deduplicar

**Sintoma:** tudo duplicando, e os ids no Notion terminando em zeros.

**Causa:** a coluna `Media ID` foi criada como **Number** em vez de **Text**. Os ids do
Instagram passam de 17 dígitos e o Notion arredonda número grande, então ids diferentes
viram o mesmo número.

**Como resolver:** mudar o tipo da coluna para Text no Notion. Os valores já
arredondados estão perdidos, mas o `state.json` continua servindo de deduplicação.

---

## O que NÃO é problema

**`Nao consegui listar colecoes (HTTP 404)`** aparece toda rodada e é esperado
(pedra 2). Pode ignorar.

**O primeiro sync demorar 10 minutos.** Com centenas de salvos, é uma página do Notion
por vez. Normal. Com o state incremental, mesmo que interrompa, o trabalho não se perde.

**Um salvo antigo com `Transcricao Status = Nao transcrever`.** É a decisão de projeto,
não um erro. Ver `transcricao.md`.
