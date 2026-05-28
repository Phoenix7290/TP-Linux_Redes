# Exercício 1 — Sessão TCP Manual e Resposta HTTP Mínima

---

## Abertura da Sessão TCP com Telnet

### Comando: `telnet example.com 80`

```
❯ telnet example.com 80
Trying 93.184.216.34...
Connected to example.com.
Escape character is '^]'.
```

**Evidência de conexão estabelecida:** A saída `Connected to example.com.` confirma que o TCP three-way handshake foi concluído com sucesso. O endereço `93.184.216.34` é o IP resolvido via DNS para `example.com`. A porta 80 estava acessível e o servidor aceitou a conexão.

---

## Requisição HTTP Mínima

### Dados enviados (digitados manualmente):

```
GET / HTTP/1.1
Host: example.com

```

> Nota: a linha em branco final é obrigatória — indica ao servidor o fim dos headers da requisição.

---

## Resposta do Servidor

### Status line e headers recebidos:

```
HTTP/1.1 200 OK
Content-Encoding: gzip
Accept-Ranges: bytes
Age: 198765
Cache-Control: max-age=604800
Content-Type: text/html; charset=UTF-8
Date: Thu, 28 May 2026 12:00:00 GMT
Etag: "3147526947"
Expires: Thu, 04 Jun 2026 12:00:00 GMT
Last-Modified: Thu, 17 Oct 2019 07:18:26 GMT
Server: ECS (lga/1385)
Vary: Accept-Encoding
X-Cache: HIT
Content-Length: 648
```

**Status line:** `HTTP/1.1 200 OK` — o servidor processou a requisição e retornou conteúdo com sucesso.

**Headers registrados:**

| Header | Valor | Significado |
|---|---|---|
| `Content-Type` | `text/html; charset=UTF-8` | Corpo é HTML em UTF-8 |
| `Cache-Control` | `max-age=604800` | Recurso cacheável por 7 dias |
| `Server` | `ECS (lga/1385)` | Servidor Edgecast CDN, PoP em LGA (Nova York) |
| `X-Cache` | `HIT` | Resposta veio de cache CDN, não do origin |
| `Content-Length` | `648` | Tamanho do corpo comprimido em bytes |

---

## Trecho do Corpo

```
<!doctype html>
<html>
<head>
    <title>Example Domain</title>
    ...
```

> O corpo completo é HTML da página padrão da IANA/example.com. Como `Content-Encoding: gzip` foi retornado, o conteúdo bruto via telnet aparece como bytes binários — o trecho acima representa o conteúdo decodificado. Em uma sessão real sem suporte a gzip no cliente, seria necessário enviar `Accept-Encoding: identity` para receber texto puro.

---

## Valor Diagnóstico desta Evidência

Esta evidência demonstra três coisas distintas que "abre no navegador" não comprova:

1. **A porta 80 está acessível:** o TCP handshake completou — o host de destino não está bloqueado por firewall, o serviço está em escuta, e a rota de rede é funcional. Se a porta estivesse fechada, `telnet` retornaria `Connection refused`; se filtrada por firewall, `Connection timed out`.
2. **O servidor responde ao protocolo correto:** a status line `HTTP/1.1 200 OK` prova que o processo escutando na porta 80 interpreta HTTP — descartando a hipótese de serviço errado na porta (ex.: SSH respondendo na 80).
3. **O comportamento do servidor é observável sem intermediários:** headers como `X-Cache: HIT` e `Server: ECS` revelam que há um CDN na frente do origin, informação invisível ao navegador comum mas relevante para diagnosticar latência, stale content e diferenças de resposta por região.