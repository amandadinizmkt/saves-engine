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

**Sintoma:** o `sync.log` para antes de `Reordenadas N linhas do topo (+N)` e nunca
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

## Vieram salvos demais na primeira rodada

**Sintoma:** a pessoa pediu 20 ou 30, e o Notion encheu de centenas ou milhares de
linhas. No log, a linha `Primeira rodada: pegando so os N salvos mais recentes.`
**não aparece**.

**Causa quase certa: o limite estava no `config.json` e sumiu.** A pessoa abriu o
arquivo num editor de texto para colar os cookies, e o editor salvou a versão que
tinha carregado — de antes de o campo existir. O `sync.py` leu "sem limite" e
trouxe o acervo inteiro.

Aconteceu em 22/09/2026: pediram 20, vieram 2.006.

**Conserto, e ele é de arquitetura, não de valor:** o limite vira constante no topo do
`sync.py` (`LIMITE_PRIMEIRA_RODADA = 30`), e sai do `config.json` de vez. O config é
editado à mão por alguém aprendendo; decisão de comportamento não pode morar lá.

**Para arrumar o que já entrou**, decida com a pessoa antes de apagar nada:
manter tudo, manter só as N primeiras, ou apagar e recomeçar. São salvos legítimos
dela — não é lixo, é só mais do que ela pediu.

Se ela quiser manter uma parte, **conserte a coluna `Ordem` primeiro**: rodadas
interrompidas deslocam a numeração, e "as 20 primeiras pela Ordem" pode não ser as 20
mais recentes de verdade. Busque a lista real do Instagram e reescreva a `Ordem` de
cada linha pelo `Media ID`.

---

## Rodadas interrompidas deixando a `Ordem` bagunçada

**Sintoma:** números de `Ordem` repetidos, com buracos, ou muito maiores que o total de
linhas. No log, várias sequências de `Pagina 1: 50 salvos` recomeçando do zero.

**Causa:** cada Ctrl+C mata o processo no meio da escrita. O lock do sistema se solta
junto (é assim que o `flock` funciona), então a rodada seguinte entra normalmente — e
recomeça a varredura **do começo**, chamando o reordenamento de novo, sobre linhas que
já tinham sido deslocadas. Três interrupções viram três deslocamentos nos mesmos dados.

**Como evitar:** depois de interromper, **não rode de novo na sequência.** Veja quantas
linhas entraram no Notion e o que o `state.json` registrou, e só então decida.

**Cuidado com o state defasado:** se a rodada morreu entre dois pontos de gravação (o
state grava a cada 25), o `state.json` conhece menos linhas do que existem no Notion.
A rodada seguinte vai tentar reescrever essa diferença. Compare os dois números antes
de rodar de novo.

**Como consertar:** busque a ordem real no Instagram (`feed/saved/posts/`, o primeiro é
o 1) e reescreva a `Ordem` de cada linha casando pelo `Media ID`. Depois confira: sem
duplicatas, e a faixa indo de 1 até o total de linhas.

---

## As transcrições voltam "Música", "Tchau" ou créditos inventados

**Sintoma:** o `transcribe.py` diz `N transcritos | 0 falharam`, mas o texto no Notion é
`Música`, `Tchau.`, `A B B A A A...`, ou algo como `Transcrição e Legendas <um nome>`.

**Isso não é erro.** O whisper transcreveu o que existia, e o que existia era música de
fundo. O texto de créditos é alucinação conhecida do modelo em áudio instrumental: ele
viu milhares de vídeos terminando assim durante o treino.

**O que isso revela é sobre o conteúdo, não sobre o sistema.** Se a maioria dos salvos
da pessoa é inspiração visual — decoração, moda, casamento, design —, a informação está
na imagem, e a transcrição rende pouco. Se são pessoas falando de negócio, ensinando ou
explicando, a transcrição é a parte mais valiosa do sistema.

**O que fazer:** diga isso a ela com franqueza, e ajuste. Numa conta visual, vale marcar
mais coisa como `Nao transcrever` e apoiar a ideação na legenda e no perfil do autor.
Não force a transcrição onde ela não tem o que capturar.

---

## O que NÃO é problema

**`Nao consegui listar colecoes (HTTP 404)`** aparece toda rodada e é esperado
(pedra 2). Pode ignorar.

**O primeiro sync demorar alguns minutos.** Cada salvo vira uma página do Notion, uma
de cada vez, a cerca de meio segundo. Com o limite padrão de 30, são menos de dois
minutos; sem limite, uma conta com mil salvos passa de dez. Com o state incremental,
mesmo que interrompa, o trabalho não se perde.

**Um salvo antigo com `Transcricao Status = Nao transcrever`.** É a decisão de projeto,
não um erro. Ver `transcricao.md`.
