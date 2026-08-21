# Módulo 9 — Ferramentas de segurança em laboratório

> **Meta:** conhecer as ferramentas normalmente associadas a testes de segurança e entender o que cada uma observa, quais evidências produz e quais riscos traz — usando somente dados fictícios e um ambiente isolado.

## Limite deste módulo

Ferramenta não concede autorização. Use apenas VMs próprias de treinamento, rede interna/host-only e dados criados para o exercício. Não use estes conceitos para enumerar, testar ou alterar redes, contas, aplicativos ou equipamentos reais.

Este módulo explica **função e leitura de resultados**. Não ensina exploração, evasão, força bruta, captura de credenciais, persistência ou ataques contra alvos.

***

## 1. Ferramentas têm papéis, não “poderes”

Uma caixa de ferramentas defensiva pode ser organizada assim:

```
Inventário ──→ "o que existe?"
Observação ──→ "o que aconteceu na rede/aplicação?"
Proxy HTTP ──→ "que conversa o cliente e o servidor tiveram?"
Logs/SIEM ──→ "que evidência ficou e o que mudou?"
Scanner ──→ "o que merece validação e correção?"
```

A resposta de uma ferramenta é uma **hipótese**. Um scanner pode errar, um pacote pode ser perdido na captura e um banner pode não refletir a versão real. O trabalho profissional é validar, entender impacto e corrigir — não apenas produzir um relatório.

***

## 2. Kit mínimo de laboratório

| Tipo                | Exemplos                           | Uso defensivo                                            |
| ------------------- | ---------------------------------- | -------------------------------------------------------- |
| Sistema operacional | Windows/Linux em VM                | Gerar tráfego controlado e inspecionar configurações.    |
| Virtualização       | Hyper-V, VirtualBox, VMware        | Isolar as VMs e trabalhar com snapshots.                 |
| Análise de pacotes  | Wireshark                          | Ver protocolos e metadados da conversa.                  |
| Cliente HTTP        | DevTools, `curl`, Postman/Insomnia | Entender requisições/respostas da aplicação de teste.    |
| Proxy web           | Burp Suite Community, OWASP ZAP    | Visualizar fluxo HTTP de um alvo autorizado.             |
| Inventário de rede  | Nmap                               | Observar serviços expostos no alvo de treinamento.       |
| Logs                | logs do SO, app e proxy            | Correlacionar ação, horário, conta fictícia e resultado. |

Comece com **DevTools + logs + Wireshark**. São suficientes para entender grande parte dos problemas de rede e web. Scanner e proxy são complementos; não substituem leitura do protocolo.

***

## 3. Ambiente modelo: duas VMs, dados fictícios

```
                 rede interna / host-only
┌────────────────────────┐     ┌─────────────────────────┐
│ VM Cliente de Estudo   │────▶│ VM App de Treinamento   │
│ dados: aluno-exemplo   │     │ dados: clientes falsos  │
│ sem bridge e sem VPN   │     │ logs locais sanitizados │
└────────────────────────┘     └─────────────────────────┘
```

### Checklist antes de abrir qualquer ferramenta

* [ ] Rede é **internal/host-only**, sem bridge para sua LAN.
* [ ] Snapshots das VMs foram criados.
* [ ] Não há pasta compartilhada, USB passthrough ou clipboard com dados reais.
* [ ] Contas, e-mails, tokens e registros são todos fictícios.
* [ ] O alvo é um app/lab deliberadamente criado para estudo.
* [ ] Há critério de parada: evidência suficiente ou comportamento inesperado.
* [ ] Você sabe restaurar o snapshot e desligar as VMs.

Se um item não estiver claro, pare no planejamento. Um laboratório seguro é parte da habilidade técnica.

***

## 4. Wireshark: observar pacotes, não “ler tudo”

O Wireshark captura e organiza pacotes. Ele é excelente para relacionar as camadas estudadas:

```
Ethernet → IP → TCP/UDP → DNS/HTTP/TLS/QUIC
```

### O que você pode aprender com uma captura fictícia

| Pergunta                       | Evidência que procura                                |
| ------------------------------ | ---------------------------------------------------- |
| O nome foi resolvido?          | consulta e resposta DNS.                             |
| A máquina alcançou o servidor? | SYN, SYN-ACK, ACK no TCP; ou tráfego QUIC.           |
| Houve retransmissão?           | indício de perda/atraso, não prova isolada de culpa. |
| A resposta é HTTP ou HTTPS?    | HTTP puro aparece legível; HTTPS cifra o conteúdo.   |
| Qual lado encerrou a conexão?  | FIN/RST e o momento na sequência.                    |

### Exemplo de leitura (inventado)

```
10:00:00.001  cliente-lab → dns-lab     consulta A app.treino.test
10:00:00.010  dns-lab → cliente-lab     resposta: 192.0.2.10
10:00:00.020  cliente-lab → app-lab     TCP SYN para 192.0.2.10:443
10:00:00.025  app-lab → cliente-lab     TCP SYN-ACK
10:00:00.030  cliente-lab → app-lab     TCP ACK
10:00:00.100  ...                       handshake TLS
10:00:00.300  ...                       dados cifrados de aplicação
```

O endereço `192.0.2.10` é reservado para documentação; não identifica uma máquina real neste exemplo.

### Cuidados

* Captura pode conter cookies, tokens, DNS interno e metadados pessoais.
* Não compartilhe arquivos `.pcap` sem revisar e sanitizar.
* HTTPS protege o conteúdo, mas não transforma uma captura em “sem informação”: horários, IPs, tamanhos e padrões ainda podem ser sensíveis.

***

## 5. DevTools e clientes HTTP: entender a aplicação

No navegador, **F12 → Network** é a ferramenta mais acessível para começar. Ela mostra o fluxo que a própria página executa.

| Aba/visão | Pergunta útil                                        |
| --------- | ---------------------------------------------------- |
| Headers   | Qual método, URL, status e política de cache?        |
| Payload   | Que campos a tela enviou?                            |
| Response  | Que representação o backend devolveu?                |
| Timing    | Onde houve espera: DNS, conexão, servidor, download? |
| Cookies   | Quais atributos de segurança foram configurados?     |

Exemplo fictício:

```http
GET /api/perfil HTTP/1.1
Host: app.treino.test
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{"nome":"Aluno Exemplo","papel":"usuario"}
```

O objetivo é verificar contrato, status e proteção de dados. Não copie cookies ou cabeçalhos de autorização de ambientes reais para qualquer ferramenta.

***

## 6. Proxy de teste: Burp Suite e OWASP ZAP

Um proxy web fica entre o cliente de laboratório e a aplicação autorizada. Ele pode tornar a conversa HTTP visível para análise.

```
Browser da VM ──→ proxy local ──→ aplicação de treinamento
                    │
                 histórico e evidência
```

### Para que serve, defensivamente?

* Conferir se a aplicação envia dados além do necessário.
* Observar status, redirecionamentos, cabeçalhos de segurança e cookies.
* Reproduzir uma requisição **benigna** contra o mesmo recurso do usuário fictício.
* Documentar o comportamento antes e depois de uma correção.

### O que não fazer

* Não enviar tráfego a site de terceiros.
* Não alterar valores para tentar acessar conta/registro de outra pessoa.
* Não ativar varreduras automatizadas fora de um alvo de treinamento com backup e autorização.
* Não guardar projeto/histórico com segredos.

A própria PortSwigger alerta que ferramentas de teste podem causar danos em alvos vulneráveis e devem ser usadas apenas com autorização e risco aceito pelo proprietário.

***

## 7. Inventário de portas e serviços: interpretando Nmap

Nmap é usado em inventário e auditoria de rede. Em um laboratório, o objetivo não é “achar uma falha”, e sim comparar o que está **deveria** estar exposto com o que está de fato exposto.

### Saída fictícia

```
Nmap scan report for alvo-lab (192.0.2.20)
Host is up.

PORT     STATE     SERVICE
22/tcp   closed    ssh
80/tcp   open      http
443/tcp  open      https
3306/tcp filtered  mysql
```

Como interpretar:

| Estado     | Significado aproximado                      | Pergunta defensiva                              |
| ---------- | ------------------------------------------- | ----------------------------------------------- |
| `open`     | há serviço aceitando conexões               | É necessário? Está atualizado e autenticado?    |
| `closed`   | host respondeu, mas não há serviço na porta | Era esperado? Há processo parado?               |
| `filtered` | algum filtro impede concluir o estado       | Qual firewall/regra intermediária explica isso? |
| \`open     | filtered\`                                  | a ferramenta não conseguiu distinguir           |

A linha `3306/tcp filtered` não significa “falha”. Pode ser exatamente a política desejada: banco de dados inacessível a clientes comuns.

O resultado deve virar inventário:

```
Serviço: HTTPS
Dono: equipe da aplicação
Necessidade: API pública
Controle: proxy + TLS + autenticação + logs
Evidência: porta 443 esperada, rota responde conforme contrato
Ação: manter monitoramento e atualização
```

***

## 8. Ferramentas de logs e correlação

A captura diz “pacotes foram vistos”. O log da aplicação pode dizer “qual rota respondeu, por qual motivo e com qual request ID”. Juntas, as evidências ficam melhores.

Modelo de evento sanitizado:

```json
{
  "timestamp": "2026-08-21T15:30:00-03:00",
  "request_id": "lab-7f3",
  "usuario": "aluno-exemplo",
  "rota": "/api/perfil",
  "status": 200,
  "duracao_ms": 18
}
```

Não registre senha, cookie completo, token de acesso, chave privada, corpo confidencial ou número de documento “por conveniência”. Um log útil tem contexto suficiente para investigar e pouco dado sensível.

### Correlação básica

```
DevTools: 15:30:00, GET /api/perfil, 200
Proxy:    mesma requisição, cabeçalhos sanitizados
App:      request_id lab-7f3, 18 ms
Rede:     conexão já estabelecida, sem retransmissões aparentes
Conclusão: resposta normal; não há evidência de problema de transporte
```

***

## 9. De ferramenta para decisão

| Resultado observado  | Conclusão errada        | Próximo passo profissional                                           |
| -------------------- | ----------------------- | -------------------------------------------------------------------- |
| Porta aberta         | “é vulnerável”          | confirmar necessidade, versão, autenticação e exposição.             |
| Scanner alerta       | “foi invadido”          | validar versão/configuração e buscar advisory oficial.               |
| 403 no proxy         | “o app quebrou”         | pode indicar autorização funcionando; comparar com a regra esperada. |
| pacote retransmitido | “a operadora é culpada” | medir repetição, ponto de captura e outras causas.                   |
| 500 no DevTools      | “o navegador causou”    | correlacionar request ID e logs do backend.                          |

Ferramentas mostram sinais. Diagnóstico transforma sinais em hipótese, teste seguro e decisão.

***

## 10. Roteiro de uma sessão segura de 30 minutos

1. Inicie as duas VMs na rede isolada.
2. Confirme que os dados são fictícios e que não há bridge.
3. Abra uma página do alvo de treinamento na VM cliente.
4. Observe uma requisição bem-sucedida no DevTools.
5. Compare método, status, `Content-Type`, cache e duração com o log sanitizado.
6. Veja na captura apenas a sequência DNS → conexão → TLS/HTTP.
7. Registre uma pergunta de segurança e a evidência que seria necessária para respondê-la.
8. Desligue as VMs ou restaure o snapshot.

Nada nesse roteiro exige atacar o alvo. Ele cria a base para reconhecer tráfego normal e discutir controles.

***

## Mapa mental

```
Ferramentas de segurança
├─ Wireshark: pacotes e camadas
├─ DevTools/curl: conversa HTTP da aplicação
├─ Proxy: histórico HTTP de laboratório
├─ Nmap: inventário de exposição autorizada
└─ Logs: contexto, correlação e resposta
        ↓
   evidência → hipótese → correção → reteste
```

## Prova curta — responda antes do próximo módulo

1. Por que uma porta `open` não prova uma vulnerabilidade?
2. Que tipo de dado sensível pode aparecer em uma captura de rede ou histórico de proxy?
3. Em que camada você investigaria primeiro se DevTools mostra 500?
4. Para que servem snapshots e uma rede host-only no laboratório?
5. Diferencie uma observação de ferramenta de uma conclusão de segurança.

***

## Referências

* [Nmap — documentação oficial](https://nmap.org/docs.html)
* [Wireshark — documentação](https://www.wireshark.org/docs/)
* [Burp Suite — documentação](https://portswigger.net/burp/documentation/contents)
* [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
