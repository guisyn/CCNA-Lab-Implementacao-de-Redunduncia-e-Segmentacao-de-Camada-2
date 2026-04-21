# CCNA-Lab-Implementação-de-Redundância-e-Segmentação-de-Camada-2
Demonstrar a configuração de uma topologia resiliente utilizando EtherChannel (LACP), Rapid-PVST+ para prevenção de loops, segmentação de rede, roteamento inter-VLAN via Router-on-a-Stick e serviço DHCP.

> **Ambiente:** Eve-NG | **Nível:** CCNA (200-301) | **Autor:** [Seu Nome]

---

## 📋 Índice

1. [Visão Geral](#visão-geral)
2. [Topologia](#topologia)
3. [Endereçamento IP e VLANs](#endereçamento-ip-e-vlans)
4. [Tecnologias Utilizadas](#tecnologias-utilizadas)
   - [VLANs](#1-vlans-virtual-local-area-networks)
   - [Rapid-PVST+](#2-rapid-pvst-rapid-per-vlan-spanning-tree)
   - [EtherChannel L2](#3-etherchannel-l2)
   - [Router-on-a-Stick](#4-router-on-a-stick)
5. [Configuração Passo a Passo](#configuração-passo-a-passo)
6. [Verificação e Testes](#verificação-e-testes)
7. [Conclusão Técnica](#conclusão-técnica)

---

## Visão Geral

Este laboratório simula um ambiente corporativo de médio porte, implementado no emulador **Eve-NG**, com foco nas tecnologias cobradas no exame **CCNA 200-301** da Cisco. O objetivo é demonstrar, de forma prática, como segmentar uma rede com VLANs, garantir redundância de camada 2 com Rapid-PVST+, aumentar a largura de banda entre switches com EtherChannel e prover roteamento inter-VLAN via Router-on-a-Stick.

A topologia reflete cenários reais de redes corporativas, onde diferentes departamentos (TI, Logística, Gerência) precisam de isolamento lógico, mas compartilham a mesma infraestrutura física.

---

## Topologia

![Topologia Eve-NG](./topologia.png)

> *Diagrama gerado no Eve-NG. Os switches de distribuição formam o núcleo da rede, conectados ao roteador para o roteamento inter-VLAN.*

### Equipamentos utilizados

| Dispositivo | Quantidade | Função |
|---|---|---|
| Router Cisco (IOSv) | 1 | Roteamento inter-VLAN (Router-on-a-Stick) |
| Switch L2 Distribuição | 2 | DSW-01, DSW-02 — Camada de distribuição |
| Switch L2 Acesso | 4 | ASW-01, ASW-02, ASW-03, ASW-04 — Camada de acesso |
| VPCs (hosts) | 8+ | VPC_C, VPC_F, VPC_G, VPC_H, VPC_I, VPC_Y, VPC_1... |

---

## Endereçamento IP e VLANs

### VLANs Configuradas

| VLAN ID | Nome | Descrição |
|---|---|---|
| 21 | TI | Departamento de Tecnologia da Informação |
| 30 | LOGISTICA | Departamento de Logística |
| 99 | GERENCIA | Departamento de Gerência |
| 199 | NATIVA | VLAN Nativa (trunk) |

### Gateways (Subinterfaces no Roteador)

| VLAN | Gateway |
|---|---|
| VLAN 21 — TI | 10.2.21.1/24 |
| VLAN 30 — LOGÍSTICA | 10.2.31.1/24 |
| VLAN 99 — GERÊNCIA | 10.2.99.1/24 |

### Hosts (VPCs)

| Host | VLAN | IP |
|---|---|---|
| VPC_6 | — | 10.2.21.2/24|
| VPC_7 | — | 10.2.31.2/24|
| VPC_8 | — | 10.2.21.3/24|
| VPC_9 | — | 10.2.31.3/24|

| DSW-01 (SVI) | — | 10.2.99.3/24 |
| DSW-02 (SVI) | — | 10.2.99.2/24 |
| ASW-01 (SVI) | — | 10.2.99.5/24 |
| ASW-02 (SVI) | — | 10.2.99.4/24 |

---

## Tecnologias Utilizadas

### 1. VLANs (Virtual Local Area Networks)

**O que é:** Uma VLAN é uma subdivisão lógica de uma rede física. Dispositivos em VLANs diferentes não se comunicam diretamente — o tráfego precisa passar por um roteador (ou switch L3). Isso proporciona segurança, organização e controle de broadcast.

**Por que foi usada:** Para segmentar os departamentos (TI, Logística, Gerência) em domínios de broadcast independentes, aumentando a segurança e reduzindo tráfego desnecessário.

**Comandos principais:**

```bash
# Criação de VLANs no switch
Switch(config)# vlan 21
Switch(config-vlan)# name TI

Switch(config)# vlan 30
Switch(config-vlan)# name LOGISTICA

Switch(config)# vlan 99
Switch(config-vlan)# name GERENCIA

Switch(config)# vlan 199
Switch(config-vlan)# name NATIVA
```

```bash
# Porta de acesso (host)
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 21

# Porta trunk (entre switches e para o roteador)
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk native vlan 199
Switch(config-if)# switchport trunk allowed vlan 21,30,99,199
```

> 📸 *[Inserir screenshot da configuração de VLANs aqui]*

---

### 2. Rapid-PVST+ (Rapid Per-VLAN Spanning Tree)

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
DSW-01(config)# spanning-tree vlan 21,30 root primary
DSW-01(config)# spanning-tree vlan 99 root secondary

DSW-02(config)# spanning-tree vlan 99 root primary
DSW-02(config)# spanning-tree vlan 21,30 root secondary

# Ajuste de prioridade manual (múltiplo de 4096)
Switch(config)# spanning-tree vlan 21 priority 4096
```

> 📸 *[Inserir screenshot do `show spanning-tree` aqui]*

---

### 3. EtherChannel L2

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
DSW-01(config-if-range)# switchport mode trunk
DSW-01(config-if-range)# switchport trunk native vlan 199
DSW-01(config-if-range)# switchport trunk allowed vlan 21,30,99,199

# Configurar a interface lógica (Port-Channel)
DSW-01(config)# interface port-channel 1
DSW-01(config-if)# switchport mode trunk
DSW-01(config-if)# switchport trunk native vlan 199
DSW-01(config-if)# switchport trunk allowed vlan 21,30,99,199
```

> 📸 *[Inserir screenshot do `show etherchannel summary` aqui]*

---

### 4. Router-on-a-Stick

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

# Subinterface para VLAN 30 — Logística
Router(config)# interface g0/0.30
Router(config-subif)# encapsulation dot1Q 30
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

### Etapa 8 — Configuração do Servidor DHCP no Roteador

Com o roteamento inter-VLAN já funcional, o próximo passo foi configurar o **servidor DHCP diretamente no R-Core1**, eliminando a necessidade de atribuição manual de IPs nas VPCs. Para cada VLAN, foi criado um pool DHCP com sua respectiva rede e gateway.

**Comandos aplicados:**

```bash
# Pool para VLAN 21 — TI
R-Core1(config)# ip dhcp pool VLAN21_POOL
R-Core1(dhcp-config)# network 10.2.21.0 255.255.255.0
R-Core1(dhcp-config)# default-router 10.2.21.1

# Pool para VLAN 30 — Logística
R-Core1(config)# ip dhcp pool VLAN31_POOL
R-Core1(dhcp-config)# network 10.2.31.0 255.255.255.0
R-Core1(dhcp-config)# default-router 10.2.31.1

# Pool para VLAN 99 — Gerência
R-Core1(config)# ip dhcp pool VLAN99_POOL
R-Core1(dhcp-config)# network 10.2.99.0 255.255.255.0
R-Core1(dhcp-config)# default-router 10.2.99.1
```

> 📸 *[Inserir screenshot da configuração dos pools DHCP no R-Core1 aqui]*

**Validação nas VPCs:**

Após a configuração, as VPCs foram configuradas para obter endereço via DHCP com o comando `ip dhcp`. O roteador respondeu corretamente, distribuindo IPs dentro da faixa de cada VLAN e informando o gateway correspondente — como ilustrado abaixo, onde a VPC recebeu `10.2.31.2/24` com gateway `10.2.31.1` (VLAN 30 — Logística) e conseguiu pingar o gateway com sucesso.

```bash
VPCS> ip dhcp
# Resultado: DDORA IP 10.2.31.2/24 GW 10.2.31.1

VPCS> ping 10.2.31.1
# 84 bytes from 10.2.31.1 — 5/5 pacotes bem-sucedidos
```

> 📸 *[Inserir screenshot do `ip dhcp` e ping na VPC aqui]*

> ✅ **Resultado:** Todas as VPCs passaram a obter IPs automaticamente, confirmando que o servidor DHCP, as subinterfaces do Router-on-a-Stick e as VLANs estão operando de forma integrada.
---

## Configuração Passo a Passo

A seguir, a ordem recomendada de configuração para replicar este laboratório:

### Etapa 1 — Criação e nomeação das VLANs (todos os switches)
> 📸 *[Screenshot aqui]*

### Etapa 2 — Configuração das portas de acesso (switches de acesso)
> 📸 *[Screenshot aqui]*

### Etapa 3 — Configuração dos trunks entre switches
> 📸 *[Screenshot aqui]*

### Etapa 4 — Configuração do EtherChannel (DSW-01 ↔ DSW-02)
> 📸 *[Screenshot aqui]*

### Etapa 5 — Configuração do Rapid-PVST+ e Root Bridge
> 📸 *[Screenshot aqui]*

### Etapa 6 — Configuração do trunk no uplink para o roteador
> 📸 *[Screenshot aqui]*

### Etapa 7 — Configuração do Router-on-a-Stick
> 📸 *[Screenshot aqui]*

### Etapa 8 — Configuração dos gateways nas VPCs
> 📸 *[Screenshot aqui]*

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
ping 10.2.31.1        # VPC na VLAN 21 pinga o gateway da VLAN 30
ping 10.2.99.1        # Pinga gateway da gerência
```

> 📸 *[Screenshot dos pings inter-VLAN aqui]*

---

## Conclusão Técnica

Este laboratório demonstra a integração de quatro pilares fundamentais da operação de redes corporativas em camada 2 e 3:

- **VLANs** isolam o tráfego por departamento, aumentando segurança e eficiência;
- **Rapid-PVST+** garante redundância sem loops, com convergência rápida em caso de falha;
- **EtherChannel** maximiza a largura de banda no core e elimina pontos únicos de falha entre os switches de distribuição;
- **Router-on-a-Stick** viabiliza a comunicação entre VLANs com um único link físico para o roteador.

A topologia está alinhada com os objetivos do **CCNA 200-301** e representa um modelo prático e escalável para ambientes reais de pequeno e médio porte.

---

> 💡 **Ferramentas utilizadas:** Eve-NG | Cisco IOSv | Cisco IOSvL2

> 📅 **Data:** Abril de 2026
