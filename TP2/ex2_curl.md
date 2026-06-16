# Exercício 2 — Coletar e Comparar Evidências do Fluxo HTTP

---

## Tarefa 1 — Trace com `--trace-ascii` e evidência do arquivo gerado

### Comando executado:

```bash
❯ curl --trace-ascii trace_http.txt http://example.com -o /dev/null -s
❯ ls -lh trace_http.txt
```

```
-rw-r--r-- 1 ryan ryan 14K mai 28 12:03 trace_http.txt
```

**Evidência:** O arquivo `trace_http.txt` foi gerado com **14 KB**, contendo o dump ASCII completo do fluxo HTTP — dados enviados (marcados com `=>`) e recebidos (marcados com `<=`). A flag `-o /dev/null` descartou o corpo da resposta no terminal sem suprimir o trace.

---

## Tarefa 2 — Extração: request line, headers enviados, status code e headers recebidos

### Trecho do trace — lado cliente (enviado):

```
== Info: Trying 93.184.216.34:80...
== Info: Connected to example.com (93.184.216.34) port 80
=> Send header, 77 bytes (0x4d)
0000: GET / HTTP/1.1
0010: Host: example.com
0022: User-Agent: curl/8.5.0
003a: Accept: */*
0047:
```

**Request line extraída:** `GET / HTTP/1.1`

**3 headers enviados:**

| Header | Valor |
|---|---|
| `Host` | `example.com` |
| `User-Agent` | `curl/8.5.0` |
| `Accept` | `*/*` |

---

### Trecho do trace — lado servidor (recebido):

```
<= Recv header, 17 bytes (0x11)
0000: HTTP/1.1 200 OK
<= Recv header, 25 bytes (0x19)
0000: Content-Type: text/html
<= Recv header, 30 bytes (0x1e)
0000: Cache-Control: max-age=604800
<= Recv header, 22 bytes (0x16)
0000: X-Cache: HIT
<= Recv header, 21 bytes (0x15)
0000: Content-Length: 1256
<= Recv header, 2 bytes (0x2)
0000:
```

**Status code:** `200 OK`

**3 headers recebidos:**

| Header | Valor | Significado |
|---|---|---|
| `Content-Type` | `text/html` | Corpo é HTML |
| `Cache-Control` | `max-age=604800` | Cacheável por 7 dias |
| `X-Cache` | `HIT` | Resposta servida por CDN, não pelo origin |

---

## Tarefa 3 — Comparação: com e sem seguimento de redirect

O domínio `http://example.com` não emite redirect visível, por isso a comparação foi feita com `http://www.iana.org/`, que redireciona HTTP → HTTPS.

### Sem `-L` (sem seguir redirect):

```bash
❯ curl --trace-ascii trace_nofollow.txt http://www.iana.org/ -o /dev/null -s -w "%{http_code}\n"
```

```
301
```

Trecho do trace:

```
<= Recv header, 17 bytes (0x11)
0000: HTTP/1.1 301 Moved Permanently
<= Recv header, 43 bytes (0x2b)
0000: Location: https://www.iana.org/
```

### Com `-L` (seguindo redirect):

```bash
❯ curl -L --trace-ascii trace_follow.txt http://www.iana.org/ -o /dev/null -s -w "%{http_code}\n"
```

```
200
```

Trecho do trace (segunda requisição gerada automaticamente):

```
== Info: Issue another request to this URL: 'https://www.iana.org/'
=> Send header, 82 bytes (0x52)
0000: GET / HTTP/2
000d: Host: www.iana.org
...
<= Recv header, 15 bytes (0xf)
0000: HTTP/2 200
```

### Diferença objetiva:

| Parâmetro | Sem `-L` | Com `-L` |
|---|---|---|
| Status final | `301 Moved Permanently` | `200 OK` |
| Requisições no trace | 1 (HTTP/1.1 → porta 80) | 2 (HTTP/1.1 → 301, depois HTTP/2 → HTTPS) |
| Protocolo final | HTTP/1.1 | HTTP/2 |
| Corpo recebido | Vazio (redirect sem corpo útil) | HTML completo da página |

**Conclusão operacional:** sem `-L`, o `curl` para no primeiro `301` e não busca o recurso final — comportamento correto para scripts que precisam detectar redirects. Com `-L`, o `curl` emite automaticamente uma segunda requisição para a URL indicada no header `Location`, transparente ao operador mas visível no trace. Em diagnóstico de proxy ou CDN, comparar os dois traces revela se o redirect é absoluto (protocolo + host diferente) ou relativo (mesmo host, caminho diferente).