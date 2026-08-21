# Fundamentos de redes

## A ideia central

Rede é o conjunto de dispositivos e regras que permite transportar informação de uma origem até um destino. O tráfego não viaja como um único bloco: ele é dividido em unidades menores e cada camada da rede cuida de uma parte do trabalho.

Uma forma prática de pensar é:

```
Aplicação → transporte → IP/roteamento → rede local → meio físico
```

* A **aplicação** define o que está sendo pedido: abrir uma página, consultar uma API, enviar um e-mail.
* O **transporte** organiza a conversa entre processos.
* O **IP** encaminha dados entre redes.
* A **rede local** entrega quadros ao próximo equipamento.
* O meio físico pode ser cabo, fibra, rádio ou rede móvel.

## Endereçamento: MAC, IP, porta e DNS

Esses termos parecem semelhantes, mas respondem perguntas diferentes.

| Elemento | Pergunta que responde              | Exemplo                      |
| -------- | ---------------------------------- | ---------------------------- |
| MAC      | Qual interface na rede local?      | placa de rede de um notebook |
| IP       | Qual equipamento/rede é o destino? | `192.168.1.10` ou IPv6       |
| Porta    | Qual serviço dentro do destino?    | 443 para HTTPS               |
| DNS      | Qual IP corresponde ao nome?       | `exemplo.com`                |

Um computador pode ter vários serviços no mesmo IP porque cada um escuta em uma porta. Uma aplicação web, por exemplo, costuma receber tráfego em 443; banco de dados e administração devem ficar em redes privadas, não expostos sem necessidade.

## TCP e UDP

**TCP** inicia uma conexão, confirma a entrega e preserva a ordem dos dados. Ele atende bem a aplicações em que perda ou troca de ordem é inaceitável: páginas web, APIs, e-mail e transferências.

**UDP** reduz a sobrecarga de confirmação e favorece rapidez. É útil quando atraso é pior que perda pontual, como voz, vídeo ou consultas DNS. Isso não significa que UDP seja “inseguro”; a aplicação pode acrescentar controles próprios quando necessário.

## Roteamento e NAT

Quando um pacote sai da rede local, o roteador escolhe o próximo salto com base no endereço IP de destino. Em casas e pequenas empresas, é comum que vários dispositivos privados usem um único IP público através de **NAT**. NAT ajuda a conservar endereços IPv4, mas não substitui firewall: a política de entrada e saída ainda precisa ser definida.

## Rede local: switch, roteador e VLAN

* **Switch:** conecta equipamentos dentro da mesma rede local e encaminha tráfego de camada 2.
* **Roteador:** conecta redes diferentes e encaminha pacotes IP.
* **VLAN:** cria separação lógica em uma infraestrutura física compartilhada.

Exemplo de organização saudável:

```
VLAN usuários      → notebooks e estações
VLAN servidores    → aplicações internas
VLAN banco         → dados; acesso só das aplicações necessárias
VLAN gestão        → administração de equipamentos
VLAN visitantes    → internet apenas, sem acesso interno
```

A VLAN, sozinha, não é uma política completa. O tráfego entre segmentos deve passar por regras explícitas de firewall ou controle equivalente.

## Como diagnosticar sem “chutar”

Ao investigar uma falha, siga o caminho da comunicação:

1. O nome resolve no DNS?
2. O destino é alcançável pela rota esperada?
3. A porta do serviço está acessível apenas a quem deveria?
4. O TLS/certificado está válido, se for HTTPS?
5. A aplicação autenticou e autorizou o usuário?
6. Os logs mostram erro de rede, aplicação ou banco?

Esse raciocínio evita resolver o sintoma no lugar errado.
