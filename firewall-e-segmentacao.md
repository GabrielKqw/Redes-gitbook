# Firewall e segmentação

## O que um firewall faz

Firewall é um controle que permite, bloqueia ou registra tráfego entre redes ou equipamentos com necessidades de segurança diferentes. Ele pode analisar origem, destino, porta, protocolo, estado da conexão e, dependendo da solução, informações da aplicação.

Firewall não é sinônimo de “bloquear a internet”. Seu valor está em explicitar quais comunicações são legítimas.

Uma regra bem definida responde:

* quem inicia a conexão;
* para qual destino;
* por qual porta/protocolo;
* em qual direção;
* para qual finalidade;
* por quanto tempo a exceção é necessária;
* quem é responsável pela revisão.

## Política de menor privilégio

Comece negando o tráfego que não foi aprovado e libere apenas os fluxos necessários. Uma regra ampla como “qualquer origem para qualquer destino” facilita operação no curto prazo, mas elimina a capacidade de conter movimento lateral.

Exemplo de política conceitual:

```
Internet → proxy web: permitir HTTPS 443
Proxy web → aplicação: permitir somente porta da aplicação
Aplicação → banco: permitir somente porta do banco
Usuário → banco: negar
Rede de visitantes → rede interna: negar
Administração → dispositivos: permitir somente via rede de gestão + MFA
```

O exemplo não é uma configuração pronta: portas, origens e controles precisam refletir o ambiente real.

## Segmentação

Segmentar é dividir uma rede em zonas menores com fronteiras controladas. Uma separação básica pode ter usuários, servidores, banco de dados, gestão e visitantes. Ambientes industriais ou ativos de alto valor podem exigir zonas ainda mais restritas e uma DMZ.

Benefícios:

* limita alcance de malware e credenciais comprometidas;
* reduz movimento lateral;
* melhora visibilidade dos fluxos;
* permite aplicar políticas diferentes por criticidade;
* reduz impacto de erro de configuração em uma área.

## Erros comuns

* Permitir “temporariamente” e nunca revisar.
* Liberar faixas inteiras quando somente um serviço precisa conversar.
* Expor banco, painel de gestão ou SSH diretamente.
* Confiar somente na rede interna; uma máquina interna pode ser comprometida.
* Não registrar tráfego bloqueado e mudanças de regra.
* Criar VLANs, mas liberar todo o tráfego entre elas.

## Processo seguro para mudança de regra

1. Descreva aplicação, origem, destino, porta, direção e justificativa.
2. Confirme se já existe fluxo aprovado.
3. Prefira a regra mais específica possível.
4. Teste a comunicação prevista.
5. Monitore o efeito e registre o responsável.
6. Agende revisão ou expiração para exceções temporárias.

## Fontes

* [NIST SP 800-41 Rev. 1 — Firewalls and Firewall Policy](https://csrc.nist.gov/pubs/sp/800/41/r1/final)
* [CISA — Layering Network Security Through Segmentation](https://www.cisa.gov/sites/default/files/publications/layering-network-security-segmentation_infographic_508_0.pdf)
