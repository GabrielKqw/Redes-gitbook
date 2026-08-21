# Módulo 5 — DNS

## Objetivo

DNS não é “apenas transformar nome em IP”. É uma base hierárquica e distribuída que permite localizar serviços, delegar responsabilidade e armazenar vários tipos de informação.

Pergunta-guia:

> Quando o navegador precisa de `www.exemplo.com`, quem pergunta a quem, onde entram os caches e como a resposta chega ao processo?

***

## 1. Quem participa de uma resolução

```
Aplicação
  ↓
stub resolver do sistema operacional
  ↓
recursive resolver configurado
  ↓ (se não houver cache)
root → TLD (.com) → servidor autoritativo da zona
  ↓
resposta + TTL
  ↓
cache do resolver / cache local / aplicação
```

* **Stub resolver:** componente local usado por aplicação/SO. Normalmente delega a recursão a outro resolvedor.
* **Recursive resolver:** recebe a pergunta do cliente e busca a resposta; pode usar cache ou consultar a hierarquia.
* **Root servers:** apontam quem é responsável pelo TLD.
* **TLD:** como `.com`, `.br` ou `.org`; aponta servidores da zona delegada.
* **Authoritative server:** possui dados definitivos da zona, por exemplo de `exemplo.com`.

Um servidor autoritativo responde com dados de sua zona. Um recursivo responde usando dados em cache ou pesquisando em nome do cliente. DNS é hierárquico, distribuído e baseado em zonas de autoridade. [RFC 1034](https://www.rfc-editor.org/rfc/rfc1034.html)

## 2. Uma consulta completa

Imagine que nenhum cache possui resposta para `www.exemplo.com A`.

1. O navegador pede ao SO: “qual é o IPv4 de `www.exemplo.com`?”
2. O stub resolver consulta o recursive resolver configurado por DHCP, VPN ou política local.
3. O recursive consulta um servidor root: “quem sabe sobre `.com`?”
4. O root devolve uma **delegação**: registros NS para `.com`, possivelmente com glue records.
5. O recursive consulta um servidor `.com`: “quem sabe sobre `exemplo.com`?”
6. O TLD devolve NS para a zona `exemplo.com`.
7. O recursive consulta um authoritative dessa zona: “qual é o A de `www.exemplo.com`?”
8. O authoritative devolve o registro A, por exemplo `198.51.100.20`, com TTL.
9. O recursive armazena a resposta por até o TTL e a devolve ao stub.
10. O navegador abre conexão para o IP e continua com TCP/TLS/HTTP.

Na próxima consulta, um cache pode encurtar vários passos. Cache melhora desempenho e reduz carga, mas também significa que mudança de DNS não é instantânea.

## 3. Registros DNS

| Registro | O que representa   | Uso                                          |
| -------- | ------------------ | -------------------------------------------- |
| A        | IPv4               | nome de host para IPv4                       |
| AAAA     | IPv6               | nome de host para IPv6                       |
| CNAME    | alias              | nome aponta para outro nome                  |
| MX       | destino de e-mail  | entrega SMTP                                 |
| TXT      | texto/políticas    | verificações, SPF, DKIM e outros usos        |
| NS       | servidores da zona | delegação/autoridade                         |
| SOA      | metadados da zona  | serial, timers e servidor primário           |
| PTR      | nome reverso       | IP para nome, sob `in-addr.arpa`/ `ip6.arpa` |

Um **CNAME** exige resolução adicional: o resolvedor encontra o alias e depois consulta/usa o tipo final do nome canônico. Não trate DNS como simples dicionário “domínio → IP”.

## 4. TTL, cache e DNS negativo

TTL é o tempo máximo, em segundos, que um resolvedor pode reutilizar um conjunto de registros antes de precisar consultar novamente.

Cache pode existir em vários lugares:

* navegador;
* sistema operacional;
* roteador;
* recursive resolver corporativo ou público;
* aplicação/SDK;
* CDN ou infraestrutura específica.

**DNS negativo** é o cache de resposta que afirma que um nome/tipo não existe, normalmente associado a informações da zona. Ele reduz consultas repetidas para nomes inexistentes, mas também pode fazer uma correção recém-publicada levar tempo para aparecer.

TTL menor não é “sempre melhor”: reduz tempo de propagação, mas aumenta consultas e dependência do authoritative. Planeje alterações com antecedência e reduza TTL gradualmente antes de migrações, quando apropriado.

## 5. DNS e portas

DNS tradicional usa normalmente porta 53:

* UDP 53 para consultas/respostas comuns;
* TCP 53 quando necessário, como respostas maiores, transferência de zona e certos casos de fallback.

O protocolo DNS possui formato próprio de pergunta e resposta. A aplicação geralmente não cria esses pacotes manualmente: chama a API de resolução, e o stub resolver/SO faz o restante.

## 6. DoH, DoT e DNSSEC são coisas diferentes

### DNS over TLS (DoT)

DoT cifra o canal entre o cliente/stub e o resolvedor usando TLS. Isso reduz espionagem e adulteração no caminho entre esses pontos. [RFC 7858](https://www.rfc-editor.org/info/rfc7858/)

### DNS over HTTPS (DoH)

DoH transporta consultas DNS dentro de HTTPS. Cada par pergunta-resposta é mapeado para uma troca HTTP protegida por TLS. [RFC 8484](https://www.rfc-editor.org/info/rfc8484/)

DoH/DoT protegem o **transporte da consulta**, mas não tornam automaticamente os dados DNS autênticos até a origem.

### DNSSEC

DNSSEC adiciona assinaturas aos dados DNS para fornecer autenticação de origem e integridade. Ele usa uma cadeia de confiança entre zona raiz, delegações e zona final.

DNSSEC **não fornece confidencialidade**: alguém que observe DNS tradicional ainda pode ver a consulta. Para privacidade do canal, DoH/DoT abordam outro problema. [RFC 4033](https://www.rfc-editor.org/info/rfc4033/)

## 7. Como DNS falha

| Sintoma                           | Causa possível                              | Onde investigar                  |
| --------------------------------- | ------------------------------------------- | -------------------------------- |
| nome não resolve                  | registro ausente, resolver indisponível     | `nslookup`/`dig`, logs DNS       |
| resolve IP errado                 | cache desatualizado, zona errada, split DNS | TTL, resposta authoritative      |
| só funciona dentro da empresa     | split-horizon DNS/VPN                       | DNS interno vs público           |
| app falha, mas IP direto funciona | DNS ou SNI/Host incorreto                   | resolvedor, URL e TLS            |
| consulta demora                   | resolver lento, timeout, cadeia longa       | tempo de resposta e rede         |
| DNSSEC falha                      | assinatura/cadeia inválida                  | resolvedor validador e zona      |
| DoH não segue política interna    | navegador usa resolvedor diferente          | configurações de browser/empresa |

**Split DNS** é quando o mesmo nome tem resposta interna e externa diferentes. Ele é útil, mas deve ser documentado; confusão entre as visões causa incidentes difíceis de reproduzir.

## 8. Observação prática

Use domínios públicos e ambientes próprios.

**Windows:**

```powershell
nslookup example.com
Resolve-DnsName example.com
ipconfig /displaydns
ipconfig /flushdns
```

* `nslookup` consulta DNS e mostra o resolvedor utilizado.
* `Resolve-DnsName` expõe tipos de registro e detalhes no PowerShell.
* `ipconfig /displaydns` mostra cache local.
* `ipconfig /flushdns` limpa somente o cache DNS local; não limpa o cache do resolver remoto.

**Linux:**

```bash
dig example.com A
dig example.com AAAA
dig +trace example.com
resolvectl query example.com
```

* `dig` permite escolher tipo de registro e observar TTL/resposta.
* `dig +trace` faz uma investigação iterativa didática, não representa necessariamente o caminho usado pelo seu resolvedor normal.
* `resolvectl` consulta o resolvedor integrado em sistemas que usam systemd-resolved.

## 9. Laboratório

1. Consulte A e AAAA de um domínio público.
2. Compare a primeira e a segunda consulta: o TTL diminui?
3. Use `dig +trace` para observar root, TLD e autoridade.
4. Veja qual servidor DNS seu sistema usa.
5. Em uma aplicação sua, compare acesso por nome e por IP — observando que HTTPS pode falhar por causa de certificado/SNI mesmo se a rota ao IP funcionar.

Não modifique registros DNS de domínios que você não controla.

## Resumo técnico

* DNS é uma base distribuída de nomes e registros, organizada por zonas e delegações.
* Stub resolver pede; recursive resolver resolve e usa cache; authoritative responde pelos dados da zona.
* A/AAAA, CNAME, MX, TXT, NS, PTR e SOA têm funções diferentes.
* TTL determina por quanto tempo a resposta pode ser reutilizada.
* DoH/DoT protegem a privacidade/integridade do transporte em trechos específicos; DNSSEC autentica dados, mas não cifra consultas.
* Troubleshooting deve distinguir cache local, resolver recursivo, delegação e dados autoritativos.

## Prova curta

1. Qual a diferença entre recursive resolver e authoritative server?
2. Por que a segunda consulta a um nome pode ser mais rápida?
3. Qual a diferença entre A, AAAA e CNAME?
4. DNSSEC cifra a consulta DNS? Explique.
5. Se um nome resolve internamente, mas não fora da VPN, qual arquitetura pode explicar isso?

**Pare aqui.** Envie suas respostas antes do Módulo 6.

## Fontes

* [RFC 1034 — DNS Concepts](https://www.rfc-editor.org/rfc/rfc1034.html)
* [RFC 1035 — DNS Specification](https://www.rfc-editor.org/info/rfc1035/)
* [RFC 7858 — DNS over TLS](https://www.rfc-editor.org/info/rfc7858/)
* [RFC 8484 — DNS over HTTPS](https://www.rfc-editor.org/info/rfc8484/)
* [RFC 4033 — DNSSEC](https://www.rfc-editor.org/info/rfc4033/)
