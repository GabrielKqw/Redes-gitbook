# Módulo 2 — OSI, TCP/IP e conexão real

## Objetivo

Modelos de rede existem para separar responsabilidades. Eles não são sete programas em sequência no computador. Numa conexão real participam processo, biblioteca, kernel, driver, NIC, switch, roteador e processo remoto — cada um cuida de uma parte.

Ao fim deste módulo você deve distinguir **IP**, **porta**, **socket**, **processo**, **conexão TCP** e **requisição HTTP**.

***

## 1. Por que dividir a comunicação em camadas?

Sem camadas, cada aplicação teria de saber transmitir sinal pelo cabo, descobrir caminhos até o destino, lidar com perdas e ainda interpretar o conteúdo. A divisão cria contratos:

```
Aplicação  → "quero enviar esta mensagem"
Transporte → "quero entregar bytes entre processos"
IP         → "quero alcançar este host"
Enlace     → "quero alcançar o próximo salto nesta rede"
Físico     → "quero converter bits em sinal"
```

Isso permite que HTTP continue funcionando se a LAN trocar Ethernet por Wi‑Fi, ou se uma rota usar IPv4 em vez de IPv6. A Internet usa, conceitualmente, as camadas aplicação, transporte, internet/IP e enlace. A própria IETF ressalta que implementações reais podem cruzar camadas para otimização. [RFC 1122](https://www.rfc-editor.org/rfc/rfc1122.html)

## 2. OSI e TCP/IP na prática

| OSI                             | TCP/IP        | Exemplos              | Quem executa                     |
| ------------------------------- | ------------- | --------------------- | -------------------------------- |
| Aplicação, apresentação, sessão | Aplicação     | HTTP, DNS, TLS, JSON  | navegador, servidor, bibliotecas |
| Transporte                      | Transporte    | TCP, UDP, QUIC        | kernel ou biblioteca             |
| Rede                            | Internet      | IPv4, IPv6, ICMP      | kernel e roteadores              |
| Enlace e física                 | Acesso à rede | Ethernet, Wi‑Fi, VLAN | NIC, driver, switch/AP           |

OSI ajuda a localizar problemas. TCP/IP descreve melhor a suíte que efetivamente opera na Internet. Um firewall moderno pode olhar IP, porta, estado TCP e HTTP: ele cruza camadas.

## 3. Conceitos que não são a mesma coisa

### IP

IP identifica a **interface** de um host no domínio de roteamento. Responde: “para qual host/interface este pacote deve ir?”.

Exemplo: `203.0.113.20`.

Um IP pode atender muitos sites e muitas conexões. Ele não identifica usuário nem processo.

### Porta

Porta identifica um serviço lógico de transporte em um host. HTTPS costuma usar 443, mas isso é convenção; uma aplicação pode escutar em outra porta.

```
203.0.113.20:443
└─────────────┘ └─ serviço lógico
     host
```

Porta não é processo. É um número usado pelo kernel, junto com o protocolo e os endereços, para demultiplexar tráfego.

### Processo e thread

* **Processo:** instância de um programa em execução, com memória, permissões e identificador próprios.
* **Thread:** unidade de execução dentro de um processo.

Um processo pode ter vários sockets. Um servidor pode atender muitas conexões com várias threads ou com I/O assíncrono/event loop.

### Socket

Socket é a interface/objeto pela qual a aplicação usa a pilha de rede do sistema operacional. Ele pode estar em escuta, conectado ou operar com datagramas.

Uma comunicação TCP é identificada conceitualmente por:

```
protocolo + IP origem + porta origem + IP destino + porta destino
```

Esse 5-tuple permite que milhares de clientes diferentes falem ao mesmo servidor em `:443`.

### Conexão TCP

Conexão TCP é estado entre dois endpoints: sequência de bytes, confirmações, buffers, janelas e timers. Ela não é HTTP.

Uma conexão TCP pode carregar várias mensagens HTTP. Em HTTP/2, diversos streams coexistem sobre uma conexão. Em HTTP/3, QUIC usa UDP, mas a ideia de uma sessão/fluxos de transporte continua.

### Requisição HTTP

É uma mensagem de aplicação:

```http
GET /login HTTP/1.1
Host: example.com
```

A requisição pode ocupar muitos segmentos TCP, uma parte de um segmento ou dividir-se entre várias leituras da aplicação. “Um pacote = uma requisição” é uma simplificação errada.

## 4. Encapsulamento

Ao acessar `https://example.com/login`, os dados ganham envoltórios:

```
HTTP GET /login
  ↓ TLS cifra e autentica o canal
registros TLS
  ↓ TCP entrega um fluxo confiável de bytes
segmentos TCP
  ↓ IP endereça origem e destino
pacotes IP
  ↓ Ethernet/Wi‑Fi entrega no enlace atual
frames
  ↓
sinais no cabo, fibra ou rádio
```

No receptor, o caminho é invertido. Cada camada remove e interpreta o seu cabeçalho antes de entregar a carga à próxima.

## 5. Follow the packet: uma conexão HTTPS

### 1. Navegador

O usuário pressiona Enter. O navegador separa esquema (`https`), host (`example.com`), porta implícita (443) e caminho (`/login`). Ele consulta cache e políticas; se necessário, inicia DNS.

### 2. Sistema operacional

O navegador pede ao SO um socket. O kernel escolhe uma rota, uma interface e uma porta efêmera local. Se o destino estiver fora da LAN, a resolução de endereço de enlace será para o MAC do gateway; se estiver na LAN, será para o MAC do host destino.

### 3. Transporte e IP

O kernel prepara unidades de transporte e IP. Em TCP, os endpoints estabelecem conexão e o kernel mantém seu estado. IP decide apenas como transportar datagramas até o destino; não garante entrega, ordem ou confiabilidade de ponta a ponta.

### 4. Enlace e roteadores

A NIC envia um frame ao switch/AP ou gateway. Cada roteador remove o frame recebido, lê o IP, reduz TTL/Hop Limit, escolhe o próximo salto e monta um **novo frame** para o próximo enlace. O pacote IP permanece endereçado ao destino final, mas MACs mudam de salto em salto.

### 5. Servidor

A NIC do servidor recebe o frame; o kernel valida Ethernet, IP e transporte; então associa os dados ao socket correto. O processo web (Nginx, Node, Java, PHP etc.) lê os bytes. Se existir reverse proxy, ele pode abrir outra conexão para o backend.

### 6. Resposta

A aplicação cria a resposta. Ela percorre o processo inverso: socket → kernel → IP → enlace → roteadores → navegador. O navegador só renderiza após interpretar a resposta e buscar recursos necessários.

## 6. Quem mantém estado?

| Componente            | Estado típico                                        |
| --------------------- | ---------------------------------------------------- |
| Navegador             | cookies, cache, conexões, DOM                        |
| Kernel do host        | sockets, buffers, rotas, estado TCP                  |
| Switch                | tabela MAC por porta/VLAN                            |
| Roteador IP comum     | rota e encaminhamento; normalmente não a conexão TCP |
| NAT/firewall stateful | mapeamentos e estado de conexões                     |
| Proxy/reverse proxy   | conexões cliente/backend e política HTTP             |
| Aplicação             | sessão, usuário, regras de negócio                   |

A confiabilidade de TCP fica, normalmente, nos hosts. Roteadores convencionais encaminham pacotes individualmente; NAT e firewall stateful são exceções porque precisam guardar estado para aplicar políticas.

## 7. Segurança por camada

```
Aplicação  → autenticação, autorização, validação, logs
Transporte → portas expostas, limitação de tentativas
IP         → roteamento, ACL, anti-spoofing
Enlace     → VLAN, Wi‑Fi seguro, controle de acesso
Infra      → firewall, segmentação, atualização e monitoramento
```

TLS protege dados em trânsito, mas não impede autorização incorreta. Firewall limita fluxos, mas não impede phishing. Segmentação reduz movimento lateral, mas não substitui backup. Segurança é uma composição de controles.

## 8. Como observar no sistema operacional

Use somente sistemas e redes sob seu controle.

**Windows:**

```powershell
netstat -ano
Get-Process -Id <PID>
route print
Test-NetConnection example.com -Port 443
```

* `netstat -ano`: conexões, portas em escuta e PID.
* `Get-Process`: relaciona PID ao processo.
* `route print`: tabela de rotas.
* `Test-NetConnection`: resolução e tentativa TCP.

**Linux:**

```bash
ss -tulpn
ip route get 1.1.1.1
curl -v https://example.com/
```

* `ss` consulta sockets expostos pelo kernel.
* `ip route get` revela a rota que o kernel escolheria.
* `curl -v` mostra resolução, conexão, TLS e cabeçalhos, sem precisar inspecionar tráfego de terceiros.

## 9. Troubleshooting por camadas

| Sintoma              | Camada provável           | Primeiro teste                   |
| -------------------- | ------------------------- | -------------------------------- |
| Nome não resolve     | DNS                       | `nslookup`/ `dig`                |
| Sem rota até destino | IP/roteamento             | `route print` / `ip route get`   |
| Porta rejeitada      | serviço/firewall          | `Test-NetConnection` / `curl -v` |
| Timeout              | rota, firewall ou serviço | rota + captura/logs              |
| Erro de certificado  | TLS                       | nome do host, validade, cadeia   |
| 502/503/504          | proxy/backend             | logs de proxy e aplicação        |

`ping` usa ICMP. Ele pode indicar alcance IP, mas não prova que DNS, TCP 443, TLS ou HTTP funcionam.

## Resumo técnico

* IP identifica interface/host; porta identifica serviço lógico; socket é o objeto do SO; processo executa código; conexão TCP é estado entre endpoints; HTTP é mensagem da aplicação.
* Frames mudam a cada enlace; pacotes IP são encaminhados entre enlaces.
* Camadas organizam responsabilidades, mas implementações reais usam componentes compartilhados.
* Ao diagnosticar, classifique onde o fluxo parou antes de tentar corrigir.

## Exercício

Abra uma conexão HTTPS própria e identifique: processo, porta local, endereço remoto, protocolo e rota escolhida. Não publique tokens, cookies ou IPs internos.

## Prova curta

1. Explique a diferença entre IP, porta, socket e processo.
2. Uma requisição HTTP é sempre exatamente um pacote IP? Por quê?
3. Um roteador mantém o mesmo frame Ethernet no próximo salto? Explique.
4. Como o kernel distingue dois clientes conectados simultaneamente a um servidor HTTPS na porta 443?
5. Escolha o primeiro teste para: sem rota, porta fechada e certificado inválido.

**Pare aqui.** Responda a prova antes de avançar ao Módulo 3.

## Fontes

* [RFC 1122 — Requirements for Internet Hosts](https://www.rfc-editor.org/rfc/rfc1122.html)
* [RFC 8200 — IPv6 Specification](https://www.rfc-editor.org/info/rfc8200/)
* [RFC 9293 — TCP](https://www.rfc-editor.org/info/rfc9293/)
