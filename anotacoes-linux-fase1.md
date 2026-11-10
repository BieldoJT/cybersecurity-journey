# Anotações de Linux: Fase 1 (Transição Dev → Cybersecurity)

Anotações das 8 missões de terminal Linux, feitas em um WSL. Para cada missão: o passo a passo, o que eu entendi certo, o que precisei corrigir e o motivo de isso importar em segurança.

---

## Missão 1: Se localizar

**Comandos:** `pwd`, `ls`, `ls -la`, `cd`

**Passo a passo**
1. Rodar `pwd` para ver onde estou.
2. Navegar com `cd` até a home, até `/` e de volta.
3. Explorar `/etc`, `/var/log` e `/tmp`.

**Caminho absoluto vs relativo**
- **Absoluto:** começa na raiz do sistema (`/`). Exemplo: `/home/gabriel/treino`.
- **Relativo:** começa na pasta onde estou (`pwd`). Exemplo: `treino/arquivo.txt`.
- O shell junta o `pwd` com o caminho relativo para chegar ao destino completo.

**Correções**
| O que eu tinha entendido | O correto |
| --- | --- |
| O absoluto começa na raiz "do repositório" | Começa na raiz **do sistema** (`/`). "Repositório" é termo do Git |
| `/etc` guarda os arquivos dos programas | Guarda a **configuração** do sistema e dos programas. Os executáveis ficam em `/usr/bin` e `/bin` |
| `/var/log` guarda tudo que foi feito | Guarda **registros de eventos** do sistema e dos serviços. Parte fica no `journalctl` |
| `/tmp` guarda cache e tokens | É **temporário** e costuma ser limpo ao reiniciar. O cache de verdade fica em `~/.cache` |

**Por que importa em segurança:** o `/tmp` é gravável por qualquer usuário, por isso é um lugar clássico onde atacantes largam arquivos. O `/etc` guarda arquivos sensíveis como `passwd`, `shadow` e `sudoers`.

---

## Missão 2: Criar e mexer em arquivos

**Comandos:** `mkdir`, `touch`, `echo`, `>`, `>>`, `cp`, `mv`, `rm`, `cat`, `head`, `tail`, `less`

**Passo a passo**
1. Criar a pasta `treino-linux` com `mkdir` e entrar nela.
2. Criar arquivos com `touch` e com `echo "texto" > arquivo.txt`.
3. Comparar `>` e `>>`.
4. Copiar (`cp`), renomear/mover (`mv`) e apagar (`rm`).
5. Ler arquivos com `cat`, `head -n 3`, `tail -n 3` e `less`.

**`>` vs `>>`**
- `>` **abre o arquivo e apaga o conteúdo** antes de escrever. Se o arquivo não existe, cria.
- `>>` escreve **no final**, sem apagar. Também cria se não existir.

**Correção:** eu tinha dito que o `>` "cria um arquivo novo sempre". Na verdade ele abre o **mesmo** arquivo e apaga o conteúdo.

**Por que importa em segurança:** um `>` no arquivo errado, como um log, destrói o conteúdo sem aviso. O `>>` é o que se usa para registrar sem perder o que já existe.

---

## Missão 3: Permissões

**Comandos:** `ls -l`, `chmod`, `chown`

**Passo a passo**
1. Ler a saída de `ls -l`, por exemplo `-rw-r--r-- 1 gabriel gabriel`.
2. Criar `oi.sh`, tentar rodar com `./oi.sh` e receber o erro de permissão negada.
3. Rodar `chmod +x oi.sh` e testar de novo.
4. Testar `chmod 600` e `chmod 755` e ver o efeito no `ls -l`.
5. Comparar `/etc/passwd` com `/etc/shadow`.

**Como ler `-rw-r--r-- 1 dono grupo`**
- **1º caractere:** tipo do item. `-` é arquivo, `d` é diretório, `l` é link simbólico.
- **3 conjuntos de 3:** permissões do dono, do grupo e dos outros.
- **Número:** quantidade de links.
- **Primeiro nome:** o dono. **Segundo nome:** o grupo.

**Permissões em número:** r=4, w=2, x=1, e os valores se somam.
- `r-x` = 5, `rw-` = 6, `-wx` = 3, `rwx` = 7
- `755` = dono `rwx`, grupo `r-x`, outros `r-x`
- `600` = dono `rw-`, grupo e outros sem nada

**Correções**
| O que eu tinha entendido | O correto |
| --- | --- |
| O primeiro traço diz se é "repositório" | Indica o **tipo** do item (arquivo, diretório, link) |
| Só identifiquei o dono | O segundo nome é o **grupo**, e é nele que o 2º conjunto de permissões se aplica |

**`passwd` vs `shadow`**
- O `/etc/passwd` precisa ser legível por todos, porque programas o consultam para traduzir UID em nome (o próprio `ls -l` faz isso).
- Antigamente os hashes das senhas ficavam no `passwd`, então foram movidos para o `/etc/shadow`, que só o root e o grupo `shadow` leem.
- Se um atacante lê o `shadow`, pode tentar quebrar os hashes **offline**. Permissão frouxa nele é uma falha séria.
- O root ignora essas permissões. O que se protege é o acesso de **usuários comuns**.

---

## Missão 4: `grep` e pipes

**Comandos:** `grep`, `grep -i`, `-n`, `-r`, `|`, `wc -l`, `cut`, `sort`

**Passo a passo**
1. `grep "root" /etc/passwd` e depois `grep -i "ROOT" /etc/passwd`.
2. `grep -n "bash" /etc/passwd`.
3. `grep -r "palavra" ~/treino-linux`.
4. Primeiro pipe: `cat /etc/passwd | grep bash`.
5. `ls /etc | wc -l`.
6. `cut -d: -f1 /etc/passwd | sort`.
7. Listar só os usuários que usam bash.

**O que aprendi**
- `-i` ignora diferença entre maiúscula e minúscula.
- `-n` mostra o número da linha.
- `-r` procura em uma pasta inteira (muito usado para caçar senhas e chaves esquecidas em código).
- O pipe `|` pega a saída de um comando e entrega como entrada do próximo, como um cano.

**Meu comando do passo 7**
```bash
grep "bash" /etc/passwd | cut -d: -f1 | sort
```

**Refinamento:** esse comando acha "bash" em **qualquer parte** da linha. O shell é o último campo, então o mais preciso é:
```bash
grep "bash$" /etc/passwd | cut -d: -f1 | sort
```
O `$` significa "fim da linha". Filtrar o campo certo, e não a linha toda, vai pesar muito na análise de logs no SOC.

---

## Missão 5: Usuários e processos

**Comandos:** `whoami`, `id`, `who`, `ps aux`, `top`, `kill`, `kill -9`, `sudo`

**Passo a passo**
1. `whoami`, `id` e `who`.
2. Analisar o `/etc/passwd` campo a campo.
3. `ps aux` e `ps aux | grep bash`.
4. `top` (sair com `q`).
5. `sleep 300` em um terminal, achar o PID com `ps aux | grep sleep` e encerrar com `kill PID`.
6. Comparar `kill` e `kill -9`.
7. `sudo whoami`.

**`whoami` vs `id`**
- `whoami` mostra só o usuário logado.
- `id` mostra UID, GID e **todos os grupos** a que pertenço. Isso importa porque as permissões de grupo dependem dessa lista.

**Os 7 campos do `/etc/passwd`**
| # | Campo |
| --- | --- |
| 1 | Nome do usuário |
| 2 | `x` (marcador: a senha está no `/etc/shadow`) |
| 3 | UID |
| 4 | GID (grupo principal) |
| 5 | Comentário (nome completo ou descrição) |
| 6 | Diretório home |
| 7 | Shell de login |

**Correção:** eu tinha dito que o campo 2 era a senha. Ele **não guarda a senha**, só um `x`.

**UID 0**
- O kernel não olha o nome, olha o **número**. Qualquer conta com UID 0 tem poder total, ignorando as permissões de arquivo.
- Verificar quem tem UID 0: `cut -d: -f1,3 /etc/passwd | grep ":0$"`. No meu sistema só apareceu `root:0`.
- Se aparecesse uma segunda conta com UID 0 (por exemplo `suporte:0`), seria um alerta sério. Atacantes criam contas assim como **backdoor**. Isso é item clássico de auditoria.

**`kill` vs `kill -9`**
- `kill PID` envia **SIGTERM**: pede para o processo encerrar, e ele pode fechar arquivos e liberar recursos. É o primeiro a tentar.
- `kill -9 PID` envia **SIGKILL**: o kernel encerra na hora, sem o processo poder reagir. Fica como último recurso, quando o processo trava ou ignora o SIGTERM.

**`sudo`**
- Executa **um único comando** com privilégios de root. O `sudo whoami` retornou `root`.
- Pede a **minha** senha, não a do root, e só funciona porque estou no grupo `sudo` (confirmável no `id`).
- Cada uso fica registrado em log.

---

## Missão 6: Rede básica

**Comandos:** `ip addr`, `hostname -I`, `ping`, `curl -I`, `nslookup`/`dig`, `ss -tuln`

**Passo a passo**
1. Descobrir o IP da máquina. No WSL o `ip a` não funcionou de início, então usei `ip addr` ou `hostname -I` (e `sudo apt install iproute2` se faltar).
2. `ping -c 4 google.com`.
3. `curl -I https://example.com`.
4. `nslookup example.com`.
5. `ss -tuln`.
6. Descrever o caminho de um acesso a `example.com`.

**O que aprendi**
- **`ping`:** o tempo é o **RTT** (ida e volta do pacote, em ms). A linha final mostra a perda de pacotes (`packet loss`).
- **`curl -I`:** retornou **200**. O `Content-Type` diz o tipo do conteúdo (html, json, imagem...).
- **Header `Server`:** informa qual **software o servidor usa** (nginx, Apache...). Em segurança entrega a tecnologia do alvo, por isso muita gente o esconde.
- **`LISTEN`:** a porta está **aberta esperando conexões**. Conexões ativas aparecem como `ESTAB`. Cada porta em `LISTEN` é superfície de ataque.

**Correção importante: o caminho de um acesso**
Eu tinha dito que o DNS verifica o IP e dá permissão de acesso, e que o 403 vinha dele. **O DNS só traduz nome em IP**, como uma lista telefônica. A sequência correta é:
1. O navegador pergunta ao **DNS** qual é o IP de `example.com`.
2. Abre uma conexão **TCP** com esse IP e, por ser HTTPS, negocia a criptografia (**TLS**).
3. Envia a requisição **HTTP** (`GET /`).
4. O **servidor** decide a resposta e devolve um código de status e o conteúdo.

O `403 Forbidden` vem do passo 4: o servidor entendeu o pedido, mas se recusou a atender (decisão do servidor ou de um firewall/WAF na frente dele). Se o navegador dá bloqueio e o `curl` dá 200, algo no caminho do navegador interfere (proxy, extensão, filtro da rede).

---

## Missão 7: SSH

**Comandos:** `ssh`, `ssh-keygen`, `ssh-copy-id`, `ls -l ~/.ssh`

**Passo a passo**
1. Instalar o servidor: `sudo apt install openssh-server` e `sudo service ssh start`.
2. Conectar em mim mesmo: `ssh usuario@localhost`.
3. Gerar chaves: `ssh-keygen -t ed25519`.
4. Ver as permissões em `ls -l ~/.ssh`.
5. Copiar a chave pública: `ssh-copy-id usuario@localhost`.
6. Conectar de novo, agora sem senha.

> **Status:** não consegui conectar no `localhost` no WSL. Os passos 2, 5 e 6 ficaram pendentes (ver "Pendências").

**Chave pública vs privada**
- **Pública (`.pub`):** pode ser compartilhada à vontade.
- **Privada:** nunca sai da minha máquina. Deve ter permissão `600`, e o SSH se recusa a usar uma privada que outros usuários conseguem ler. Vale definir uma *passphrase* no `ssh-keygen`.

**Se a chave privada vazasse:** alguém poderia **se passar por mim** em todos os servidores que têm minha chave pública instalada (`authorized_keys`).

**Correções sobre "por que chave é mais seguro"**
| O que eu tinha entendido | O correto |
| --- | --- |
| A chave garante que só quem a tem abre um arquivo | A chave serve para **autenticar**, não protege nem criptografa arquivo. O servidor manda um desafio, eu assino com a privada e ele confere com a pública. A privada nunca viaja pela rede |
| O servidor guarda um hash para ambas as partes | O servidor guarda **só a chave pública**. Se vazar, ela não serve para entrar. Numa senha, o servidor guarda o hash, e é ele que o atacante quebra offline |
| Só domínios reais têm chave | Quem prova a identidade do **servidor** é a *host key* (o fingerprint). Isso é outra coisa |

**Por que chave ganha da senha:** não dá para adivinhar nem fazer força bruta, nada secreto trafega pela rede e dá para desligar o login por senha. O risco muda de lugar: quem roubar a privada entra, daí o `600` e a passphrase.

**Fingerprint (host key)**
- É um resumo (hash) da chave pública do **servidor**. Na primeira conexão o SSH pergunta se confio, e guarda em `~/.ssh/known_hosts`.
- Se o valor mudar, aparece `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`. Pode ser só reinstalação, mas também pode ser um ataque *man-in-the-middle*.
- O ideal é conferir o fingerprint por um canal confiável, e não digitar `yes` no automático.

---

## Missão 8: Logs (ponte para o SOC)

**Comandos:** `ls -lh /var/log`, `tail -f`, `grep`, `journalctl`

**Passo a passo**
1. Listar `/var/log` e identificar os arquivos principais.
2. Acompanhar `tail -f /var/log/auth.log` em um terminal e gerar eventos em outro (`sudo ls`, `ssh localhost`).
3. Achar meus registros de `sudo`: `sudo grep "sudo" /var/log/auth.log | tail`.
4. Gerar falhas de propósito com `ssh usuario_inexistente@localhost`.
5. Contar tentativas por IP.
6. Pensar como analista de SOC.

> No WSL, o `auth.log` só existe se o `rsyslog` estiver instalado (`sudo apt install rsyslog` e `sudo service rsyslog start`). `journalctl` é a alternativa.

**Arquivos de log**
- `syslog`: logs gerais do sistema.
- `auth.log`: autenticação e segurança.
- `dpkg.log`: histórico de gerenciamento de pacotes.

**Lendo uma linha de `sudo` no log**
Mostra o **hostname** da máquina (não o IP), o usuário que rodou, `USER=root` (como quem rodou), o `COMMAND`, o `PWD` e o `TTY` (`pts/4` é um pseudo-terminal). Com a data e a hora, responde: **quem fez o quê, quando e com qual privilégio**.

**Contagem de tentativas por IP**
```bash
sudo grep "Failed password" /var/log/auth.log | grep -oE "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" | sort | uniq -c | sort -rn
```
- `grep "Failed password"`: filtra só as falhas de senha.
- `grep -oE "..."`: `-E` aceita expressão regular (formato de IP) e `-o` imprime **só o trecho que casou**.
- `sort`: ordena, obrigatório porque o `uniq` só junta linhas **iguais e vizinhas**.
- `uniq -c`: junta as repetições e coloca a contagem na frente.
- `sort -rn`: ordena pelo número, do maior para o menor.

**Sinais de ataque de força bruta**
- Muitas falhas do **mesmo IP** em pouco tempo (automação).
- Falhas para **muitos usuários diferentes** (`admin`, `root`, `test`), com `invalid user`: o atacante testa quais contas existem.
- Muitas falhas para **um único usuário**: ele já sabe quem atacar.
- O mais grave: falhas **seguidas de `Accepted password`** do mesmo IP, ou seja, o ataque funcionou.

---

## Desafio final: primeiro script bash

**Objetivo:** um script `desafio.sh` que, sozinho:
1. Cria a pasta `desafio-linux` (`mkdir -p`).
2. Gera 5 arquivos com um laço `for`.
3. Dá `600` para dois arquivos e `755` para um.
4. Lista as permissões com `ls -l`.
5. Salva a listagem em `relatorio.txt`.

**Regras:** começar com `#!/bin/bash`, dar `chmod +x desafio.sh` e rodar com `./desafio.sh`.

**Resultado:** consegui rodar e funcionou.

---

## Resumo dos meus erros mais comuns (para não repetir)

1. **Misturar termos de Git com Linux:** no Linux é "diretório", não "repositório".
2. **Confundir configuração com programa:** `/etc` guarda configuração, não executáveis.
3. **Atribuir ao DNS o que é do servidor:** o DNS só traduz nome em IP.
4. **Confundir autenticação com criptografia:** a chave SSH serve para provar quem eu sou.
5. **Ler só o que aparece, e não o que está faltando:** por exemplo `USER=root` no log e o campo `x` no `passwd`.

---

## Pendências

- [ ] Fazer o SSH funcionar no `localhost` (`sudo service ssh status`, `ssh -v usuario@localhost`) e completar os passos 2, 5 e 6 da missão 7.
- [ ] Gerar falhas de login reais e rodar a contagem por IP da missão 8 (passos 4 e 5).
- [ ] Subir estas anotações para um repositório no GitHub (entregável da Fase 1).
- [ ] Próxima etapa: trilha **Pre Security** do TryHackMe.
