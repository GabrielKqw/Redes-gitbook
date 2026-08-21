# Módulo 11 — Ferramentas Linux para entender a rede

> **Meta:** aprender as ferramentas Linux mais úteis para observar a própria máquina e diagnosticar rede. Todos os exemplos são fictícios; os comandos são de leitura.

## Privacidade e escopo

Use somente na sua máquina, VM ou rede autorizada. Saídas podem revelar IPs, MACs, hostnames, DNS e serviços. Nunca publique resultados completos, senhas, tokens, cookies ou chaves.

## 1. A sequência de diagnóstico

```
interface → IP → rota → gateway → DNS → serviço → logs
```

| Ferramenta        | Função                             |
| ----------------- | ---------------------------------- |
| ip addr           | interfaces e endereços             |
| ip route          | gateway e rotas                    |
| ip neigh          | vizinhos ARP/NDP locais            |
| ping              | sonda básica de conectividade      |
| resolvectl ou dig | DNS                                |
| ss                | portas, sockets e processos locais |
| curl              | resposta HTTP/HTTPS                |
| journalctl        | logs do sistema e serviços         |
| nmcli             | Wi-Fi e perfis no NetworkManager   |

## 2. ip: interfaces, IPs e rotas

### Interface e endereço

```
ip addr
```

Saída fictícia:

```
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.0.2.25/24 scope global dynamic enp0s3
    inet6 2001:db8:10::25/64 scope global
3: wlp2s0: <BROADCAST,MULTICAST> state DOWN
```

* UP e LOWER\_UP: interface habilitada e enlace ativo.
* inet: endereço IPv4; inet6: endereço IPv6.
* state DOWN: interface sem conexão naquele cenário.
* Nomes variam: eth0, enp3s0, wlan0 e wlp2s0 são comuns.

### Rota padrão

```
ip route
```

Saída fictícia:

```
default via 192.0.2.1 dev enp0s3 proto dhcp metric 100
192.0.2.0/24 dev enp0s3 proto kernel scope link src 192.0.2.25
```

A linha default via informa para qual gateway o Linux envia pacotes destinados a fora da rede local.

Para ver o caminho que o kernel escolheria, sem enviar tráfego:

```
ip route get 198.51.100.20
```

### Vizinhos locais

```
ip neigh
```

Saída fictícia:

```
192.0.2.1 dev enp0s3 lladdr 02:00:00:00:00:01 REACHABLE
192.0.2.60 dev enp0s3 lladdr 02:00:00:00:00:60 STALE
```

A tabela associa IP local e endereço de enlace. Não é inventário completo; mostra vizinhos que seu computador observou.

## 3. ping: hipótese de conectividade

```
ping -c 4 <seu-gateway>
ping -c 4 1.1.1.1
```

* -c 4 envia quatro tentativas e encerra.
* Gateway responde e IP externo falha: investigar WAN, rota ou operadora.
* IP externo responde e nome falha: investigar DNS.
* Ping não é veredito: alguns serviços bloqueiam ICMP e seguem funcionando.

## 4. DNS: resolvectl e dig

Em sistemas com systemd-resolved:

```
resolvectl status
resolvectl query example.com
```

Alternativa quando disponível:

```
dig example.com
```

Use para descobrir o DNS em uso e confirmar se um nome resolve. Antes de alterar DNS, observe a interface e a configuração.

## 5. ss: portas e processos

```
ss -tulpn
```

* -t: TCP; -u: UDP; -l: em escuta; -p: processo; -n: formato numérico.

Saída fictícia:

```
Netid State  Local Address:Port    Process
tcp   LISTEN 127.0.0.1:8080        users:((app-lab,pid=4242,fd=7))
udp   UNCONN 127.0.0.53:53         users:((systemd-resolved,pid=610,fd=12))
```

127.0.0.1:8080 só aceita conexões da própria máquina. 0.0.0.0:8080 escuta nas interfaces IPv4 disponíveis. A ferramenta informa exposição; não é prova de vulnerabilidade.

## 6. curl: HTTP e HTTPS sem navegador

```
curl -I https://example.com/
curl -v https://example.com/
```

* -I solicita cabeçalhos.
* -v mostra detalhes de conexão e cabeçalhos.

Exemplo fictício:

```
HTTP/2 200
content-type: text/html
cache-control: max-age=600
```

Use para conferir status, redirecionamento, tipo de conteúdo e cache. Não publique detalhes que contenham sessão ou URLs internas.

## 7. journalctl: logs

```
journalctl -b -p warning
journalctl -u NetworkManager --since "30 minutes ago"
```

* -b: boot atual.
* -p warning: avisos e erros.
* -u: filtra serviço.
* \--since: limita período.

Exemplo fictício:

```
Aug 21 19:05:12 host-lab NetworkManager[700]: device enp0s3: unavailable → activated
Aug 21 19:05:13 host-lab NetworkManager[700]: dhcp4: bound 192.0.2.25
```

Compare horário do log com o sintoma. Procure DHCP, quedas, DNS e processos que não iniciaram.

## 8. nmcli: Linux desktop e Wi-Fi

```
nmcli device status
nmcli connection show
```

Saída fictícia:

```
DEVICE  TYPE      STATE      CONNECTION
wlp2s0  wifi      connected  wifi-lab
enp0s3 ethernet  disconnected --
```

nmcli confirma a interface e o perfil ativo. Neste curso, use apenas consultas; ele também pode mudar configurações.

## 9. Roteiro seguro

```
ip addr
ip route
ping -c 4 <seu-gateway>
ping -c 4 1.1.1.1
resolvectl query example.com
curl -I https://example.com/
journalctl -b -p warning
```

Se a interface não tem IP, investigue adaptador e DHCP. Se não há rota padrão, investigue gateway/perfil. Se IP funciona e DNS falha, investigue DNS. Se DNS funciona, investigue HTTP, TLS ou o destino.

## Prova curta — responda antes do próximo módulo

1. Qual ferramenta mostra a rota padrão?
2. Se IP externo responde mas o nome não resolve, qual é a hipótese inicial?
3. O que significa um processo em 127.0.0.1:8080?
4. Por que ping não prova sozinho indisponibilidade?
5. Quais dados devem ser removidos antes de compartilhar saída de comandos?

## Referências

* [ip — manual Linux](https://man7.org/linux/man-pages/man8/ip.8.html)
* [ip route — tabela de roteamento](https://man7.org/linux/man-pages/man8/ip-route.8.html)
* [resolvectl — DNS](https://www.freedesktop.org/software/systemd/man/devel/resolvectl.html)
* [ss — sockets](https://man7.org/linux/man-pages/man8/ss.8.html)
* [journalctl — logs](https://man7.org/linux/man-pages/man1/journalctl.1.html)
