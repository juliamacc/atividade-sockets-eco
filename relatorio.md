# Relatório — Atividade Prática: Sockets TCP

**Disciplina:** Computação Distribuída
**Nome:** Julia Macedo de Castro
**Data:** 23/11/2026

---

## Nível 1 — Inspecionar o código

### 1.1 Servidor (`servidor.py`)

Descreva com suas palavras o que cada chamada abaixo faz e por que ela
é necessária:

| Chamada | O que ela faz? |
|---------|---------------|
| `socket.socket(AF_INET, SOCK_STREAM)` | Cria o socket que o servidor vai usar. `AF_INET` indica que o endereçamento é IPv4 e `SOCK_STREAM` indica TCP (comunicação orientada a conexão, confiável e com os bytes chegando em ordem). Sem ele não existe "ponta" de comunicação no sistema operacional. |
| `servidor_socket.bind((HOST, PORTA))` | Associa o socket a um endereço IP e a uma porta (`0.0.0.0:5000`). É necessário para que o SO saiba que os pacotes que chegam nessa porta devem ser entregues a esse processo, e para que o cliente saiba onde se conectar. |
| `servidor_socket.listen(5)` | Coloca o socket em modo passivo, ou seja, ele passa a esperar conexões em vez de iniciá-las. O `5` é o tamanho da fila de conexões pendentes que ainda não foram aceitas pelo `accept()`. |
| `servidor_socket.accept()` | Bloqueia até um cliente se conectar. Quando isso acontece, retorna um **novo** socket (`conn`) só para conversar com aquele cliente e o endereço dele. O socket original continua escutando novas conexões. |
| `conn.recv(TAMANHO_BUFFER)` | Lê até 1024 bytes que o cliente enviou pela conexão. Bloqueia até chegar algum dado. Se retornar `b''` significa que o cliente fechou a conexão. |
| `conn.sendall(resposta.encode("utf-8"))` | Converte a resposta para bytes e envia tudo para o cliente. Diferente do `send()`, que pode enviar só uma parte dos bytes e retorna quantos foram enviados, o `sendall()` fica repetindo o envio até mandar todos os dados (ou dá erro). |
| `conn.close()` | Fecha a conexão com aquele cliente e libera os recursos do socket. O cliente recebe o aviso de fim de conexão. O servidor continua rodando e volta para o `accept()`. |

### 1.2 Cliente (`cliente.py`)

| Chamada | O que ela faz? |
|---------|---------------|
| `socket.socket(AF_INET, SOCK_STREAM)` | Cria o socket TCP/IPv4 do cliente, igual ao do servidor. O cliente não precisa de `bind`, o SO escolhe uma porta efêmera automaticamente. |
| `cliente_socket.connect((HOST_SERVIDOR, PORTA_SERVIDOR))` | Resolve o nome `servidor` para o IP do contêiner e inicia a conexão TCP (three-way handshake) com a porta 5000. Se não tiver ninguém escutando, dá `ConnectionRefusedError`. |
| `cliente_socket.sendall(mensagem.encode("utf-8"))` | Converte a mensagem em bytes (UTF-8) e envia todos os bytes para o servidor. |
| `cliente_socket.recv(TAMANHO_BUFFER)` | Espera e lê até 1024 bytes da resposta do servidor (o eco), que depois é decodificada de volta para string. |

### 1.3 Rede e contêineres

Por que o cliente usa o hostname `"servidor"` em vez de `"localhost"`
para se conectar? O que aconteceria se usasse `"localhost"`?

> _Resposta:_ Cada contêiner tem a sua própria pilha de rede (namespace de rede), com interface e IP próprios, então `localhost` (127.0.0.1) dentro do contêiner do cliente aponta para o próprio contêiner do cliente, e não para o do servidor. Mesmo rodando no mesmo computador, eles se comportam como máquinas diferentes na rede `rede-eco`, por isso é preciso endereço (IP/hostname) e porta. O Docker tem um DNS interno que resolve o nome `servidor` para o IP do contêiner do servidor nessa rede. Se o cliente usasse `localhost`, ele tentaria conectar na porta 5000 dele mesmo, onde não tem nada escutando, e daria `ConnectionRefusedError`.

---

## Nível 2 — Modificar o servidor

Descreva as mudanças que você fez no `servidor.py` para:

1. Devolver a mensagem **em maiúsculas**:

   > _O que foi alterado e por quê:_ A linha `resposta = mensagem` virou `resposta = mensagem.upper()`. Como a mensagem já foi decodificada para `str`, dá pra usar o `upper()` direto antes de codificar de novo e enviar. Funciona com acentos também ("Olá" vira "OLÁ") porque a conversão é feita na string e não nos bytes.

2. Exibir no log o **contador de mensagens recebidas** acumulado:

   > _O que foi alterado e por quê:_ Ao receber uma mensagem o servidor faz `total_mensagens += 1` (a variável é global, por isso o `global total_mensagens` no início da função). O log de recebimento passou a mostrar o número da mensagem (`Mensagem #N recebida`) e depois do envio tem uma linha com o total acumulado. Como o servidor fica no `while True`, o contador continua somando entre conexões diferentes, só zera se o servidor reiniciar.

Após a modificação, você precisou rodar `docker compose up --build`.
Por que o `--build` é necessário aqui?

> _Resposta:_ Porque o `servidor.py` é copiado para dentro da imagem no momento do build (`COPY servidor.py .` no Dockerfile), não é montado como volume. Sem o `--build` o Compose reaproveitaria a imagem antiga que já existe, com o código antigo, e a mudança não apareceria. O `--build` força a reconstrução da imagem com o arquivo novo.

---

## Observações livres

Anote aqui qualquer comportamento inesperado que observaram, erros que
apareceram e como resolveram, ou dúvidas que ainda têm:

> - Os logs do cliente e do servidor aparecem misturados, às vezes o cliente mostra que recebeu o eco antes do servidor mostrar que enviou. Isso acontece porque são processos independentes rodando em paralelo, a ordem de impressão não segue a ordem real dos eventos. Os timestamps ajudam a entender.
> - Sobre o `depends_on`: ele só garante que o contêiner do servidor foi *iniciado* antes do cliente, não que o servidor já chegou no `listen()`. Sem ele, o cliente pode subir primeiro e o `connect()` falhar (erro de resolução de nome ou `ConnectionRefusedError`), e o cliente termina sem tentar de novo. Mesmo com o `depends_on` existe essa corrida, só funciona porque o servidor inicia muito rápido. O certo seria um healthcheck ou o cliente tentar reconectar algumas vezes.
> - O `recv(1024)` lê no máximo 1024 bytes de uma vez. Como TCP é um fluxo de bytes e não de mensagens, uma mensagem maior (ou com muitos caracteres acentuados, que ocupam 2 bytes em UTF-8) poderia chegar cortada, e um caractere poderia até ser dividido no meio, dando erro no `decode`.

---

## Dúvida para a próxima aula

Escreva **uma pergunta** que surgiu durante a atividade e que vocês
gostariam de ver respondida na aula seguinte:

> Como o servidor sabe onde termina uma mensagem e começa a outra se o TCP é um fluxo de bytes? Qual a forma mais usada na prática para delimitar mensagens (tamanho no cabeçalho, caractere delimitador, etc.)?
