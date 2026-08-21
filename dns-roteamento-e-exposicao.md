# DNS, roteamento e exposição

## DNS é a agenda distribuída da internet

DNS transforma nomes em informações que aplicações usam. Para abrir `app.exemplo.com`, o dispositivo consulta um resolvedor, que encontra os servidores autoritativos responsáveis pela zona e devolve o registro necessário.

Registros comuns:

| Registro | Uso típico                       |
| -------- | -------------------------------- |
| A        | nome para IPv4                   |
| AAAA     | nome para IPv6                   |
| CNAME    | alias para outro nome            |
| MX       | recebimento de e-mail            |
| TXT      | verificações e políticas         |
| NS       | servidores autoritativos da zona |

A resposta pode ficar em cache por um período definido pelo TTL. Isso melhora desempenho, mas significa que mudanças de DNS podem levar tempo para se propagar.

## Riscos de DNS

Controle de DNS é controle de onde o usuário será enviado. Proteja a conta do provedor com MFA, menor privilégio e registro de alterações. Revise registros esquecidos, subdomínios de ambientes antigos e delegações que não pertencem mais à organização.

Evite confundir DNS com controle de acesso: apontar um nome para um IP não protege o serviço. O servidor, firewall e aplicação ainda devem validar origem e identidade.

## Exposição à internet

Todo serviço público deve ter um dono, uma finalidade e uma política clara. Perguntas úteis:

* Precisa ser acessível por qualquer pessoa ou apenas por uma rede/parceiro?
* A porta está aberta por necessidade atual ou por herança?
* Há autenticação, TLS, atualização e logs?
* O serviço administrativo pode ficar atrás de VPN, bastion ou controle de identidade?
* O endereço revela um ambiente de teste ou dado sensível?

A regra prática é: **o que não precisa receber tráfego externo não deve recebê-lo**.

## Fonte

[DNS — Domain Concepts and Facilities (RFC 1034)](https://www.rfc-editor.org/rfc/rfc1034.html).
