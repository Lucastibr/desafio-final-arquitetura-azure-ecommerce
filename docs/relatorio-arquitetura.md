# Relatório Arquitetural da Solução

## 1. Contexto

Este relatório descreve a arquitetura de solução em nuvem proposta para uma aplicação de e-commerce com operação 24/7, alta disponibilidade, escalabilidade, segurança, monitoramento e recuperação de desastres.

A arquitetura foi desenhada utilizando serviços da Microsoft Azure e considera uma aplicação distribuída, capaz de lidar com variações de demanda, falhas parciais de infraestrutura e necessidade de continuidade operacional.

A solução contempla múltiplas zonas de disponibilidade, balanceamento de carga, escalabilidade automática de máquinas virtuais Linux, banco de dados gerenciado PaaS, controle de acesso baseado em identidade, monitoramento centralizado e estratégia de Disaster Recovery em região secundária.

---

## 2. Objetivo da Solução

O objetivo da arquitetura é prover uma base cloud resiliente e escalável para uma aplicação de vendas online, garantindo:

* Alta disponibilidade da aplicação.
* Distribuição de carga entre múltiplas instâncias.
* Escalabilidade automática da camada computacional.
* Uso de banco de dados gerenciado.
* Segurança na comunicação entre aplicação e banco de dados.
* Controle de acesso baseado em identidade.
* Observabilidade da infraestrutura e da aplicação.
* Recuperação de desastres em caso de falha regional.

---

## 3. Cloud Provider

A cloud escolhida para a solução foi a **Microsoft Azure**.

A escolha se justifica pela disponibilidade de serviços gerenciados para entrada global, balanceamento regional, redes privadas, banco de dados PaaS, identidade, segurança, monitoramento e recuperação de desastres.

---

## 4. Visão Geral da Arquitetura

A arquitetura foi organizada em camadas, separando responsabilidades e reduzindo acoplamento entre os componentes.

As principais camadas são:

1. **Internet / Usuários**
2. **Camada Global / Edge**
3. **Camada Regional de Entrada**
4. **Virtual Network**
5. **Camada de Aplicação**
6. **Camada de Dados**
7. **Identidade e Segredos**
8. **Segurança de Rede**
9. **Observabilidade**
10. **Disaster Recovery**

O fluxo principal da solução segue o modelo:

```text
Usuários da Internet
    ↓
Azure Front Door + WAF Policy
    ↓
Application Gateway
    ↓
VM Scale Set Linux
    ↓
Private Endpoint / Private Link
    ↓
Azure SQL Database
    ↓
Geo-replicação para região secundária
```

---

## 5. Regiões Utilizadas

A arquitetura considera duas regiões Azure:

| Região                 | Função                                   |
| ---------------------- | ---------------------------------------- |
| Azure Brazil South     | Região primária da aplicação             |
| Azure Brazil Southeast | Região secundária para Disaster Recovery |

A região primária concentra a aplicação, rede, balanceamento, identidade, segurança, observabilidade e banco principal.

A região secundária é utilizada para recuperação de desastres, contendo a geo-réplica do Azure SQL Database, Auto-Failover Group e backups/PITR.

---

## 6. Componentes da Arquitetura

### 6.1 Azure Front Door + WAF Policy

O **Azure Front Door** atua como ponto de entrada global da aplicação.

Responsabilidades:

* Entrada HTTPS global.
* Roteamento na borda.
* Encaminhamento para origin regional.
* Proteção com WAF Policy.
* Redução de latência por meio de presença global.
* Proteção contra ameaças comuns de camada web.

A WAF Policy aplicada ao Front Door ajuda a proteger a aplicação contra ataques como injeção, exploração de vulnerabilidades comuns e tráfego malicioso em camada HTTP/HTTPS.

---

### 6.2 Application Gateway

O **Application Gateway** atua como balanceador regional de camada 7.

Responsabilidades:

* Receber tráfego do Azure Front Door.
* Atuar como origin regional da aplicação.
* Distribuir requisições para o VM Scale Set Linux.
* Realizar balanceamento baseado em HTTP/HTTPS.
* Utilizar health probes para verificar a saúde das instâncias.

O Application Gateway está posicionado em uma subnet dedicada:

```text
ApplicationGatewaySubnet — 10.0.1.0/24
```

Essa separação melhora a organização da rede e segue o princípio de segmentação por função.

---

### 6.3 Virtual Network

A arquitetura utiliza uma Virtual Network principal:

```text
Virtual Network — 10.0.0.0/16
```

A VNet é responsável por isolar logicamente os recursos de rede da aplicação e permitir segmentação por subnets.

Subnets consideradas:

| Subnet                   |        CIDR | Função                                 |
| ------------------------ | ----------: | -------------------------------------- |
| ApplicationGatewaySubnet | 10.0.1.0/24 | Subnet dedicada ao Application Gateway |
| Subnet Aplicação         | 10.0.2.0/24 | Subnet do VM Scale Set Linux           |
| Subnet Private Endpoints | 10.0.3.0/24 | Subnet dedicada ao Private Endpoint    |
| AzureBastionSubnet       | 10.0.4.0/27 | Subnet dedicada ao Azure Bastion       |
| AzureFirewallSubnet      | 10.0.5.0/26 | Subnet dedicada ao Azure Firewall      |

---

### 6.4 VM Scale Set Linux

A camada de aplicação utiliza **Virtual Machine Scale Set com Linux**.

Configuração considerada:

* Sistema operacional: Linux.
* Escalabilidade automática.
* Mínimo de instâncias: 3.
* Máximo de instâncias: 6.
* Distribuição entre 3 Availability Zones.
* Acesso administrativo sem SSH público.
* Integração com Managed Identity.

Distribuição lógica:

| Zona                | Instância |
| ------------------- | --------- |
| Availability Zone 1 | VM Linux  |
| Availability Zone 2 | VM Linux  |
| Availability Zone 3 | VM Linux  |

O uso de VM Scale Set permite ajustar automaticamente a quantidade de instâncias de acordo com a demanda da aplicação.

Métricas possíveis para autoscaling:

* CPU.
* Memória.
* Número de requisições.
* Métricas customizadas da aplicação.
* Latência.
* Fila de processamento, se aplicável.

---

### 6.5 Azure SQL Database

O banco de dados escolhido foi o **Azure SQL Database**, serviço PaaS gerenciado da Microsoft Azure.

Responsabilidades:

* Persistência dos dados da aplicação.
* Alta disponibilidade gerenciada.
* Backups automáticos.
* Point-in-Time Restore.
* Criptografia em repouso.
* Geo-replicação para região secundária.
* Integração com Microsoft Entra ID.

O Azure SQL Database não é representado dentro da VNet, pois é um serviço PaaS gerenciado. O acesso privado é realizado por meio de Private Endpoint.

---

### 6.6 Private Endpoint e Private Link

O acesso ao Azure SQL Database ocorre por meio de **Private Endpoint** e **Private Link**.

Fluxo:

```text
VM Scale Set Linux
    ↓
Private Endpoint
    ↓
Private Link
    ↓
Azure SQL Database
```

Benefícios:

* Evita exposição pública do banco.
* Mantém o tráfego em caminho privado.
* Reduz superfície de ataque.
* Permite controle de rede mais restritivo.
* Facilita segmentação e isolamento.

O Private Endpoint está localizado na subnet:

```text
Subnet Private Endpoints — 10.0.3.0/24
```

---

### 6.7 Private DNS Zone

A **Private DNS Zone** é utilizada para resolução DNS privada do Azure SQL Database por meio do Private Endpoint.

Ela garante que o nome do banco seja resolvido para o endereço privado associado ao Private Endpoint, permitindo que as VMs acessem o banco sem utilizar endpoint público.

---

### 6.8 Microsoft Entra ID

O **Microsoft Entra ID** é utilizado como provedor de identidade da arquitetura.

Responsabilidades:

* Controle de identidade.
* Autenticação.
* Integração com Managed Identity.
* Apoio à autenticação no Azure SQL Database.
* Governança de acesso aos recursos Azure.

---

### 6.9 Managed Identity

A arquitetura utiliza **Managed Identity** para permitir que o VM Scale Set acesse serviços Azure sem necessidade de credenciais fixas.

Uso previsto:

* Acesso ao Azure SQL Database com Entra ID Auth.
* Acesso ao Azure Key Vault.
* Redução de segredos hardcoded.
* Melhoria da segurança operacional.

No caso do Azure SQL Database, a permissão de leitura e escrita deve ser controlada por roles dentro do banco de dados, associadas à identidade gerenciada.

Modelo conceitual:

```text
VM Scale Set Linux
    ↓
Managed Identity
    ↓
Microsoft Entra ID Auth
    ↓
Database Roles
    ↓
Azure SQL Database
```

---

### 6.10 Azure Key Vault

O **Azure Key Vault** é utilizado para armazenamento seguro de segredos, certificados e informações sensíveis.

Exemplos de itens armazenados:

* Certificados TLS.
* Segredos de aplicação.
* Connection strings, quando necessárias.
* Chaves criptográficas.

O acesso ao Key Vault deve ser realizado por Managed Identity, evitando credenciais estáticas na aplicação.

---

### 6.11 Network Security Groups

A arquitetura utiliza **Network Security Groups** aplicados às subnets para controle de tráfego.

Objetivo:

* Restringir comunicação entre subnets.
* Permitir somente fluxos necessários.
* Bloquear acessos indevidos.
* Reduzir superfície de ataque.

NSGs são aplicados de forma lógica às subnets principais, como:

* ApplicationGatewaySubnet.
* Subnet Aplicação.
* Subnet Private Endpoints.

---

### 6.12 Azure Firewall

O **Azure Firewall** é utilizado para controle e inspeção de tráfego de saída e tráfego norte-sul, quando aplicável.

Ele está localizado em subnet dedicada:

```text
AzureFirewallSubnet — 10.0.5.0/26
```

Responsabilidades:

* Controle de egress.
* Inspeção de tráfego.
* Centralização de políticas de rede.
* Apoio à estratégia de segurança em profundidade.

---

### 6.13 Azure Bastion

O **Azure Bastion** é utilizado para acesso administrativo seguro às VMs Linux.

Ele está localizado em subnet dedicada:

```text
AzureBastionSubnet — 10.0.4.0/27
```

Benefícios:

* Acesso administrativo sem exposição de SSH público.
* Redução da superfície de ataque.
* Administração segura via portal Azure.
* Evita necessidade de IP público nas VMs.

---

### 6.14 Azure Monitor

O **Azure Monitor** é utilizado para coleta e análise de métricas da infraestrutura.

Responsabilidades:

* Monitorar recursos Azure.
* Acompanhar métricas de disponibilidade.
* Criar alertas.
* Apoiar análise operacional.

---

### 6.15 Log Analytics Workspace

O **Log Analytics Workspace** centraliza logs dos serviços e recursos da arquitetura.

Fontes de logs consideradas:

* Application Gateway.
* VM Scale Set Linux.
* Azure SQL Database.
* Azure Firewall.
* Azure Monitor.
* Logs de diagnóstico.

---

### 6.16 Application Insights

O **Application Insights** é utilizado para telemetria da aplicação.

Métricas e dados observáveis:

* Latência de requisições.
* Erros HTTP.
* Exceções.
* Dependências externas.
* Performance da aplicação.
* Disponibilidade da aplicação.

---

### 6.17 Disaster Recovery

A estratégia de Disaster Recovery considera uma região secundária:

```text
Azure Brazil Southeast
```

Componentes utilizados:

* Azure SQL Geo-replica.
* Auto-Failover Group.
* Backups automáticos.
* Point-in-Time Restore.

O objetivo é permitir recuperação do banco de dados em caso de falha relevante na região primária.

---

## 7. Fluxo de Tráfego da Aplicação

O fluxo principal da aplicação é:

1. O usuário acessa a aplicação pela Internet utilizando HTTPS.
2. A requisição chega ao Azure Front Door.
3. A WAF Policy realiza inspeção e proteção na borda.
4. O Azure Front Door encaminha a requisição para o Application Gateway como origin regional.
5. O Application Gateway distribui o tráfego para as instâncias Linux do VM Scale Set.
6. As VMs processam a lógica da aplicação.
7. A aplicação acessa o Azure SQL Database por meio de Private Endpoint.
8. A resolução privada do banco é feita pela Private DNS Zone.
9. Logs, métricas e telemetria são enviados para Azure Monitor, Log Analytics e Application Insights.

---

## 8. Alta Disponibilidade

A alta disponibilidade foi aplicada em múltiplas camadas.

### Entrada Global

O Azure Front Door fornece ponto de entrada global e reduz dependência de um único ponto regional de exposição.

### Entrada Regional

O Application Gateway atua como balanceador regional e utiliza health probes para encaminhar tráfego apenas para instâncias saudáveis.

### Camada de Aplicação

O VM Scale Set Linux está distribuído em três Availability Zones:

* AZ 1
* AZ 2
* AZ 3

Essa distribuição reduz o impacto de falhas em uma única zona.

### Banco de Dados

O Azure SQL Database é um serviço PaaS gerenciado com recursos de alta disponibilidade, backups e recuperação pontual.

### Disaster Recovery

A geo-replicação do banco para região secundária permite continuidade em caso de indisponibilidade regional.

---

## 9. Escalabilidade

A escalabilidade horizontal é implementada com VM Scale Set Linux.

Configuração proposta:

```text
Mínimo: 3 instâncias
Máximo: 6 instâncias
```

Essa configuração permite manter pelo menos uma instância por zona de disponibilidade e ampliar capacidade quando houver aumento de demanda.

Possíveis regras de autoscaling:

| Métrica                 | Ação                              |
| ----------------------- | --------------------------------- |
| CPU média acima de 70%  | Aumentar quantidade de instâncias |
| CPU média abaixo de 30% | Reduzir quantidade de instâncias  |
| Alta latência HTTP      | Aumentar capacidade               |
| Aumento de requisições  | Escalar horizontalmente           |

O limite máximo de 6 instâncias evita crescimento descontrolado de custo.

---

## 10. Segurança

A arquitetura segue princípios de segurança em profundidade.

### Segurança de Entrada

* HTTPS na entrada.
* WAF Policy no Azure Front Door.
* Application Gateway como camada regional L7.
* Exposição controlada dos serviços.

### Segurança de Rede

* Segmentação por subnets.
* NSG aplicado às subnets.
* Azure Firewall para controle e inspeção.
* Private Endpoint para acesso ao banco.
* Ausência de SSH público nas VMs.
* Administração via Azure Bastion.

### Segurança de Identidade

* Microsoft Entra ID.
* Managed Identity.
* Acesso ao Azure SQL com Entra ID Auth.
* Roles no banco para leitura/escrita.
* Acesso ao Key Vault sem credenciais fixas.

### Segurança de Dados

* Azure SQL Database PaaS.
* Criptografia em repouso.
* Backups automáticos.
* Point-in-Time Restore.
* Acesso privado via Private Link.

---

## 11. IAM e Controle de Acesso

O controle de acesso foi desenhado para reduzir uso de credenciais estáticas.

A estratégia é:

1. O VM Scale Set recebe uma Managed Identity.
2. A Managed Identity é reconhecida pelo Microsoft Entra ID.
3. A aplicação utiliza essa identidade para acessar o Azure SQL Database.
4. As permissões de leitura e escrita são atribuídas no banco por meio de roles.
5. A mesma identidade pode acessar segredos e certificados no Azure Key Vault, conforme permissões definidas.

Essa abordagem evita armazenamento de senhas na aplicação e melhora a rastreabilidade do acesso.

---

## 12. Monitoramento e Observabilidade

A observabilidade foi estruturada em três componentes principais:

| Serviço                 | Finalidade                                     |
| ----------------------- | ---------------------------------------------- |
| Azure Monitor           | Métricas, alertas e acompanhamento de recursos |
| Log Analytics Workspace | Centralização e consulta de logs               |
| Application Insights    | Telemetria da aplicação                        |

### Fontes monitoradas

* Azure Front Door.
* Application Gateway.
* VM Scale Set Linux.
* Azure SQL Database.
* Azure Firewall.
* Aplicação.
* Logs de diagnóstico.

### Alertas recomendados

* Falha em health probe do Application Gateway.
* Erros HTTP 5xx elevados.
* Latência acima do limite aceitável.
* CPU elevada nas VMs.
* Escalabilidade acionada com frequência.
* Falha de conexão com o banco.
* Consumo elevado no Azure SQL.
* Eventos de failover.
* Falhas de autenticação.
* Aumento anormal de tráfego bloqueado pelo WAF.

---

## 13. Disaster Recovery

A estratégia de recuperação de desastres é baseada em replicação do banco de dados para uma região secundária.

### Região Primária

```text
Azure Brazil South
```

### Região Secundária

```text
Azure Brazil Southeast
```

### Componentes de DR

* Azure SQL Geo-replica.
* Auto-Failover Group.
* Backups automáticos.
* Point-in-Time Restore.

### Fluxo de DR

1. O Azure SQL Database primário opera na região Azure Brazil South.
2. Uma geo-réplica é mantida na região Azure Brazil Southeast.
3. O Auto-Failover Group permite redirecionamento em caso de falha.
4. Backups/PITR permitem recuperação pontual de dados.
5. Em caso de desastre regional, a aplicação pode ser direcionada para uso do banco secundário.

### RTO e RPO estimados

Para fins arquiteturais, podem ser considerados:

| Indicador | Estimativa                                                                         |
| --------- | ---------------------------------------------------------------------------------- |
| RTO       | Baixo a moderado, dependendo do processo de failover e reconfiguração da aplicação |
| RPO       | Baixo, devido à geo-replicação e backups automáticos                               |

Esses valores devem ser refinados em um projeto real com base em requisitos de negócio, criticidade da aplicação, orçamento e testes de continuidade.

---

## 14. Justificativas Arquiteturais

### Uso de Azure Front Door

Foi escolhido para fornecer entrada global, roteamento de borda e proteção WAF antes da chegada do tráfego à região primária.

### Uso de Application Gateway

Foi utilizado como balanceador regional L7, permitindo distribuição de requisições HTTP/HTTPS para as instâncias da aplicação.

### Uso de VM Scale Set

Foi escolhido para atender ao requisito de escalabilidade dinâmica de VMs Linux com mínimo de 3 e máximo de 6 instâncias.

### Uso de Availability Zones

A distribuição das VMs em três zonas aumenta a resiliência contra falhas de datacenter/zona.

### Uso de Azure SQL Database

Foi escolhido por ser um banco gerenciado PaaS, reduzindo esforço operacional e oferecendo recursos de alta disponibilidade, backup e replicação.

### Uso de Private Endpoint

Foi escolhido para evitar exposição pública do banco de dados e manter a comunicação de dados por caminho privado.

### Uso de Managed Identity

Foi escolhido para evitar credenciais estáticas e melhorar a segurança no acesso da aplicação ao banco e ao Key Vault.

### Uso de Azure Monitor, Log Analytics e Application Insights

Foram escolhidos para garantir visibilidade operacional da infraestrutura, aplicação e banco de dados.

### Uso de Geo-replica e Auto-Failover Group

Foram escolhidos para suportar recuperação de desastres em região secundária.

---

## 15. Requisitos Atendidos

| Requisito                          | Implementação                                       |
| ---------------------------------- | --------------------------------------------------- |
| Múltiplas zonas de disponibilidade | VM Scale Set Linux distribuído em AZ 1, AZ 2 e AZ 3 |
| Balanceamento de carga             | Azure Front Door e Application Gateway              |
| Autoscaling de VMs                 | VM Scale Set com mínimo 3 e máximo 6                |
| Imagens Linux                      | VMs Linux na camada de aplicação                    |
| Banco PaaS                         | Azure SQL Database                                  |
| IAM para acesso ao banco           | Managed Identity + Entra ID Auth + DB Roles         |
| Segurança de comunicação           | Private Endpoint, Private Link e Private DNS Zone   |
| Segurança de rede                  | NSG, Azure Firewall e Bastion                       |
| Monitoramento                      | Azure Monitor, Log Analytics e Application Insights |
| Disaster Recovery                  | Geo-replica, Auto-Failover Group e Backups/PITR     |

---

## 16. Limitações e Considerações

Esta arquitetura representa uma proposta conceitual e arquitetural para o cenário do desafio.

Em um projeto real, ainda seria necessário detalhar:

* Estimativa de custos.
* Sizing das VMs.
* SKU dos serviços Azure.
* Políticas detalhadas de NSG.
* Regras específicas do Azure Firewall.
* Estratégia de CI/CD.
* Hardening do sistema operacional.
* Gestão de patches.
* Testes de carga.
* Testes de failover.
* Definição formal de RTO e RPO.
* Plano de continuidade operacional.
* Governança com Azure Policy.
* Gestão de tags e custos.

Esses pontos não foram aprofundados por estarem fora do escopo principal do desafio, cujo foco é o desenho e documentação da arquitetura de solução.

---

## 17. Conclusão

A arquitetura proposta atende aos requisitos de uma aplicação de e-commerce 24/7 em nuvem, utilizando serviços da Microsoft Azure para alta disponibilidade, escalabilidade, segurança, observabilidade e recuperação de desastres.

A solução utiliza múltiplas zonas de disponibilidade, balanceamento de carga, VM Scale Set Linux com autoscaling, banco de dados gerenciado PaaS, identidade gerenciada, acesso privado ao banco, monitoramento centralizado e replicação para região secundária.

Com isso, a arquitetura oferece uma base sólida para aplicações distribuídas que exigem resiliência, continuidade operacional e capacidade de adaptação a variações de demanda.
