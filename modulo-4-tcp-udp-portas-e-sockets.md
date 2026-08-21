# Módulo 4 — TCP, UDP, portas e sockets

## Objetivo

Agora que você sabe como IP e roteamento levam um pacote até um host, falta entender como o host entrega os bytes ao **processo correto**. Este módulo explica portas, sockets, TCP, UDP e o que o kernel mantém para uma conexão.

Pergunta-guia:

> Quando um programa executa algo equivalente a `connect("142.250.x.x", 443)`, o que o sistema operacional faz e como ele sabe para qual processo entregar a resposta?

***

## 1. Transporte: host não é processo

IP entrega um pacote a uma interface/host. Um servidor pode ter navegador, banco, proxy, SSH e dezenas de aplicações no mesmo IP. A camada de transporte acrescenta **portas** para que o kernel escolha o destino lógico.

```
IP destino:    203.0.113.10
Porta destino: 443
Protocolo:     TCP
                 ↓
kernel procura o socket que corresponde a esse tráfego
                 ↓
processo servidor recebe bytes
```

Uma porta é um número de 16 bits. Ela identifica serviço lógico dentro de um protocolo — TCP e UDP têm espaços de portas separados. Porta 443/TCP não é o mesmo objeto que 443/UDP.

* **Portas conhecidas:** convenções como 53/DNS, 80/HTTP, 443/HTTPS.
* **Portas efêmeras:** escolhidas temporariamente pelo sistema operacional para o lado cliente.
* **Porta em escuta:** socket servidor preparado para aceitar novas conexões.
* **Porta conectada:** associação concreta entre dois endpoints.

## 2. Socket, bind, listen, accept e connect

A API de sockets é a ponte entre aplicação e kernel.

### Servidor TCP

Fluxo conceitual:

```
socket() → bind() → listen() → accept() → read()/write()
```

1. **socket():** cria um objeto de comunicação no processo e uma estrutura associada no kernel.
2. **bind():** associa o socket a IP/porta local, por exemplo `0.0.0.0:443` ou `192.168.10.20:443`.
3. **listen():** transforma o socket em ponto de escuta para novos pedidos TCP.
4. **accept():** após uma conexão ser estabelecida, cria/retorna um socket específico daquela conexão.
5. **read/write:** troca bytes com aquele cliente.

O socket em escuta não é a mesma coisa que os sockets aceitos. Um servidor HTTP pode ter um único listener em 443 e milhares de sockets de clientes.

### Cliente TCP

```
socket() → connect() → read()/write() → close()
```

Ao chamar `connect(IP, 443)`, o kernel:

1. escolhe uma interface/rota;
2. reserva uma porta efêmera local;
3. cria estado da conexão;
4. envia SYN;
5. espera a resposta ou timeout;
6. informa sucesso/erro ao processo.

Exemplo conceitual:

```
cliente: 192.168.10.15:51523
servidor: 142.250.x.x:443
protocolo: TCP
```

Esse conjunto, com protocolo, é o **5-tuple**. Ele permite que vários clientes usem a mesma porta 443 no servidor sem confusão.

## 3. TCP: por que ele existe

IP entrega datagramas sem garantir ordem, entrega ou ausência de duplicidade. TCP cria um fluxo confiável de bytes entre endpoints.

TCP oferece, entre outras coisas:

* estabelecimento de conexão;
* entrega ordenada de bytes;
* confirmação (ACK);
* retransmissão de dados perdidos;
* controle de fluxo para não sobrecarregar o receptor;
* controle de congestionamento para não sobrecarregar a rede;
* encerramento ordenado.

TCP é orientado a conexão, bidirecional e usa portas para multiplexar fluxos. [RFC 9293](https://www.rfc-editor.org/info/rfc9293/)

## 4. Three-way handshake

Antes de enviar dados de aplicação, cliente e servidor sincronizam números de sequência:

```
Cliente                                      Servidor
SYN, seq=x       ───────────────────────────►
                 ◄─────────────────────────── SYN-ACK, seq=y, ack=x+1
ACK, ack=y+1     ───────────────────────────►
                  conexão estabelecida
```

* **SYN:** pedido para iniciar; carrega número inicial de sequência.
* **SYN-ACK:** servidor confirma o SYN e envia seu próprio SYN.
* **ACK:** cliente confirma o SYN do servidor.

O handshake evita tratar bytes antigos/perdidos como pertencentes a uma nova conversa e permite negociar opções. Não é autenticação de usuário nem cifra; HTTPS adiciona TLS depois de TCP no caso de HTTP/1.1 e HTTP/2.

### O que pode falhar

* **SYN sem resposta:** rota, firewall silencioso, host indisponível ou perda; normalmente resulta em timeout.
* **RST em resposta:** host alcançado, mas conexão rejeitada porque nenhuma aplicação aceita aquele destino/porta ou uma política rejeita ativamente.
* **SYN-ACK não confirmado:** resposta não volta ao cliente, comum em rota assimétrica, filtro ou perda.

## 5. Sequence Number, ACK e retransmissão

TCP trabalha com posição no fluxo de bytes, não com “mensagem 1, mensagem 2”. O número de sequência indica o primeiro byte em um segmento; o ACK normalmente informa o próximo byte esperado.

```
A envia bytes 1000–1499
B responde ACK 1500
```

Se A não recebe confirmação no tempo esperado, ele pode retransmitir. Se segmentos chegam fora de ordem, B pode armazená-los e continuar confirmando o ponto que falta.

A aplicação pode fazer uma única escrita grande e o kernel dividi-la em vários segmentos; ou várias escritas pequenas podem ser agrupadas. Por isso, servidor HTTP não deve supor que uma chamada `read()` corresponde a uma requisição completa.

## 6. Flow control e congestion control

### Flow control

O receptor informa quanto espaço ainda tem no buffer de recepção, através da janela anunciada. Isso evita que um emissor rápido ocupe toda a memória do receptor.

### Congestion control

O emissor também limita quanto pode colocar na rede sem confirmação, por meio de uma janela de congestionamento. Perdas, confirmações e tempo de ida e volta influenciam esse limite.

Esses controles são diferentes:

```
janela de recepção  → protege o receptor
janela congestionamento → protege a rede
limite efetivo = o menor dos dois
```

Implementações modernas usam algoritmos e timers do kernel. A especificação TCP requer mecanismos básicos para evitar colapso de congestionamento. [RFC 9293](https://www.rfc-editor.org/info/rfc9293/)

## 7. Encerramento, FIN, RST e TIME\_WAIT

### Encerramento normal

```
A → B: FIN
B → A: ACK
B → A: FIN, quando terminou de enviar
A → B: ACK
```

FIN significa “não enviarei mais bytes”. Cada direção pode ser encerrada separadamente.

### RST

RST interrompe uma conexão de forma abrupta. Pode surgir quando não há socket compatível, quando aplicação/kernel aborta a sessão ou por política intermediária. Não confunda RST com timeout: RST é resposta explícita; timeout é ausência de resposta útil.

### TIME\_WAIT

Após encerramento ativo, o kernel pode manter estado por um período para lidar com segmentos atrasados e evitar colisão com uma conexão nova usando o mesmo 5-tuple. Isso é normal; não “mate” TIME\_WAIT como primeira reação a um problema.

## 8. Keep-alive

TCP não garante, sozinho, que a outra aplicação continua saudável. Keep-alive TCP e health checks da aplicação servem para detectar conexões ociosas/quebradas, mas têm objetivos e temporizações diferentes.

* **TCP keep-alive:** recurso da pilha de transporte, frequentemente com padrão de tempo longo.
* **health check HTTP/app:** pergunta se o serviço realmente responde de forma adequada.
* **heartbeat de aplicação:** mensagem definida pelo protocolo da própria aplicação.

## 9. UDP: quando não há conexão

UDP envia datagramas sem handshake, garantia de ordem ou retransmissão pela camada de transporte. Ele é útil quando baixa latência, mensagens independentes ou controle próprio da aplicação fazem mais sentido.

DNS tradicional e muitas aplicações de mídia usam UDP. QUIC usa UDP como base, mas implementa confiabilidade, segurança e multiplexação em espaço de usuário.

UDP não é “sempre mais rápido” e TCP não é “sempre pesado”. A escolha depende da semântica necessária: fluxo confiável de bytes, mensagens independentes, latência, perda e controle da aplicação.

## 10. Como o kernel entrega um pacote ao processo certo

Ao receber um segmento TCP, o kernel verifica, entre outros dados:

```
protocolo = TCP
IP local/destino
porta local/destino
IP remoto/origem
porta remota/origem
estado da conexão
```

Para uma conexão existente, procura a entrada pelo 5-tuple. Para um SYN novo, procura um socket que esteja em escuta na porta/IP de destino. Depois do handshake, a conexão possui estado separado do listener.

Isso explica por que:

* dois navegadores podem acessar o mesmo site simultaneamente;
* um servidor pode atender muitos clientes em 443;
* a mesma porta pode ser usada por TCP e UDP;
* um processo não recebe automaticamente todo pacote destinado ao IP da máquina.

## 11. Observação prática

Use apenas seu ambiente.

**Windows:**

```powershell
netstat -ano -p TCP
Get-Process -Id <PID>
Test-NetConnection example.com -Port 443
```

**Linux:**

```bash
ss -tanp
ss -ltnp
curl -v https://example.com/
```

Procure estados como `LISTEN`, `ESTABLISHED`, `TIME-WAIT` e `SYN-SENT`. Eles são pistas do estado no kernel, não diagnóstico completo por si só.

No Wireshark, filtre uma conexão que você controla:

```
tcp.port == 443
tcp.flags.syn == 1
tcp.flags.reset == 1
```

Observe SYN, SYN-ACK, ACK, retransmissões, FIN e RST. Tráfego HTTPS terá o conteúdo HTTP cifrado após o handshake TLS.

## Cenário de troubleshooting

**“Connection refused”**: há uma resposta ativa; verifique porta, processo em escuta, IP de bind e firewall que rejeita.

**“Connection timeout”**: não houve resposta útil; verifique rota, firewall silencioso, host, NAT e perda.

**“Connection reset”**: conexão foi abortada; verifique logs da aplicação/proxy, política e estabilidade do peer.

## Resumo técnico

* Porta identifica serviço lógico; socket é o objeto/API de comunicação; processo executa código.
* TCP mantém estado por conexão e entrega um fluxo ordenado de bytes.
* SYN/SYN-ACK/ACK estabelecem a conexão; ACKs e números de sequência sustentam confiabilidade.
* Flow control protege receptor; congestion control protege a rede.
* FIN encerra ordenadamente; RST aborta; TIME\_WAIT é estado esperado.
* UDP entrega datagramas sem a confiabilidade de TCP.

## Exercício

Execute uma conexão HTTPS para um serviço seu e encontre, com `netstat` ou `ss`: IP/porta local, IP/porta remota, PID/processo e estado. Depois explique qual parte desse conjunto permite ao kernel distinguir sua conexão das demais.

## Prova curta

1. Por que `443` não identifica sozinho uma conexão TCP?
2. O que `accept()` retorna em relação ao socket de escuta?
3. Diferencie timeout, refused e reset.
4. Quais são as duas finalidades distintas de flow control e congestion control?
5. Por que uma chamada `read()` não representa necessariamente uma requisição HTTP inteira?

**Pare aqui.** Envie as respostas antes de continuar.
