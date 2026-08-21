# Módulo 1 — Comunicação em uma LAN

## Objetivo do módulo

Ao terminar este módulo, você deverá conseguir explicar o caminho de uma mensagem entre dois computadores na **mesma rede local (LAN)** e identificar quem executa cada etapa: aplicação, sistema operacional, driver, NIC, switch e equipamento de destino.

A pergunta-guia será:

> O computador A, em `192.168.1.10`, precisa entregar uma mensagem a um serviço no computador B, em `192.168.1.20`. O que acontece antes, durante e depois da transmissão?

Este módulo usa uma LAN IPv4 com Ethernet como cenário principal. Wi-Fi e IPv6 aparecem como comparação. Ainda não vamos aprofundar TCP, DNS, TLS ou HTTP; eles serão módulos próprios. Aqui, o objetivo é construir o chão sobre o qual todos eles trabalham.

***

## 1. O cenário e os participantes

```
Aplicação A
    │ chama a rede
    ▼
Sistema operacional A ── Driver ── NIC A ── Switch ── NIC B ── Driver ── Sistema operacional B
                                                                        │
                                                                        ▼
                                                                   Aplicação B

A: 192.168.1.10 / MAC AA:AA:AA:AA:AA:10
B: 192.168.1.20 / MAC BB:BB:BB:BB:BB:20
Máscara: 255.255.255.0 (/24)
```

### LAN, host e interface

* **LAN (Local Area Network):** rede de alcance local, como casa, empresa, laboratório ou sala de aula. É onde dispositivos podem comunicar-se diretamente no enlace ou por equipamentos locais.
* **Host:** dispositivo que implementa IP e participa da rede. Um notebook, servidor, celular ou VM pode ser host.
* **Interface de rede:** o ponto pelo qual o host se liga a uma rede. Um notebook pode ter Ethernet, Wi-Fi, VPN e adaptadores virtuais — cada um é uma interface.
* **NIC (Network Interface Controller):** hardware (ou interface virtual equivalente) que envia e recebe sinais/quados da rede. Ela possui um endereço de enlace, normalmente um MAC em Ethernet/Wi-Fi.
* **Switch:** equipamento que encaminha frames dentro da LAN. Ele aprende por qual porta viu cada MAC e evita, quando possível, enviar o frame para todas as portas.

### Cliente e servidor

Esses termos descrevem **papéis**, não tipos fixos de máquina. O cliente inicia uma solicitação; o servidor aceita ou responde. O mesmo computador pode ser cliente em uma conexão e servidor em outra.

No nosso cenário, a aplicação A é cliente e a aplicação B é servidor. O sistema operacional não “conhece” o negócio da aplicação; ele conhece endereços, protocolos, portas, sockets, interfaces e regras de encaminhamento.

***

## 2. Os recipientes: dados, segmento, pacote e frame

A mesma informação recebe nomes diferentes conforme ganha cabeçalhos em cada camada.

```
dados da aplicação
  ↓
[ cabeçalho de transporte | dados ]                 = segmento TCP ou datagrama UDP
  ↓
[ cabeçalho IP | cabeçalho de transporte | dados ]  = pacote/datagrama IP
  ↓
[ cabeçalho Ethernet | pacote IP | FCS ]            = frame Ethernet
```

* **Dados da aplicação:** texto, JSON, arquivo, comando ou qualquer conteúdo produzido pelo programa.
* **Segmento:** nome usual quando TCP adiciona seu cabeçalho.
* **Datagrama:** termo comum para uma unidade UDP ou IP, conforme o contexto.
* **Pacote IP:** unidade encaminhada entre redes; contém IP de origem e destino.
* **Frame Ethernet:** unidade que atravessa o enlace local atual; contém MAC de origem e destino.

A distinção importa porque o IP de destino final normalmente permanece igual ao longo de uma rota, enquanto o frame é reconstruído a cada enlace. Entre A e B na mesma LAN há um único enlace lógico, então o frame vai de A para B. Se B estivesse em outra rede, A criaria o frame para o **gateway**, não para o MAC do host remoto.

***

## 3. Antes de enviar: como A decide se B é local

A não começa procurando um MAC aleatoriamente. Primeiro o kernel consulta a configuração da interface e a **tabela de rotas**.

Com `192.168.1.10/24`, a rede local é `192.168.1.0/24`. O endereço `192.168.1.20` pertence a esse mesmo prefixo. Logo, A escolhe uma rota **diretamente conectada**: não precisa entregar o pacote a um gateway.

O gateway padrão seria usado se o destino não pertencesse à rede local. Por exemplo, para alcançar `8.8.8.8`, A enviaria o pacote IP destinado a `8.8.8.8`, mas colocaria no frame Ethernet o MAC do roteador/gateway local.

### CIDR e máscara, de forma prática

`/24` significa que os primeiros 24 bits identificam a rede. A máscara equivalente é `255.255.255.0`.

```
192.168.1.10  = rede 192.168.1 | host 10
192.168.1.20  = rede 192.168.1 | host 20
```

Eles podem tentar comunicação direta no enlace. A máscara não “cria segurança”; ela ajuda o host a decidir se um destino parece local e qual rota consultar.

***

## 4. ARP: descobrir o MAC de B no IPv4

A sabe o IP de B, mas Ethernet precisa de um MAC de destino para formar um frame. Em IPv4 sobre Ethernet, essa tradução costuma ser feita por **ARP (Address Resolution Protocol)**.

O sistema operacional mantém uma tabela/cache de vizinhos: pares IP ↔ MAC já conhecidos. Se A já possui `192.168.1.20 → BB:BB:BB:BB:BB:20`, pode enviar o frame imediatamente.

Se não possui, a sequência conceitual é:

```
A → broadcast Ethernet:
"Quem possui 192.168.1.20? Informe 192.168.1.10."

B → A:
"192.168.1.20 está em BB:BB:BB:BB:BB:20."
```

O pedido ARP é enviado para o MAC de broadcast `FF:FF:FF:FF:FF:FF`. O switch o distribui pelas portas da VLAN porque todos precisam poder verificar se possuem aquele IP. Apenas B responde; A então grava a associação no cache por um período.

ARP resolve somente o **próximo salto no enlace local**. Ele não encontra MACs na internet. Se o destino é remoto, A faz ARP para o gateway, não para o host remoto.

> ARP foi criado para associar endereços de protocolo, como IP, a endereços Ethernet necessários para transmissão local. [RFC 826](https://www.rfc-editor.org/info/rfc826/)

### IPv6: Neighbor Discovery, não ARP

IPv6 usa **Neighbor Discovery (ND)**, baseado em ICMPv6, para descobrir endereços de enlace, roteadores e alcance de vizinhos. Em vez do broadcast ARP tradicional, ele usa mensagens de descoberta e multicast apropriado. O objetivo continua parecido: o host precisa saber como alcançar o próximo salto no enlace. [RFC 4861](https://www.rfc-editor.org/info/rfc4861/)

***

## 5. A transmissão: do processo A até o processo B

Agora acompanhe a mensagem por completo.

### Etapa 1 — a aplicação A pede uma comunicação

A aplicação cria ou usa uma interface de comunicação do sistema operacional. Em APIs de programação, isso aparece como um **socket**. Um socket representa um ponto de comunicação que o kernel associa a protocolo, endereços e portas.

A aplicação não fala diretamente com o cabo nem com o switch. Ela entrega bytes ao kernel por uma chamada de sistema ou API; o sistema operacional assume a montagem e envio.

### Etapa 2 — o kernel escolhe rota e interface

O kernel consulta a tabela de rotas e conclui:

```
destino 192.168.1.20 → rede diretamente conectada → interface Ethernet/Wi-Fi local
```

Ele determina que precisa do MAC de B. Se o cache de vizinhos não possuir a entrada, dispara ARP e aguarda a resolução ou falha por timeout.

### Etapa 3 — o kernel encapsula

O sistema operacional adiciona cabeçalhos conforme o protocolo usado:

```
Frame Ethernet
  MAC origem:  AA:AA:AA:AA:AA:10
  MAC destino: BB:BB:BB:BB:BB:20
  EtherType:   IPv4
  └─ Pacote IP
       IP origem:  192.168.1.10
       IP destino: 192.168.1.20
       TTL:        valor inicial do host
       └─ Unidade de transporte
            porta origem/destino e dados da aplicação
```

O **TTL** é um contador no cabeçalho IP que roteadores decrementam quando encaminham pacotes. Ele evita que um pacote fique circulando indefinidamente em caso de rota errada. Dentro de uma LAN direta, nenhum roteador precisa decrementá-lo.

### Etapa 4 — driver e NIC enviam

O driver traduz a solicitação do kernel em operações que a NIC entende. A NIC transmite o frame pelo meio físico. Em Ethernet com cabo, são sinais elétricos ou ópticos; em Wi-Fi, ondas de rádio sob regras próprias de acesso ao meio.

Antes do envio, o hardware/lógica de enlace trata detalhes como frame e verificação de integridade do enlace. O campo FCS ajuda a detectar corrupção do frame no caminho local. Isso não substitui a integridade de camadas superiores nem proteção criptográfica.

### Etapa 5 — o switch encaminha

O switch aprende que o MAC de A está na porta em que recebeu o frame. Como já aprendeu ou descobre que o MAC de B está em outra porta, encaminha o frame apenas por essa porta. Se não conhecer o MAC de destino, ele pode inundar o frame dentro da VLAN; B o aceita e os demais o descartam por não serem o destino.

O switch não decide, em regra, se a aplicação tem permissão para falar com B. Esse controle pode estar em um firewall, ACL, host firewall ou política de segmentação.

### Etapa 6 — NIC e kernel B recebem

A NIC B recebe os sinais, valida a integridade de enlace e entrega o frame ao driver. O kernel então descapsula em ordem:

1. verifica que o MAC corresponde à interface B ou a um destino aceito;
2. identifica pelo EtherType que há IPv4;
3. verifica que o IP de destino é local;
4. identifica o protocolo de transporte;
5. usa IP, portas e protocolo para localizar o socket correto;
6. entrega os bytes à fila de recepção da aplicação B.

A aplicação B lê os dados quando seu processo é agendado pelo sistema operacional. A rede e o escalonamento do processo são partes diferentes: os bytes podem já estar na fila do socket enquanto o processo ainda não recebeu tempo de CPU.

### Etapa 7 — a resposta percorre o caminho inverso

B cria uma resposta, consulta rota para A, usa o MAC de A no cache de vizinhos (ou faz ARP se necessário), monta um novo frame e envia pelo switch. O frame da resposta não é “o mesmo frame voltando”; é uma nova unidade com MACs e dados no sentido inverso.

***

## 6. Ethernet e Wi-Fi: mesma função, meios diferentes

Ethernet e Wi-Fi atendem à camada de enlace, mas não funcionam igual fisicamente.

* **Ethernet:** normalmente usa cabo e switches; a comunicação moderna é geralmente full-duplex.
* **Wi-Fi:** usa rádio e ponto de acesso. Dispositivos compartilham o meio, lidam com interferência, potência do sinal e retransmissões próprias do enlace.

Para IP, a ideia continua: há uma interface local, um próximo salto, um endereço de enlace e um pacote IP encapsulado. A observação no Wireshark, entretanto, pode diferir conforme a placa, o modo de captura e o driver.

***

## 7. Onde a comunicação pode falhar

| Sintoma                                  | Camada provável           | Perguntas para investigar                                        |
| ---------------------------------------- | ------------------------- | ---------------------------------------------------------------- |
| Interface sem IP                         | configuração              | DHCP falhou? cabo/Wi-Fi está ativo?                              |
| Não encontra MAC do vizinho              | enlace/ARP                | hosts estão na mesma VLAN? há duplicidade de IP?                 |
| IPs parecem locais, mas não comunicam    | máscara/VLAN/firewall     | a máscara está correta? há isolamento de cliente?                |
| Frame chega, aplicação não responde      | host/transporte/aplicação | serviço está em execução? porta correta? firewall local permite? |
| Funciona para um destino, não para outro | rota/política             | qual rota foi escolhida? há ACL ou regra de firewall?            |
| Quedas intermitentes                     | meio físico/Wi-Fi         | sinal, cabo, duplex, erros da NIC, saturação?                    |

Erros de rede não devem ser investigados por tentativa aleatória. Comece na camada mais próxima do sintoma e avance: interface → IP/rota → vizinho/enlace → porta/serviço → aplicação.

***

## 8. Como observar isso em Windows e Linux

Execute somente na sua própria máquina ou ambiente autorizado.

### Ver interfaces, IP e gateway

**Windows (PowerShell):**

```powershell
ipconfig /all
route print
arp -a
```

* `ipconfig /all` pede ao sistema os detalhes das interfaces: endereços, DNS, DHCP e gateway.
* `route print` mostra a tabela de rotas usada para decidir o próximo salto.
* `arp -a` mostra o cache IP ↔ MAC IPv4.

**Linux:**

```bash
ip addr
ip route
ip neigh
```

* `ip addr` mostra interfaces e endereços.
* `ip route` revela qual rota seria preferida.
* `ip neigh` mostra vizinhos ARP/ND e seus estados.

### Ver uma rota escolhida

Em Linux, use seu próprio destino de teste:

```bash
ip route get 192.168.1.20
```

A saída informa interface, endereço de origem e próximo salto que o kernel pretende usar.

### Produzir tráfego e observar o cache

1. Escolha outro dispositivo que você controla na LAN.
2. Veja o cache de vizinhos.
3. Faça um teste de conectividade autorizado.
4. Veja o cache novamente.

No Windows:

```powershell
Test-Connection 192.168.1.20 -Count 2
arp -a
```

No Linux:

```bash
ping -c 2 192.168.1.20
ip neigh
```

`ping` envia mensagens ICMP Echo Request; ele não prova que HTTP, DNS ou uma porta específica funcionam. Ele ajuda a observar alcance IP e tempo de ida e volta quando ICMP é permitido.

### Ver sockets e processos

**Windows:**

```powershell
netstat -ano
```

`netstat -ano` solicita ao sistema conexões e portas em escuta, exibindo também o PID. Para descobrir o processo do PID, use `Get-Process -Id <PID>`. A documentação da Microsoft confirma que `netstat` pode mostrar conexões TCP, portas em escuta e tabela de rotas. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netstat)

**Linux:**

```bash
ss -tulpn
```

`ss` consulta estruturas do kernel relacionadas a sockets. As opções pedem TCP/UDP, portas em escuta e associação a processos; permissões podem limitar a visualização de outros processos.

### Wireshark: o que procurar

Em uma captura da sua LAN, filtros úteis para aprendizado:

```
arp
icmp
ip.addr == 192.168.1.20
```

Observe, nesta ordem:

1. uma requisição ARP broadcast, se a entrada não estava no cache;
2. a resposta ARP de B;
3. ICMP Echo Request/Reply após o ping;
4. MACs no cabeçalho Ethernet;
5. IPs no cabeçalho IP.

Não capture ou compartilhe tráfego de terceiros. Pacotes podem conter dados privados, cookies ou credenciais.

***

## 9. Modelos OSI e TCP/IP ligados ao caso real

Os modelos são mapas mentais, não uma descrição de arquivos ou processos separados no computador.

| Caso real                        | TCP/IP        | OSI aproximado | Quem costuma executar                    |
| -------------------------------- | ------------- | -------------- | ---------------------------------------- |
| Dados da aplicação               | Aplicação     | 5–7            | processo, biblioteca, navegador/servidor |
| Portas e entrega entre processos | Transporte    | 4              | kernel e pilha TCP/UDP                   |
| Endereços e roteamento           | Internet      | 3              | kernel e roteadores                      |
| MAC e frame local                | Acesso à rede | 1–2            | NIC, driver, switch/AP                   |
| Sinal/cabo/rádio                 | Acesso à rede | 1              | hardware e meio físico                   |

Quando você abre um navegador, não existe uma “camada 7” como programa separado. Há um processo que usa APIs do SO; o kernel implementa partes da pilha; o driver e NIC lidam com enlace; switches e roteadores participam apenas das funções que lhes cabem.

***

## Resumo técnico

* Uma aplicação entrega bytes ao sistema operacional por um socket.
* O kernel escolhe rota e interface com base na tabela de rotas.
* Se o destino IPv4 está na mesma LAN, ARP converte o IP do próximo salto em MAC.
* O pacote IP é colocado dentro de um frame Ethernet destinado ao MAC do próximo salto.
* O switch encaminha frames na VLAN; B recebe, o kernel descapsula e entrega ao socket correto.
* IP, MAC, porta, processo e socket são conceitos relacionados, mas cada um identifica uma parte diferente do caminho.
* Segurança começa aqui: segmentação, firewall local, controle de acesso ao switch, proteção contra rede Wi-Fi insegura e observabilidade reduzem risco, mas não eliminam a necessidade de segurança na aplicação.

## Mapa mental

```
Processo A
  └─ socket → kernel
      ├─ rota: B é local?
      ├─ ARP: IP de B → MAC de B
      ├─ IP: origem/destino
      └─ Ethernet: MAC origem/destino
           └─ NIC → switch → NIC B
                └─ kernel B → socket → processo B
```

## Exercício prático

Em uma rede que você controla:

1. Anote IP, máscara, gateway e MAC da sua máquina.
2. Identifique um segundo dispositivo autorizado na mesma LAN.
3. Compare a rota escolhida e o cache ARP/neighbor antes e depois de um `ping`.
4. Capture apenas esse tráfego no Wireshark e localize ARP e ICMP.
5. Explique, em cinco linhas, por que o MAC usado muda quando o destino está fora da sua sub-rede.

## Perguntas de revisão

1. Por que A precisa de ARP antes de criar um frame IPv4 para B na mesma LAN?
2. Qual é a diferença entre MAC, IP, porta e socket?
3. O switch escolhe a rota IP até a internet? Por quê?
4. Para um destino remoto, qual MAC aparece no frame inicial de A?
5. O que `ping` prova e o que ele não prova?
6. Por que uma aplicação pode receber os dados depois de a NIC já tê-los entregue ao kernel?
7. Qual a diferença central entre ARP e Neighbor Discovery?
8. Cite duas causas possíveis para “IP parece correto, mas o host não responde”.

## Prova curta — responda antes de avançar

1. A possui `192.168.10.15/24` e quer alcançar `192.168.10.200`. Ela procura o MAC de quem: do host destino ou do gateway? Explique.
2. A possui `192.168.10.15/24` e quer alcançar `10.0.0.20`. Ela procura o MAC de quem? Explique.
3. Ordene: frame Ethernet, dados da aplicação, pacote IP.
4. Um switch sabe, por padrão, qual processo está escutando uma porta em B? Explique.
5. Diga uma observação que você faria no Windows ou Linux para confirmar a rota escolhida.

**Não avance ainda.** Envie suas respostas aqui; eu vou corrigi-las e só então seguiremos para o Módulo 2.
