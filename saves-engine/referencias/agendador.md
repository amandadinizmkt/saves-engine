# O agendador: rodar sozinho, duas vezes por dia

Sem agendador, o sistema é um script que a pessoa lembra de rodar. Com agendador, é
um sistema. É a diferença entre a coisa funcionar na semana da aula e funcionar em
março do ano que vem.

**Horário padrão: 9h e 21h.** Duas vezes por dia, não de hora em hora. O motivo não é
técnico, é de discrição: um volume baixo e em horários humanos é o que mantém a conta
do Instagram fora de qualquer radar. Ver `problemas-conhecidos.md`.

---

## Mac — launchd

O launchd é o agendador nativo do macOS. Ele lê um arquivo `.plist` e dispara o
programa nos horários pedidos.

### O arquivo

Gere em `~/Library/LaunchAgents/com.{{usuario}}.saves-engine.plist`, trocando
`{{usuario}}` por algo sem espaço e sem acento (o nome dela, minúsculo) e `{{pasta}}`
pelo caminho absoluto da pasta do projeto.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.{{usuario}}.saves-engine</string>

    <key>ProgramArguments</key>
    <array>
        <string>{{pasta}}/.venv/bin/python3</string>
        <string>{{pasta}}/sync.py</string>
    </array>

    <key>WorkingDirectory</key>
    <string>{{pasta}}</string>

    <key>StartCalendarInterval</key>
    <array>
        <dict>
            <key>Hour</key>
            <integer>9</integer>
            <key>Minute</key>
            <integer>0</integer>
        </dict>
        <dict>
            <key>Hour</key>
            <integer>21</integer>
            <key>Minute</key>
            <integer>0</integer>
        </dict>
    </array>

    <key>StandardOutPath</key>
    <string>{{pasta}}/launchd-stdout.log</string>

    <key>StandardErrorPath</key>
    <string>{{pasta}}/launchd-stderr.log</string>

    <key>RunAtLoad</key>
    <false/>
</dict>
</plist>
```

### Carregar

```bash
launchctl load ~/Library/LaunchAgents/com.{{usuario}}.saves-engine.plist
```

### Conferir que carregou

```bash
launchctl list | grep saves-engine
```

Se aparecer uma linha, está carregado. A primeira coluna é o PID (ou `-` se não estiver
rodando agora), a segunda é o código de saída da última execução. **Segunda coluna
diferente de 0 significa que a última rodada deu erro**: olhe o `launchd-stderr.log`.

### Descarregar (para parar ou editar)

```bash
launchctl unload ~/Library/LaunchAgents/com.{{usuario}}.saves-engine.plist
```

Sempre descarregue antes de editar o `.plist`, e carregue de novo depois. Editar com
ele carregado não tem efeito.

### Detalhes do launchd que pegam as pessoas

**Caminhos absolutos, sempre.** O launchd não roda com o mesmo ambiente do terminal.
`python3` sozinho não funciona: tem que ser o caminho completo do `.venv`.

**Se o Mac estiver dormindo no horário**, o launchd dispara assim que ele acordar. Não
perde a rodada, mas pode rodar às 10h30 em vez das 9h. É por isso que o `sync.py` tem
retry de rede: nesse momento o wi-fi ainda está subindo.

**Se o Mac estiver desligado**, a rodada é perdida e a próxima pega tudo junto. Não é
problema: o sync compara com o `state.json` e traz só o que falta.

**Permissão de disco.** Na primeira execução o macOS pode pedir autorização para o
programa acessar a pasta. Se o `launchd-stderr.log` mostrar erro de permissão, vá em
Ajustes do Sistema > Privacidade e Segurança > Acesso total ao disco e adicione o
`python3` do `.venv`.

---

## Windows — Agendador de Tarefas

O equivalente do launchd. Dá para criar pela interface, mas o comando é mais rápido e
mais fácil de conferir.

### Criar as duas tarefas

Abra o **Prompt de Comando como administrador** e rode os dois comandos, trocando
`{{pasta}}` pelo caminho da pasta do projeto (ex.: `C:\Users\Ana\saves-engine`):

```
schtasks /create /tn "SavesEngine-Manha" /tr "{{pasta}}\.venv\Scripts\python.exe {{pasta}}\sync.py" /sc daily /st 09:00 /f

schtasks /create /tn "SavesEngine-Noite" /tr "{{pasta}}\.venv\Scripts\python.exe {{pasta}}\sync.py" /sc daily /st 21:00 /f
```

São duas tarefas porque o `schtasks` cria um horário por tarefa. O `/f` sobrescreve se
já existir, então dá para rodar de novo sem erro.

### Conferir

```
schtasks /query /tn "SavesEngine-Manha"
```

Mostra a próxima execução e o resultado da última.

### Rodar agora, para testar sem esperar

```
schtasks /run /tn "SavesEngine-Manha"
```

Isso dispara na hora. É o jeito de confirmar que funcionou sem esperar até as 9h.

### Apagar

```
schtasks /delete /tn "SavesEngine-Manha" /f
schtasks /delete /tn "SavesEngine-Noite" /f
```

### Detalhes do Windows que pegam as pessoas

**O caminho do Python é diferente do Mac.** No Windows a venv guarda em
`.venv\Scripts\python.exe`, não em `.venv/bin/python3`.

**Caminho com espaço quebra o comando.** Se a pasta for
`C:\Users\Ana Paula\saves-engine`, o `/tr` precisa de aspas internas:

```
/tr "'C:\Users\Ana Paula\saves-engine\.venv\Scripts\python.exe' 'C:\Users\Ana Paula\saves-engine\sync.py'"
```

O jeito mais simples de evitar isso é criar a pasta num caminho sem espaço, como
`C:\saves-engine`.

**Janela preta piscando.** Por padrão o Windows abre uma janela do prompt quando a
tarefa roda. Para rodar em silêncio, use `pythonw.exe` no lugar de `python.exe`. A
contrapartida é que erros deixam de aparecer na tela: confira pelo `sync.log`.

**Computador desligado no horário.** Igual ao Mac: a rodada é perdida e a próxima traz
tudo. Se quiser que o Windows rode ao ligar caso tenha perdido, marque "Executar a
tarefa o mais rápido possível após uma inicialização agendada perdida" nas
propriedades da tarefa (só pela interface, o `schtasks` não expõe isso).

---

## Como saber que está funcionando (os dois sistemas)

O melhor sinal não é o agendador: é o log do próprio sync.

```bash
tail -20 sync.log
```

Uma rodada saudável termina com uma linha assim:

```
2026-09-22 09:01:50  Sync completo: 5 novos | 824 ja existiam | 829 total | 0 erros
```

**Se a última linha do log não for um "Sync completo", a rodada morreu no meio.** O
motivo costuma estar no `launchd-stderr.log` (Mac). Esse é o primeiro lugar pra olhar,
e é o que o prompt de diagnóstico da aula faz.
