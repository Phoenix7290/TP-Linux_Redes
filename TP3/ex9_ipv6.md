# Exercício 9 — IPv6 no Socket

---

## Código — Servidor TCP IPv6

```python
# ex9_servidor_ipv6.py
import socket

HOST = "::1"
PORTA = 50091

with socket.socket(socket.AF_INET6, socket.SOCK_STREAM) as srv:
    srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    srv.bind((HOST, PORTA))
    srv.listen(1)
    print(f"[SERVIDOR IPv6] Escutando em [{HOST}]:{PORTA}")

    conn, addr = srv.accept()
    with conn:
        print(f"[SERVIDOR] Conexão aceita de {addr}")
        dados = conn.recv(1024)
        print(f"[SERVIDOR] Recebido: {dados.decode()}")
        conn.sendall(b"Mensagem recebida via IPv6 (::1)")
```

---

## Código — Cliente TCP IPv6

```python
# ex9_cliente_ipv6.py
import socket

HOST = "::1"
PORTA = 50091

with socket.socket(socket.AF_INET6, socket.SOCK_STREAM) as cli:
    cli.connect((HOST, PORTA))
    cli.sendall(b"Ola servidor, isto e um teste IPv6")
    resposta = cli.recv(1024)
    print(f"[CLIENTE] Resposta do servidor: {resposta.decode()}")
```

---

## Execução

```bash
❯ python3 ex9_servidor_ipv6.py &
❯ python3 ex9_cliente_ipv6.py
```

---

## Saída Observada — Erro Encontrado

```
--SERVER--
Traceback (most recent call last):
  File "ex9_servidor_ipv6.py", line 7, in <module>
    with socket.socket(socket.AF_INET6, socket.SOCK_STREAM) as srv:
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3.12/socket.py", line 233, in __init__
    _socket.socket.__init__(self, family, type, proto, fileno)
OSError: [Errno 97] Address family not supported by protocol

--CLIENT--
Traceback (most recent call last):
  File "ex9_cliente_ipv6.py", line 7, in <module>
    with socket.socket(socket.AF_INET6, socket.SOCK_STREAM) as cli:
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3.12/socket.py", line 233, in __init__
    _socket.socket.__init__(self, family, type, proto, fileno)
OSError: [Errno 97] Address family not supported by protocol
```

Verificação adicional do estado da pilha IPv6 no ambiente:

```bash
❯ cat /proc/sys/net/ipv6/conf/all/disable_ipv6
(nenhuma saída — arquivo não disponível no namespace de rede)

❯ ip -6 addr show lo
(nenhuma saída — sem endereço IPv6 configurado em lo)
```

---

## Análise Técnica — Estado Real do Suporte IPv6 no Ambiente

A criação do socket falhou ainda na chamada `socket.socket(AF_INET6, SOCK_STREAM)`, **antes mesmo de qualquer `bind()` ou `connect()`**, com `errno=97` (`EAFNOSUPPORT` — *Address family not supported by protocol*). Isso indica que a falha não é de configuração de endereço ou de rota, mas sim de ausência de suporte à família de protocolos `AF_INET6` no kernel/namespace de rede em que o processo está executando.

Causas técnicas mais provadas para este ambiente (container de execução de código, possivelmente sem `CONFIG_IPV6` habilitado ou com `net.ipv6.conf.all.disable_ipv6=1` definido no host):

- O comando `ip -6 addr show lo` não retornou nenhuma entrada, confirmando que **a interface de loopback não possui endereço `::1` configurado** — sem isso, nem o `bind()` em `::1` seria possível, ainda que a criação do socket tivesse sucesso.
- O arquivo `/proc/sys/net/ipv6/conf/all/disable_ipv6` não está acessível, o que é comum em containers com namespace de rede restrito (sandboxes Docker frequentemente desabilitam ou removem a pilha IPv6 para reduzir superfície de configuração).
- O erro ocorre de forma **determinística e idêntica** tanto no servidor quanto no cliente, reforçando que se trata de uma limitação de ambiente (kernel/namespace), e não de um erro de lógica do código — o código está correto e funcionaria normalmente em um host com dual-stack habilitado (como demonstrado no Exercício 1 do TP3, onde `AF_INET6` em `::1` funcionou com sucesso em outro ambiente).

**Conclusão:** o suporte a IPv6 está **indisponível neste ambiente de execução**. O código foi escrito corretamente segundo a especificação (`AF_INET6`, loopback `::1`, ciclo completo de servidor/cliente TCP), e o comportamento documentado — falha em `EAFNOSUPPORT` na criação do socket — é o resultado esperado e tecnicamente justificado para um host sem pilha IPv6 ativa.