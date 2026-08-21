# Módulo 3 — IP, sub-redes e roteamento

## Objetivo

Neste módulo você vai entender como um host decide se o destino está na rede local, quando usa o gateway e como roteadores encaminham pacotes entre redes.

A pergunta-guia é:

> Se meu computador quer enviar dados para um IP, como ele decide o próximo salto e o que cada roteador faz até a entrega?

***

## 1. IP não é “o endereço da internet”

Um endereço IP identifica uma interface no domínio de roteamento. Ele permite que hosts e roteadores decidam **para onde encaminhar um pacote**.

Um endereço IPv4 tem 32 bits. Uma forma em pontos, como `192.168.10.15`, é só uma leitura humana de quatro grupos de 8 bits.

```
192 . 168 . 10 . 15
 8      8    8    8 bits = 32 bits
```

IPv6 tem 128 bits e foi criado como sucessor de IPv4, com espaço de endereçamento muito maior e mudanças no formato do protocolo. [RFC 8200](https://www.rfc-editor.org/info/rfc8200/)

Um IP não identifica uma pessoa, um site ou um processo. O site é distinguido mais acima, por nomes, TLS/SNI e cabeçalhos HTTP; o processo é distinguido por protocolo, portas e sockets.

## 2. Prefixo, máscara e CIDR

Todo IP é interpretado junto com um **prefixo**. O prefixo indica quais bits representam a rede e quais identificam hosts dentro daquela rede.

```
192.168.10.15/24
└───────────┘ └─ 24 bits são o prefixo de rede

rede: 192.168.10.0/24
hosts possíveis: 192.168.10.1 até 192.168.10.254
broadcast IPv4: 192.168.10.255
```

`/24` equivale à máscara `255.255.255.0`. Em vez de decorar classes A/B/C, use CIDR: `/16`, `/24`, `/27` e assim por diante descrevem precisamente quantos bits pertencem ao prefixo.

CIDR permite alocar blocos de tamanhos variados e agregar rotas, reduzindo desperdício de endereços e o tamanho das tabelas globais de roteamento. [RFC 4632](https://www.rfc-editor.org/rfc/rfc4632.html)

### Exemplo de sub-rede

Para `192.168.10.0/27`:

```
/27 = 255.255.255.224
bloco = 32 endereços

192.168.10.0/27    hosts .1–.30       broadcast .31
192.168.10.32/27   hosts .33–.62      broadcast .63
192.168.10.64/27   hosts .65–.94      broadcast .95
```

Em IPv4 tradicional, endereço de rede e broadcast não são atribuídos a hosts. Não crie sub-redes apenas para “organizar números”: cada limite adiciona regras de firewall, DHCP, DNS, roteamento e troubleshooting.

## 3. Unicast, broadcast, multicast e anycast

* **Unicast:** um emissor para um destino específico; é o tráfego mais comum.
* **Broadcast IPv4:** um emissor para todos no domínio de broadcast local. Roteadores não encaminham broadcast normal entre sub-redes.
* **Multicast:** um emissor para um grupo interessado. Evita enviar uma cópia separada a cada participante quando há suporte.
* **Anycast:** um endereço associado a múltiplas interfaces; a rede entrega ao membro considerado mais próximo pela rota. IPv6 define anycast como tipo de endereço. [RFC 8200](https://www.rfc-editor.org/info/rfc8200/)

IPv6 não usa broadcast; usa multicast e Neighbor Discovery para funções que, no IPv4/Ethernet, costumam envolver ARP e broadcast.

## 4. Público, privado, NAT e CGNAT

Os blocos privados IPv4 são:

```
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

Eles podem ser usados internamente por muitas organizações, mas não são globalmente roteáveis. [RFC 1918](https://www.rfc-editor.org/info/rfc1918/)

Um roteador com **NAT** pode traduzir endereço/porta privados para um endereço público ao sair para a internet. Ele mantém uma tabela de traduções, por exemplo:

```
192.168.10.15:51523  →  198.51.100.8:40001
```

Quando a resposta chega a `198.51.100.8:40001`, o NAT consulta o estado e devolve para o host privado correspondente.

NAT ajuda a compartilhar IPv4 público, mas não é uma fronteira de segurança completa. Uma política de firewall ainda define fluxos permitidos; uma aplicação ainda precisa autenticar e autorizar.

**CGNAT** é NAT operado pelo provedor, frequentemente depois do NAT do seu roteador: NAT444. Ele pode dificultar encaminhamento de porta e acesso de entrada. O bloco `100.64.0.0/10` é reservado como espaço compartilhado para esse uso. [RFC 6598](https://www.rfc-editor.org/rfc/rfc6598.html)

## 5. Tabela de rotas: a decisão do kernel

O kernel não escolhe um caminho “porque parece certo”. Ele consulta a tabela de rotas e aplica, em geral, a regra do **prefixo mais específico**.

Exemplo:

```
Destino            Próximo salto      Interface
192.168.10.0/24    direto             Ethernet
10.20.0.0/16       192.168.10.1       Ethernet
0.0.0.0/0          192.168.10.1       Ethernet
```

Para `10.20.5.8`, a rota `10.20.0.0/16` vence a rota padrão `0.0.0.0/0`, pois é mais específica.

Para `8.8.8.8`, não há rota específica; o kernel usa a rota padrão e envia o frame ao gateway `192.168.10.1`.

### Gateway padrão

Gateway é o próximo roteador conhecido pelo host. Ele não “é a internet”; é apenas a primeira porta de saída da sua sub-rede.

```
Host A: 192.168.10.15/24
Gateway: 192.168.10.1

Destino 192.168.10.20 → direto; ARP para .20
Destino 1.1.1.1       → remoto; ARP para .1
```

Nos dois casos o pacote IP mantém o destino final. O MAC do frame inicial muda: no destino local, é o MAC do host B; no destino remoto, é o MAC do gateway.

## 6. O que faz um roteador

Quando recebe um frame, o roteador:

1. valida e remove o cabeçalho de enlace;
2. lê o pacote IP;
3. verifica se pode encaminhá-lo pela política e rota;
4. decrementa TTL (IPv4) ou Hop Limit (IPv6);
5. escolhe próximo salto;
6. resolve o endereço de enlace do próximo salto, se necessário;
7. cria um novo frame e envia pela interface de saída.

```
Host A → [frame A→gateway | pacote IP A→servidor]
Gateway → [frame gateway→ISP | mesmo pacote IP A→servidor]
```

TTL/Hop Limit evita loops infinitos. Quando chega a zero, o roteador descarta e normalmente sinaliza um erro ICMP. `tracert`/`traceroute` exploram esse comportamento com cuidado para revelar saltos.

## 7. MTU e fragmentação

**MTU** é o maior pacote que um enlace pode carregar sem fragmentação. Ethernet comum costuma trabalhar com MTU de 1500 bytes, mas o caminho completo pode conter enlaces menores.

O **Path MTU** é o menor MTU ao longo do caminho. Um pacote que cabe na LAN pode não caber em outro enlace. IPv4 possui mecanismo de fragmentação; IPv6 desloca a fragmentação para endpoints e depende de sinalização/descoberta de MTU do caminho.

Sinais típicos de problema de MTU:

* site abre parcialmente;
* VPN funciona para mensagens pequenas, mas falha em uploads;
* conexão estabelece, mas certas respostas travam;
* ICMP necessário foi bloqueado e impede descoberta de MTU.

Não resolva isso reduzindo MTU sem medir. Verifique interface, túnel/VPN, MSS TCP, ICMP e documentação do caminho.

## 8. Segurança no roteamento

A rota correta não é automaticamente confiável. Controles úteis incluem:

* filtros de entrada/saída para endereços privados ou impossíveis na borda;
* segmentação por VLAN/sub-rede e regras explícitas entre zonas;
* autenticação e auditoria para alterações em roteadores;
* proteção do plano de gerenciamento;
* monitoramento de mudanças de rota, NAT e firewall;
* uso de TLS/SSH na aplicação, pois IP por si só não cifra nem autentica o conteúdo.

BGP será estudado mais adiante: ele troca informações de alcance entre Sistemas Autônomos (ASNs) e forma parte da rota global entre provedores. Dentro de casa/empresa, sua tabela local e gateway resolvem o primeiro passo.

## 9. Laboratório seguro

Em sua própria rede, execute:

**Windows**

```powershell
ipconfig /all
route print
Test-NetConnection 1.1.1.1 -InformationLevel Detailed
tracert -d 1.1.1.1
```

**Linux**

```bash
ip addr
ip route
ip route get 1.1.1.1
traceroute -n 1.1.1.1
```

Interprete:

* qual interface possui rota padrão;
* qual é o gateway;
* qual rota foi escolhida;
* onde aparece o primeiro salto;
* se o IP da WAN é privado, público ou possivelmente CGNAT.

Não altere rotas, regras ou interfaces em uma máquina de produção durante o estudo.

## Resumo técnico

* CIDR/prefixo diz quais endereços pertencem a uma rede e permite rotas específicas.
* A tabela de rotas decide o próximo salto; a rota mais específica vence.
* Gateway é o primeiro roteador para redes remotas.
* IP público, privado, NAT e CGNAT explicam por que hosts internos conseguem sair, mas normalmente não recebem conexões diretas de entrada.
* Roteadores reencapsulam pacotes em frames novos a cada enlace e reduzem TTL/Hop Limit.
* MTU é uma propriedade do enlace; Path MTU é uma propriedade do caminho.

## Exercício

Desenhe sua rede doméstica ou de laboratório, com: host, prefixo, gateway, roteador, WAN e um destino externo. Para cada seta, indique o MAC do próximo salto e o IP final do pacote.

## Prova curta

1. Por que `192.168.10.15/24` envia diretamente para `192.168.10.20`, mas usa gateway para `10.0.0.20`?
2. O NAT muda necessariamente o destino IP de uma conexão que sai? Explique o que ele pode traduzir.
3. Qual rota vence para `10.20.5.8`: `10.20.0.0/16` ou `0.0.0.0/0`? Por quê?
4. Qual MAC vai no primeiro frame enviado a um servidor remoto?
5. Diferencie MTU de Path MTU.

**Pare aqui.** Envie as respostas antes de continuar.

## Fontes

* [RFC 1918 — Private IPv4](https://www.rfc-editor.org/info/rfc1918/)
* [RFC 4632 — CIDR](https://www.rfc-editor.org/rfc/rfc4632.html)
* [RFC 6598 — Shared Address Space/CGNAT](https://www.rfc-editor.org/rfc/rfc6598.html)
* [RFC 8200 — IPv6](https://www.rfc-editor.org/info/rfc8200/)
