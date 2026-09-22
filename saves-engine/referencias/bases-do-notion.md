# As duas bases do Notion

Este arquivo é a fonte da verdade sobre a estrutura. Se a pessoa criar as colunas
diferentes daqui, o `sync.py` quebra, porque ele escreve por nome de coluna.

## Por que duas bases, e não uma

O depósito pode ser bagunçado. O calendário não pode.

Se fosse tudo na mesma base, o calendário de conteúdo teria centenas de linhas de
pesquisa misturadas com as cinco coisas que a pessoa vai gravar essa semana. A
separação é o que mantém o calendário utilizável.

- **Base 1 (Instagram Saves)** é o depósito bruto. Tudo que a pessoa salva cai aqui,
  sem filtro, sem curadoria.
- **Base 2 (Content Ideas)** é o calendário limpo. Só entra o que ela aprovou.

---

## Base 1 — Instagram Saves

O nome pode ser outro, mas as **colunas têm que ter exatamente estes nomes**,
incluindo os acentos onde eles aparecem.

| Coluna | Tipo no Notion | Para que serve |
|---|---|---|
| `Ordem` | Number | Posição na lista de salvos do Instagram. 1 = o último que ela salvou, sempre no topo |
| `Name` | Title | `@autor/codigo-do-post` |
| `Author` | Text | O @ de quem publicou |
| `URL` | URL | Link direto pra abrir o post |
| `Type` | Select | `Reel`, `Carousel`, `Post`, `IGTV` |
| `Status` | Select | `New`, `Reviewed`, `Used` |
| `Caption` | Text | A legenda original, cortada em 1900 caracteres |
| `Transcricao` | Text | O que a pessoa fala no vídeo |
| `Transcricao Status` | Select | `Pendente`, `Transcrito`, `Sem audio`, `Falhou`, `Nao transcrever` |
| `Duracao` | Text | Tamanho do vídeo, formato `m:ss` |
| `Media ID` | Text | Identificador do Instagram. É o que impede duplicata |
| `Saved` | Date | Quando o sync capturou |

### Detalhes que quebram se forem ignorados

**`Transcricao` e `Duracao` sem acento e sem cedilha.** O código escreve nesses nomes
exatos. `Transcrição` com ç é outra coluna, e o Notion devolve erro de propriedade
inexistente.

**Os valores dos Select têm que existir antes.** O Notion cria valor novo de select
automaticamente na escrita via API, mas o filtro por um valor que nunca existiu
devolve lista vazia sem erro nenhum, o que é pior que quebrar. Crie os cinco valores
de `Transcricao Status` na mão, uma vez.

**`Media ID` é Text, não Number.** Os ids do Instagram passam de 17 dígitos e o Notion
arredonda number grande. Arredondou, a deduplicação para de funcionar e tudo duplica.

**Limite de 2000 caracteres por campo de texto.** O código corta em 1900 por segurança.
Transcrição de vídeo longo vem truncada, e isso é aceitável: o objetivo é saber do que
o vídeo trata, não ter a transcrição completa.

---

## Base 2 — Content Ideas

| Coluna | Tipo no Notion | Para que serve |
|---|---|---|
| `Name` | Title | Título da ideia |
| `Platform` | Multi-select | `Instagram`, `TikTok`, `YouTube` |
| `Format` | Select | `Carousel`, `Reel`, `Short Video`, `Long-form Video` |
| `Status` | Select | `Not started`, `Em producao`, `Pronto`, `Publicado` |
| `Angle` | Text | O ângulo do conteúdo, adaptado pro público dela |
| `Hook Options` | Text | Os 3 ganchos, unidos por ` \| ` |
| `Priority` | Select | `High`, `Medium`, `Low` |
| `Week Of` | Date | A segunda-feira da semana planejada |
| `Pillar` | Select | Os pilares editoriais **dela**, não os da Amanda |

O `Pillar` é o único campo que muda de pessoa pra pessoa. A skill pergunta os pilares
na entrevista e cria os valores de select com os nomes que ela deu.

---

## As 4 visões que deixam isso utilizável

Uma base sem visão é uma lista de 800 linhas que ninguém abre. São estas:

**1. Fila de ideação** (na base Instagram Saves)
Filtro: `Status` = `New`. Ordenar por `Ordem` crescente.
É o que ainda não virou conteúdo. É a visão que ela abre quando senta pra produzir.

**2. Fila de transcrição** (na base Instagram Saves)
Filtro: `Transcricao Status` = `Pendente`.
Os vídeos que ainda vão ser transcritos. Serve pra ela ver se a transcrição está em dia.

**3. Escolher p/ transcrever** (na base Instagram Saves)
Filtro: `Transcricao Status` = `Nao transcrever`. Ordenar por `Ordem` crescente.
O acervo antigo. Quando ela quiser pescar um salvo velho, muda o status aqui pra
`Pendente` e ele entra na fila.

**4. Produção** (na base Content Ideas)
Visão em quadro (kanban), agrupada por `Status`.
É o calendário de verdade.

---

## A integração do Notion

1. A pessoa cria em `notion.so/my-integrations`, botão "New integration".
2. Copia o token, que começa com `ntn_`.
3. **Conecta a integração nas DUAS bases**, uma de cada vez: abre a base, menu `...`
   no canto superior direito, `Conexões`, escolhe a integração.

**O erro mais comum do sistema inteiro é conectar só numa base.** O sync funciona,
a ideação quebra, e a mensagem de erro do Notion fala em "could not find database",
o que faz a pessoa procurar defeito no id em vez de na permissão.

## Onde achar os database ids

Abre a base no navegador. A URL fica assim:

```
https://www.notion.so/workspace/1f2a3b4c5d6e7f8091a2b3c4d5e6f708?v=...
                                └──────── é isso, 32 caracteres ────────┘
```

O id é o bloco de 32 caracteres antes do `?v=`. Se a base estiver dentro de uma página,
é preciso abrir a base em página cheia primeiro (ícone de expandir), senão a URL mostra
o id da página, não o da base.
