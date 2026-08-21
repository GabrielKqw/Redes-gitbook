# Defesa e boas práticas

## Segurança é redução de risco

O objetivo não é prometer que um ambiente nunca será atacado. É reduzir a chance de incidente, limitar o impacto e recuperar com evidência e rapidez.

O NIST CSF organiza esse ciclo em Governar, Identificar, Proteger, Detectar, Responder e Recuperar. Para uma equipe pequena, isso pode começar de forma simples: inventário, atualizações, MFA, backups, logs e procedimento de incidente.

## Defesa em camadas

| Camada     | Pergunta                     | Controles práticos                                        |
| ---------- | ---------------------------- | --------------------------------------------------------- |
| Identidade | Quem pode entrar?            | MFA, menor privilégio, revisão de acesso                  |
| Rede       | Que caminhos são permitidos? | firewall, segmentação, VPN/acesso administrativo restrito |
| Endpoint   | O dispositivo é confiável?   | atualizações, EDR/antivírus, criptografia de disco        |
| Aplicação  | A operação é permitida?      | validação, autorização, limitação de tentativas           |
| Dados      | O que precisa ser protegido? | classificação, cifra, backup, retenção mínima             |
| Operação   | Como percebemos o problema?  | logs, alertas, inventário e resposta a incidentes         |

## Identidade: autenticação não é autorização

**Autenticação** confirma quem é a pessoa. **Autorização** define o que ela pode fazer. Um usuário autenticado não deve ganhar acesso administrativo automaticamente.

Aplique MFA em contas administrativas e ações sensíveis, como mudar e-mail, senha, fatores de MFA, permissões ou dados de pagamento. Não use contas técnicas de banco ou infraestrutura como contas de login em interfaces públicas.

Controle tentativas repetidas de login com limitação de taxa, monitoramento e mecanismos proporcionais de bloqueio. O objetivo é desestimular tentativa automatizada sem facilitar negação de serviço contra usuários legítimos.

## Atualização e inventário

Não se atualiza o que não se conhece. Mantenha uma lista de:

* ativos expostos à internet;
* sistemas operacionais e versões;
* aplicações, bibliotecas e dependências;
* responsáveis por cada serviço;
* dados processados e localização;
* portas e fluxos autorizados.

Priorize correções que afetam ativos expostos, credenciais, acesso remoto ou dados sensíveis. Antes de atualização crítica, valide backup, janela de mudança e plano de retorno.

## Logs que realmente ajudam

Registre eventos de autenticação, falhas de login, mudança de privilégios, alterações administrativas, erros críticos, chamadas sensíveis e ações de backup/restauração. Inclua horário confiável, origem, ação, resultado e identificação mínima necessária.

Evite registrar senha, token de sessão, chave privada ou dados pessoais desnecessários. Logs são úteis para investigar, mas também precisam de controle de acesso e retenção adequada.

## Resposta inicial a incidente

1. Preserve horário, contas, sistemas e evidências.
2. Reduza impacto isolando o ativo ou bloqueando a credencial comprometida.
3. Avalie alcance por logs e inventário.
4. Corrija a causa antes de restaurar.
5. Restaure a partir de backup validado.
6. Documente o que aconteceu, a decisão tomada e o que muda no processo.

Fontes: [NIST CSF 2.0](https://csrc.nist.gov/CSRC/media/Projects/cybersecurity-framework/documents/Framework_Quick%20Start_Guide.pdf), [OWASP Authentication](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) e [OWASP MFA](https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html).
