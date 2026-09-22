# O cérebro que transforma: o comando de ideação

Esta é a peça esperta do sistema. O sync é burro de propósito: ele só captura, rápido
e sem pensar. Pensar acontece quando a pessoa está pronta pra pensar, e é aqui.

A skill gera este arquivo em `.claude/commands/saves-engine.md`, **dentro da pasta do
projeto dela**, com o público e os pilares **dela** preenchidos. Não entregue o
comando com os pilares da Amanda dentro.

---

## Como gerar

Copie o template abaixo e substitua:

- `{{PASTA}}` — caminho absoluto da pasta do projeto
- `{{BASE_SAVES}}` — id da base Instagram Saves
- `{{BASE_IDEAS}}` — id da base Content Ideas
- `{{PUBLICO}}` — a resposta da pergunta 2 da entrevista, em uma frase
- `{{PILARES}}` — os pilares da pergunta 3, separados por vírgula
- `{{PILAR_FERRAMENTA}}` — o pilar da pergunta 4, o que fala de ferramenta e IA
- `{{PYTHON}}` — `.venv/bin/python3` no Mac, `.venv\Scripts\python.exe` no Windows

Se a pessoa não tiver um pilar de ferramenta (pergunta 4 respondida como "nenhum"),
apague o parágrafo da regra dos pilares e a menção ao `{{PILAR_FERRAMENTA}}`.

---

## O template

````markdown
---
description: Sincroniza os salvos do Instagram e transforma em ideias de conteudo no Notion
---

# /saves-engine

Voce opera o Saves Engine. Leia o argumento do comando e execute a acao correspondente.

**Argumento recebido:** $ARGUMENTS

Se nenhum argumento vier, pergunte qual das seis acoes ela quer.

## Constantes do projeto

- Pasta do projeto: `{{PASTA}}`
- Base **Instagram Saves** (salvos brutos): `{{BASE_SAVES}}`
- Base **Content Ideas** (ideias prontas): `{{BASE_IDEAS}}`
- Publico: {{PUBLICO}}
- Pilares editoriais: {{PILARES}}

Regra dos pilares: so **{{PILAR_FERRAMENTA}}** fala de ferramenta e IA. Os outros nao
precisam citar IA nenhuma.

## Acao 1 - sync (rodar a sincronizacao agora)

Rode `{{PYTHON}} sync.py` a partir da pasta do projeto. Mostre o resumo final.
Em seguida, se houver videos com `Transcricao Status = "Pendente"`, rode
`{{PYTHON}} transcribe.py` para transcrever antes de idear.
Se o resultado tiver algum salvo com Status = New, siga direto para a acao **ideate**.

## Acao 1b - transcribe (transcrever os videos salvos)

Rode `{{PYTHON}} transcribe.py` a partir da pasta do projeto. Ele busca os salvos com
`Transcricao Status = "Pendente"`, baixa o audio, transcreve e grava na coluna
`Transcricao`.

Por padrao transcreve 10 por rodada. Use `--all` para todos ou `--limit N` para uma
quantidade especifica.

## Acao 2 - ideate (transformar salvos em ideias)

Pipeline de 6 passos:

**Passo 1.** Consulte a base Instagram Saves filtrando `Status = "New"`. Traga Caption,
Author, Type, URL e **Transcricao** de cada um.

Antes de seguir: se algum desses salvos for video e estiver com `Transcricao Status =
"Pendente"`, rode `{{PYTHON}} transcribe.py` primeiro. A transcricao e a melhor
materia-prima que existe aqui. Em reel, a legenda costuma ser so "link na bio"
enquanto o conteudo real esta na fala.

Se houver mais de 10 pendentes, processe em lotes de 10 com revisao entre os lotes.

**Passo 2.** Para cada salvo, gere **uma** ideia de conteudo:

- Baseie a ideia na **Transcricao** quando ela existir, usando a Caption como apoio.
  Sem transcricao, use a Caption.
- Reframe a ideia original para o publico dela (nunca copie o post: traduza o conceito
  para a realidade de quem {{PUBLICO}}).
- Defina o pilar entre {{PILARES}}.
- Escreva **3 opcoes de hook**: uma de curiosidade, uma de valor, uma emocional.
- Escreva um outline estruturado: **HOOK**, **3 a 4 PONTOS PRINCIPAIS**, **CTA**.
- Escreva os desdobramentos por plataforma:
  - Instagram: carrossel ou reel. Se for carrossel, escreva o texto slide a slide,
    **tudo em letras minusculas e curto**.
  - TikTok: roteiro falado de 30 a 60 segundos.
  - YouTube: so inclua se o tema tiver profundidade de video longo.
- Defina a Priority: **High** se for tema central ou urgente, **Medium** se tiver forte
  aderencia ao publico, **Low** se for inspiracao tangencial.

Tom: portugues do Brasil, linguagem neutra, sem travessoes, sem promessa exagerada.

**Passo 3.** Apresente todas as ideias com titulo, pilar, prioridade, plataformas, as 3
hooks, o outline e os desdobramentos. Pergunte: aprovar todas, escolher por numero,
pular todas, ou pedir ajustes.

**Passo 4.** Para cada ideia aprovada, crie uma pagina na base Content Ideas com:
- `Name`: titulo da ideia
- `Platform`: multi-select com as plataformas
- `Format`: Carousel, Reel, Short Video ou Long-form Video
- `Status`: "Not started"
- `Angle`: o angulo do conteudo
- `Hook Options`: as 3 hooks unidas por " | "
- `Priority`: High, Medium ou Low
- `Week Of`: a segunda-feira da semana atual
- `Pillar`: o pilar definido

No corpo da pagina: o angulo, o outline completo, e uma secao para cada plataforma.

**Passo 5.** Atualize o Status de cada salvo processado na base Instagram Saves:
**"Used"** se a ideia foi aprovada, **"Reviewed"** se foi pulada.

**Passo 6.** Imprima o resumo **SAVES IDEATION COMPLETE** com: salvos processados,
ideias geradas, ideias gravadas no Notion, os titulos gravados e os proximos comandos
sugeridos.

## Acao 3 - status (conferir a sincronizacao)

Leia o `state.json` e as ultimas 20 linhas do `sync.log`. Informe a data do ultimo
sync, o total ja sincronizado e os erros recentes.

**Se a ultima linha do sync.log nao for um "Sync completo", a rodada morreu no meio.**
Olhe o `launchd-stderr.log` (Mac) e diga em linguagem simples o que aconteceu.

## Acao 4 - scheduler (conferir o agendador)

No Mac: `launchctl list | grep saves-engine`.
No Windows: `schtasks /query /tn "SavesEngine-Manha"`.
Se nao voltar nada, o agendador nao esta carregado: explique como recarregar.

## Acao 5 - refresh session (renovar os cookies)

Conduza passo a passo:
1. Abrir o navegador logado no instagram.com
2. Abrir o DevTools (Cmd+Option+I no Mac, F12 no Windows)
3. Aba **Application** > menu lateral **Cookies** > `https://www.instagram.com`
4. Copiar os valores de `sessionid`, `csrftoken` e `ds_user_id`
5. Atualizar o `config.json` com os tres valores
6. Rodar `{{PYTHON}} sync.py` para confirmar que voltou a funcionar

Nunca escreva o valor do sessionid de volta na conversa nem em nenhum log.

## Acao 6 - recent (ver os salvos recentes)

Consulte a base Instagram Saves ordenada por `Ordem` crescente. Mostre os 15 primeiros.
````

---

## Por que o reframe é a regra mais importante

O passo 2 tem uma instrução que parece detalhe e não é: **nunca copie o post, traduza
o conceito**.

Sem ela, o sistema vira uma máquina de replicar conteúdo dos outros, e isso é pior que
não ter sistema nenhum. Com ela, o salvo é matéria-prima: o que a pessoa aproveita é o
conceito, aplicado à realidade do público dela.

Um exemplo real: um reel salvo falava de um aplicativo open source que mostra o limite
de uso das LLMs. O conteúdo não vira "olha esse app". Vira "a ferramenta que você usa
todo dia tem um limite que você não enxerga, e isso vale pra mais coisa que você
imagina" — o conceito, traduzido.

## Por que a prioridade é do cérebro, não do robô

O sync não tenta adivinhar prioridade nem pilar. Ele grava tudo como `New` e pronto.

Isso é de propósito. Classificar exige entender o conteúdo, e entender exige a
transcrição, que chega depois. Um robô que tenta classificar na captura erra, e erro
de classificação é pior que classificação nenhuma: a pessoa passa a não confiar na
base.
