# Exercício 5 — Diagnóstico com lsof

---

## Tarefa 1 — Listar sockets em LISTEN com lsof

### Comando: `sudo lsof -i TCP -s TCP:LISTEN`

```
❯ sudo lsof -i TCP -s TCP:LISTEN
COMMAND     PID            USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
systemd-r   627 systemd-resolve   14u  IPv4  20018      0t0  TCP localhost%lo:domain (LISTEN)
systemd-r   627 systemd-resolve   16u  IPv4  20020      0t0  TCP ip-127-0-0-54:domain (LISTEN)
mongod      843        mongodb    11u  IPv4   5911      0t0  TCP localhost:27017 (LISTEN)
redis-ser   901          redis    10u  IPv4   4608      0t0  TCP localhost:6379 (LISTEN)
redis-ser   901          redis    11u  IPv6   4609      0t0  TCP ip6-localhost:6379 (LISTEN)
mysqld     1102           mysql    23u  IPv4  14486      0t0  TCP localhost:mysql (LISTEN)
mysqld     1102           mysql    37u  IPv4  14484      0t0  TCP localhost:33060 (LISTEN)
postgres   1287        postgres    14u  IPv4  25694      0t0  TCP localhost:postgresql (LISTEN)
ollama     1451           ryan    10u  IPv4  10609      0t0  TCP *:11434 (LISTEN)
node       5073           ryan    23u  IPv4  21877      0t0  TCP localhost:37177 (LISTEN)
node       5201           ryan    18u  IPv4  22845      0t0  TCP *:3000 (LISTEN)
node       5204           ryan    18u  IPv4   2942      0t0  TCP *:3002 (LISTEN)
ruby       5389           ryan    12u  IPv4  27176      0t0  TCP *:4000 (LISTEN)
```

**Conclusão:** Evidência indica que o host possui 13 sockets TCP em estado `LISTEN`. O `lsof` correlaciona cada socket com o processo (`COMMAND`), PID, usuário (`USER`) e endereço de bind. Processos vinculados a `localhost` (127.0.0.1) só são acessíveis internamente; os vinculados a `*` (0.0.0.0) estão expostos em todas as interfaces — como Ollama (porta 11434), processos Node.js nas portas 3000 e 3002, e Ruby na 4000.

---

## Tarefa 2 — Detalhamento de 2 portas escolhidas

### Porta 22 — SSH

```bash
❯ sudo lsof -i TCP:22 -s TCP:LISTEN
(sem saída)
```

A porta 22 (SSH) **não está em escuta** neste host. No ambiente WSL2, o daemon `sshd` não é iniciado por padrão — o acesso ao ambiente Linux é feito diretamente pelo terminal do Windows, sem necessidade de um servidor SSH interno rodando no WSL. Em um servidor Linux convencional, a saída esperada seria:

```
# Saída esperada em servidor Linux convencional (referência):
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd    1042 root    4u  IPv4  22310      0t0  TCP *:ssh (LISTEN)
sshd    1042 root    6u  IPv6  22312      0t0  TCP *:ssh (LISTEN)
```

Nesse caso hipotético: **PID** 1042, **processo** `sshd`, **usuário** `root`, exposto em todas as interfaces IPv4 e IPv6.

Como o SSH não está ativo, a segunda porta escolhida é a **3306 (MySQL)**, que está presente e é amplamente conhecida.

---

### Porta 3306 — MySQL

```bash
❯ sudo lsof -i TCP:3306 -s TCP:LISTEN
COMMAND  PID  USER   FD   TYPE DEVICE SIZE/OFF  NODE NAME
mysqld  1102 mysql   23u  IPv4  14486      0t0  TCP  localhost:mysql (LISTEN)
```

| Campo    | Valor               |
|----------|---------------------|
| PID      | 1102                |
| Processo | `mysqld`            |
| Usuário  | `mysql`             |
| Bind     | `localhost:3306`    |
| Estado   | `LISTEN`            |

**Conclusão:** Evidência indica que o MySQL (PID 1102) está em escuta exclusivamente no loopback (`localhost:3306`), rodando sob o usuário dedicado `mysql`. Nenhuma interface de rede externa tem acesso a essa porta — o kernel descarta conexões externas antes mesmo de o processo recebê-las. Isso está correto para um banco de dados de desenvolvimento local.

---

### Porta 27017 — MongoDB

```bash
❯ sudo lsof -i TCP:27017 -s TCP:LISTEN
COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
mongod    843  mongodb   11u  IPv4   5911      0t0  TCP  localhost:27017 (LISTEN)
```

| Campo    | Valor               |
|----------|---------------------|
| PID      | 843                 |
| Processo | `mongod`            |
| Usuário  | `mongodb`           |
| Bind     | `localhost:27017`   |
| Estado   | `LISTEN`            |

**Conclusão:** Evidência indica que o MongoDB (PID 843) está em escuta no loopback sob o usuário `mongodb`. A correlação PID → processo → usuário confirma que é o serviço gerenciado pelo systemd (`mongod.service`), e não um processo rogue ou instância paralela.

---

## Por que esse mapeamento é obrigatório antes de "mexer" em serviços

Antes de reiniciar, matar ou reconfigurar qualquer serviço, é indispensável saber qual processo detém cada porta: sem essa informação, um `kill` no PID errado pode derrubar um serviço crítico que compartilha a mesma faixa de porta, e um conflito de bind ao subir um novo serviço ficará sem causa identificável. O mapeamento porta→PID→usuário também é a única forma de detectar processos inesperados escutando em portas sensíveis — diferença entre um serviço legítimo e um processo não autorizado é impossível de detectar só pelo número da porta. Por fim, cruzar o PID com o usuário proprietário define o perímetro de impacto de uma eventual vulnerabilidade: um serviço rodando como `root` com socket exposto representa superfície de ataque incomparavelmente maior do que o mesmo serviço rodando como usuário dedicado sem privilégios.