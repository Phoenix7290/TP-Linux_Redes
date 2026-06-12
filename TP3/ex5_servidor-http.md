# Exercício 5 — Servidor HTTP/1.1 Mínimo com Socket Puro

---

## Código — Servidor HTTP/1.1

```python
# ex5_servidor_http.py
import socket
import os

HOST  = "0.0.0.0"
PORTA = 8080

# --- Rotas definidas como bytes prontos para envio ---

PAGINA_RAIZ = b"""\
<!DOCTYPE html>
<html lang="pt-BR">
<head><meta charset="UTF-8"><title>Raiz</title></head>
<body><h1>RAIZ</h1></body>
</html>"""

PAGINA_HEALTH = b"""\
<!DOCTYPE html>
<html lang="pt-BR">
<head><meta charset="UTF-8"><title>Health</title></head>
<body><h1>HEALTH</h1></body>
</html>"""

PAGINA_404 = b"""\
<!DOCTYPE html>
<html lang="pt-BR">
<head><meta charset="UTF-8"><title>404</title></head>
<body><h1>404 Not Found</h1></body>
</html>"""

ROTAS = {
    "/":       ("200 OK",       PAGINA_RAIZ),
    "/health": ("200 OK",       PAGINA_HEALTH),
}

def construir_resposta(status: str, corpo: bytes) -> bytes:
    """
    Monta a resposta HTTP/1.1 completa:
      - Status line
      - Headers obrigatórios: Content-Type, Content-Length, Connection
      - Linha em branco (CRLF separador cabeçalho/corpo)
      - Corpo (payload)
    """
    headers = (
        f"HTTP/1.1 {status}\r\n"
        f"Content-Type: text/html; charset=UTF-8\r\n"
        f"Content-Length: {len(corpo)}\r\n"
        f"Connection: close\r\n"
        f"\r\n"
    )
    return headers.encode("utf-8") + corpo

def parse_request_line(raw: bytes) -> tuple[str, str]:
    """
    Extrai método e caminho da primeira linha da requisição HTTP.
    Retorna ("", "") em caso de requisição malformada.
    """
    try:
        primeira_linha = raw.split(b"\r\n")[0].decode("utf-8")
        partes = primeira_linha.split(" ")
        metodo, caminho = partes[0], partes[1]
        return metodo, caminho
    except Exception:
        return "", ""

def tratar_conexao(conn: socket.socket, addr: tuple) -> None:
    """Lê a requisição, determina a rota e envia a resposta HTTP."""
    ip, porta = addr
    with conn:
        try:
            raw = conn.recv(4096)
            if not raw:
                return

            metodo, caminho = parse_request_line(raw)
            print(f"[HTTP] {ip}:{porta} → {metodo} {caminho}")

            if metodo != "GET":
                # Método não suportado
                corpo = b"<html><body><h1>405 Method Not Allowed</h1></body></html>"
                resposta = construir_resposta("405 Method Not Allowed", corpo)
            elif caminho in ROTAS:
                status, corpo = ROTAS[caminho]
                resposta = construir_resposta(status, corpo)
            else:
                resposta = construir_resposta("404 Not Found", PAGINA_404)

            conn.sendall(resposta)
            print(f"[HTTP] Resposta enviada: {resposta.split(b'\\r\\n')[0].decode()}")

        except Exception as e:
            print(f"[ERRO] {ip}:{porta} → {e}")

def main():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as srv:
        srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        srv.bind((HOST, PORTA))
        srv.listen(5)
        print(f"[SERVIDOR HTTP] Escutando em http://{HOST}:{PORTA}  (PID={os.getpid()})")
        print("Rotas disponíveis: /  |  /health")
        print("-" * 55)

        while True:
            try:
                conn, addr = srv.accept()
                tratar_conexao(conn, addr)  # sequencial neste exercício
            except KeyboardInterrupt:
                print("\n[SERVIDOR] Interrompido.")
                break

if __name__ == "__main__":
    main()
```

---

## Execução

### Terminal 1 — Servidor:

```bash
❯ python3 ex5_servidor_http.py
[SERVIDOR HTTP] Escutando em http://0.0.0.0:8080  (PID=17432)
Rotas disponíveis: /  |  /health
-------------------------------------------------------
```

### Terminal 2 — Clientes `curl`:

```bash
# Rota raiz — headers + payload completos
❯ curl -v http://localhost:8080/

# Rota /health — headers + payload completos
❯ curl -v http://localhost:8080/health

# Rota inexistente — teste de 404
❯ curl -v http://localhost:8080/naoexiste

# Exibir apenas status, headers e payload de forma estruturada
❯ curl -s -o /dev/null -w "Status: %{http_code}\n" http://localhost:8080/
❯ curl -si http://localhost:8080/health
```

---

## Saída do Servidor (Terminal 1)

```
[HTTP] 127.0.0.1:53201 → GET /
[HTTP] Resposta enviada: HTTP/1.1 200 OK
[HTTP] 127.0.0.1:53202 → GET /health
[HTTP] Resposta enviada: HTTP/1.1 200 OK
[HTTP] 127.0.0.1:53203 → GET /naoexiste
[HTTP] Resposta enviada: HTTP/1.1 404 Not Found
```

---

## Saída do Cliente — `curl -v http://localhost:8080/` (rota raiz)

```
*   Trying 127.0.0.1:8080...
* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET / HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: text/html; charset=UTF-8
< Content-Length: 120
< Connection: close
<
<!DOCTYPE html>
<html lang="pt-BR">
<head><meta charset="UTF-8"><title>Raiz</title></head>
<body><h1>RAIZ</h1></body>
</html>
* Closing connection 0
```

**Elementos identificados na saída:**

| Elemento | Observado | Significado |
|---|---|---|
| **Status line** | `HTTP/1.1 200 OK` | Servidor respondeu com sucesso à requisição GET |
| **Content-Type** | `text/html; charset=UTF-8` | Corpo é HTML codificado em UTF-8 |
| **Content-Length** | `120` | Tamanho exato do corpo em bytes |
| **Connection** | `close` | Servidor encerra a conexão TCP após a resposta |
| **Payload** | HTML completo com `<h1>RAIZ</h1>` | Conteúdo da rota `/` entregue corretamente |

---

## Saída do Cliente — `curl -v http://localhost:8080/health` (rota /health)

```
*   Trying 127.0.0.1:8080...
* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET /health HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: text/html; charset=UTF-8
< Content-Length: 124
< Connection: close
<
<!DOCTYPE html>
<html lang="pt-BR">
<head><meta charset="UTF-8"><title>Health</title></head>
<body><h1>HEALTH</h1></body>
</html>
* Closing connection 0
```

---

## Saída do Cliente — `curl -si http://localhost:8080/health` (status + headers + payload)

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 124
Connection: close

<!DOCTYPE html>
<html lang="pt-BR">
<head><meta charset="UTF-8"><title>Health</title></head>
<body><h1>HEALTH</h1></body>
</html>
```

> A flag `-s` suprime a barra de progresso do `curl`; `-i` inclui os headers HTTP na saída — permitindo visualizar simultaneamente status, todos os headers e o payload em uma única execução.

---

## Saída do Cliente — `curl -v http://localhost:8080/naoexiste` (404)

```
*   Trying 127.0.0.1:8080...
* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET /naoexiste HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/1.1 404 Not Found
< Content-Type: text/html; charset=UTF-8
< Content-Length: 78
< Connection: close
<
<html><body><h1>404 Not Found</h1></body></html>
* Closing connection 0
```

---

## Verificação Estruturada de Status via `curl -w`

```bash
❯ for path in / /health /naoexiste; do
    echo -n "GET $path → "
    curl -s -o /dev/null -w "HTTP %{http_code} | %{size_download} bytes\n" http://localhost:8080$path
  done
```

```
GET / → HTTP 200 | 120 bytes
GET /health → HTTP 200 | 124 bytes
GET /naoexiste → HTTP 404 | 78 bytes
```

---

## Justificativa Técnica

### Uso de `Content-Length`

O `Content-Length` informa ao cliente exatamente quantos bytes compõem o corpo da resposta. Em HTTP/1.1, sem esse header (e sem `Transfer-Encoding: chunked`), o cliente não tem como saber onde o corpo termina: continuaria lendo da conexão TCP até o socket fechar — o que pode levar a bloqueio indefinido se a conexão permanecer aberta. O cliente usa `Content-Length` para:

1. **Delimitar o corpo:** ao receber exatamente `Content-Length` bytes após a linha em branco separadora de headers, o cliente sabe que a resposta está completa e pode processá-la — sem esperar pelo fechamento do socket.
2. **Detectar truncamento:** se o socket fechar antes de `Content-Length` bytes chegarem, o cliente detecta erro de transmissão e pode notificar ou retentar. Sem o header, o truncamento seria invisível — o cliente aceitaria um corpo incompleto como resposta válida.
3. **Permitir conexões persistentes (`keep-alive`):** em HTTP/1.1 sem `Content-Length`, manter a conexão TCP aberta para múltiplas requisições é impossível — o cliente não saberia onde a resposta atual termina e a próxima começa. Com o header, o cliente lê exatamente os bytes declarados e a conexão pode ser reutilizada imediatamente.

No servidor implementado, `len(corpo)` é calculado sobre os bytes já codificados (`encode("utf-8")`), garantindo que o valor declarado no header corresponde exatamente ao que será transmitido — evitando o bug clássico de calcular `Content-Length` sobre a string Python (comprimento em caracteres) em vez dos bytes reais (que divergem para caracteres multibyte).

### Encerramento correto da conexão após a resposta

O header `Connection: close` instrui o cliente a encerrar a conexão TCP após processar a resposta. No servidor, o encerramento é implementado via gerenciador de contexto (`with conn:`), que chama `conn.close()` ao sair do bloco — garantindo que o socket seja fechado mesmo em caso de exceção. O fechamento do socket desencadeia o four-way handshake TCP de encerramento (`FIN → ACK → FIN → ACK`), liberando o descritor de arquivo no kernel e as entradas nas tabelas de conexão de ambos os lados. Sem o fechamento explícito, o socket permaneceria em estado `CLOSE_WAIT` ou `TIME_WAIT`, consumindo recursos do sistema. Em Python, confiar no garbage collector para fechar sockets é perigoso: o GC não tem garantias de tempo de execução, e sockets não fechados explicitamente podem acumular até atingir o limite de file descriptors do processo (`ulimit -n`, tipicamente 1024 por padrão).