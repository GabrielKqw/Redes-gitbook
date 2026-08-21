# Módulo 7 — Fibra, operadoras, ONT/ONU, OLT e CGNAT

> **Meta:** entender como a Internet sai do seu roteador, percorre a rede de acesso da operadora e alcança a Internet — sem confundir fibra, ONT, modem, Wi‑Fi, IP público e CGNAT.

## Ao terminar, você deverá conseguir

* Desenhar o caminho físico e lógico da sua conexão.
* Diferenciar **OLT**, **ODN**, **splitter**, **ONU/ONT**, roteador e ponto de acesso Wi‑Fi.
* Explicar GPON, XGS-PON, PPPoE, DHCP, VLAN, NAT e CGNAT em nível operacional.
* Investigar luz vermelha/LOS, falha de autenticação, latência e ausência de IP público.
* Separar problema de fibra, rede da operadora, roteador, Wi‑Fi, DNS e serviço remoto.

***

## 1. Da sua casa até a Internet

Uma conexão residencial de fibra não é “um cabo direto até o Google”. Há uma rede de acesso compartilhada, equipamentos de agregação e conexões entre provedores.

```
Seu dispositivo
   │ Wi‑Fi ou cabo Ethernet
Roteador / Access Point
   │ Ethernet
ONT ou ONU
   │ fibra de acesso
Caixa / splitter passivo ── ODN ── OLT da operadora
                                     │
                              agregação / BNG / CGN
                                     │
                     backbone, peering, trânsito e CDNs
                                     │
                              servidor de destino
```

* **Rede de acesso**: trecho que conecta assinantes ao provedor.
* **Backbone**: rede de alta capacidade que interliga regiões e pontos de presença.
* **Peering**: troca direta de tráfego entre redes/autonomous systems.
* **Trânsito IP**: conectividade comprada de outra rede para alcançar o restante da Internet.
* **CDN**: cópia de conteúdo próxima do usuário; muitas vezes o vídeo/site não sai da cidade ou do estado.

A rota real varia por operadora, destino, horário e engenharia de tráfego. Um `tracert` ajuda a formar hipótese, mas não é um mapa completo: roteadores podem filtrar ou limitar respostas ICMP.

***

## 2. Fibra óptica: o sinal é luz

A fibra guia luz através de um núcleo muito fino. Ela permite alta capacidade e é imune a interferência eletromagnética comum, mas é sensível a curvaturas excessivas, conectores contaminados, emendas ruins e perda óptica.

### O que é potência óptica?

A intensidade do sinal é frequentemente mostrada em **dBm**. Como é uma escala logarítmica, valores mais negativos representam sinal mais fraco. A faixa aceitável depende da tecnologia, do módulo óptico, do projeto e da operadora. Portanto:

> Não conclua “está bom” ou “está ruim” por um número isolado visto na Internet. Compare com a especificação e o limite operacional do equipamento/operadora.

**Nunca olhe para a ponta da fibra ou para um conector óptico.** Mesmo luz invisível pode causar dano aos olhos. Limpeza, medição e reparo devem seguir procedimento apropriado e, quando necessário, ser feitos pelo técnico.

***

## 3. PON: uma fibra compartilhada de forma inteligente

**PON** significa _Passive Optical Network_ — rede óptica passiva. “Passiva” aqui se refere ao trecho de distribuição óptica, que usa fibra e divisores sem equipamento elétrico ativo no caminho até o assinante.

```
                ┌──────── ONU/ONT cliente A
OLT ─ fibra ─ splitter ─ ONU/ONT cliente B
                └──────── ONU/ONT cliente C
```

### Principais siglas

| Termo                                    | Onde fica                | Papel                                                                 |
| ---------------------------------------- | ------------------------ | --------------------------------------------------------------------- |
| **OLT** (_Optical Line Terminal_)        | central/POP da operadora | Controla a porta PON, comunica e gerencia as unidades dos assinantes. |
| **ODN** (_Optical Distribution Network_) | entre OLT e cliente      | Fibra, conectores, caixas e splitters.                                |
| **Splitter**                             | ODN                      | Divide/combina o sinal óptico; é passivo.                             |
| **ONU** (_Optical Network Unit_)         | ponta do assinante       | Termina a rede PON no lado do cliente.                                |
| **ONT** (_Optical Network Terminal_)     | forma de ONU no cliente  | Geralmente o equipamento instalado na casa.                           |
| **CPE**                                  | casa/empresa             | Equipamento do cliente; pode incluir ONT e roteador.                  |

Na prática, “ONU” e “ONT” são usados de modo próximo. A documentação da operadora/equipamento define a nomenclatura exata. Uma ONT pode ser:

* apenas ponte/conversor óptico, entregando Ethernet a um roteador separado;
* uma **ONU/ONT roteada**, acumulando roteador, NAT, DHCP e Wi‑Fi;
* parte de um equipamento da operadora com recursos de telefonia/IPTV.

### GPON e XGS-PON

| Tecnologia  |                   Capacidade de linha típica | Ideia                                                                  |
| ----------- | -------------------------------------------: | ---------------------------------------------------------------------- |
| **GPON**    | \~2,5 Gb/s downstream e \~1,25 Gb/s upstream | Muito presente em FTTH; a capacidade é compartilhada na PON.           |
| **XGS-PON** |                         \~10 Gb/s simétricos | Evolução com maior capacidade, útil para planos e demandas superiores. |

Capacidade de linha não é automaticamente velocidade garantida por assinante. Há divisão por splitter, perfil contratado, concorrência, uplinks e políticas da operadora. A ITU-T padroniza a arquitetura e as tecnologias PON; o dimensionamento comercial é decisão de cada provedor.

### Como vários clientes enviam pela mesma fibra?

No downstream, a OLT envia informação para a PON e cada ONU/ONT processa somente o que lhe cabe. No upstream, as ONUs transmitem em janelas de tempo coordenadas para não colidirem. Isso exige sincronização, _ranging_ (medição/compensação de distância) e gerenciamento pela OLT.

***

## 4. O que cada caixa da sua casa faz

```
Internet da operadora
       │ fibra
     [ONT] ── Ethernet ── [roteador] ))) Wi‑Fi ))) notebook/celular
                              │
                           LAN cabeada
```

| Equipamento/função | Não é a mesma coisa que | O que faz                                                              |
| ------------------ | ----------------------- | ---------------------------------------------------------------------- |
| ONT/ONU            | roteador                | Converte/termina a rede PON para Ethernet/serviço do assinante.        |
| Roteador           | Wi‑Fi                   | Encaminha entre WAN e LAN; geralmente faz NAT, DHCP e firewall básico. |
| Access Point       | roteador                | Oferece Wi‑Fi; pode estar integrado ao roteador.                       |
| Switch             | roteador                | Expande portas Ethernet dentro da mesma LAN.                           |
| “Modem”            | sempre uma ONT          | Nome popular. Em fibra, o equipamento pode ser ONT, roteador ou ambos. |

Quando a ONT e o roteador estão no mesmo aparelho, a interface pode esconder essas fronteiras — mas elas continuam existindo logicamente.

***

## 5. Como a operadora reconhece e configura o assinante

Depois que a fibra está com sinal e a ONT foi autorizada, a conexão IP pode ser entregue de formas diferentes. Duas comuns são **PPPoE** e **IPoE/DHCP**.

### PPPoE

**PPPoE** encapsula sessões PPP sobre Ethernet. Ele permite estabelecer uma sessão entre o cliente e o concentrador de acesso; historicamente é usado para controle de acesso, autenticação e cobrança.

Fluxo simplificado:

```
Roteador/ONT                 Concentrador da operadora
  PADI  (procura)       ───>
  <─── PADO  (oferta)
  PADR  (pedido)        ───>
  <─── PADS  (sessão)
  autenticação/configuração PPP
  recebe IP, rota e DNS (conforme perfil)
```

O usuário e senha PPPoE são segredos. Não envie print com esses dados e não “clone” configurações de terceiros.

### DHCP/IPoE

Em muitos cenários, o cliente recebe configuração IP via DHCP sobre Ethernet/IP, possivelmente com identificação por VLAN, porta, serial da ONT ou outro mecanismo da operadora. O usuário pode nem enxergar credencial PPPoE.

### VLAN

Uma **VLAN** separa logicamente redes no mesmo meio Ethernet. A operadora pode exigir uma VLAN específica na WAN para identificar/entregar o serviço. Configurar a VLAN errada costuma resultar em “fibra sincronizada, mas sem navegar”.

> Não altere VLAN, modo bridge ou credenciais sem salvar a configuração atual e sem saber o procedimento de retorno. Você pode perder acesso de gerenciamento e, em alguns contratos, precisar de suporte para reconfigurar a ONT.

***

## 6. IP público, NAT e CGNAT

Dentro da sua casa, dispositivos normalmente recebem IP privado, como `192.168.1.10`. O roteador traduz muitas conexões internas para o endereço da WAN usando **NAT**.

```
Notebook 192.168.1.10:51500
      │ NAT no roteador
WAN público 200.x.y.z:40001
      │
Servidor web 203.0.113.20:443
```

### O que é CGNAT?

IPv4 público é escasso. No **CGNAT** (_Carrier-Grade NAT_), a própria operadora também traduz conexões de vários clientes para um ou mais IPv4 públicos.

```
sua LAN privada
 → NAT do roteador
 → endereço compartilhado da operadora (muitas vezes 100.64.0.0/10)
 → CGNAT
 → IPv4 público da operadora
 → Internet IPv4
```

O bloco `100.64.0.0/10` foi reservado para _Shared Address Space_ em redes de operadoras. Ele não é roteável globalmente e não é o mesmo bloco privado RFC 1918.

### Efeito prático do CGNAT

* Navegar, jogar e consumir serviços costuma funcionar.
* Encaminhar uma porta da sua casa para a Internet pode **não** funcionar, pois o IPv4 público não está diretamente na sua WAN.
* Alguns serviços P2P, acesso remoto e jogos podem precisar de IPv6, relay, VPN/túnel autorizado, ou um IPv4 público fornecido pela operadora.
* Não tente “abrir portas” em dispositivos que você não controla. A solicitação correta é confirmar com o provedor se há CGNAT e quais opções legítimas existem.

IPv6 reduz a necessidade de NAT para endereços globais, mas exige firewall bem configurado: ter endereço alcançável não autoriza expor serviços sem proteção.

***

## 7. Bridge, roteamento e duplo NAT

| Modo        | Desenho                                     | Quando aparece                                                         |
| ----------- | ------------------------------------------- | ---------------------------------------------------------------------- |
| ONT roteada | ONT faz PPPoE/DHCP, NAT e talvez Wi‑Fi      | Instalação simples da operadora.                                       |
| Bridge      | ONT entrega o enlace ao seu roteador        | Seu roteador assume PPPoE/DHCP e a política da LAN.                    |
| Duplo NAT   | ONT roteada → seu roteador roteando de novo | Pode funcionar para navegação, mas complica portas, VPN e diagnóstico. |

**Duplo NAT não é sempre um defeito.** É um desenho que precisa ser conhecido. Antes de mudar para bridge, avalie telefonia/IPTV, suporte da operadora, acesso à ONT e a configuração de retorno.

***

## 8. Diagnóstico em camadas

### LEDs da ONT

Nomes e comportamento variam, mas um padrão comum é:

| Sinal                     | Hipótese inicial              | Próxima ação segura                             |
| ------------------------- | ----------------------------- | ----------------------------------------------- |
| **PON aceso estável**     | ONU/ONT registrou na OLT      | Verificar WAN, DNS, roteador e Wi‑Fi.           |
| **LOS vermelho/piscando** | perda de sinal óptico         | Não dobrar/manusear fibra; acionar a operadora. |
| LAN apagada               | cabo/porta/interface sem link | Testar cabo e porta sob seu controle.           |
| Wi‑Fi ruim, cabo bom      | problema local sem fio        | Canal, distância, interferência, posição do AP. |

Esses LEDs são indícios, não diagnóstico completo. A operadora consegue verificar potência, estado de registro e eventos na OLT.

### Sequência curta de investigação

1. **Uma máquina por cabo navega?** Se sim, concentre-se no Wi‑Fi/dispositivo.
2. **A ONT está com PON normal e sem LOS?** Se não, é provável acesso óptico.
3. **A WAN recebeu endereço e gateway?** Verifique painel do roteador/ONT.
4. **Um IP funciona mas nome falha?** Teste DNS.
5. **Só um site falha?** Compare com outros destinos e procure status do serviço.
6. **Há perda/latência?** Meça em horário, destino e meio de acesso controlados.

No Windows:

```powershell
ipconfig /all
ping 1.1.1.1
nslookup example.com
tracert 1.1.1.1
```

No Linux:

```bash
ip addr
ip route
ping -c 4 1.1.1.1
dig example.com
traceroute 1.1.1.1
```

* Se `ping 1.1.1.1` funciona e `nslookup example.com` falha, DNS é uma hipótese forte.
* Se ambos falham, não conclua de imediato que “a fibra caiu”: teste gateway, cabo/Wi‑Fi, estado WAN e indicadores da ONT.
* `tracert`/`traceroute` com um salto que não responde não prova falha; muitos equipamentos não respondem a esse tipo de sonda.

***

## 9. Exemplo: “minha Internet está lenta”

“Lenta” não é uma causa; é um sintoma. Transforme em medida:

```
Ruim:   "A Internet está péssima."
Útil:   "Em 21/08 às 20:30 BRT, por cabo, três testes ao destino X
         tiveram 3% de perda e 180 ms; pelo 4G não ocorreu."
```

Colete sem expor dados sensíveis:

* horário e fuso;
* teste por cabo versus Wi‑Fi;
* destino e resultado;
* plano contratado e resultado de teste próximo à operadora;
* LED/estado PON/LOS;
* se o problema é geral ou ocorre em um aplicativo.

A operadora pode então separar saturação local, defeito de acesso, rota/peering, problema em CDN/destino ou falha no seu Wi‑Fi.

***

## Mapa mental

```
Operadora de Internet
├─ acesso: fibra → ODN/splitter → ONT/ONU → OLT
├─ ativação: PON + VLAN + PPPoE ou DHCP
├─ endereço: IPv4/IPv6, NAT e possivelmente CGNAT
├─ transporte: backbone, peering, trânsito e CDN
└─ diagnóstico: óptico → WAN → LAN/Wi‑Fi → DNS → destino
```

## Exercício prático seguro

1. No painel do seu roteador, anote sem publicar: estado da WAN, tipo de conexão e se há IPv4/IPv6.
2. Compare `ping 1.1.1.1` e `nslookup example.com`.
3. Faça o mesmo teste por cabo e Wi‑Fi, se possível.
4. Observe os LEDs da ONT sem tocar na fibra.
5. Classifique o cenário: acesso óptico, WAN/autenticação, LAN/Wi‑Fi, DNS ou destino.

## Prova curta — responda antes do Módulo 8

1. Qual é o caminho entre a ONT e a OLT e qual elemento permite atender vários clientes sem energia no meio?
2. Por que ONT e roteador não são necessariamente o mesmo equipamento?
3. Em PPPoE, para que serve a fase de descoberta antes da sessão?
4. Qual problema o CGNAT cria para alguém que quer receber conexões da Internet em casa?
5. Uma ONT tem PON estável, não há LOS, mas somente o Wi‑Fi está lento. Em qual camada você começaria a investigar?

***

## Referências técnicas

* [ITU-T G.984.1 — GPON: características gerais](https://www.itu.int/rec/dologin_pub.asp?id=T-REC-G.984.1-200803-I%21%21PDF-E\&lang=e\&type=items)
* [ITU-T G.9807.1 — XGS-PON](https://www.itu.int/rec/T-REC-G.9807.1)
* [RFC 2516 — PPPoE](https://www.rfc-editor.org/rfc/rfc2516.html)
* [RFC 6598 — Shared Address Space para CGNAT](https://www.rfc-editor.org/rfc/rfc6598.html)
