# Módulo 8 — Vulnerabilidades e laboratório virtual teórico

> **Meta:** aprender a reconhecer vulnerabilidades, avaliar risco e estruturar uma simulação segura em máquina virtual — sem atacar redes reais, sem instruções de exploração e sem transformar o laboratório em risco.

## Regra de ouro

Teste somente sistemas que são seus ou para os quais existe autorização explícita, escopo documentado e janela definida. Uma VM isolada é um ambiente de estudo; não é autorização para testar a rede do condomínio, a operadora, um site público ou qualquer máquina “que respondeu”.

Este módulo é **teórico e defensivo**. Ele ensina modelo mental, prevenção, observação e relato. Não inclui payloads, bypass, roubo de credenciais, persistência, evasão ou instruções de intrusão.

***

## Ao terminar, você deverá conseguir

* Diferenciar falha, vulnerabilidade, ameaça, exposição, risco e incidente.
* Ler os termos CVE, CWE, CVSS e advisory sem tratá-los como sinônimos.
* Relacionar vulnerabilidades a ativos, controles, evidências e impacto.
* Desenhar uma VM de estudo que não toque sua LAN nem a Internet.
* Descrever um cenário de ataque **em teoria** do ponto de vista de quem defende.
* Produzir um achado útil: evidência, impacto, prioridade e correção verificável.

***

## 1. Não é tudo “um hack”

| Termo               | Pergunta que responde                         | Exemplo defensivo                                    |
| ------------------- | --------------------------------------------- | ---------------------------------------------------- |
| **Ativo**           | O que precisa ser protegido?                  | API, banco, notebook, conta administrativa.          |
| **Ameaça**          | O que pode causar dano?                       | Erro humano, ransomware, invasor, falha elétrica.    |
| **Vulnerabilidade** | Qual fraqueza pode ser explorada?             | Serviço sem atualização, autorização ausente.        |
| **Exposição**       | Onde essa fraqueza está alcançável?           | Painel administrativo aberto à Internet.             |
| **Exploit**         | Meio de acionar uma vulnerabilidade           | Conceito de técnica/código; não é sinônimo de CVE.   |
| **Risco**           | Qual a combinação de probabilidade e impacto? | Dados de clientes acessíveis por falha de permissão. |
| **Incidente**       | O dano/violação aconteceu?                    | Conta comprometida ou dados exfiltrados.             |

Uma vulnerabilidade sem caminho de acesso, sem ativo relevante ou sem impacto pode ter prioridade menor que uma configuração banal exposta na Internet. Segurança é priorização baseada em evidência, não coleção de termos.

***

## 2. CVE, CWE, CVSS e advisory

### CVE: um identificador de vulnerabilidade publicada

Uma **CVE** normalmente identifica uma vulnerabilidade conhecida em produto/versão específicos. Ela ajuda a correlacionar fornecedor, scanner, ticket e correção. Ter uma CVE não prova que a sua empresa está vulnerável: é preciso confirmar se o componente existe, qual versão está instalada, se a configuração é afetada e se há mitigação.

### CWE: a classe do erro

**CWE** classifica o tipo de fraqueza, por exemplo validação insuficiente, controle de acesso inadequado ou uso incorreto de criptografia. Várias CVEs podem ter a mesma causa-raiz CWE.

### CVSS: medida padronizada, não decisão automática

**CVSS** descreve características de severidade técnica, como vetor de ataque, complexidade, privilégios, interação do usuário e impacto em confidencialidade/integridade/disponibilidade. Ele não conhece, sozinho, o valor do seu ativo, a exposição real ou compensações existentes.

```
Prioridade de correção =
severidade técnica
+ exposição
+ importância do ativo
+ evidência de exploração
− controles compensatórios confiáveis
```

### Advisory

Um _advisory_ é o comunicado de fornecedor, projeto ou autoridade sobre uma vulnerabilidade/atualização. Prefira fontes primárias: fornecedor do componente, projeto oficial, NVD/CVE e equipes de segurança reconhecidas. Não aplique “correções” a partir de posts aleatórios.

***

## 3. Como vulnerabilidades aparecem

Vulnerabilidades quase nunca são magia. Em geral, elas nascem de combinação entre decisão de projeto, implementação, configuração, dependência e operação.

```
Requisito mal definido
    ↓
Código/configuração inadequados
    ↓
Serviço exposto ou atualização atrasada
    ↓
Ausência de logs, limites ou revisão
    ↓
Impacto possível
```

### Categorias importantes para aprender

O OWASP Top 10 é uma referência de conscientização para riscos de aplicações web. Exemplos de categorias que merecem atenção:

| Categoria                      | O erro de fundo                                     | Defesa essencial                                      |
| ------------------------------ | --------------------------------------------------- | ----------------------------------------------------- |
| Controle de acesso quebrado    | servidor não confirma “quem pode fazer o quê”       | autorização no backend em toda ação sensível.         |
| Falhas criptográficas          | segredo/dado sensível protegido de forma inadequada | TLS, gestão de chaves, senhas com hash apropriado.    |
| Injeção                        | entrada vira instrução para outro interpretador     | validação + parametrização + menor privilégio.        |
| Design inseguro                | requisito/fluxo já nasce sem controle               | modelagem de ameaça e casos de abuso.                 |
| Configuração insegura          | padrão, serviço, debug ou permissão mal ajustados   | baseline seguro, revisão e hardening.                 |
| Componentes desatualizados     | dependência afetada sem correção                    | inventário, atualização e gestão de vulnerabilidades. |
| Falha de autenticação          | identidade/sessão mal protegida                     | MFA, rate limit, expiração e gestão segura de sessão. |
| Falha de logging/monitoramento | evento ocorre sem evidência/alerta                  | logs úteis, retenção e resposta a incidentes.         |

Isso é um mapa de riscos, não uma receita para “testar tudo”. Cada categoria deve levar a uma pergunta defensiva: **qual controle deveria impedir, detectar ou limitar este cenário?**

***

## 4. Simulação de “máquina atacante”: modelo seguro

Em um curso, o termo “máquina atacante” significa uma VM que representa a origem de um cenário simulado. Ela só deve conversar com alvos intencionalmente vulneráveis, criados para treinamento, dentro de uma rede isolada.

```
                 REDE HOST-ONLY / ISOLADA
┌─────────────────────┐        ┌──────────────────────┐
│ VM de observação    │        │ VM alvo de treinamento│
│ "cliente de teste"  │  ────  │ app/dados fictícios   │
└─────────────────────┘        └──────────────────────┘

Sem bridge para sua LAN
Sem acesso à rede da operadora
Sem portas encaminhadas
Sem dados reais
```

A VM de observação não precisa ter ferramentas ofensivas para este módulo. Ela pode apenas representar o cliente e gerar requisições benignas previstas pelo laboratório.

### Três tipos de rede de VM

| Modo de rede                 | O que faz                                                                  | Uso neste módulo                                                                   |
| ---------------------------- | -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| **Host-only / rede interna** | VMs conversam entre si e/ou com o host, sem sair para a LAN                | Preferido.                                                                         |
| **NAT**                      | VM pode sair à Internet por meio do host, mas não fica diretamente exposta | Evitar durante a prática de simulação; usar apenas para atualizar, então desligar. |
| **Bridge**                   | VM entra como outro dispositivo na LAN física                              | Não usar no laboratório teórico.                                                   |

**Isolamento mínimo:**

* snapshots antes de qualquer alteração;
* adaptador em host-only/rede interna;
* sem clipboard/arrastar-e-soltar compartilhados com dados pessoais;
* sem pastas compartilhadas;
* sem USB passthrough;
* sem credenciais reais, chaves, tokens ou backups;
* alvo com dados inteiramente fictícios;
* desligar a VM ao fim e guardar anotações, não artefatos perigosos.

O isolamento reduz risco, mas não é garantia absoluta. Atualizações do hipervisor, permissões do host e disciplina de escopo continuam importantes.

***

## 5. Cenário teórico: painel com autorização incorreta

Imagine uma aplicação de treinamento com usuários fictícios:

```
Usuário comum: pode ler o próprio perfil.
Administrador: pode ler todos os perfis e gerenciar contas.
```

O defeito teórico seria: o endpoint recebe um identificador de perfil e retorna dados sem verificar se o usuário autenticado tem permissão para aquele identificador.

### A leitura defensiva do cenário

| Etapa       | O que um defensor verifica               | Evidência esperada                                      |
| ----------- | ---------------------------------------- | ------------------------------------------------------- |
| Ativo       | quais dados o endpoint devolve?          | classificação de dados e contrato da API.               |
| Identidade  | como o usuário é autenticado?            | sessão/token válido de conta fictícia.                  |
| Autorização | onde a permissão é verificada?           | regra no backend, não somente na tela.                  |
| Exposição   | quem consegue chamar a rota?             | gateway, rota, logs e política de acesso.               |
| Detecção    | há registro da tentativa e do resultado? | log com usuário/ação/request ID sem segredos.           |
| Correção    | como a regra muda?                       | verificação por dono/tenant/papel no servidor.          |
| Regressão   | como provar que voltou a funcionar?      | teste autorizado para próprio recurso e recurso negado. |

O ponto não é “conseguir acessar dados”: é reconhecer que a proteção correta deve existir no servidor e ser testada como requisito de negócio.

***

## 6. O ciclo profissional de avaliação

O NIST recomenda planejamento, execução segura, análise e relatório. Um ciclo enxuto:

```
1. Autorização e escopo
2. Inventário e hipótese
3. Observação não destrutiva
4. Validação controlada
5. Evidência mínima necessária
6. Impacto e prioridade
7. Correção
8. Reteste e encerramento
```

### 1. Autorização e escopo

Antes de qualquer atividade, registre:

* donos do ambiente e contatos de emergência;
* sistemas, IPs, aplicações e contas fictícias permitidas;
* ações proibidas, limites de volume e janela;
* como interromper se algo sair do esperado;
* onde evidências serão guardadas e por quanto tempo.

### 2. Observação antes de mudança

Colete versões, configurações e logs permitidos. Começar pelo inventário frequentemente encontra mais valor do que “rodar ferramentas”. Perguntas úteis:

* Qual serviço está ativo e por quê?
* Ele precisa estar acessível da rede?
* Qual conta executa o processo?
* Qual dado ele trata?
* Há patch, configuração segura e backup testado?

### 3. Validação controlada

Validação não é devastar o sistema. A meta é confirmar se a hipótese é verdadeira com a menor ação possível e parar ao obter evidência suficiente. Em produção, a validação pode exigir aprovação adicional.

### 4. Relatório que ajuda a corrigir

Um achado bom contém:

```
Título: autorização ausente na rota de perfil (ambiente de treinamento)
Ativo/escopo: aplicação fictícia X, rede isolada
Evidência: resposta do teste controlado + horário + request ID
Impacto: possível leitura cruzada de dados fictícios
Causa-raiz: ausência de verificação de dono/papel no backend
Correção: centralizar política de autorização e negar por padrão
Validação: teste de regressão aprovado
Risco residual: o que ainda não foi verificado
```

Evite anexar segredos, bases de dados completas ou detalhes que facilitem abuso.

***

## 7. Prevenção e detecção: controles por camada

| Camada         | Previna                                                    | Detecte                                            |
| -------------- | ---------------------------------------------------------- | -------------------------------------------------- |
| Identidade     | MFA, senhas protegidas, sessão curta/revogável             | logins anômalos, falhas repetidas, mudança de MFA. |
| Aplicação      | validação, autorização no servidor, queries parametrizadas | erros, negações de acesso, padrões de abuso.       |
| Dependências   | SBOM/inventário, atualização, fontes confiáveis            | alertas de CVE que atingem versões usadas.         |
| Infraestrutura | menor privilégio, segmentação, patching, serviço mínimo    | mudanças de configuração, portas inesperadas.      |
| Dados          | minimização, criptografia adequada, backup testado         | acesso fora do padrão, exportações incomuns.       |
| Operação       | runbooks, revisão, resposta a incidente                    | alertas com contexto e correlação.                 |

O melhor controle é o que reduz a chance de falha **e** produz evidência quando a falha acontece.

***

## 8. MITRE ATT\&CK: uma linguagem para descrever comportamento

MITRE ATT\&CK organiza táticas pelo objetivo do adversário, como acesso inicial, execução, persistência, elevação de privilégio, descoberta, movimento lateral, coleta, exfiltração e impacto.

Use isso como linguagem de defesa:

```
Tática (por quê?) → técnica (como, em alto nível?) → controle → telemetria → resposta
```

Exemplo conceitual:

```
Objetivo: acesso a conta
Possível técnica: uso de credenciais comprometidas
Prevenção: MFA + proteção contra tentativas automatizadas
Detecção: login impossível/atípico + muitas falhas
Resposta: revogar sessões, investigar origem, redefinir credencial
```

Não é necessário executar uma técnica para aprender a defendê-la.

***

## 9. Laboratório de papel: antes de criar VMs

Preencha este roteiro como exercício teórico.

| Item                     | Resposta                                                               |
| ------------------------ | ---------------------------------------------------------------------- |
| Objetivo de aprendizagem | Ex.: verificar que autorização não pode depender da interface.         |
| Alvo                     | aplicação fictícia em VM isolada.                                      |
| Origem                   | VM cliente de teste.                                                   |
| Rede                     | host-only/rede interna, sem bridge.                                    |
| Dados                    | perfis e e-mails inventados.                                           |
| Ação permitida           | observar requisições e validar controle de acesso em cenário aprovado. |
| Ação proibida            | Internet/LAN real, força bruta, exfiltração, persistência, evasão.     |
| Evidência                | status HTTP, logs sanitizados, prints sem segredo.                     |
| Critério de parada       | evidência suficiente ou comportamento inesperado.                      |
| Correção esperada        | negar por padrão e validar permissão no backend.                       |

Se não conseguir preencher escopo, rede e critério de parada, o laboratório ainda não está pronto.

***

## Mapa mental

```
Vulnerabilidade
├─ contexto: ativo + ameaça + exposição + impacto
├─ linguagem: CVE (instância) | CWE (classe) | CVSS (severidade)
├─ causas: design | código | configuração | dependência | operação
├─ laboratório: VMs isoladas + dados fictícios + snapshots
├─ avaliação: autorização → hipótese → evidência mínima → correção → reteste
└─ defesa: prevenir + detectar + responder
```

## Prova curta — responda antes do próximo módulo

1. Diferencie CVE, CWE e CVSS em uma frase para cada.
2. Por que uma CVE com nota alta não vira automaticamente prioridade máxima?
3. Qual configuração de rede de VM é mais apropriada aqui e por quê?
4. Em um caso de autorização quebrada, por que esconder um botão na interface não é uma correção suficiente?
5. Quais três itens precisam existir antes de validar uma hipótese em laboratório?

***

## Referências

* [OWASP Top 10 — riscos de aplicações web](https://owasp.org/www-project-top-ten/)
* [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/IndexTopTen.html)
* [NIST SP 800-115 — avaliação e teste de segurança](https://csrc.nist.gov/pubs/sp/800/115/final)
* [MITRE ATT\&CK — táticas Enterprise](https://attack.mitre.org/tactics/)
