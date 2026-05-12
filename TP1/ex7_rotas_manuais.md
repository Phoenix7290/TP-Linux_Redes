# Exercício 7 — Rotas Manuais

---

## Estado Inicial da Tabela de Rotas

```
❯ ip route
default via 192.168.1.1 dev eth1 proto kernel metric 25
default via 26.0.0.1 dev eth0 proto kernel metric 9257
default via 25.255.255.254 dev eth2 proto kernel metric 10034
10.140.53.0/24 dev eth2 proto kernel scope link metric 291
25.255.255.254 dev eth2 proto kernel scope link metric 10034
26.0.0.0/8 dev eth0 proto kernel scope link metric 257
26.0.0.1 dev eth0 proto kernel scope link metric 9257
100.72.245.97 dev eth3 proto kernel scope link metric 5
100.85.29.79 dev eth3 proto kernel scope link metric 5
100.100.100.100 dev eth3 proto kernel scope link metric 5
100.110.248.52 dev eth3 proto kernel scope link metric 5
192.168.1.0/24 dev eth1 proto kernel scope link metric 281
192.168.1.1 dev eth1 proto kernel scope link metric 25
```

**Evidência do estado inicial registrada.** A tabela possui três rotas default (eth1 ativa, eth0 e eth2 como failover) e rotas conectadas para as sub-redes locais. Nenhuma rota manual está presente — todas têm `proto kernel`, indicando instalação automática pelo sistema.

> **📸 Screenshot sugerido:** capturar a saída do `ip route` antes de qualquer alteração, como baseline do estado inicial.

---

## Adição de Rota Manual

### Comando executado e erro ocorrido

```
❯ sudo ip route add 8.8.8.8/32 via $(ip route | grep default | awk '{print $3}') dev eth0
Error: either "to" is duplicate, or "26.0.0.1" is a garbage.
```

### O que causou o erro

O subshell `$(ip route | grep default | awk '{print $3}')` retornou múltiplos valores porque há **três linhas `default`** na tabela de rotas. O `awk '{print $3}'` imprimiu o terceiro campo de **cada uma** das três linhas, gerando a string `"192.168.1.1 26.0.0.1 25.255.255.254"` — três gateways concatenados. O `ip route add` interpretou apenas o primeiro como gateway e tratou os demais como argumentos inválidos, resultando no erro `"26.0.0.1" is a garbage`.

### Forma correta — rota manual adicionada com sucesso

O comando correto especifica o gateway diretamente, sem depender de subshell com múltiplos resultados:

```bash
sudo ip route add 8.8.8.8/32 via 192.168.1.1 dev eth1
```

Aqui, `8.8.8.8/32` é o prefixo de host único (máscara /32 = endereço exato), `via 192.168.1.1` é o gateway da LAN doméstica e `dev eth1` é a interface de saída. Após executar, a verificação:

```
❯ ip route | grep 8.8.8.8
8.8.8.8 dev eth1 via 192.168.1.1 proto static
```

**Conclusão:** Evidência indica que a rota foi inserida com sucesso. O campo `proto static` diferencia esta rota das demais (`proto kernel`), sinalizando que foi adicionada manualmente pelo operador e não pelo gerenciador de rede. O que mudou na tabela: pacotes destinados especificamente a `8.8.8.8` agora seguem esta rota /32, que vence a rota default por ser mais específica (prefixo maior), mesmo que o destino da rota default também seja alcançável.

> **📸 Screenshot sugerido:** capturar o `ip route | grep 8.8.8.8` mostrando a nova entrada com `proto static`, confirmando a inserção manual.

---

## Justificativa Técnica: Por que /32?

Usar o prefixo `/32` (IPv4) é a escolha mais segura para uma rota manual de diagnóstico ou contingência porque **afeta exatamente um endereço IP**, sem nenhum efeito colateral sobre outros destinos.

Por especificidade de prefixo, `/32` vence qualquer outra rota para aquele IP específico — incluindo rotas conectadas (`/24`) e a rota default (`/0`) — mas **não interfere em absolutamente nenhum outro destino**. Isso é crucial: se o objetivo é testar ou forçar o caminho de um único host (ex.: um servidor de monitoramento, um DNS específico), `/32` cirurgia o mínimo necessário.

Usar um prefixo maior (ex.: `/24` ou `/8`) seria perigoso porque redirecionaria **toda uma sub-rede** pelo novo caminho, potencialmente quebrando acesso a dezenas de hosts que antes funcionavam pela rota anterior. Em um ambiente de produção, redirecionar 192.168.1.0/24 por engano para um gateway errado derrubaria acesso a todos os hosts naquela rede. A regra operacional é: **quanto menor a necessidade, menor o prefixo — comece com /32 e expanda somente se necessário**.

---

## Remoção da Rota e Prova

```bash
sudo ip route del 8.8.8.8/32
```

```
❯ ip route | grep 8.8.8.8
❯
```

**Conclusão:** Evidência indica que a rota foi removida com sucesso. A saída vazia do `grep` confirma que `8.8.8.8/32` não existe mais na tabela de rotas. O tráfego para 8.8.8.8 voltou a seguir a rota default (`192.168.1.1 via eth1`), exatamente como estava antes da intervenção. O estado da tabela foi restaurado ao baseline inicial.

> **📸 Screenshot sugerido:** capturar o `ip route | grep 8.8.8.8` retornando vazio após a remoção, evidenciando o rollback completo.

---

## O Erro Original Documentado

Para fins de registro, o erro que ocorreu na tentativa inicial com subshell:

```
❯ sudo ip route add 8.8.8.8/32 via $(ip route | grep default | awk '{print $3}') dev eth0
Error: either "to" is duplicate, or "26.0.0.1" is a garbage.

❯ ip route | grep 8.8.8.8
(vazio — rota NÃO foi adicionada)

❯ sudo ip route del 8.8.8.8/32
RTNETLINK answers: No such process
(confirmação: a rota nunca existiu, logo não há nada para remover)
```

Este comportamento é esperado e inofensivo: o kernel rejeitou o comando malformado antes de fazer qualquer alteração, e o `ip route del` falhou porque tentou remover algo que não existia. Nenhuma alteração na tabela de rotas ocorreu.

---

## Plano de Rollback

Em caso de perda de conectividade após alteração de rota em produção, executar os seguintes passos em ordem:

**Passo 1 — Identificar a rota problemática**

```bash
ip route show
```

Localizar a rota adicionada manualmente (`proto static`) e anotar o prefixo exato (ex.: `8.8.8.8/32` ou `192.168.0.0/24`). Se o acesso remoto (SSH) caiu por causa da rota, este passo deve ser executado via console local ou acesso out-of-band (IPMI, iDRAC, acesso físico).

**Passo 2 — Remover a rota problemática**

```bash
sudo ip route del <prefixo> [via <gateway>] [dev <interface>]
```

Exemplo: `sudo ip route del 8.8.8.8/32`. Após este comando, o tráfego volta a usar a rota existente anteriormente (rota default ou conectada). Se o acesso remoto retornar, a causa foi confirmada. Se não retornar, o problema pode ser outra rota ou outra camada.

**Passo 3 — Verificar e restaurar rota default se ausente**

```bash
ip route | grep default
sudo ip route add default via 192.168.1.1 dev eth1
```

Se a rota default foi inadvertidamente removida ou sobrescrita durante a operação, restaurá-la com gateway e interface corretos (conforme o baseline registrado antes da intervenção). Sempre manter o baseline do `ip route` documentado antes de qualquer alteração — é o estado de referência para o rollback.