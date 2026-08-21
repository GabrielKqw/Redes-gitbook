# Roteiro de estudo e prática segura

## Trilha sugerida

| Etapa | Tema                         | Prática segura                                   | Critério de aprendizado                  |
| ----- | ---------------------------- | ------------------------------------------------ | ---------------------------------------- |
| 1     | IP, DNS, TCP/UDP e portas    | mapear sua própria rede e serviços autorizados   | explicar o caminho de uma requisição     |
| 2     | Switch, roteador, NAT e VLAN | desenhar uma topologia simples                   | distinguir rede local de roteamento      |
| 3     | HTTP, cookies e APIs         | observar requisições da sua própria aplicação    | interpretar método, cabeçalho e status   |
| 4     | TLS e certificados           | verificar certificado de um site seu             | explicar cadeia de confiança e expiração |
| 5     | Firewall e segmentação       | escrever uma matriz de fluxos permitidos         | justificar cada regra de rede            |
| 6     | Identidade e sessão          | revisar MFA e permissões de uma conta de teste   | separar autenticação de autorização      |
| 7     | Logs, backup e incidentes    | testar restauração de backup em ambiente próprio | registrar e responder a um cenário       |

## Laboratório responsável

Pratique somente em ambiente próprio, máquinas virtuais, captura de tráfego da sua rede ou plataformas de treinamento autorizadas. Não teste portas, credenciais ou falhas de sistemas de terceiros sem permissão explícita.

## Checklist de aplicação exposta

### Rede e infraestrutura

* [ ] Todo ativo exposto tem responsável definido.
* [ ] Portas externas são mínimas e justificadas.
* [ ] Administração não está exposta sem controles fortes.
* [ ] Firewall e regras de segmentação foram revisados.
* [ ] Atualizações e inventário estão em dia.

### Aplicação e identidade

* [ ] HTTPS funciona com certificado válido.
* [ ] HTTP redireciona para HTTPS.
* [ ] Cookies de sessão usam atributos adequados.
* [ ] MFA protege contas administrativas.
* [ ] Autorização é validada em cada operação sensível.
* [ ] Logs não armazenam segredos.

### Resiliência

* [ ] Backups são restauráveis e testados.
* [ ] Há alerta para falhas críticas e login suspeito.
* [ ] Existe responsável e procedimento inicial de incidente.
* [ ] Mudanças importantes possuem plano de retorno.

## Próximo passo

Escolha um serviço real que você administra e complete o checklist com evidências: onde está hospedado, quais portas usa, quem administra, como autentica, onde guarda logs e quando o backup foi restaurado pela última vez.
