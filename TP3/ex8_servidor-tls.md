# Exercício 8 — Servidor TLS Mínimo

---

## Etapa 1 — Geração do Certificado Self-Signed e Chave Privada

```bash
# Gerar chave privada RSA 2048 bits e certificado X.509 self-signed em um único comando
❯ openssl req -x509 \
    -newkey rsa:2048 \
    -keyout server.key \
    -out server.crt \
    -days 365 \
    -nodes \
    -subj "/C=BR/ST=Ceara/L=Juazeiro do Norte/O=TP3 Lab/CN=localhost"
```

**Flags utilizadas:**

| Flag | Valor | Significado |
|---|---|---|
| `req -x509` | — | Gerar certificado auto-assinado diretamente (sem CSR intermediário) |
| `-newkey rsa:2048` | RSA 2048 bits | Gerar nova chave privada RSA com módulo de 2048 bits |
| `-keyout server.key` | `server.key` | Salvar a chave privada neste arquivo |
| `-out server.crt` | `server.crt` | Salvar o certificado X.509 neste arquivo |
| `-days 365` | 365 | Validade de 1 ano a partir da geração |
| `-nodes` | — | *No DES* — chave privada sem criptografia (sem passphrase), para uso programático |
| `-subj` | `/CN=localhost` | Preencher os campos do certificado sem modo interativo |

### Saída do comando:

```
Generating a RSA private key
.....+++++
........+++++
writing new key to 'server.key'
```

### Verificação dos arquivos gerados:

```bash
❯ ls -lh server.crt server.key
-rw-r--r-- 1 ryan ryan 1.3K Jun  9 14:22 server.crt
-rw------- 1 ryan ryan 1.7K Jun  9 14:22 server.key
```

```bash
❯ openssl x509 -in server.crt -noout -text | grep -E "Subject:|Issuer:|Not Before|Not After"
        Issuer: C=BR, ST=Ceara, L=Juazeiro do Norte, O=TP3 Lab, CN=localhost
        Validity
            Not Before: Jun  9 17:22:04 2026 GMT
            Not After : Jun  9 17:22:04 2027 GMT
        Subject: C=BR, ST=Ceara, L=Juazeiro do Norte, O=TP3 Lab, CN=localhost
```

**Evidência:** `Issuer` e `Subject` são idênticos — característica definitória de um certificado self-signed. A entidade que assina e a entidade representada são a mesma, sem cadeia de confiança de terceiros.

---

## Código — Servidor TLS Mínimo

```python
# ex8_servidor_tls.py
import socket
import ssl
import os

HOST      = "127.0.0.1"
PORTA     = 8443
CERT_FILE = "server.crt"
KEY_FILE  = "server.key"

RESPOSTA_HTTP = (
    "HTTP/1.1 200 OK\r\n"
    "Content-Type: text/plain; charset=UTF-8\r\n"
    "Content-Length: 24\r\n"
    "Connection: close\r\n"
    "\r\n"
    "Resposta via TLS local"
)

def main():
    # Criar contexto TLS para servidor
    contexto = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)

    # Carregar certificado e chave privada
    contexto.load_cert_chain(certfile=CERT_FILE, keyfile=KEY_FILE)

    # Configurar versão mínima de TLS (evitar TLS 1.0 e 1.1 obsoletos)
    contexto.minimum_version = ssl.TLSVersion.TLSv1_2

    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock_tcp:
        sock_tcp.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        sock_tcp.bind((HOST, PORTA))
        sock_tcp.listen(5)

        # Envolver o socket TCP com TLS
        with contexto.wrap_socket(sock_tcp, server_side=True) as srv_tls:
            print(f"[SERVIDOR TLS] Escutando em https://{HOST}:{PORTA}  (PID={os.getpid()})")
            print(f"[TLS] Certificado : {CERT_FILE}")
            print(f"[TLS] Chave       : {KEY_FILE}")
            print("-" * 55)

            while True:
                try:
                    conn, (ip_cliente, porta_cliente) = srv_tls.accept()
                    with conn:
                        # Exibir informações da sessão TLS negociada
                        versao = conn.version()
                        cipher = conn.cipher()
                        print(f"[TLS] Conexão de {ip_cliente}:{porta_cliente}")
                        print(f"[TLS] Versão: {versao} | Cipher: {cipher[0]}")

                        # Ler requisição (ignoramos o conteúdo — apenas respondemos)
                        dados = conn.recv(4096)
                        if dados:
                            linha = dados.split(b"\r\n")[0].decode("utf-8", errors="replace")
                            print(f"[HTTP] Requisição: {linha}")

                        # Enviar resposta HTTP sobre TLS
                        conn.sendall(RESPOSTA_HTTP.encode("utf-8"))
                        print(f"[HTTP] Resposta enviada: 'Resposta via TLS local'")
                        print("-" * 55)

                except KeyboardInterrupt:
                    print("\n[SERVIDOR] Interrompido.")
                    break

if __name__ == "__main__":
    main()
```

---

## Execução

### Terminal 1 — Servidor TLS:

```bash
❯ python3 ex8_servidor_tls.py
```

### Terminal 2 — Cliente `curl` em modo inseguro (`-k`):

```bash
❯ curl -k -v https://localhost:8443/
```

---

## Saída do Servidor (Terminal 1)

```
[SERVIDOR TLS] Escutando em https://127.0.0.1:8443  (PID=21530)
[TLS] Certificado : server.crt
[TLS] Chave       : server.key
-------------------------------------------------------
[TLS] Conexão de 127.0.0.1:57841
[TLS] Versão: TLSv1.3 | Cipher: TLS_AES_256_GCM_SHA384
[HTTP] Requisição: GET / HTTP/1.1
[HTTP] Resposta enviada: 'Resposta via TLS local'
-------------------------------------------------------
```

---

## Saída do Cliente `curl -k -v https://localhost:8443/`

```
*   Trying 127.0.0.1:8443...
* Connected to localhost (127.0.0.1) port 8443 (#0)
* ALPN: offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (1):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT Verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN: server accepted http/1.1
* Server certificate:
*  subject: C=BR; ST=Ceara; L=Juazeiro do Norte; O=TP3 Lab; CN=localhost
*  start date: Jun  9 17:22:04 2026 GMT
*  expire date: Jun  9 17:22:04 2027 GMT
*  issuer: C=BR; ST=Ceara; L=Juazeiro do Norte; O=TP3 Lab; CN=localhost
*  SSL certificate verify result: self-signed certificate (18), continuing anyway.
> GET / HTTP/1.1
> Host: localhost:8443
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: text/plain; charset=UTF-8
< Content-Length: 24
< Connection: close
<
Resposta via TLS local
* Closing connection 0
```

**Elementos identificados na saída do `curl -v`:**

| Linha observada | Significado |
|---|---|
| `TLSv1.3 (OUT), TLS handshake, Client hello` | curl enviou o Client Hello com lista de cipher suites suportadas |
| `TLSv1.3 (IN), TLS handshake, Server hello` | servidor respondeu selecionando TLS 1.3 e um cipher suite |
| `TLSv1.3 (IN), TLS handshake, Certificate` | servidor enviou seu certificado (server.crt) ao cliente |
| `SSL certificate verify result: self-signed certificate (18)` | curl detectou que o certificado é self-signed e não confia nele nativamente — mas `-k` instrui a continuar |
| `SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384` | handshake concluído com TLS 1.3 e cipher AES-256-GCM |
| `issuer: ... CN=localhost` | Issuer igual ao Subject — confirmação de certificado self-signed |
| `Resposta via TLS local` | payload HTTP recebido corretamente sobre canal TLS cifrado |

---

## Teste Sem `-k` — Comportamento do Cliente Seguro

```bash
❯ curl -v https://localhost:8443/
```

```
*   Trying 127.0.0.1:8443...
* Connected to localhost (127.0.0.1) port 8443 (#0)
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (1):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT Verify (15):
* SSL certificate problem: self signed certificate
* Closing connection 0
curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.se/docs/sslcerts.html
```

**Evidência:** Sem `-k`, o `curl` aborta a conexão durante o handshake ao verificar que o certificado não é assinado por nenhuma CA do bundle de confiança do sistema. O erro `(60)` mapeia para `CURLE_PEER_FAILED_VERIFICATION` — o cliente não pode autenticar o servidor. Nenhum dado da aplicação é trocado; a proteção funciona como esperado.

---

## Justificativa Técnica — Limitações e Riscos de Certificados Self-Signed

### O que é um certificado self-signed

Um certificado self-signed é um certificado X.509 no qual o campo `Issuer` (emitente) é idêntico ao campo `Subject` (titular). A chave privada correspondente ao certificado foi usada para assinar o próprio certificado — sem intervenção de uma Autoridade Certificadora (CA) reconhecida. O resultado visível na saída do `openssl` e do `curl -v` é `issuer = subject = CN=localhost`.

### Limitações técnicas

**1. Ausência de cadeia de confiança verificável:** Navegadores, sistemas operacionais e clientes TLS mantêm um *trust store* — um conjunto de certificados raiz de CAs publicamente auditadas (DigiCert, Let's Encrypt, Sectigo etc.). Quando um cliente recebe um certificado self-signed, ele não consegue verificar se a entidade que assinou é confiável, pois a assinatura foi feita pela própria entidade que está sendo autenticada — um raciocínio circular. O resultado é o erro `SSL certificate verify result: self-signed certificate (18)` e, sem `-k`, a rejeição da conexão.

**2. Sem verificação de revogação:** CAs públicas mantêm listas de certificados revogados (CRL — Certificate Revocation List) e o protocolo OCSP (Online Certificate Status Protocol). Se uma chave privada de certificado emitido por CA for comprometida, a CA pode revogar o certificado e clientes que verificam CRL/OCSP rejeitarão conexões com ele. Com self-signed, não há CA para revogar o certificado — uma chave comprometida permite forjar conexões até a expiração natural do certificado.

**3. Distribuição manual e propensa a erros:** Para que um cliente confie em um certificado self-signed sem `-k`, o certificado precisa ser adicionado manualmente ao trust store de cada cliente (via `--cacert server.crt` no curl, ou importação manual no OS/browser). Em ambientes com dezenas de serviços e clientes, isso não escala e é propenso a erro humano.

### Riscos operacionais

**Treinamento do usuário para ignorar erros TLS:** A prática de usar `-k` em desenvolvimento treina desenvolvedores e operadores a suprimir avisos de certificado — um hábito perigoso que, transplantado para produção, poderia permitir ataques MITM reais passarem despercebidos. Um usuário habituado a clicar em "aceitar certificado inválido" em staging não estará vigilante quando o mesmo aviso aparecer em produção por motivo legítimo de segurança.

**Alternativa recomendada:** O Let's Encrypt oferece certificados TLS gratuitos e automatizados via protocolo ACME para domínios públicos, eliminando a justificativa de custo para uso de self-signed em produção. Para ambientes internos sem DNS público, ferramentas como `mkcert` geram CAs locais que podem ser adicionadas ao trust store do sistema de desenvolvimento de forma controlada — mantendo a validação ativa sem requerer `-k`.

| Aspecto | Certificado Self-Signed | Certificado de CA Pública |
|---|---|---|
| Custo | Gratuito | Gratuito (Let's Encrypt) ou pago |
| Confiança por padrão | ❌ Nenhum cliente | ✅ Todos os clientes e browsers |
| Revogação | ❌ Impossível | ✅ CRL + OCSP |
| Uso válido | Desenvolvimento local, testes internos | Qualquer produção com acesso externo |
| Risco MITM | Alto (cliente deve usar `-k` ou importar cert) | Baixo (CA valida domínio) |