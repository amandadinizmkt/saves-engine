# Criar as duas bases automaticamente

Quando o conector do Notion está ligado, você cria as duas bases direto, com as colunas
e os valores de select já prontos. É o caminho padrão: leva segundos e não tem erro de
digitação.

Se o conector não estiver disponível, veja `bases-do-notion.md` para o caminho manual.

---

## Antes de criar

**Pergunte onde criar.** As bases precisam morar em algum lugar do workspace dela.
Ofereça duas opções:

1. **Numa página nova** chamada "Saves Engine" (recomendado: fica tudo junto)
2. **Dentro de uma página que ela já tem** — peça o link ou o nome

Se ela não souber, crie a página nova. Não trave a construção nisso.

---

## Base 1 — Instagram Saves

Use a ferramenta de criar base do conector, com este schema:

```sql
CREATE TABLE (
  "Name" TITLE,
  "Ordem" NUMBER,
  "Author" RICH_TEXT,
  "URL" URL,
  "Type" SELECT('Reel':blue, 'Carousel':purple, 'Post':gray, 'IGTV':pink),
  "Status" SELECT('New':red, 'Reviewed':yellow, 'Used':green),
  "Caption" RICH_TEXT,
  "Transcricao" RICH_TEXT,
  "Transcricao Status" SELECT('Pendente':yellow, 'Transcrito':green, 'Sem audio':gray, 'Falhou':red, 'Nao transcrever':brown),
  "Duracao" RICH_TEXT,
  "Media ID" RICH_TEXT,
  "Saved" DATE
)
```

Título: **Instagram Saves**

> **Testado de verdade em 22/09/2026.** Este schema foi executado no conector e as 12
> colunas saíram com os tipos e os valores de select corretos.
>
> Uma curiosidade que aparece no resultado e **não é erro**: a coluna `URL` aparece
> como `userDefined:URL` na definição SQL que o conector devolve. Isso é só um apelido
> interno (o Notion reserva os nomes "url" e "id"). O nome real da propriedade continua
> sendo `URL`, que é o que o `sync.py` usa para escrever. Não renomeie a coluna.

### Três coisas que não podem mudar neste schema

1. **`Transcricao` e `Duracao` sem acento e sem cedilha.** O `sync.py` e o
   `transcribe.py` escrevem nesses nomes exatos. Com cedilha, é outra coluna pro Notion
   e a escrita falha com erro de propriedade inexistente.
2. **`Media ID` é RICH_TEXT, nunca NUMBER.** Os ids do Instagram passam de 17 dígitos e
   o Notion arredonda número grande. Arredondou, dois posts viram o mesmo id e a
   deduplicação morre: tudo duplica.
3. **Os cinco valores de `Transcricao Status` precisam existir desde o começo.**
   Filtrar por um valor que nunca existiu devolve lista vazia **sem dar erro**, o que é
   pior que quebrar: a pessoa acha que não tem nada pendente quando tem.

---

## Base 2 — Content Ideas

```sql
CREATE TABLE (
  "Name" TITLE,
  "Platform" MULTI_SELECT('Instagram':pink, 'TikTok':gray, 'YouTube':red),
  "Format" SELECT('Carousel':purple, 'Reel':blue, 'Short Video':orange, 'Long-form Video':brown),
  "Status" SELECT('Not started':gray, 'Em producao':yellow, 'Pronto':blue, 'Publicado':green),
  "Angle" RICH_TEXT,
  "Hook Options" RICH_TEXT,
  "Priority" SELECT('High':red, 'Medium':yellow, 'Low':gray),
  "Week Of" DATE,
  "Pillar" SELECT(<OS PILARES DELA>)
)
```

Título: **Content Ideas**

**O `Pillar` é o único campo que muda de pessoa pra pessoa.** Monte os valores com os
pilares que ela respondeu na pergunta 3, não com os da Amanda. Escolha cores diferentes
pra cada um.

Exemplo, se ela respondeu "Fé, Bastidores, Empreendedorismo, Mindset, IA na Prática":

```sql
"Pillar" SELECT('Fe':purple, 'Bastidores':orange, 'Empreendedorismo':blue, 'Mindset':green, 'IA na Pratica':pink)
```

Repare: **sem acento nos valores de select também**, pra evitar qualquer divergência
entre o que está no Notion e o que o comando de ideação escreve.

---

## Depois de criar

**1. Pegue os dois ids.** A ferramenta devolve o id de cada base ao criar. Guarde os
dois: eles vão pro `config.json`.

**2. Mostre pra ela o que foi criado.** Diga o nome das duas bases, onde elas ficaram,
e ofereça o link. É o momento em que o sistema começa a ficar real.

**3. Avise sobre a integração.** Esta é a parte que confunde, e você precisa explicar
a diferença:

> "As bases já estão criadas. Mas falta uma coisa: eu criei elas usando o **conector**,
> que é eu falando com o seu Notion agora, nesta conversa. O robô que vai rodar às 9h
> da manhã, quando você não estiver aqui, precisa de um acesso próprio: é a
> **integração**. São duas coisas diferentes, e você precisa das duas."

Conduza então: `notion.so/my-integrations` > New integration > copiar o token `ntn_` >
conectar **nas duas bases**.

**4. As quatro visões.** O conector cria a base, mas as visões (filtros salvos) valem
mais a pena criar na mão, porque ela vai querer mexer nelas depois. Liste as quatro de
`bases-do-notion.md` e deixe ela criar, ou crie e mostre.

---

## Se o conector não estiver ligado

Conduza ela a ligar, é rápido:

1. Abrir **claude.ai** no navegador
2. **Configurações** > **Conectores**
3. Procurar **Notion** > **Conectar**
4. Autorizar o workspace onde as bases vão morar
5. Voltar pro Claude Code e rodar `/saves-engine` de novo

**Se não der certo** (workspace do trabalho onde ela não é admin, plano sem conector,
erro de autorização): não insista. Vá para o caminho manual de `bases-do-notion.md` e
diga que o resultado é exatamente o mesmo, só leva uns dez minutos a mais.

**Nunca deixe a pessoa parada por causa do conector.** As bases são o pré-requisito de
tudo: sem elas, nada do resto funciona.
