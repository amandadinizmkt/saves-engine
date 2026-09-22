# As duas bases do Notion — estrutura para copiar

Crie as duas bases antes de rodar a skill. Os nomes das colunas precisam ser
exatamente estes: o programa escreve por nome de coluna.

---

## Base 1 — Instagram Saves

E o deposito bruto. Tudo que voce salva cai aqui, sem filtro.

| Coluna | Tipo no Notion | Valores (quando for Select) |
|---|---|---|
| `Ordem` | Number | |
| `Name` | Title | |
| `Author` | Text | |
| `URL` | URL | |
| `Type` | Select | `Reel`, `Carousel`, `Post`, `IGTV` |
| `Status` | Select | `New`, `Reviewed`, `Used` |
| `Caption` | Text | |
| `Transcricao` | Text | |
| `Transcricao Status` | Select | `Pendente`, `Transcrito`, `Sem audio`, `Falhou`, `Nao transcrever` |
| `Duracao` | Text | |
| `Media ID` | Text | |
| `Saved` | Date | |

### Tres detalhes que quebram o sistema se forem ignorados

**1. `Transcricao` e `Duracao` vao sem acento e sem cedilha.**
O programa escreve nesses nomes exatos. `Transcrição` com c-cedilha e uma coluna
diferente pro Notion, e a escrita falha com erro de propriedade inexistente.

**2. `Media ID` precisa ser Text, nunca Number.**
Os identificadores do Instagram passam de 17 digitos, e o Notion arredonda numero
grande. Arredondou, dois posts diferentes viram o mesmo numero, e ai tudo duplica.

**3. Crie os valores dos Select na mao, antes de rodar.**
O Notion cria valor novo sozinho quando o programa escreve, mas filtrar por um valor
que nunca existiu devolve lista vazia **sem dar erro nenhum**, o que e pior do que
quebrar: voce acha que nao tem nada pendente quando tem.

---

## Base 2 — Content Ideas

E o calendario limpo. So entra o que voce aprovou.

| Coluna | Tipo no Notion | Valores (quando for Select) |
|---|---|---|
| `Name` | Title | |
| `Platform` | Multi-select | `Instagram`, `TikTok`, `YouTube` |
| `Format` | Select | `Carousel`, `Reel`, `Short Video`, `Long-form Video` |
| `Status` | Select | `Not started`, `Em producao`, `Pronto`, `Publicado` |
| `Angle` | Text | |
| `Hook Options` | Text | |
| `Priority` | Select | `High`, `Medium`, `Low` |
| `Week Of` | Date | |
| `Pillar` | Select | **os seus pilares** |

O `Pillar` e o unico campo que muda de pessoa pra pessoa. Coloque os seus pilares
de conteudo, nao os da Amanda. A skill pergunta quais sao na entrevista.

---

## Por que duas bases, e nao uma

O deposito pode ser bagunçado. O calendario nao pode.

Se fosse tudo na mesma base, o seu calendario de conteudo teria centenas de linhas
de pesquisa misturadas com as cinco coisas que voce vai gravar essa semana.

---

## As 4 visoes que deixam isso utilizavel

Uma base sem visao e uma lista de 800 linhas que ninguem abre. Crie estas quatro:

**1. Fila de ideacao** (na base Instagram Saves)
Filtro: `Status` = `New`. Ordenar por `Ordem`, crescente.
E o que ainda nao virou conteudo. E a visao que voce abre quando senta pra produzir.

**2. Fila de transcricao** (na base Instagram Saves)
Filtro: `Transcricao Status` = `Pendente`.
Os videos que ainda vao ser transcritos.

**3. Escolher p/ transcrever** (na base Instagram Saves)
Filtro: `Transcricao Status` = `Nao transcrever`. Ordenar por `Ordem`, crescente.
O acervo antigo. Quando voce quiser pescar um salvo velho, muda o status aqui pra
`Pendente` e ele entra na fila.

**4. Producao** (na base Content Ideas)
Visao em quadro (kanban), agrupada por `Status`.
E o calendario de verdade.

---

## Onde achar o id de cada base

Abra a base no navegador, em pagina cheia (icone de expandir, se ela estiver dentro
de outra pagina). A URL fica assim:

```
https://www.notion.so/seuworkspace/1f2a3b4c5d6e7f8091a2b3c4d5e6f708?v=abc123
                                   └───────── o id, 32 caracteres ─────────┘
```

O id e o bloco de 32 caracteres antes do `?v=`.

Se voce nao abrir em pagina cheia, a URL mostra o id da pagina que contem a base,
nao o da base. Ai o programa reclama que nao encontrou o banco de dados.

---

## Depois de criar as duas

1. Va em `notion.so/my-integrations`
2. Clique em "New integration", de um nome e crie
3. Copie o token (comeca com `ntn_`)
4. Abra a base Instagram Saves, menu `...` no canto superior direito,
   `Conexoes`, e escolha a sua integracao
5. **Repita o passo 4 na base Content Ideas**

O passo 5 e o que todo mundo esquece.
