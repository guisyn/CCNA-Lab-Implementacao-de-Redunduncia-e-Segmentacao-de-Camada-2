# CCNA-Lab-Implementação-de-Redundância-e-Segmentação-de-Camada-2
Demonstrar a configuração de uma topologia resiliente utilizando EtherChannel (LACP), Rapid-PVST+ para prevenção de loops, segmentação de rede, roteamento inter-VLAN via Router-on-a-Stick e serviço DHCP.

> **Ambiente:** Eve-NG | **Nível:** CCNA (200-301) | **Autor:** Guilherme Rodrigues

---

## 📋 Índice

1. [Visão Geral](#visão-geral)
2. [Topologia](#topologia)
3. [Endereçamento IP e VLANs](#endereçamento-ip-e-vlans)
4. [Tecnologias Utilizadas](#tecnologias-utilizadas)
   - [Configuração Inicial](#1--configuração-inicial-e-segurança-básica)
   - [EtherChannel L2](#2-etherchannel-l2)
   - [VLANs](#3-vlans-virtual-local-area-networks)
   - [Rapid-PVST+](#4-rapid-pvst-rapid-per-vlan-spanning-tree)
   - [Portas de Acesso e Segurança STP](#5-portas-de-acesso-e-segurança-stp-portfast--bpdu-guard)
   - [Router-on-a-Stick](#6-router-on-a-stick)
   - [Serviço DHCP](#7-serviço-dhcp)
6. [Configuração Passo a Passo](#configuração-passo-a-passo)
7. [Verificação e Testes](#verificação-e-testes)
8. [Conclusão Técnica](#conclusão-técnica)

---

## Visão Geral

Este laboratório simula um ambiente corporativo de médio porte, implementado no emulador **Eve-NG**, com foco nas tecnologias cobradas no exame **CCNA 200-301** da Cisco. O objetivo é demonstrar, de forma prática, como segmentar uma rede com VLANs, garantir redundância de camada 2 com Rapid-PVST+, aumentar a largura de banda entre switches com EtherChannel e prover roteamento inter-VLAN via Router-on-a-Stick.

A topologia reflete cenários reais de redes corporativas, onde diferentes departamentos (TI, Logística, Gerência) precisam de isolamento lógico, mas compartilham a mesma infraestrutura física.

---

## Topologia

![Topologia Eve-NG]
<img width="1365" height="767" alt="TOPOLOGIA" src="https://github.com/user-attachments/assets/90914031-a444-4197-867c-415d74a1fbd9" />


> *Diagrama gerado no Eve-NG. Os switches de distribuição formam o núcleo da rede, conectados ao roteador para o roteamento inter-VLAN.*

### Equipamentos utilizados

| Dispositivo | Quantidade | Função |
|---|---|---|
| Router Cisco (IOSv) | 1 | Roteamento inter-VLAN (Router-on-a-Stick) |
| Switch L2 Distribuição | 2 | DSW-01, DSW-02 — Camada de distribuição |
| Switch L2 Acesso | 2 | ASW-01, ASW-02 — Camada de acesso |
| VPCs (hosts) | 4 | VPC_6, VPC_7, VPC_8, VPC_9 |

---

## Endereçamento IP e VLANs

### VLANs Configuradas

| VLAN ID | Nome | Descrição |
|---|---|---|
| 21 | TI | Departamento de Tecnologia da Informação |
| 31 | LOGISTICA | Departamento de Logística |
| 99 | GERENCIA | Departamento de Gerência |
| 199 | NATIVA | VLAN Nativa (untagged) |

### Gateways (Subinterfaces no Roteador)

| VLAN | Gateway |
|---|---|
| VLAN 21 — TI | 10.2.21.1/24 |
| VLAN 31 — LOGÍSTICA | 10.2.31.1/24 |
| VLAN 99 — GERÊNCIA | 10.2.99.1/24 |

### Hosts (VPCs)
| Host  | VLAN | IP           |
|-------|------|--------------|
| VPC_6 | 21   | 10.2.21.2/24 |
| VPC_7 | 31   | 10.2.31.2/24 |
| VPC_8 | 21   | 10.2.21.3/24 |
| VPC_9 | 31   | 10.2.31.3/24 |

### Endereços de Gerenciamento (SVIs — VLAN 99)
| Dispositivo | IP           |
|-------------|--------------|
| DSW-01      | 10.2.99.3/24 |
| DSW-02      | 10.2.99.2/24 |
| ASW-01      | 10.2.99.5/24 |
| ASW-02      | 10.2.99.4/24 |

---


---

## Tecnologias Utilizadas

### 1 — Configuração Inicial e Segurança Básica (todos os dispositivos)

Antes de qualquer configuração de rede, todos os dispositivos passaram por uma **hardening inicial**, definindo identidade, controle de acesso e proteção de credenciais. Essa etapa é considerada boa prática obrigatória em ambientes corporativos e é cobrada diretamente no CCNA.

**Comandos aplicados (exemplo no DSW-01):**

```bash
# Acesso ao modo privilegiado e configuração global
Switch> en
Switch# conf t

# 1. Definir o hostname do dispositivo
Switch(config)# hostname DSW-01

# 2. Criar usuário local com senha criptografada (secret = MD5)
DSW-01(config)# username admin secret ccna

# 3. Exigir autenticação local na porta de console
DSW-01(config)# line con 0
DSW-01(config-line)# login local

# 4. Exigir autenticação local nas linhas VTY (acesso remoto SSH/Telnet)
DSW-01(config)# line vty 0 15
DSW-01(config-line)# login local

# 5. Definir senha de enable (modo privilegiado) — criptografada com secret
DSW-01(config)# enable secret ccna

# 6. Ativar o serviço de criptografia de senhas no arquivo de configuração
DSW-01(config)# service password-encryption
```

<img width="1365" height="767" alt="CONFIG-INICIAL" src="https://github.com/user-attachments/assets/3f6e8dbc-84e9-48f5-a27c-7d85c58d0ffb" />

> *Configurações iniciais, replicadas em cada switch e também no roteador.*

**Aspectos técnicos relevantes:**

- **`hostname`** — Nomeia o dispositivo, facilitando a identificação em ambientes com múltiplos equipamentos e aparecendo no prompt de cada sessão CLI.
- **`username secret`** — Armazena a senha com hash MD5, mais seguro do que `password` (que usa cifra reversível de tipo 7).
- **`login local`** — Vincula as linhas de acesso (console e VTY) ao banco de usuários local do dispositivo, exigindo usuário + senha para qualquer sessão.
- **`line vty 0 15`** — Cobre todas as 16 sessões remotas simultâneas possíveis no IOS.
- **`enable secret`** — Protege o acesso ao modo privilegiado (`#`), onde o operador tem controle total sobre o dispositivo.
- **`service password-encryption`** — Aplica criptografia tipo 7 sobre todas as senhas em texto plano ainda presentes no `running-config` (como `line password`), impedindo visualização direta por quem tiver acesso ao arquivo de configuração.

> ✅ **Resultado:** Todos os dispositivos da topologia ficaram protegidos contra acesso não autorizado, tanto pelo console físico quanto por sessões remotas.

### 2. EtherChannel L2

**O que é:** O EtherChannel (também conhecido como Link Aggregation) agrupa múltiplos links físicos entre dois switches em um único link lógico. Isso dobra (ou multiplica) a largura de banda disponível e oferece redundância — se um link físico falhar, o tráfego continua pelos demais sem interrupção perceptível.

**Por que foi usado:** Entre os switches de distribuição (DSW-01 e DSW-02), foram utilizados dois links físicos agrupados em um EtherChannel. Isso garante maior largura de banda no núcleo da rede e elimina o bloqueio que o STP aplicaria a um dos links caso não houvesse o agrupamento.

**Modos de negociação:**
- **LACP (802.3ad):** Protocolo aberto (IEEE). Modos: `active` / `passive`.
- **PAgP:** Protocolo proprietário Cisco. Modos: `desirable` / `auto`.

**Comandos principais (LACP):**

```bash
# Nos dois switches — interfaces a serem agrupadas
DSW-01(config)# interface range g0/0 - 1
DSW-01(config-if-range)# channel-group 1 mode active

# Configurar a interface lógica (Port-Channel)
DSW-01(config)# interface port-channel 1
DSW-01(config-if)# switchport mode trunk
DSW-01(config-if)# switchport trunk native vlan 199
```

<img width="1365" height="767" alt="ETHERCHANNEL-SUM" src="https://github.com/user-attachments/assets/f65a1e08-9562-4abf-9eb4-17136b71f6ec" />

> *Verificação com o comando 'show etherchannel summary'.*

---

### 3. VLANs (Virtual Local Area Networks)

**O que é:** Uma VLAN é uma subdivisão lógica de uma rede física. Dispositivos em VLANs diferentes não se comunicam diretamente — o tráfego precisa passar por um roteador (ou switch L3). Isso proporciona segurança, organização e controle de broadcast.

**Por que foi usada:** Para segmentar os departamentos (TI, Logística, Gerência) em domínios de broadcast independentes, aumentando a segurança e reduzindo tráfego desnecessário.

**Comandos principais:**

```bash
# Criação de VLANs no switch
Switch(config)# vlan 21
Switch(config-vlan)# name TI

Switch(config)# vlan 31
Switch(config-vlan)# name LOGISTICA

Switch(config)# vlan 99
Switch(config-vlan)# name GERENCIA

Switch(config)# vlan 199
Switch(config-vlan)# name NATIVA
```

<img width="1365" height="767" alt="VLANs" src="https://github.com/user-attachments/assets/1d2001b8-93aa-40b5-ae9b-5250f2ae7d23" />
> *Verificação com o comando 'show vlan'.*

```bash

# Links trunk (entre switches e para o roteador)
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk native vlan 199

```

<img width="1365" height="767" alt="TRUNKS" src="https://github.com/user-attachments/assets/e5da066b-27dc-4f97-a325-7b1a40738cce" />
> *Verificação com o comando 'show interface trunk'.*

---

### 4. Rapid-PVST+ (Rapid Per-VLAN Spanning Tree)

**O que é:** O Rapid-PVST+ é uma versão aprimorada do Spanning Tree Protocol (STP). Ele opera por VLAN — ou seja, cada VLAN tem sua própria instância de STP — e converge muito mais rápido que o STP clássico (802.1D), reduzindo o tempo de recuperação após uma falha de segundos para milissegundos.

**Por que foi usado:** A topologia possui links redundantes entre os switches de distribuição e acesso. Sem o STP, esses loops causariam tempestades de broadcast e tornariam a rede inoperante. O Rapid-PVST+ garante que apenas os caminhos necessários estejam ativos, mantendo os demais em standby para failover automático.

**Conceitos-chave:**
- **Root Bridge:** O switch "eleito" como raiz da árvore. Todos os caminhos são calculados a partir dele.
- **Port States:** Discarding, Learning, Forwarding (convergência mais rápida que STP clássico).
- **Port Roles:** Root Port, Designated Port, Alternate Port.

**Comandos principais:**

```bash
# Ativar Rapid-PVST+ (padrão nos switches Cisco modernos)
Switch(config)# spanning-tree mode rapid-pvst

# Definir Root Bridge primário e secundário por VLAN
DSW-01(config)# spanning-tree vlan 21,99 root primary
DSW-01(config)# spanning-tree vlan 31 root secondary

DSW-02(config)# spanning-tree vlan 31 root primary
DSW-02(config)# spanning-tree vlan 21,99 root secondary


```

<img width="1365" height="767" alt="DSW1-BRIDGE-CONFIG" src="https://github.com/user-attachments/assets/9fd30619-1e1e-4e3c-b30a-5c605d21a044" />

---

### 5. Portas de Acesso e Segurança STP (PortFast + BPDU Guard)

Com as VLANs criadas e os trunks configurados, o próximo passo foi atribuir as **portas de acesso** nos switches de acesso (ASW) às suas respectivas VLANs e aplicar os mecanismos de segurança do Spanning Tree: **PortFast** e **BPDU Guard**.

**Comandos aplicados (exemplo no ASW-01):**

```bash
# Porta g2/1 — acesso na VLAN 21 (TI)
ASW-01(config)# interface g2/1
ASW-01(config-if)# switchport mode access
ASW-01(config-if)# switchport access vlan 21

# Porta g2/2 — acesso na VLAN 31 (Logística)
ASW-01(config)# interface g2/2
ASW-01(config-if)# switchport mode access
ASW-01(config-if)# switchport access vlan 31

# Aplicar PortFast e BPDU Guard nas portas de acesso (range)
ASW-01(config)# interface range g2/1-2
ASW-01(config-if-range)# spanning-tree portfast
ASW-01(config-if-range)# spanning-tree bpduguard enable
```

<img width="1363" height="767" alt="ACCESS-PORTFAST" src="https://github.com/user-attachments/assets/e738444d-152d-49b3-beaa-a927a9a215aa" />



**Aspectos técnicos relevantes:**

- **`switchport mode access`** — Define a porta como acesso, ou seja, ela pertence a uma única VLAN e não carrega tag 802.1Q. É o modo correto para portas conectadas a hosts finais (VPCs, PCs, servidores).

- **`switchport access vlan X`** — Associa a porta à VLAN correspondente ao departamento do host conectado.

- **PortFast (`spanning-tree portfast`)** — Por padrão, quando uma porta sobe, o STP a mantém nos estados *Listening* e *Learning* por até 30 segundos antes de liberar tráfego (*Forwarding*). O PortFast elimina essa espera em portas de acesso, colocando-as diretamente em *Forwarding*. Isso é seguro e recomendado **somente em portas conectadas a hosts** — nunca entre switches, pois poderia causar loops. O próprio IOS emite um aviso (*Warning*) ao ser habilitado, reforçando esse cuidado.

- **BPDU Guard (`spanning-tree bpduguard enable`)** — Complementa o PortFast com uma camada de segurança: se a porta receber um **BPDU** (pacote de controle do STP, gerado por switches), ela é imediatamente colocada em estado **err-disabled**, desligando-se automaticamente. Isso impede que um switch não autorizado seja conectado à rede por um usuário e cause instabilidade na topologia STP.

> ⚠️ **Nota sobre o aviso do IOS:** A mensagem `%Warning: portfast should only be enabled on ports connected to a single host` exibida no terminal é esperada e não indica erro — é apenas o IOS reforçando que o recurso deve ser usado exclusivamente em portas de acesso para hosts, o que é exatamente o caso aqui.

> ✅ **Resultado:** As VPCs passaram a obter conectividade imediatamente ao serem conectadas, sem aguardar a convergência do STP, e qualquer tentativa de inserir um switch não autorizado nessas portas resulta em desligamento automático da interface.


---

### 6. Router-on-a-Stick

**O que é:** Router-on-a-Stick é uma técnica de roteamento inter-VLAN que utiliza um único link físico entre o roteador e um switch, dividido em múltiplas **subinterfaces lógicas** — uma para cada VLAN. Cada subinterface possui seu próprio endereço IP, que serve de gateway para os hosts daquela VLAN.

**Por que foi usado:** Como o ambiente não possui switch Layer 3, o roteamento entre as VLANs (TI, Logística, Gerência) é feito pelo roteador. Com um único cabo físico e a configuração de subinterfaces, o roteador consegue rotear o tráfego entre todas as VLANs de forma eficiente.

**Comandos principais:**

```bash
# Interface física — sem IP, apenas "ligada"
Router(config)# interface g0/0
Router(config-if)# no shutdown

# Subinterface para VLAN 21 — TI
Router(config)# interface g0/0.21
Router(config-subif)# encapsulation dot1Q 21
Router(config-subif)# ip address 10.2.21.1 255.255.255.0

# Subinterface para VLAN 31 — Logística
Router(config)# interface g0/0.31
Router(config-subif)# encapsulation dot1Q 31
Router(config-subif)# ip address 10.2.31.1 255.255.255.0

# Subinterface para VLAN 99 — Gerência
Router(config)# interface g0/0.99
Router(config-subif)# encapsulation dot1Q 99
Router(config-subif)# ip address 10.2.99.1 255.255.255.0

# Subinterface para VLAN Nativa
Router(config)# interface g0/0.199
Router(config-subif)# encapsulation dot1Q 199 native
```

> 📸 *[Inserir screenshot das subinterfaces (`show ip interface brief`) aqui]*

---

### 7. Serviço DHCP

Com o roteamento inter-VLAN já funcional, o próximo passo foi configurar o **servidor DHCP diretamente no R-Core1**, eliminando a necessidade de atribuição manual de IPs nas VPCs e switches. Para cada VLAN, foi criado um pool DHCP com sua respectiva rede e gateway.

**Comandos aplicados:**

```bash
# Pool para VLAN 21 — TI
R-Core1(config)# ip dhcp pool VLAN21_POOL
R-Core1(dhcp-config)# network 10.2.21.0 255.255.255.0
R-Core1(dhcp-config)# default-router 10.2.21.1

# Pool para VLAN 31 — Logística
R-Core1(config)# ip dhcp pool VLAN31_POOL
R-Core1(dhcp-config)# network 10.2.31.0 255.255.255.0
R-Core1(dhcp-config)# default-router 10.2.31.1

# Pool para VLAN 99 — Gerência
R-Core1(config)# ip dhcp pool VLAN99_POOL
R-Core1(dhcp-config)# network 10.2.99.0 255.255.255.0
R-Core1(dhcp-config)# default-router 10.2.99.1
```


<img width="1365" height="767" alt="DCHP-POOL" src="https://github.com/user-attachments/assets/a413df40-c9f4-47c6-8821-8e78df7d3717" />


**Validação nas VPCs:**

Após a configuração, as VPCs foram configuradas para obter endereço via DHCP com o comando `ip dhcp`. O roteador respondeu corretamente, distribuindo IPs dentro da faixa de cada VLAN e informando o gateway correspondente — como ilustrado abaixo, onde a VPC recebeu `10.2.31.2/24` com gateway `10.2.31.1` (VLAN 31 — Logística) e conseguiu pingar o gateway com sucesso.

```bash
VPCS> ip dhcp
# Resultado: DDORA IP 10.2.31.2/24 GW 10.2.31.1

VPCS> ping 10.2.31.1
# 84 bytes from 10.2.31.1 — 5/5 pacotes bem-sucedidos
```
<img width="1365" height="767" alt="VPC6-VER" src="https://github.com/user-attachments/assets/da1f62af-9092-4103-bcde-896c0121ae58" />


Além das VPCs, as **SVIs (Switch Virtual Interfaces)** dos switches de distribuição e acesso também foram configuradas para obter endereço IP via DHCP, na VLAN 99 (Gerência). Isso permite o **gerenciamento remoto** de cada switch via Telnet/SSH, sem necessidade de IP estático em cada dispositivo.
 
Uma SVI é uma interface lógica criada dentro do switch associada a uma VLAN. Ela não corresponde a nenhuma porta física — é a "interface de gerenciamento" do switch na camada 3.
 
**Comandos aplicados (exemplo no DSW-01):**
 
```bash
DSW-01(config)# interface vlan 99
DSW-01(config-if)# ip address dhcp
DSW-01(config-if)# no shutdown
```
 
<img width="1365" height="767" alt="SVI-IP-CONFIG" src="https://github.com/user-attachments/assets/d6672e81-e944-4ddb-9dfa-bbf5f50a6738" />

 
**O que os logs do IOS confirmam, em sequência:**
 
1. `%LINEPROTO-5-UPDOWN: Interface Vlan99, changed state to down` — A SVI foi criada mas ainda não tinha conectividade (aguardando porta ativa na VLAN 99).
2. `%LINK-3-UPDOWN: Interface Vlan99, changed state to up` — Uma porta trunk com a VLAN 99 permitida subiu, ativando a SVI.
3. `%LINEPROTO-5-UPDOWN: Interface Vlan99, changed state to up` — O protocolo de linha ficou ativo.
4. `%DHCP-6-ADDRESS_ASSIGN: Interface Vlan99 assigned DHCP address 10.2.99.3, mask 255.255.255.0, hostname DSW-01` — O roteador respondeu ao DHCP Discover e atribuiu o IP `10.2.99.3/24` ao DSW-01.
**Aspectos técnicos relevantes:**
 
- **SVI (`interface vlan X`)** — Interface virtual que representa a VLAN na camada 3 do switch. Para subir (`up/up`), é necessário que pelo menos uma porta física no switch esteja ativa e pertencente àquela VLAN (seja em modo access ou trunk com a VLAN permitida).
- **`ip address dhcp`** — Faz a SVI se comportar como um cliente DHCP, enviando um *Discover* e aguardando oferta do servidor — no caso, o R-Core1.
- **VLAN 99 como VLAN de gerência** — Isolar o tráfego de gerenciamento em uma VLAN dedicada é uma prática de segurança recomendada, impedindo que usuários em VLANs de produção (TI, Logística) acessem diretamente as interfaces de administração dos switches.


> ✅ **Resultado:** Todas as VPCs as SVIs dos switches passaram a obter IPs automaticamente, confirmando que o servidor DHCP, as subinterfaces do Router-on-a-Stick e as VLANs estão operando de forma integrada.
---

## Configuração Passo a Passo

A seguir, a ordem recomendada de configuração para replicar este laboratório:

### Etapa 1 — Atribuição de hostnames, username e senha para as linhas (todos os switches)

### Etapa 2 — Criação e nomeação das VLANs (todos os switches)

### Etapa 3 — Configuração do EtherChannel (DSW-01 ↔ DSW-02)

### Etapa 4 — Configuração dos links trunk

### Etapa 5 — Configuração do Rapid-PVST+ e Root Bridge

### Etapa 6 — Configuração das portas de acesso (switches de acesso)

### Etapa 7 — Atribuir PortFast e BPDU Guard nas portas de acesso

### Etapa 8 — Configuração do Router-on-a-Stick e serviço DHCP

### Etapa 9 — Configuração dos gateways nas VPCs

---

## Verificação e Testes

### Comandos de verificação essenciais

```bash
# VLANs
show vlan brief
show interfaces trunk

# Spanning Tree
show spanning-tree
show spanning-tree vlan 21
show spanning-tree summary

# EtherChannel
show etherchannel summary
show etherchannel port-channel

# Roteamento
show ip interface brief
show ip route

# Conectividade entre hosts
ping 10.2.31.1        # VPC na VLAN 21 ou 31 pinga o gateway da VLAN 31
ping 10.2.21.1        # VPC na VLAN 21 ou 31 pinga o gateway da VLAN 21
ping 10.2.99.1        # Pinga gateway da gerência
```


---

## Conclusão Técnica

Este laboratório demonstra a integração de cinco pilares fundamentais da operação de redes corporativas em camada 2 e 3:

- **VLANs** isolam o tráfego por departamento, aumentando segurança e eficiência;
- **Rapid-PVST+** garante redundância sem loops, com convergência rápida em caso de falha;
- **EtherChannel** maximiza a largura de banda no core e elimina pontos únicos de falha entre os switches de distribuição;
- **Router-on-a-Stick** viabiliza a comunicação entre VLANs com um único link físico para o roteador.
- **DHCP centralizado** elimina configuração manual de IPs e viabiliza o gerenciamento remoto dos switches via SVIs.

A topologia está alinhada com os objetivos do **CCNA 200-301** e representa um modelo prático e escalável para ambientes reais de pequeno e médio porte.

---

> 💡 **Ferramentas utilizadas:** Eve-NG | Cisco IOSv | Cisco IOSvL2

> 📅 **Data:** Abril de 2026
