# Módulo 6 — HTTP: requisições, respostas e cache

> **Meta:** enxergar uma página ou API como uma conversa HTTP concreta — do clique no navegador até a resposta do backend — e saber investigar erros sem adivinhar.

## Ao terminar, você deve conseguir

* Ler uma requisição e uma resposta HTTP/1.1.
* Diferenciar **método**, **URL**, **cabeçalho**, **corpo**, **status** e **cookie**.
* Explicar quando entram HTTP/1.1, HTTP/2 e HTTP/3.
* Usar DevTools e `curl` para observar uma troca HTTP de forma segura.
* Começar uma investigação de 4xx, 5xx, cache e redirecionamento.

***

## 1. O que o HTTP resolve

HTTP (_Hypertext Transfer Protocol_) é o protocolo de aplicação usado para clientes — navegador, app mobile, integração ou `curl` — pedirem uma **representação** de um recurso a um servidor. O recurso pode ser uma página, uma imagem, um pedido, ou o resultado de uma API.

Ele segue o modelo **requisição → resposta**:

```
Cliente                         Servidor
  |  GET /produtos  ---------->    |
  |  <----------  200 + JSON       |
```

HTTP é **stateless**: cada requisição deve trazer os dados necessários para ser entendida. Isso não quer dizer que a aplicação “não guarda estado”; sessão, carrinho e login podem existir em banco ou cache. Quer dizer que HTTP, por si, não presume que a requisição anterior foi da mesma pessoa.

**Ideia central:** HTTP define o significado da conversa. TCP, TLS e QUIC cuidam de partes diferentes do transporte e da proteção dessa conversa.

### Quem participa no caminho real?

```
Browser/app
  → DNS encontra o IP
  → conexão (TCP ou QUIC)
  → TLS, quando há HTTPS
  → CDN / WAF / balanceador / proxy reverso
  → aplicação (Node, Java, .NET, Python...)
  → banco, cache ou outro serviço
  ← resposta volta pelo caminho lógico
```

Um único endereço pode passar por vários intermediários. Por isso, “a API retornou 502” não prova automaticamente que o código da API é o culpado: o erro pode ter sido gerado pelo proxy ao não conseguir falar com ela.

***

## 2. Anatomia de uma requisição

Veja uma requisição HTTP/1.1 didática:

```http
POST /api/sessoes HTTP/1.1
Host: app.exemplo.test
Content-Type: application/json
Accept: application/json
User-Agent: navegador-exemplo
Cookie: tema=escuro

{"usuario":"ana","senha":"<nunca-use-senha-real-em-exemplos-ou-logs>"}
```

| Parte          | O que informa             | Por que importa                                      |
| -------------- | ------------------------- | ---------------------------------------------------- |
| `POST`         | o método                  | A intenção: enviar dados para processamento/criação. |
| `/api/sessoes` | alvo da requisição        | Qual recurso/rota será tratado.                      |
| `HTTP/1.1`     | versão de mensagem        | Nesta versão a primeira linha é textual.             |
| `Host`         | autoridade/host desejado  | Permite vários sites no mesmo IP.                    |
| Cabeçalhos     | metadados e controles     | Formato aceito, cache, autenticação, origem etc.     |
| Linha vazia    | separa cabeçalhos e corpo | Faz parte do formato HTTP/1.1.                       |
| Corpo          | conteúdo enviado          | Aqui, JSON. Nem toda requisição possui corpo.        |

Não coloque senhas, tokens de acesso, cookies de sessão ou dados de clientes em tickets, prints públicos, commits ou logs de diagnóstico. Mascarar não é só estética: reduz risco de vazamento.

### A resposta correspondente

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/sessoes/ab12
Cache-Control: no-store

{"id":"ab12","usuario":"ana"}
```

* `201 Created`: a operação criou um recurso.
* `Content-Type`: ensina ao cliente como interpretar o corpo.
* `Location`: aponta para o recurso criado, quando aplicável.
* `Cache-Control: no-store`: evita armazenar uma resposta sensível em cache.

Em HTTP/2 e HTTP/3 o significado continua sendo o mesmo, mas a mensagem não aparece “em linhas” no fio como no HTTP/1.1. Há frames e campos especiais do protocolo.

***

## 3. Métodos: intenção, segurança e repetição

Método não é autorização. Um `DELETE` só deve funcionar se a aplicação validar identidade e permissão do solicitante.

| Método    | Uso usual                        | Seguro (sem mudar estado)? |        Idempotente? |
| --------- | -------------------------------- | -------------------------: | ------------------: |
| `GET`     | Ler recurso                      |                        Sim |                 Sim |
| `HEAD`    | Ler apenas cabeçalhos            |                        Sim |                 Sim |
| `POST`    | Processar/criar ação subordinada |                        Não | Não necessariamente |
| `PUT`     | Substituir estado no alvo        |                        Não |       Em geral, sim |
| `PATCH`   | Alteração parcial                |                        Não |  Depende do desenho |
| `DELETE`  | Remover recurso                  |                        Não |       Em geral, sim |
| `OPTIONS` | Descobrir opções/negociação      |                        Sim |                 Sim |

**Idempotente** significa que repetir a mesma requisição pretendida produz o mesmo estado final no servidor — não que a resposta, os logs ou o horário sejam iguais. Isso é relevante para retries: repetir um pagamento via `POST`, por exemplo, precisa de uma estratégia explícita de idempotência; não se deve presumir que será seguro.

***

## 4. Status HTTP: o primeiro sinal para investigar

| Faixa | Significado                                     | Exemplos comuns                                |
| ----- | ----------------------------------------------- | ---------------------------------------------- |
| `1xx` | informação provisória                           | `100 Continue`                                 |
| `2xx` | sucesso                                         | `200 OK`, `201 Created`, `204 No Content`      |
| `3xx` | redirecionamento ou cache condicional           | `301`, `302`, `307`, `308`, `304 Not Modified` |
| `4xx` | problema na requisição, credencial ou permissão | `400`, `401`, `403`, `404`, `429`              |
| `5xx` | servidor/intermediário não concluiu a operação  | `500`, `502`, `503`, `504`                     |

Leitura prática:

* **400**: formato, campo ou regra de entrada inválida.
* **401**: faltou credencial válida; normalmente vem um desafio/fluxo de autenticação.
* **403**: a identidade pode ser conhecida, mas não possui autorização para aquela ação.
* **404**: rota/recurso não encontrado — ou deliberadamente ocultado.
* **429**: limite de requisições atingido; respeite `Retry-After` se fornecido.
* **500**: erro não tratado ou falha interna da aplicação.
* **502**: gateway/proxy recebeu resposta inválida ou não alcançou corretamente o upstream.
* **503**: serviço indisponível/sobrecarregado/manutenção.
* **504**: gateway aguardou o upstream e excedeu o tempo.

**Regra de diagnóstico:** guarde horário com fuso, URL sem segredos, método, status, request ID/correlation ID e a camada que respondeu. Isso é mais útil que “deu erro”.

***

## 5. Cabeçalhos que você verá todos os dias

| Cabeçalho                | Direção típica     | Função                                                            |
| ------------------------ | ------------------ | ----------------------------------------------------------------- |
| `Content-Type`           | ambos              | Formato do conteúdo: `application/json`, `text/html`, imagem etc. |
| `Accept`                 | cliente → servidor | Formatos que o cliente aceita receber.                            |
| `Authorization`          | cliente → servidor | Credencial de requisição; trate como segredo.                     |
| `Cookie` / `Set-Cookie`  | ambos              | Cliente envia cookie; servidor instrui criação/atualização.       |
| `Cache-Control`          | ambos              | Política de cache.                                                |
| `ETag` / `If-None-Match` | ambos              | Validação de versão para reuso de resposta.                       |
| `Location`               | servidor → cliente | Destino de redirecionamento ou recurso criado.                    |
| `Origin`                 | cliente → servidor | Origem em certos contextos web; importante para CORS.             |
| `Referer`                | cliente → servidor | Página de onde a navegação veio, quando enviado.                  |
| `User-Agent`             | cliente → servidor | Identificação declarada do agente; não é prova de identidade.     |
| `Content-Encoding`       | ambos              | Codificação do conteúdo, por exemplo compressão.                  |

Cabeçalhos são texto no HTTP/1.1, mas **não confie apenas na aparência**: validação de tamanho, formato e repetição de campos é responsabilidade séria de servidores e proxies.

***

## 6. Cookies, sessão e cache

### Cookies não são “o login”

Cookie é um mecanismo pelo qual o servidor pede ao navegador que guarde um pequeno valor e o envie depois em certos escopos. Uma aplicação pode usar isso para referenciar uma sessão armazenada no servidor.

Para cookies de sessão, considere:

```http
Set-Cookie: session=<valor-opaco>; Secure; HttpOnly; SameSite=Lax; Path=/
```

* `Secure`: o navegador envia apenas por HTTPS.
* `HttpOnly`: JavaScript da página não lê o cookie; ajuda a reduzir impacto de certos XSS.
* `SameSite`: limita envio em contextos entre sites; ajuda contra alguns cenários de CSRF.
* Isso não substitui validação de sessão, expiração, revogação, HTTPS, autorização nem defesa contra XSS. Autenticação e autorização serão aprofundadas em módulo próprio.

### Cache: desempenho com responsabilidade

Cache evita buscar ou recalcular a mesma representação sempre. Há cache no navegador, CDN, proxy e aplicação.

```
1. Cliente recebe: ETag: "v42"
2. Depois envia:   If-None-Match: "v42"
3. Se nada mudou:  304 Not Modified (sem novo corpo)
4. Cliente reutiliza a cópia já armazenada
```

Use `Cache-Control: no-store` em respostas que carregam dados altamente sensíveis. Já recursos estáticos versionados, como `app.8ca1.js`, normalmente podem ter cache longo. Cache errado causa a clássica situação: “publiquei, mas o usuário ainda vê a versão antiga”.

***

## 7. HTTP/1.0, 1.1, 2 e 3: o que mudou

As semânticas de métodos, status e cabeçalhos são compartilhadas pelas versões modernas. O que muda sobretudo é **como** as mensagens são transportadas e multiplexadas.

| Versão   | Ideia principal                                                     | Consequência prática                                                                            |
| -------- | ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| HTTP/1.0 | Conexões simples, muitas vezes encerradas após resposta             | Mais conexões e menos eficiência.                                                               |
| HTTP/1.1 | Conexões persistentes, `Host`, framing explícito, chunked           | Base textual ainda muito visível em ferramentas.                                                |
| HTTP/2   | Frames binários, multiplexação de streams, compressão de cabeçalhos | Diversos recursos podem avançar na mesma conexão TCP.                                           |
| HTTP/3   | HTTP sobre QUIC, que usa UDP como transporte                        | Streams independentes no transporte e melhor recuperação de mudança de rede em certos cenários. |

### Por que HTTP/2 não “elimina” todo bloqueio?

HTTP/2 multiplexa vários streams em uma conexão TCP. Porém, se um pacote TCP se perde, a entrega ordenada do TCP pode atrasar dados posteriores na conexão. HTTP/3 usa QUIC, com streams próprios, para reduzir esse bloqueio entre streams no transporte. HTTP/3 não significa “sem perda, sem latência ou automaticamente mais rápido”; depende da rede, do servidor e do cliente.

### HTTP/1.1 e chunked

Quando o tamanho do corpo ainda não é conhecido, HTTP/1.1 pode usar `Transfer-Encoding: chunked`. O corpo chega em partes. A aplicação não deve “inventar” um `Content-Length` concorrente: framing inconsistente pode causar falhas graves em proxies e servidores.

***

## 8. O percurso até o backend

Imagine `https://loja.exemplo.test/produtos?categoria=rede`.

```
1. Browser separa esquema, host, porta, caminho e query.
2. DNS resolve loja.exemplo.test.
3. Browser conecta: TCP para HTTPS tradicional ou QUIC para HTTP/3.
4. TLS autentica o servidor e cifra a conversa.
5. Proxy/WAF pode aplicar limites e encaminhar.
6. Framework escolhe middleware e rota.
7. Código valida entrada e autorização.
8. Serviço consulta cache/banco/outro serviço.
9. Backend produz status, cabeçalhos e corpo.
10. Proxy e browser recebem, registram e exibem a resposta.
```

No código do servidor, a rota não é “a internet inteira”: ela recebe uma requisição já entregue pelo sistema operacional e pelo servidor HTTP. Mesmo assim, ela precisa tratar tudo o que vem do cliente como entrada não confiável.

***

## 9. Laboratório seguro: observar, não atacar

Use apenas máquinas e serviços sob sua autorização.

### A. Inspecione uma resposta pública simples

No Windows PowerShell ou Linux:

```bash
curl -I https://example.com/
curl -v https://example.com/
```

* `-I` solicita apenas cabeçalhos (HEAD).
* `-v` mostra detalhes da negociação e cabeçalhos. Revise antes de compartilhar: pode haver cookies ou tokens em outros cenários.

### B. Suba um servidor local de arquivos

Em uma pasta sem dados sensíveis:

```bash
python -m http.server 8080
```

Em outro terminal:

```bash
curl -v http://localhost:8080/
curl -I http://localhost:8080/
```

Pare com `Ctrl+C`. Esse servidor é didático, não é para expor à rede ou usar em produção.

### C. DevTools do navegador

1. Abra **F12 → Network**.
2. Recarregue uma página sua ou de teste.
3. Clique em uma requisição.
4. Compare **Headers**, **Payload**, **Response**, **Timing** e **Status**.
5. Não cole tokens, cookies ou respostas de clientes em local público.

No Wireshark, HTTP puro pode aparecer legível. Com HTTPS, o conteúdo HTTP fica cifrado; você ainda pode observar DNS, IP, TCP/QUIC e metadados de rede, mas não deve esperar ler a URL completa ou o corpo apenas capturando pacotes.

***

## 10. Guia de falhas: do sintoma à camada

| Sintoma              | Primeiro teste                       | Hipóteses iniciais                                   |
| -------------------- | ------------------------------------ | ---------------------------------------------------- |
| Nome não resolve     | `nslookup host` ou `dig host`        | DNS, nome errado, VPN/rede.                          |
| Conexão recusada     | conferir host/porta e serviço        | Processo não escuta, firewall, porta errada.         |
| Certificado inválido | conferir horário e nome do host      | Cadeia TLS, hostname, data/hora, proxy corporativo.  |
| 404                  | conferir método e rota               | URL, prefixo de proxy, deploy, rota ausente.         |
| 401/403              | conferir identidade e permissão      | Token/cookie expirado, escopo, regra de autorização. |
| 429                  | ler `Retry-After`                    | Rate limit, loop de retry, credencial compartilhada. |
| 502/504              | checar logs do proxy e upstream      | Serviço downstream, DNS interno, timeout, conexão.   |
| resposta antiga      | olhar `Cache-Control`, `Age`, `ETag` | CDN/navegador/proxy com cache.                       |

Faça uma alteração por vez e registre o resultado. “Tentei tudo” é difícil de reproduzir; “às 14:03 BRT, GET X retornou 504 no proxy Y, request ID Z” é investigável.

***

## Mapa mental

```
HTTP
├─ Semântica: método + alvo + status
├─ Mensagem: controle + cabeçalhos + corpo
├─ Estado aplicado: cookie/sessão e cache
├─ Versões: 1.1 textual | 2 frames/TCP | 3 frames/QUIC
├─ Caminho: cliente → DNS → transporte/TLS → proxy → aplicação → dados
└─ Observação: DevTools, curl, logs correlacionados
```

## Exercício

Escolha uma página ou API que você controla e responda:

1. Qual foi o método e o status de uma requisição bem-sucedida?
2. Quais três cabeçalhos ajudam a explicar a resposta?
3. Há cache? Mostre `Cache-Control`, `ETag` ou explique por que não existe.
4. Se essa requisição retornasse 502, quais duas camadas você investigaria antes de alterar código?

## Prova curta — responda antes do Módulo 7

1. Qual é a diferença entre HTTP ser stateless e uma aplicação ter sessão?
2. Em uma resposta `304 Not Modified`, de onde vem o corpo que o usuário verá?
3. Por que `Authorization` e `Cookie` não devem aparecer em logs ou prints públicos?
4. Compare HTTP/2 e HTTP/3 em uma frase técnica.
5. Relacione: **502**, **504** e **429** a uma hipótese inicial diferente.

***

## Referências técnicas

* [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
* [RFC 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html)
* [RFC 9113 — HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html)
* [RFC 9114 — HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)
