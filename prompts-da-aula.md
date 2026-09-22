# Prompts da aula — Saves Engine

Todos os comandos e prompts que apareceram na tela, na ordem em que apareceram.

---

## 1. Instalar a skill

**Mac:**
```bash
mkdir -p ~/.claude/skills && cp -r saves-engine ~/.claude/skills/
```

**Windows:**
```
xcopy /E /I saves-engine %USERPROFILE%\.claude\skills\saves-engine
```

Conferir:
```bash
ls ~/.claude/skills/saves-engine
```

---

## 2. Construir o sistema

```
/saves-engine
```

A skill conduz as 8 perguntas e escreve o sistema inteiro. Voce nao precisa
digitar codigo nenhum.

---

## 3. Rodar a captura na mao

**Mac:**
```bash
.venv/bin/python3 sync.py
```

**Windows:**
```
.venv\Scripts\python.exe sync.py
```

Uma rodada saudavel termina assim:

```
Sync completo: 5 novos | 824 ja existiam | 829 total | 0 erros
```

---

## 4. Transcrever os videos

```bash
.venv/bin/python3 transcribe.py --limit 3   # so tres, pra testar
.venv/bin/python3 transcribe.py             # ate dez
.venv/bin/python3 transcribe.py --all       # todos os pendentes
```

---

## 5. Transformar salvos em ideias

```
/saves-engine ideate
```

Le os salvos novos, entende do que cada um trata pela transcricao, e devolve
angulo, tres ganchos, roteiro e desdobramento por plataforma.

---

## 6. Os outros comandos

```
/saves-engine status            mostra se esta funcionando
/saves-engine scheduler         confere o agendador
/saves-engine refresh session   renova os cookies quando a sessao expira
/saves-engine recent            mostra os 15 salvos mais recentes
```

---

## 7. Prompt de diagnostico

Guarde este. E o que voce usa quando o sistema parar e voce nao souber o porque.
Rode dentro da pasta do projeto:

> O meu Saves Engine parou. Leia o sync.log e o state.json da pasta, me diga em
> linguagem simples o que aconteceu e o que eu preciso fazer.

---

## 8. Prompt de construcao sem a skill (opcional)

Se voce quiser entender o sistema por dentro, ou construir uma versao sua pra
outra fonte que nao seja o Instagram, este e o prompt. Ele descreve o sistema
inteiro pro Claude Code.

> Quero construir um sistema que captura conteudo de uma fonte externa e
> transforma em ideias de conteudo no Notion. Ele tem tres pecas separadas:
>
> **1. Um script de captura** que roda sozinho pelo agendador do sistema, busca
> os itens novos na fonte, compara com um arquivo de estado local pra nao
> duplicar, e grava cada item novo numa base do Notion com status "New".
>
> Esse script precisa de quatro protecoes, que nao sao opcionais:
> - retry de rede com espera crescente, porque o agendador roda quando a maquina
>   pode estar acordando com o wi-fi ainda subindo
> - qualquer etapa cosmetica (ordenacao, enriquecimento) com teto de itens e com
>   tratamento de erro que avisa e segue, nunca derrubando a captura
> - lock de processo, pra duas copias nao rodarem juntas e duplicarem
> - gravacao do estado a cada 25 itens, e so depois que o Notion confirmar a
>   escrita de cada um
>
> **2. Um script de transcricao** separado, que busca no Notion os itens com
> status de transcricao "Pendente", baixa o audio, transcreve com um modelo
> local e grava o texto de volta. Separado de proposito: a captura precisa ser
> rapida, a transcricao e lenta, e se a transcricao travar a captura continua.
>
> **3. Um comando do Claude Code** que le os itens novos, entende do que cada um
> trata pela transcricao (nao pelo titulo ou legenda, que costumam mentir), e
> gera pra cada um: um angulo adaptado ao meu publico, tres opcoes de gancho, um
> roteiro estruturado e o desdobramento por plataforma. Nunca copiar o original:
> traduzir o conceito pra realidade do meu publico.
>
> Meu publico e [DESCREVA EM UMA FRASE].
> Meus pilares de conteudo sao [LISTE].
>
> Use duas bases no Notion, nao uma: um deposito bruto onde tudo cai sem filtro,
> e um calendario limpo onde so entra o que eu aprovei.
>
> Comece me perguntando qual e a fonte e em qual sistema operacional eu estou.

---

## 9. Prompt pra adaptar o sistema a outra fonte

Depois que o seu Saves Engine estiver rodando, este prompt adapta a mesma ideia
pra outra fonte:

> Meu Saves Engine esta funcionando pros salvos do Instagram. Quero fazer o mesmo
> com [YouTube / e-mails marcados / artigos salvos / mensagens de um grupo].
>
> Leia o sync.py e o comando de ideacao que ja existem na pasta e me diga: o que
> da pra reaproveitar igual, o que muda na captura, e o que eu preciso decidir
> antes de a gente comecar. Nao escreva codigo ainda.
