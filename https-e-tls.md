# HTTPS e TLS

## HTTPS não é só “cadeado no navegador”

HTTPS é HTTP dentro de uma conexão protegida por TLS. O padrão HTTP define a semântica de requisições e respostas; o TLS acrescenta proteção criptográfica ao canal.

Quando bem configurado, TLS oferece:

* **Confidencialidade:** terceiros no caminho não conseguem ler o conteúdo.
* **Integridade:** alterações no tráfego são detectadas.
* **Autenticação do servidor:** o navegador verifica que o certificado corresponde ao serviço acessado.

O cliente autentica o servidor por padrão. A autenticação do cliente só ocorre quando há mecanismo adicional, como certificado de cliente/mTLS ou uma camada de login.

## O que acontece ao abrir uma URL HTTPS

1. O navegador consulta o DNS para descobrir o endereço do domínio.
2. Ele abre conexão com o destino e inicia a negociação TLS.
3. O servidor envia certificado e informações para a negociação.
4. O navegador verifica validade, cadeia de confiança e nome do domínio.
5. São derivadas chaves temporárias de sessão.
6. Só então o navegador envia requisições HTTP protegidas.

Se o certificado não corresponde ao domínio, expirou ou não possui cadeia confiável, o navegador deve alertar. Ignorar avisos de certificado em sistemas reais é um risco: pode mascarar um servidor impostor, uma configuração errada ou inspeção TLS não autorizada.

## Certificados e autoridade certificadora

Um certificado associa uma chave pública a uma identidade, geralmente um nome de domínio. A cadeia de confiança liga esse certificado a uma autoridade certificadora confiada pelo dispositivo.

Boas práticas operacionais:

* emitir certificados para todos os nomes usados;
* renovar antes do vencimento;
* monitorar falhas de renovação;
* proteger a chave privada como um segredo;
* limitar quem pode alterar DNS, certificados e configuração do proxy/reverse proxy.

## Cookies, sessão e HTTPS

HTTP é sem estado. Por isso aplicações usam cookies ou tokens para vincular requisições à sessão autenticada. Um identificador de sessão roubado pode permitir que outra pessoa se passe pelo usuário; trate-o como uma credencial temporária.

Para cookies de sessão:

* `Secure`: envia somente em HTTPS;
* `HttpOnly`: reduz exposição ao JavaScript do navegador;
* `SameSite`: ajuda a controlar envio em navegação entre sites;
* escopo de domínio e caminho: mantenha o menor necessário;
* expiração e revogação: encerre sessões ao sair, trocar senha ou detectar risco.

## Conteúdo misto, redirecionamento e HSTS

Uma página HTTPS não deve carregar scripts, estilos ou recursos sensíveis por HTTP. Esse **conteúdo misto** pode abrir caminho para manipulação ou vazamento.

O servidor pode redirecionar HTTP para HTTPS. Depois de validar que todos os subdomínios necessários funcionam exclusivamente com HTTPS, HSTS instrui navegadores compatíveis a preferir HTTPS em acessos futuros.

```http
Strict-Transport-Security: max-age=63072000; includeSubDomains
```

Use `includeSubDomains` e `preload` apenas após inventariar os subdomínios. Uma política ampla mal planejada pode indisponibilizar serviços legítimos que ainda dependem de HTTP.

## Limites do HTTPS

HTTPS não impede:

* vulnerabilidades de autorização;
* injeção, upload malicioso ou falhas de lógica;
* roubo de sessão no próprio dispositivo do usuário;
* vazamento de dados no servidor;
* credenciais obtidas por phishing;
* acesso legítimo, porém excessivo, de uma conta.

Por isso HTTPS é uma camada essencial, mas deve trabalhar com autenticação forte, autorização, validação de entrada, logs e proteção de dados.

## Fontes

* [IETF RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
* [OWASP — Transport Layer Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Security_Cheat_Sheet.html)
* [OWASP — Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
* [OWASP — HSTS Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Strict_Transport_Security_Cheat_Sheet.html)
