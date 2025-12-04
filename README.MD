# Tuning TCP/IP para TrueNAS/SMB 

## 1. Introdução
Este documento descreve ajustes avançados de TCP/IP aplicáveis em sistemas Linux, originalmente configurados em uma VM TrueNAS para otimização de servidores SMB. O objetivo principal é melhorar a estabilidade de rede, reduzir perda de pacotes e aumentar o desempenho em ambientes com múltiplos usuários, especialmente em conexões via Wi-Fi.

Embora estas configurações tenham sido testadas em servidores NAS, elas são amplamente aplicáveis em qualquer cenário que envolva:

Servidores de arquivos (SMB, NFS) com múltiplos clientes simultâneos;

Servidores web e aplicações em tempo real com alto número de conexões simultâneas;

Bancos de dados e replicações que exigem alta confiabilidade TCP;

Servidores de e-mail com grande volume de contas ativas;

Soluções de backup e sincronização de dados de grande porte;

Serviços de VPN e túneis de rede com múltiplos clientes conectados;

Ambientes virtualizados ou containers com comunicação intensiva entre hosts ou VMs.

As otimizações contemplam filas de rede, buffers TCP, parâmetros de KeepAlive, backlog de conexões, retransmissões e gerenciamento de memória para muitas conexões simultâneas.
Essas alterações permitem que o sistema lide de forma mais eficiente com tráfego intenso, múltiplas conexões simultâneas, latência variável e transferências de grandes volumes de dados, resultando em maior desempenho, estabilidade e confiabilidade.

---

## 2. Objetivo
- Garantir maior estabilidade nas operações SMB.  
- Reduzir descartes de pacotes e resets de conexão.  
- Melhorar throughput em transferências de grande volume.  
- Maximizar eficiência do TCP em cenários com múltiplas conexões e tráfego variável.

---

## 3. Resumo dos Ajustes

O documento apresenta os ajustes em três grandes categorias:

1. **Filas de rede e offloads:** otimização do txqueuelen, ring buffers, multiqueue e offloads TCP.  
2. **Buffers e autotuning TCP:** aumento dos buffers de envio/recepção e definição de limites máximos, permitindo melhor performance do autotuning TCP.  
3. **Parâmetros TCP complementares:** backlog, KeepAlive, TIME_WAIT, retransmissões e gestão de memória.

---

## 4. Ajustes Detalhados

### 4.1 Filas de Rede e Offloads

| Comando | Comentário / Objetivo |
|---------|---------------------|
| `ip link set eth0 txqueuelen 3000` | Aumenta a fila de transmissão do kernel para reduzir descartes de pacotes. |
| `ip link show eth0` | Verifica o valor atual do txqueuelen. |
| `ethtool -G eth0 rx 4096 tx 4096` | Ajusta ring buffers RX/TX (se suportado pelo driver). |
| `ethtool -L eth0 combined 4` | Define 4 filas combinadas (multiqueue) para melhor distribuição de tráfego. |
| `ethtool -K eth0 gro on` | Habilita Generic Receive Offload, reduzindo CPU em recebimento de pacotes. |
| `ethtool -K eth0 gso on` | Habilita Generic Segmentation Offload, reduzindo CPU em envio de pacotes grandes. |
| `ethtool -K eth0 tso on` | Habilita TCP Segmentation Offload para envio eficiente de pacotes grandes. |

---

### 4.2 Buffers TCP e Autotuning

| Parâmetro | Valor | Comentário / Objetivo |
|-----------|-------|---------------------|
| `net.core.rmem_max` | 268435456 | Máximo buffer de recepção do kernel. |
| `net.core.wmem_max` | 268435456 | Máximo buffer de envio do kernel. |
| `net.core.rmem_default` | 26214400 | Buffer de recepção padrão. |
| `net.core.wmem_default` | 26214400 | Buffer de envio padrão. |
| `net.ipv4.tcp_rmem` | 4096 87380 67108864 | Autotuning TCP de recepção: min, default, max. |
| `net.ipv4.tcp_wmem` | 4096 65536 67108864 | Autotuning TCP de envio: min, default, max. |

---

### 4.3 Backlog de Conexões, KeepAlive e TIME_WAIT

| Parâmetro | Valor | Comentário / Objetivo |
|-----------|-------|---------------------|
| `net.core.somaxconn` | 4096 | Máximo de conexões pendentes em listen(). |
| `net.ipv4.tcp_max_syn_backlog` | 4096 | Máximo de SYNs pendentes aguardando accept(). |
| `net.ipv4.tcp_keepalive_time` | 30 | Tempo de inatividade antes do primeiro KeepAlive. |
| `net.ipv4.tcp_keepalive_intvl` | 10 | Intervalo entre sondas KeepAlive. |
| `net.ipv4.tcp_keepalive_probes` | 5 | Tentativas antes de considerar socket morto. |
| `net.ipv4.tcp_fin_timeout` | 15 | Tempo que socket fica em FIN-WAIT-2 antes de fechar. |
| `net.ipv4.tcp_tw_reuse` | 1 | Permite reutilizar sockets em TIME_WAIT. |

---

### 4.4 Retransmissões TCP

| Parâmetro | Valor | Comentário / Objetivo |
|-----------|-------|---------------------|
| `net.ipv4.tcp_retries1` | 3 | Tentativas iniciais de retransmissão TCP. |
| `net.ipv4.tcp_retries2` | 8 | Tentativas totais de retransmissão TCP antes de fechar conexão. |

---

### 4.5 Gestão de Memória TCP para Muitas Conexões

| Parâmetro | Valor | Comentário / Objetivo |
|-----------|-------|---------------------|
| `net.ipv4.tcp_max_orphans` | 8192 | Máximo de sockets órfãos (sem processo) permitidos. |
| `net.ipv4.tcp_orphan_retries` | 2 | Tentativas de retransmissão para sockets órfãos. |
| `net.ipv4.tcp_max_tw_buckets` | 262144 | Máximo de sockets em TIME_WAIT simultâneos. |

---

## 5. Persistência das Configurações

Adicionar todos os parâmetros ao `/etc/sysctl.conf` para garantir que sejam aplicados após reinicialização:

```bash
cat <<EOF >> /etc/sysctl.conf
# Filas de rede e buffers TCP
net.core.rmem_max=268435456
net.core.wmem_max=268435456
net.core.rmem_default=26214400
net.core.wmem_default=26214400
net.ipv4.tcp_rmem=4096 87380 67108864
net.ipv4.tcp_wmem=4096 65536 67108864

# Backlog e KeepAlive
net.core.somaxconn=4096
net.ipv4.tcp_max_syn_backlog=4096
net.ipv4.tcp_keepalive_time=30
net.ipv4.tcp_keepalive_intvl=10
net.ipv4.tcp_keepalive_probes=5

# TIME_WAIT e reutilização
net.ipv4.tcp_fin_timeout=15
net.ipv4.tcp_tw_reuse=1

# Retransmissões
net.ipv4.tcp_retries1=3
net.ipv4.tcp_retries2=8

# Gestão de memória
net.ipv4.tcp_max_orphans=8192
net.ipv4.tcp_orphan_retries=2
net.ipv4.tcp_max_tw_buckets=262144
EOF

sysctl --system   # Aplica todas as configurações persistentes
