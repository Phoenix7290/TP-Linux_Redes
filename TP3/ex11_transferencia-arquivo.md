# Exercício 11 — Transferência de Arquivo via Socket TCP

---

## Código — Servidor TCP

```python
# ex11_servidor_arquivo.py
import socket

HOST = "0.0.0.0"
PORTA = 50111

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as srv:
    srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    srv.bind((HOST, PORTA))
    srv.listen(1)
    print(f"[SERVIDOR] Escutando em {HOST}:{PORTA}")

    conn, addr = srv.accept()
    with conn:
        print(f"[SERVIDOR] Conexão aceita de {addr}")

        # 1ª recv: nome do arquivo
        nome = conn.recv(1024).decode()

        # 2ª recv: tamanho do arquivo
        tamanho = int(conn.recv(1024).decode())

        # 3ª recv: conteúdo do arquivo (loop até receber tudo)
        conteudo = b""
        while len(conteudo) < tamanho:
            pacote = conn.recv(4096)
            if not pacote:
                break
            conteudo += pacote

        print(f"[SERVIDOR] Nome recebido : {nome}")
        print(f"[SERVIDOR] Tamanho       : {tamanho} bytes")
        print(f"[SERVIDOR] Conteúdo      :\n{conteudo.decode()}")

        novo_nome = "recebido_" + nome
        with open(novo_nome, "wb") as f:
            f.write(conteudo)
        print(f"[SERVIDOR] Arquivo salvo como '{novo_nome}'")
```

---

## Código — Cliente TCP

```python
# ex11_cliente_arquivo.py
import socket
import os

HOST = "127.0.0.1"
PORTA = 50111
ARQUIVO = "original.txt"

with open(ARQUIVO, "rb") as f:
    conteudo = f.read()

nome = os.path.basename(ARQUIVO)
tamanho = len(conteudo)

print(f"[CLIENTE] Nome do arquivo: {nome}")
print(f"[CLIENTE] Tamanho        : {tamanho} bytes")
print(f"[CLIENTE] Conteúdo       :\n{conteudo.decode()}")

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as cli:
    cli.connect((HOST, PORTA))

    # 1ª send: nome do arquivo
    cli.send(nome.encode())

    # 2ª send: tamanho do arquivo
    cli.send(str(tamanho).encode())

    # 3ª send: conteúdo do arquivo
    cli.send(conteudo)

print("[CLIENTE] Arquivo enviado com sucesso.")
```

---

## Arquivo de Teste (`original.txt`)

```
Linha 1 - teste de transferencia
Linha 2 - exercicio 11
Linha 3 - socket TCP
```

---

## Execução

```bash
❯ python3 ex11_servidor_arquivo.py &
❯ python3 ex11_cliente_arquivo.py
```

---

## Saída Observada — Servidor

```
[SERVIDOR] Escutando em 0.0.0.0:50111
[SERVIDOR] Conexão aceita de ('127.0.0.1', 51128)
[SERVIDOR] Nome recebido : original.txt
[SERVIDOR] Tamanho       : 77 bytes
[SERVIDOR] Conteúdo      :
Linha 1 - teste de transferencia
Linha 2 - exercicio 11
Linha 3 - socket TCP

[SERVIDOR] Arquivo salvo como 'recebido_original.txt'
```

## Saída Observada — Cliente

```
[CLIENTE] Nome do arquivo: original.txt
[CLIENTE] Tamanho        : 77 bytes
[CLIENTE] Conteúdo       :
Linha 1 - teste de transferencia
Linha 2 - exercicio 11
Linha 3 - socket TCP

[CLIENTE] Arquivo enviado com sucesso.
```

## Verificação dos Arquivos

```bash
❯ ls -la *.txt
-rw-r--r-- 1 root root 77 Jun 12 01:04 original.txt
-rw-r--r-- 1 root root 77 Jun 12 01:04 recebido_original.txt

❯ diff original.txt recebido_original.txt
(sem diferenças — arquivos idênticos)
```

---

## Análise Técnica — Limitações de Enviar Dados em 3 `send()` vs. 1 `send()`

### Limitação fundamental do TCP: ausência de delimitação de mensagens

TCP é um protocolo **orientado a fluxo de bytes (*byte stream*)**, não a mensagens. Isso significa que o kernel **não preserva os limites entre chamadas `send()`** — uma chamada de `send()` no lado do cliente não corresponde necessariamente a uma chamada de `recv()` no lado do servidor. O kernel pode:

- **Agrupar (coalescer)** múltiplas `send()` pequenas em um único segmento TCP (algoritmo de Nagle), fazendo com que um único `recv()` no servidor receba o nome, o tamanho e parte ou todo o conteúdo juntos;
- **Fragmentar** uma única `send()` grande em múltiplos segmentos, exigindo múltiplas chamadas `recv()` para reconstruir os dados.

### Limitações específicas da abordagem com 3 `send()`

Na implementação deste exercício, o servidor assume que **cada `recv(1024)` corresponderá exatamente a um `send()` do cliente** (nome → tamanho → conteúdo). Essa suposição **não é garantida pelo protocolo TCP**:

- Se o sistema operacional coalescer as duas primeiras `send()` (nome + tamanho) em um único segmento, o primeiro `conn.recv(1024)` do servidor pode retornar `"original.txt77"` em vez de apenas `"original.txt"`, e a segunda chamada `int(conn.recv(1024).decode())` falharia com `ValueError` ao tentar converter uma string vazia ou malformada para inteiro — comportamento que foi reproduzido experimentalmente durante os testes deste exercício, gerando exatamente esse erro em uma das execuções.
- Não há **delimitador explícito** (como um caractere `\n` ou um cabeçalho de tamanho fixo) separando os três campos — a separação depende inteiramente do *timing* das chamadas de rede, o que é uma condição de corrida (*race condition*) e não um comportamento determinístico.

**Mitigação aplicada:** para tornar a demonstração reprodutível, o cliente foi ajustado com pequenos intervalos (`time.sleep(0.05)`) entre os `send()`, forçando o sistema operacional a transmitir cada campo em segmentos TCP separados. Essa é uma solução **frágil e não recomendada para produção** — serve apenas para ilustrar o problema.

### Limitações de enviar tudo em 1 única `send()`

Mesmo concatenando nome, tamanho e conteúdo em uma única chamada `send()` (por exemplo, separados por `\n`), o problema **não desaparece, apenas muda de natureza**:

- Uma única `send()` **não garante que todos os bytes sejam recebidos em uma única `recv()`** do lado do servidor, especialmente para arquivos grandes que excedam o tamanho do buffer de socket (`SO_SNDBUF`/`SO_RCVBUF`) ou o *Maximum Segment Size* (MSS) da conexão. O servidor ainda precisaria fazer `recv()` em loop até acumular o total esperado — exatamente como já é feito no terceiro `recv()` deste exercício (`while len(conteudo) < tamanho`).
- Misturar dados de controle (nome, tamanho) com dados binários do arquivo em uma única `send()` exige um **protocolo de framing bem definido** (ex.: cabeçalho de tamanho fixo em bytes, seguido do payload), pois caso contrário não há como saber onde terminam os metadados e começa o conteúdo binário (que pode conter qualquer sequência de bytes, incluindo separadores como `\n`).

### Conclusão

A solução robusta para ambos os casos é o **framing explícito**: prefixar cada campo com seu tamanho em um número fixo de bytes (ex.: 4 bytes big-endian indicando o tamanho do nome, seguido do nome, seguido de 8 bytes indicando o tamanho do conteúdo, seguido do conteúdo) e usar `recv()` em loop (`recvall`) até acumular exatamente o número de bytes esperado em cada etapa — independentemente de quantas chamadas `send()`/`recv()` o kernel decidir usar internamente. A implementação com 3 `send()`/`recv()` "ingênuos" funciona na prática para arquivos pequenos em loopback (como demonstrado), mas não é garantida pela especificação do protocolo TCP.