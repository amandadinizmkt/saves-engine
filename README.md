# Saves Engine

**Uma skill do Claude Code que transforma os seus posts salvos do Instagram em um banco de ideias de conteúdo que se alimenta sozinho.**

Você salva post no Instagram achando que vai voltar depois. Nunca volta.
O conteúdo que você mais consome é justamente o que menos vira conteúdo seu.

Esta skill constrói, junto com você, o sistema que resolve isso: ele puxa os seus
salvos pro Notion duas vezes por dia, transcreve o que as pessoas falam nos reels,
e te devolve ideia de conteúdo pronta, adaptada pro seu público.

Roda no seu computador. Sem mensalidade, sem Zapier, sem n8n.

> Material da aula **Saves Engine**, do Clube Divos da IA.

---

## Por que transcrever

Num reel, a legenda quase nunca diz do que o vídeo trata. É "comenta EU QUERO" ou
"link na bio". A ideia de verdade está na fala, e a fala não é pesquisável, não é
filtrável, não existe em lugar nenhum além do vídeo.

Um exemplo real:

| A legenda dizia | A transcrição capturou |
|---|---|
| "Comenta EU QUERO aqui que eu mando o link." | "...esse aplicativo que mostra os status e os limites de uso de cada uma das LLMs. Ele é open source, tem 1.700 estrelas no GitHub..." |

Sem transcrição, esse salvo seria uma linha vazia. Com ela, virou matéria-prima.

---

## Como instalar

### 1. Clone este repositório

```bash
git clone https://github.com/amandadinizmkt/saves-engine.git
cd saves-engine
```

### 2. Copie a skill para o Claude Code

**Mac / Linux:**
```bash
cp -r saves-engine ~/.claude/skills/
```

**Windows (Prompt de Comando):**
```
xcopy /E /I saves-engine %USERPROFILE%\.claude\skills\saves-engine
```

### 3. Confira que deu certo

```bash
ls ~/.claude/skills/saves-engine
```

Você deve ver `SKILL.md` e a pasta `referencias`.

### 4. Rode

Abra o Claude Code em qualquer pasta e digite:

```
/saves-engine
```

A skill conduz o resto.

---

## Antes de rodar, você precisa de

| Item | Observação |
|---|---|
| **Claude Code** | É quem escreve o código e faz a ideação |
| **Conta no Notion** | A gratuita resolve |
| **Duas bases no Notion** | As colunas estão em [`estrutura-das-bases.md`](estrutura-das-bases.md) |
| **Uma integração do Notion** | Criada em dois minutos, gera um token |
| **Instagram com posts salvos** | Pode ser uma conta secundária |
| **Python 3** | Já vem no Mac. No Windows, baixe em python.org e marque "Add Python to PATH" |

A parte manual leva uns dez minutos. O resto o Claude Code escreve.
O conjunto todo leva uma tarde, e tem pedra no caminho. Este repositório não vai
te dizer que leva cinco minutos, porque não leva.

---

## Como usar no dia a dia

Depois de montado, o sistema captura sozinho nos horários que você escolheu. Quando
quiser produzir conteúdo, abra o Claude Code **na pasta do projeto** e rode:

```
/saves-engine ideate
```

Ele lê os salvos novos, entende do que cada um trata (pela transcrição, não pela
legenda) e devolve, pra cada um: um ângulo adaptado pro seu público, três opções de
gancho, um roteiro estruturado e o desdobramento por plataforma. Você aprova o que
faz sentido, e só o aprovado vai pro calendário.

### Todos os comandos

| Comando | O que faz |
|---|---|
| `/saves-engine` | Constrói o sistema (a entrevista de 8 perguntas) |
| `/saves-engine ideate` | Transforma os salvos novos em ideias de conteúdo |
| `/saves-engine sync` | Roda a captura agora, sem esperar o horário |
| `/saves-engine transcribe` | Transcreve os vídeos pendentes |
| `/saves-engine status` | Mostra se está funcionando e quando foi o último sync |
| `/saves-engine scheduler` | Confere se o agendador está carregado |
| `/saves-engine refresh session` | Renova os cookies quando a sessão expira |
| `/saves-engine recent` | Mostra os 15 salvos mais recentes |

### O que você vai ter na pasta

```
saves-engine/
├── sync.py              o robô que busca os salvos
├── transcribe.py        o robô que escuta os reels
├── config.json          os seus acessos (nunca compartilhe)
├── .gitignore           protege o config.json
├── LEIA-ME.md           o que fazer quando parar de funcionar
└── .claude/commands/
    └── saves-engine.md  o cérebro que transforma salvo em ideia
```

Mais o agendador instalado, rodando duas vezes por dia sozinho.

---

## O que tem dentro da skill

```
saves-engine/
├── SKILL.md                          o cérebro: entrevista, regras e a ordem de construção
└── referencias/
    ├── bases-do-notion.md            as colunas das duas bases e as 4 visões
    ├── sync-py.md                    o robô que busca, com as 4 proteções
    ├── transcricao.md                o robô que escuta (Mac, Windows e API)
    ├── agendador.md                  launchd e Agendador de Tarefas
    ├── comando-de-ideacao.md         o cérebro que vira salvo em ideia
    └── problemas-conhecidos.md       as pedras do caminho e como sair delas
```

### `SKILL.md` — a entrevista e as regras

É o arquivo que o Claude Code lê. Ele faz **8 perguntas, uma de cada vez**, e a
partir das respostas escreve o sistema inteiro:

| # | Pergunta | Pra que serve |
|---|---|---|
| 1 | Mac ou Windows? | Define a transcrição e o agendador |
| 2 | Pra quem você cria conteúdo? | Entra no comando de ideação |
| 3 | Quais são os seus pilares? | Viram o campo Pillar no Notion |
| 4 | Qual pilar fala de ferramenta e IA? | Separa os assuntos |
| 5 | Token do Notion e os dois ids | Vão pro arquivo de configuração |
| 6 | Transcrever os antigos ou só os novos? | Padrão: só os novos |
| 7 | Que horas rodar? | Padrão: 9h e 21h |
| 8 | No Windows: grátis e lento, ou pago e rápido? | Define a transcrição |

O SKILL.md também carrega as **regras de ouro** que a skill segue sempre. A mais
importante: o `sessionid` do Instagram nunca é escrito de volta na conversa, num log
ou num arquivo versionado, e o `.gitignore` é criado junto com o arquivo de
configuração.

### `referencias/bases-do-notion.md`

A estrutura das duas bases, com os tipos de cada coluna e os detalhes que quebram o
sistema se forem ignorados:

- `Transcricao` e `Duracao` vão **sem acento e sem cedilha** (o código escreve nesses nomes exatos)
- `Media ID` precisa ser **Text, nunca Number** (o Notion arredonda número grande e a deduplicação morre)
- Os valores dos Select têm que existir **antes**, criados na mão

Explica também **por que são duas bases e não uma**: o depósito pode ser bagunçado,
o calendário não pode.

### `referencias/sync-py.md` — o robô que busca

O código completo e comentado, e as **quatro proteções** que não existem nos tutoriais
que ensinam esse sistema. Cada uma nasceu de um problema real, em produção:

| Proteção | Por que existe |
|---|---|
| **Retry de rede** | O agendador acorda o programa quando o computador pode estar saindo do sono com o wi-fi ainda subindo |
| **Teto no reordenamento** | A etapa que reorganiza a lista percorria a base inteira *antes* de gravar os salvos novos. Falha ali derrubava a rodada sem gravar nada |
| **Lock de processo** | Duas cópias rodando juntas duplicam páginas |
| **Estado a cada 25** | A mais importante. Uma interrupção no meio de 800 posts deixava 600 linhas escritas e zero registradas, e a rodada seguinte recriava todas |

A lição que vale pra qualquer sistema: **uma etapa cosmética nunca pode bloquear a
etapa essencial. Se o bonito falhar, o importante continua.**

### `referencias/transcricao.md` — o robô que escuta

Os três caminhos, com custo e velocidade de cada um:

| Caminho | Sistema | Custo | Velocidade | O áudio sai da máquina? |
|---|---|---|---|---|
| `mlx-whisper` | Mac com chip M | R$ 0 | 7 a 8 segundos por reel | Não |
| `faster-whisper` | Windows e Mac Intel | R$ 0 | 15 a 40 segundos | Não |
| API da OpenAI | Qualquer um | US$ 0,006 por minuto | Rápido | **Sim** |

Traz também a decisão de projeto que vale mais que o código: **não transcreva o seu
acervo antigo**. Sistema bom não é o que processa tudo, é o que processa o que você
vai usar.

### `referencias/agendador.md`

Como deixar rodando sozinho, nos dois sistemas: `launchd` no Mac (com o `.plist`
pronto) e Agendador de Tarefas no Windows (com os comandos `schtasks`). Inclui os
detalhes que pegam as pessoas: caminho absoluto no Mac, espaço no caminho no Windows,
e o que acontece quando o computador está dormindo ou desligado na hora.

**Por que 9h e 21h, e não de hora em hora:** o motivo não é técnico, é de discrição.

### `referencias/comando-de-ideacao.md` — o cérebro

O template do comando que transforma salvo em ideia, com lacunas pro seu público e os
seus pilares. Pra cada salvo, ele devolve:

- Um ângulo adaptado ao seu público
- Três opções de gancho (curiosidade, valor, emocional)
- Um roteiro estruturado: hook, 3 a 4 pontos, CTA
- O desdobramento por plataforma (Instagram, TikTok, YouTube)

A regra mais importante do comando: **nunca copiar o post, traduzir o conceito.** Sem
ela, o sistema vira uma máquina de replicar conteúdo dos outros, e isso é pior do que
não ter sistema nenhum.

### `referencias/problemas-conhecidos.md`

O que fazer quando quebrar. Os sintomas reais, com a causa e a saída de cada um:
sessão expirada, "could not find database", transcrição falhando, páginas duplicadas,
agendador que não dispara.

---

## A arquitetura, em uma frase

**O robô que busca é burro. O cérebro é esperto. De propósito.**

Buscar tem que ser rápido e rodar sem você. Pensar tem que acontecer quando você
estiver pronta pra pensar.

```
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │   sync.py    │ ──▶ │ transcribe.py│ ──▶ │   ideação    │
  │  busca, 2x   │     │   escuta e   │     │ você aprova  │
  │   por dia    │     │  transcreve  │     │  o que quer  │
  └──────────────┘     └──────────────┘     └──────────────┘
         │                     │                    │
         ▼                     ▼                    ▼
    ┌─────────────────────────────┐        ┌────────────────┐
    │  Notion · Instagram Saves   │        │ Content Ideas  │
    │  o depósito, tudo cai aqui  │        │  o calendário  │
    └─────────────────────────────┘        └────────────────┘
```

Isso não é sobre Instagram. É o padrão de qualquer sistema desse tipo: e-mail,
YouTube, mensagens de cliente. **A fonte muda. A arquitetura é essa.**

---

## Um aviso honesto sobre segurança

São três partes, e você merece saber as três antes de montar.

**O acesso à sua conta.** O Instagram não tem integração oficial para posts salvos.
O sistema usa o mesmo crachá (cookie de sessão) que o seu navegador já usa quando
você está logada. Esse crachá fica **só na sua máquina** e **vale tanto quanto a sua
senha**. Nunca mande pra ninguém, não cole em grupo, não deixe aparecer num print.

**O risco de chateação.** Isso vai contra os termos de uso do Instagram. Na prática,
o que costuma acontecer com um uso pessoal e de baixo volume é a sessão expirar e
você precisar logar de novo. Rodar duas vezes por dia, e não de hora em hora, é o que
mantém isso discreto.

**O Notion.** Essa parte é oficial e documentada. A integração enxerga só as duas
bases que você autorizar, e você revoga quando quiser.

Se tiver receio, teste numa conta secundária primeiro. Muda um número no arquivo de
configuração e nada mais.

---

## Quando parar de funcionar

Vai parar em algum momento: a sessão do Instagram expira sozinha. Não é bloqueio,
não é defeito seu. O sinal é uma linha assim no `sync.log`:

```
Sessao do Instagram invalida (HTTP 401)
```

A solução é pegar os cookies de novo:

```
/saves-engine refresh session
```

Se for outra coisa, rode este prompt dentro da pasta do projeto:

> O meu Saves Engine parou. Leia o sync.log e o state.json da pasta, me diga em
> linguagem simples o que aconteceu e o que eu preciso fazer.

Os sintomas mais comuns, com a causa e a saída de cada um, estão em
[`saves-engine/referencias/problemas-conhecidos.md`](saves-engine/referencias/problemas-conhecidos.md).

### Se você é aluna do Clube

Traga a dúvida no **plantão ao vivo**, com as últimas 20 linhas do `sync.log`:

```bash
tail -20 sync.log
```

**Não leve o `config.json`**: ele tem os seus acessos. O `sync.log` não tem segredo
nenhum e é o que mostra o que aconteceu.

---

## Crédito

Este sistema é adaptado do tutorial **"The Instagram Saves Engine"**, de Justyn Berk.

Quatro coisas dele não funcionaram como descrito, e essas quatro correções estão
dentro desta skill: a biblioteca do Notion mudou, o Instagram descontinuou o endereço
das coleções, o controle de duplicatas só era salvo no fim, e o reordenamento derrubava
a rodada inteira.

**Isso não é detalhe técnico. É a diferença entre assistir a uma aula e conseguir
fazer funcionar.**

---

## Licença

MIT. Use, adapte e ensine à vontade.

---

<sub>Clube Divos da IA · Amanda Diniz · 2026</sub>
