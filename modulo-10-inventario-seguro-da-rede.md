# Módulo 10 — Inventário seguro da rede

> **Meta:** aprender a identificar e organizar os dispositivos da sua própria rede de forma defensiva, começando por fontes confiáveis e dados fictícios. Inventário é a base para proteger, atualizar e investigar.

## Compromisso deste módulo

Todos os nomes, endereços, MACs, contas e dados abaixo são **fictícios**. Os blocos `192.0.2.0/24`, `198.51.100.0/24` e `203.0.113.0/24` são reservados para documentação.

Faça qualquer coleta somente em uma rede que você administra ou possui autorização explícita para administrar. Não use inventário para invadir privacidade: evite coletar conteúdo de tráfego, mensagens, senhas, históricos de navegação ou dados pessoais sem necessidade e autorização.

***

## Ao terminar, você deverá conseguir

* Explicar por que inventário é uma atividade de segurança.
* Diferenciar endereço IP, MAC, hostname, fabricante, usuário e serviço.
* Usar fontes de baixo impacto: painel do roteador, DHCP, tabela ARP e informações do próprio dispositivo.
* Montar uma planilha de inventário sem registrar segredos.
* Separar “dispositivo desconhecido” de “dispositivo malicioso”.
* Criar uma rotina simples para atualizar e proteger sua rede.

***

## 1. Você não protege o que não conhece

Inventário de ativos é a lista confiável de aparelhos, sistemas, aplicações, contas e fluxos relevantes. Sem ela, perguntas simples viram chute:

```
"Este aparelho deveria estar conectado?"
"Quem é responsável por atualizar esta câmera?"
"Este serviço aberto é necessário?"
"Quais dispositivos guardam arquivos importantes?"
```

O NIST trata a identificação e gestão de dados, pessoas, dispositivos, sistemas e instalações como parte central da função **Identify** do Cybersecurity Framework. Inventário não é burocracia: ele permite priorizar atualizações, limitar acesso, localizar um incidente e recuperar um ambiente.

### Nem toda informação tem o mesmo risco

| Dado de inventário                  | Utilidade                    |         Trate como sensível? |
| ----------------------------------- | ---------------------------- | ---------------------------: |
| Apelido do ativo: `notebook-estudo` | alta                         |                moderadamente |
| Tipo/modelo                         | alta                         |                moderadamente |
| IP privado interno                  | alta                         |            sim, não publique |
| MAC completo                        | média                        |            sim, não publique |
| Versão de firmware                  | alta                         | sim, em ambiente corporativo |
| Senha do Wi‑Fi/roteador             | não deve estar no inventário |                  **segredo** |
| Token/cookie/chave privada          | não deve estar no inventário |                  **segredo** |
| Conteúdo de arquivos/mensagens      | normalmente desnecessário    |         privado/confidencial |

Um inventário deve guardar o **mínimo necessário**. Saber que existe um NAS com backup é útil; copiar os arquivos do NAS para a planilha não é.

***

## 2. As cinco identidades de um aparelho

Um mesmo dispositivo pode aparecer de várias formas.

```
Ativo:        notebook-estudo
Hostname:     NOTE-ALUNO
IP privado:   192.0.2.25     (fictício)
MAC:          02:00:00:00:00:25  (fictício)
Papel:        computador de uso pessoal
```

| Identificador         | O que é                       |                                                 Muda? |
| --------------------- | ----------------------------- | ----------------------------------------------------: |
| Apelido de inventário | nome escolhido por você       |                        não deveria mudar sem registro |
| Hostname              | nome anunciado pelo sistema   |                                            pode mudar |
| IP                    | endereço lógico na rede       |                                   pode mudar com DHCP |
| MAC                   | endereço da interface de rede | pode variar; dispositivos usam aleatorização em Wi‑Fi |
| Porta/interface       | onde está conectado           |                                            pode mudar |
| Conta/usuário         | identidade de quem opera      |                     deve ser controlada separadamente |

**Importante:** MAC não é identidade humana. Ele identifica uma interface em um contexto de rede, pode ser aleatorizado e não prova propriedade. Não conclua “é do fulano” apenas pelo fabricante ou pelo MAC.

***

## 3. Onde identificar aparelhos sem “varrer”

Comece pelas fontes que já administram a rede. Elas reduzem impacto e falsos positivos.

### A. Painel do roteador ou da ONT

Em geral mostra:

* dispositivos conectados por cabo e Wi‑Fi;
* nome/hostname, IP atribuído e MAC;
* SSID/banda Wi‑Fi;
* tabela DHCP;
* clientes bloqueados/liberados;
* versão de firmware e estado da WAN.

Exemplo **fictício**:

| Dispositivo       | IP           | Conexão       | Observação           |
| ----------------- | ------------ | ------------- | -------------------- |
| `notebook-estudo` | `192.0.2.25` | Wi‑Fi 5 GHz   | conhecido            |
| `celular-teste`   | `192.0.2.41` | Wi‑Fi 5 GHz   | conhecido            |
| `tv-lab`          | `192.0.2.60` | cabo          | conhecido            |
| `desconhecido-01` | `192.0.2.88` | Wi‑Fi 2,4 GHz | investigar com calma |

O painel pode mostrar aparelhos recentemente desconectados; “aparece na lista” não significa que esteja ativo neste segundo.

### B. Tabela DHCP

DHCP é quem empresta configurações IP temporárias aos clientes. É excelente para responder:

```
Quem recebeu um endereço?
Quando termina o lease?
Qual hostname foi informado?
```

Mas DHCP não mostra tudo: dispositivo com IP fixo pode não aparecer; um aparelho desligado pode continuar listado até o lease expirar.

### C. Tabela ARP/neighbor do seu computador

ARP (IPv4) relaciona IPs locais e MACs observados. No Windows:

```powershell
arp -a
```

No Linux:

```bash
ip neigh
```

Exemplo fictício de leitura:

```
192.0.2.1    02:00:00:00:00:01   dynamic
192.0.2.25   02:00:00:00:00:25   dynamic
```

Ela mostra principalmente vizinhos com os quais seu dispositivo conversou recentemente; não é uma lista completa de todos os aparelhos da rede.

### D. Informações do próprio dispositivo

No Windows:

```powershell
ipconfig /all
getmac /v
```

No Linux:

```bash
ip addr
ip route
```

Use esses comandos para entender a **sua própria** interface, gateway, DNS e endereço. O resultado possui dados do seu ambiente; não publique o texto bruto.

***

## 4. Monte um inventário que realmente serve

Uma planilha simples é melhor que uma ferramenta sofisticada abandonada.

| Campo            | Exemplo fictício                      | Motivo                   |
| ---------------- | ------------------------------------- | ------------------------ |
| ID               | `ATV-004`                             | referência estável       |
| Nome             | `camera-entrada-lab`                  | reconhecível             |
| Tipo             | câmera IP                             | agrupa controles         |
| Dono             | responsável fictício                  | deixa alguém responsável |
| Local lógico     | VLAN IoT / Wi‑Fi convidados           | ajuda a segmentar        |
| IP/MAC           | mascarado ou guardado em local seguro | correlação técnica       |
| Sistema/firmware | `firmware 2.x`                        | atualização              |
| Criticidade      | média                                 | prioridade               |
| Dados tratados   | vídeo de teste                        | risco e privacidade      |
| Acesso remoto    | não                                   | superfície de ataque     |
| Última revisão   | `2026-08-21`                          | manutenção               |

### Campos que não entram

* senha do Wi‑Fi, senha de administrador, chave SSH;
* token de API ou cookie de sessão;
* conteúdo de câmera, áudio, documentos ou conversas;
* dados pessoais sem necessidade;
* arquivos de configuração completos não protegidos.

Se precisar guardar um segredo, use um gerenciador de senhas confiável, com controles de acesso; não uma planilha compartilhada.

***

## 5. Classifique os ativos antes de agir

Nem todo dispositivo precisa da mesma proteção.

```
Crítico: roteador, NAS de backup, computador administrativo
Importante: notebook pessoal, celular principal
IoT: câmera, TV, lâmpada, assistente de voz
Convidado: celular de visitante
Desconhecido: ainda não associado a um ativo legítimo
```

| Classe       | Controle inicial razoável                                              |
| ------------ | ---------------------------------------------------------------------- |
| Roteador/ONT | senha administrativa única, firmware acompanhado, administração local. |
| Computador   | atualização automática, conta sem privilégio diário, backup.           |
| IoT          | rede/VLAN separada quando possível, firmware e conta exclusiva.        |
| Convidado    | rede de convidados, sem acesso aos dispositivos internos.              |
| Desconhecido | identificar sem expor pânico; revogar acesso se não for reconhecido.   |

A segmentação reduz impacto: uma TV comprometida não deveria alcançar diretamente o NAS ou a máquina de trabalho.

***

## 6. “Dispositivo desconhecido”: investigação sem pânico

Um item desconhecido pode ser:

* seu celular com nome aleatório e MAC privado no Wi‑Fi;
* uma impressora esquecida;
* um repetidor, TV ou console;
* um visitante autorizado;
* um dispositivo realmente não autorizado.

Fluxo seguro:

```
1. Registrar horário, IP, MAC mascarado, banda/porta e hostname.
2. Conferir os próprios aparelhos e pessoas autorizadas.
3. Verificar se o nome muda quando o aparelho é desligado.
4. Atualizar senha Wi‑Fi/admin se não houver identificação legítima.
5. Remover/bloquear o aparelho desconhecido no roteador.
6. Revisar firmware, WPS, rede de convidados e logs.
```

Não acuse ninguém com base em um hostname ou fabricante. Identifique por evidência e trate a rede como um sistema que pode ter dispositivos aleatórios/temporários.

***

## 7. Dados que indicam exposição indevida

Inventário também ajuda a procurar “informações que não deveriam estar ali”.

| Sinal                                              | Pergunta defensiva                        | Ação segura                              |
| -------------------------------------------------- | ----------------------------------------- | ---------------------------------------- |
| Interface administrativa acessível por Wi‑Fi comum | quem precisa disso?                       | restringir à rede/admin apropriada.      |
| Câmera e notebook na mesma rede plana              | a câmera precisa falar com o notebook?    | considerar segmentação.                  |
| Firmware muito antigo                              | há atualização e suporte?                 | confirmar versão e planejar atualização. |
| Serviço remoto habilitado sem motivo               | qual é o uso e quem autorizou?            | desabilitar ou restringir após validar.  |
| Senha padrão                                       | o manual ainda tem a credencial original? | trocar por senha única forte.            |
| WPS ativo sem necessidade                          | há benefício suficiente?                  | desabilitar se não for necessário.       |
| Sem backup de configuração                         | como recuperar após falha?                | exportar backup protegido, sem publicar. |

“Encontrar dados” não deve significar coletar tudo. O objetivo é identificar riscos de configuração e dados que precisam de proteção.

***

## 8. Exemplo completo: rede fictícia

```
Rede: 192.0.2.0/24 (fictícia)
Gateway: 192.0.2.1
DNS: 192.0.2.1
```

| ID      | Ativo              | Classe       | Estado    | Próxima ação                                    |
| ------- | ------------------ | ------------ | --------- | ----------------------------------------------- |
| ATV-001 | roteador-lab       | crítico      | conhecido | verificar atualização e backup protegido.       |
| ATV-002 | notebook-estudo    | importante   | conhecido | atualizar sistema e revisar backup.             |
| ATV-003 | camera-entrada-lab | IoT          | conhecido | colocar em rede IoT e trocar credencial padrão. |
| ATV-004 | celular-teste      | importante   | conhecido | confirmar atualizações.                         |
| ATV-005 | desconhecido-01    | desconhecido | pendente  | identificar; bloquear se não for legítimo.      |

O inventário transforma “tem coisas no Wi‑Fi” em uma lista de decisões verificáveis.

***

## 9. Rotina mensal de 15 minutos

1. Abra a lista de clientes do roteador.
2. Compare com o inventário.
3. Identifique novos dispositivos e retire os que não existem mais.
4. Veja atualizações do roteador/ONT e ativos críticos.
5. Revise contas administrativas e acesso remoto.
6. Confirme que backup está funcionando.
7. Registre o que mudou e a data da revisão.

Pequena frequência vence uma auditoria enorme feita uma vez e esquecida.

***

## Mapa mental

```
Inventário de rede
├─ fontes: roteador | DHCP | ARP | próprio dispositivo
├─ identidade: nome | hostname | IP | MAC | papel
├─ classificação: crítico | importante | IoT | convidado | desconhecido
├─ privacidade: coletar mínimo, nunca senhas/tokens/conteúdo
├─ decisão: permitir | segmentar | atualizar | bloquear | documentar
└─ rotina: revisar mudanças todo mês
```

## Prova curta — responda antes do Módulo 11

1. Por que IP e MAC, sozinhos, não identificam uma pessoa?
2. Qual a diferença entre tabela DHCP e tabela ARP?
3. Cite três informações que **não** devem ser colocadas no inventário.
4. Qual deve ser a primeira reação profissional a um dispositivo desconhecido?
5. Por que uma rede de convidados/IoT segmentada melhora a segurança?

***

## Referências

* [NIST CSF — Identify](https://www.nist.gov/cyberframework/identify)
* [NIST IR 7693 — identificação de ativos](https://csrc.nist.gov/pubs/ir/7693/final)
* [CISA — gestão de inventário de ativos](https://www.cisa.gov/stopransomware/ransomware-guide)
* [RFC 1918 — endereçamento privado](https://datatracker.ietf.org/doc/rfc1918/)
