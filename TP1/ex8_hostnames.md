# Exercício 8 — Hostnames

---

## Estado Atual do Hostname

### Comando 1: `hostname`

```
❯ hostname
WinCore
```

**Conclusão:** Evidência indica que o hostname atual é `WinCore`. Este nome foi definido automaticamente pelo WSL com base no nome da máquina Windows host. O comando `hostname` consulta o valor em tempo real diretamente do kernel (via syscall `gethostname`), sem ler arquivos de configuração.

---

### Comando 2: `hostnamectl`

```
❯ hostnamectl
 Static hostname: WinCore
       Icon name: computer-container
         Chassis: container ☐
      Machine ID: 67749eaa5e6940e39337d9f4a42ecf81
         Boot ID: ca29dbbc00bd45c0a9e9fb5fc97a3e23
  Virtualization: wsl
Operating System: Ubuntu 24.04.4 LTS
          Kernel: Linux 6.6.87.2-microsoft-standard-WSL2
    Architecture: x86-64
```

**Conclusão:** Evidência indica que o hostname estático é `WinCore`, rodando sobre Ubuntu 24.04.4 LTS em virtualização WSL2 (kernel Microsoft). O campo `Static hostname` é o nome persistente gravado em `/etc/hostname`. O campo `Chassis: container` reflete que o WSL é detectado como ambiente de container pelo systemd. O `Machine ID` é um identificador único da instância, gerado na instalação e usado por serviços como journald e systemd para correlacionar logs.

> **📸 Screenshot sugerido:** capturar a saída de `hostnamectl` completa para evidenciar todos os campos, em especial `Static hostname` e `Virtualization: wsl`.

---

## Convenção de Hostname Definida

**Convenção proposta:** `pb-<usuario>-<papel>`

Exemplos: `pb-ryansubu-dev`, `pb-ryansubu-db`, `pb-ryansubu-web`

**Justificativa:** Uma convenção de hostname deve ser legível, informativa e sem ambiguidade em logs e inventários. O prefixo `pb` identifica o bloco da disciplina (Projeto de Bloco), tornando imediato o contexto organizacional ao ver o nome em qualquer log centralizado. O campo `<usuario>` identifica o responsável ou dono do host — crucial em ambientes compartilhados ou de laboratório onde múltiplos alunos operam máquinas na mesma rede. O campo `<papel>` descreve a função do host (`dev` para desenvolvimento, `db` para banco de dados, `web` para servidor web), permitindo que automações, scripts de inventário e alertas de monitoramento filtrem por papel sem precisar consultar documentação adicional. Hostnames sem convenção (como `WinCore` ou `ubuntu`) geram ambiguidade em logs: "qual máquina é essa?" é uma pergunta que nenhum engenheiro de plantão deveria precisar fazer.

---

## Aplicação do Novo Hostname

### Alterando o hostname de forma persistente

```bash
sudo hostnamectl set-hostname pb-ryansubu-dev
```

O `hostnamectl set-hostname` é o método correto em sistemas com systemd: ele atualiza **simultaneamente** o hostname em runtime (equivalente ao `hostname` antigo) e de forma persistente em `/etc/hostname`. Sem o `hostnamectl`, uma alteração via `hostname <nome>` seria perdida no próximo boot.

### Prova da alteração

```bash
❯ hostname
pb-ryansubu-dev

❯ hostnamectl
 Static hostname: pb-ryansubu-dev
       Icon name: computer-container
         Chassis: container ☐
      Machine ID: 67749eaa5e6940e39337d9f4a42ecf81
         Boot ID: ca29dbbc00bd45c0a9e9fb5fc97a3e23
  Virtualization: wsl
Operating System: Ubuntu 24.04.4 LTS
          Kernel: Linux 6.6.87.2-microsoft-standard-WSL2
    Architecture: x86-64

❯ cat /etc/hostname
pb-ryansubu-dev
```

**Conclusão:** Evidência indica que as três fontes de verdade estão consistentes: `hostname` (runtime), `hostnamectl` (systemd) e `/etc/hostname` (arquivo persistente) retornam o mesmo valor `pb-ryansubu-dev`. A alteração é persistente — sobreviverá a reinicializações do WSL.

> **📸 Screenshot sugerido:** capturar o `hostname` e o `hostnamectl` lado a lado após a alteração, evidenciando o novo nome em `Static hostname`.

---

## Verificação e Ajuste do `/etc/hosts`

Após alterar o hostname, é necessário verificar se o antigo nome (`WinCore`) está mapeado em `/etc/hosts` — se estiver, pode causar erros em aplicações que resolvem o próprio hostname (ex.: sudo, postfix, alguns bancos de dados).

```bash
❯ cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       WinCore.localdomain     WinCore
::1     ip6-localhost ip6-loopback
```

O hostname antigo `WinCore` ainda está na linha `127.0.1.1`. Atualizar para o novo nome:

```bash
sudo sed -i 's/WinCore.localdomain\s*WinCore/pb-ryansubu-dev.localdomain\t pb-ryansubu-dev/' /etc/hosts
```

Resultado após ajuste:

```
127.0.0.1       localhost
127.0.1.1       pb-ryansubu-dev.localdomain    pb-ryansubu-dev
::1     ip6-localhost ip6-loopback
```

**Justificativa do ajuste:** Aplicações que chamam `gethostbyname(hostname())` — ou seja, tentam resolver o próprio hostname — consultam `/etc/hosts` primeiro. Se o arquivo retorna o nome antigo ou falha na resolução do nome novo, comandos como `sudo` podem apresentar o aviso `unable to resolve host pb-ryansubu-dev`, e serviços como Postfix ou RabbitMQ podem recusar iniciar. A consistência entre `/etc/hostname` e `/etc/hosts` é requisito operacional básico.

> **📸 Screenshot sugerido:** capturar o `cat /etc/hosts` após o ajuste, mostrando o novo hostname na linha `127.0.1.1`.

---

## Reversão ao Hostname Original

```bash
sudo hostnamectl set-hostname WinCore
```

```bash
sudo sed -i 's/pb-ryansubu-dev.localdomain.*pb-ryansubu-dev/WinCore.localdomain\tWinCore/' /etc/hosts
```

### Prova da reversão

```bash
❯ hostname
WinCore

❯ hostnamectl | grep "Static hostname"
 Static hostname: WinCore

❯ cat /etc/hostname
WinCore
```

**Conclusão:** Evidência indica que o hostname foi revertido com sucesso ao valor original `WinCore` em todas as camadas — runtime, systemd e arquivo persistente. O `/etc/hosts` também foi restaurado, mantendo a consistência do ambiente. A operação foi completamente reversível, sem efeito residual.

> **📸 Screenshot sugerido:** capturar o `hostname` retornando `WinCore` após a reversão, para evidenciar o ciclo completo: estado inicial → alteração → reversão.