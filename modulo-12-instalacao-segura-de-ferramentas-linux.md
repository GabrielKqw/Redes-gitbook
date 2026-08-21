# Módulo 12 — Instalação segura de ferramentas Linux

> **Meta:** preparar uma VM Linux para estudo defensivo de redes e cybersegurança usando repositórios oficiais, pacotes mínimos e ambiente isolado.

## Escopo seguro

Instale em uma VM de estudo, não no computador de trabalho principal. Use ferramentas apenas em máquinas, VMs e redes autorizadas. O objetivo é diagnóstico, inventário e observação — não exploração.

Não é necessário instalar Kali Linux nem uma coleção enorme de ferramentas para começar. Ubuntu ou Debian em uma VM atualizada, com um conjunto pequeno de utilitários, cobre os módulos deste curso.

***

## 1. Antes da instalação

Checklist:

* Crie um snapshot da VM.
* Use rede NAT apenas para baixar atualizações; retorne a host-only/rede interna para o laboratório.
* Confirme espaço em disco e versão do sistema.
* Use conta normal e sudo somente quando necessário.
* Não adicione repositórios aleatórios, scripts copiados de vídeos ou pacotes piratas.

Confira o sistema:

```
cat /etc/os-release
uname -r
df -h
```

Exemplo fictício:

```
PRETTY_NAME="Ubuntu 24.04 LTS"
Linux 6.x
/dev/vda1  30G  8G  21G  28% /
```

***

## 2. Atualize pelo repositório oficial

Em Ubuntu/Debian:

```
sudo apt update
sudo apt upgrade
```

* apt update atualiza o índice de pacotes disponíveis.
* apt upgrade instala atualizações compatíveis.
* Leia a lista antes de confirmar; não use comandos de atualização sem entender o que será alterado.

Para verificar a origem e versão de um pacote antes de instalar:

```
apt policy nmap
apt show nmap
```

APT registra ações de instalação e remoção. Isso é útil para saber o que mudou na VM.

***

## 3. Kit inicial mínimo

Instale apenas ferramentas de observação e diagnóstico:

```
sudo apt install curl dnsutils iproute2 iputils-ping traceroute mtr-tiny
```

| Pacote/ferramenta | O que permite aprender                                     |
| ----------------- | ---------------------------------------------------------- |
| curl              | HTTP, HTTPS, cabeçalhos e APIs de teste                    |
| dnsutils          | dig e consultas DNS                                        |
| iproute2          | ip addr, ip route e ip neigh                               |
| iputils-ping      | ping de conectividade                                      |
| traceroute        | observar caminho lógico até um destino                     |
| mtr-tiny          | perda/latência ao longo do caminho, em contexto autorizado |

Verifique a instalação:

```
curl --version
dig -v
ip -V
ping -V
```

***

## 4. Análise local de rede

Para entender protocolos e processos da sua própria VM:

```
sudo apt install wireshark tcpdump
```

| Ferramenta | Uso defensivo                | Cuidado                                           |
| ---------- | ---------------------------- | ------------------------------------------------- |
| Wireshark  | visualizar pacotes e camadas | capturas podem conter metadados e segredos        |
| tcpdump    | captura simples no terminal  | não compartilhe arquivos de captura sem sanitizar |

Comece por tráfego local e dados fictícios. Capturar redes sem autorização pode invadir privacidade mesmo que você não altere nada.

Teste didático de loopback, sem rede externa:

```
curl http://127.0.0.1:8080/
```

Esse comando só faz sentido se você tiver um servidor de estudo local já autorizado. O objetivo é observar a conversa da própria VM, não capturar a LAN.

***

## 5. Inventário de um alvo de treinamento

Para inventariar uma VM de laboratório autorizada:

```
sudo apt install nmap
```

Nmap serve para entender quais serviços estão expostos em um ativo permitido. Neste curso, ele é usado para comparar o que deveria estar disponível com o que foi documentado no laboratório.

Antes de qualquer inventário, registre:

* alvo: VM de treinamento, não Internet/LAN real;
* rede: host-only ou interna;
* janela e responsável;
* serviços esperados;
* critério de parada;
* destino dos relatórios e capturas sanitizadas.

Não rode scanners contra a Internet, roteador da operadora ou dispositivos de outras pessoas. Porta aberta não é sinônimo de vulnerabilidade.

***

## 6. Ferramentas de criptografia e formatos

```
sudo apt install openssl ca-certificates jq
```

| Ferramenta      | Uso seguro no curso                                |
| --------------- | -------------------------------------------------- |
| openssl         | observar certificado TLS e formatos criptográficos |
| ca-certificates | manter autoridades certificadoras do sistema       |
| jq              | ler JSON de respostas e logs fictícios             |

Exemplos de estudo local:

```
openssl version
jq --version
```

Chaves privadas, tokens e senhas nunca devem ser usados como dados de exemplo. Use marcadores como TOKEN\_DE\_EXEMPLO e nunca um valor verdadeiro.

***

## 7. Como verificar sem instalar às cegas

| Pergunta                                    | Comando           |
| ------------------------------------------- | ----------------- |
| O pacote existe no repositório configurado? | apt policy NOME   |
| O que ele faz e do que depende?             | apt show NOME     |
| Ele foi instalado?                          | dpkg -l NOME      |
| Quais arquivos o pacote adicionou?          | dpkg -L NOME      |
| Que versão da ferramenta está em uso?       | NOME --version    |
| Onde estão os registros de instalação?      | /var/log/dpkg.log |

Evite estes atalhos:

* curl seguido de shell para instalar algo sem revisão;
* repositório de terceiro sem verificar mantenedor e assinatura;
* desativar verificações TLS para “fazer funcionar”;
* executar tudo como root;
* usar ferramentas de segurança em ambiente fora do escopo.

***

## 8. Atualização e remoção

Verifique atualizações periodicamente:

```
sudo apt update
apt list --upgradable
```

Para remover algo que não será usado:

```
sudo apt remove NOME_DO_PACOTE
```

Remover reduz superfície de ataque e confusão. Use purge apenas quando entender que configurações também serão removidas.

Faça snapshot antes de mudanças importantes. Uma VM descartável facilita aprender sem transformar erro de laboratório em problema permanente.

***

## 9. Configuração recomendada de laboratório

```
VM Linux de estudo
  ├─ atualização feita em NAT
  ├─ snapshot limpo
  ├─ ferramentas mínimas instaladas
  ├─ rede interna/host-only para práticas
  ├─ dados fictícios
  └─ sem pastas compartilhadas ou credenciais reais
```

## Prova curta — responda antes do próximo módulo

1. Por que a VM é preferível ao computador principal para instalar ferramentas de estudo?
2. Qual a diferença entre apt update e apt upgrade?
3. Cite três ferramentas do kit mínimo e suas finalidades.
4. Por que uma captura Wireshark deve ser tratada como dado sensível?
5. Antes de usar uma ferramenta de inventário, quais limites de escopo você registra?

## Referências

* [Ubuntu — gestão de pacotes](https://ubuntu.com/server/docs/how-to/software/package-management/)
* [Nmap — documentação oficial](https://nmap.org/docs.html)
* [Wireshark — documentação](https://www.wireshark.org/docs/)
