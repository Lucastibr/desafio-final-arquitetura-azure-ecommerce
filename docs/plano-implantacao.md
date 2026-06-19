# Plano de Implantação da Arquitetura

## 1. Objetivo

Este documento apresenta o plano lógico de implantação da arquitetura de e-commerce 24/7 proposta para Microsoft Azure.

O objetivo é descrever a sequência recomendada para provisionamento dos recursos necessários para suportar:

* Alta disponibilidade.
* Escalabilidade automática.
* Balanceamento de carga.
* Banco de dados gerenciado PaaS.
* Segurança de rede.
* Controle de acesso com identidade gerenciada.
* Monitoramento e observabilidade.
* Recuperação de desastres.

Este plano representa uma proposta arquitetural de implantação. A execução prática pode ser realizada pelo Portal Azure, Azure CLI, Azure PowerShell, Terraform, Bicep ou pipelines de CI/CD.

---

## 2. Escopo

O plano contempla os principais recursos da arquitetura:

* Resource Group.
* Virtual Network.
* Subnets.
* Network Security Groups.
* Azure Firewall.
* Azure Bastion.
* Azure Front Door com WAF Policy.
* Application Gateway.
* VM Scale Set Linux.
* Autoscaling entre 3 e 6 instâncias.
* Azure SQL Database.
* Private Endpoint.
* Private DNS Zone.
* Managed Identity.
* Azure Key Vault.
* Azure Monitor.
* Log Analytics Workspace.
* Application Insights.
* Azure SQL Geo-replica.
* Auto-Failover Group.
* Backups/PITR.

---

## 3. Premissas da Implantação

A arquitetura considera as seguintes premissas:

| Item                        | Definição                                            |
| --------------------------- | ---------------------------------------------------- |
| Cloud provider              | Microsoft Azure                                      |
| Região primária             | Azure Brazil South                                   |
| Região secundária           | Azure Brazil Southeast                               |
| Sistema operacional das VMs | Linux                                                |
| Modelo de computação        | VM Scale Set                                         |
| Mínimo de instâncias        | 3                                                    |
| Máximo de instâncias        | 6                                                    |
| Banco de dados              | Azure SQL Database                                   |
| Modelo do banco             | PaaS gerenciado                                      |
| Acesso ao banco             | Private Endpoint / Private Link                      |
| Administração das VMs       | Azure Bastion                                        |
| SSH público                 | Não permitido                                        |
| Identidade                  | Managed Identity + Microsoft Entra ID                |
| Monitoramento               | Azure Monitor + Log Analytics + Application Insights |
| Recuperação de desastres    | Geo-replica + Auto-Failover Group + Backups/PITR     |

---

## 4. Estratégia Geral de Implantação

A implantação deve seguir uma ordem lógica, respeitando dependências entre os serviços.

Ordem recomendada:

1. Criar Resource Group.
2. Criar Virtual Network.
3. Criar subnets.
4. Criar Network Security Groups.
5. Criar Azure Firewall.
6. Criar Azure Bastion.
7. Criar Log Analytics Workspace.
8. Criar Application Insights.
9. Criar Azure Key Vault.
10. Criar Azure SQL Database.
11. Criar Private DNS Zone.
12. Criar Private Endpoint para o Azure SQL Database.
13. Criar ou habilitar Managed Identity.
14. Criar VM Scale Set Linux.
15. Associar Managed Identity ao VM Scale Set.
16. Configurar permissões da Managed Identity no banco.
17. Configurar Application Gateway.
18. Configurar Azure Front Door com WAF Policy.
19. Configurar autoscaling do VM Scale Set.
20. Configurar Azure SQL Geo-replica.
21. Configurar Auto-Failover Group.
22. Configurar monitoramento, logs e alertas.
23. Validar tráfego, segurança, escalabilidade e DR.

---

## 5. Resource Group

O primeiro passo é criar um Resource Group para organizar os recursos da solução.

Sugestão de nome:

```text
rg-ecommerce-prod-brazilsouth
```

Finalidades do Resource Group:

* Agrupar recursos da arquitetura.
* Facilitar governança.
* Facilitar controle de acesso.
* Facilitar acompanhamento de custos.
* Organizar recursos por ambiente e projeto.

Tags recomendadas:

| Tag         | Valor sugerido |
| ----------- | -------------- |
| Project     | ecommerce      |
| Environment | production     |
| Owner       | architecture   |
| CostCenter  | academic       |
| Region      | brazilsouth    |

---

## 6. Virtual Network e Subnets

Criar a Virtual Network principal da solução.

```text
vnet-ecommerce-prod
CIDR: 10.0.0.0/16
Região: Azure Brazil South
```

Subnets propostas:

| Subnet                   |        CIDR | Finalidade                             |
| ------------------------ | ----------: | -------------------------------------- |
| ApplicationGatewaySubnet | 10.0.1.0/24 | Subnet dedicada ao Application Gateway |
| Subnet Aplicação         | 10.0.2.0/24 | Subnet do VM Scale Set Linux           |
| Subnet Private Endpoints | 10.0.3.0/24 | Subnet dedicada aos Private Endpoints  |
| AzureBastionSubnet       | 10.0.4.0/27 | Subnet dedicada ao Azure Bastion       |
| AzureFirewallSubnet      | 10.0.5.0/26 | Subnet dedicada ao Azure Firewall      |

A segmentação por subnets permite separar responsabilidades, aplicar regras de segurança específicas e reduzir a superfície de ataque.

---

## 7. Network Security Groups

Criar Network Security Groups para controlar o tráfego das subnets.

NSGs sugeridos:

| NSG                     | Associação               |
| ----------------------- | ------------------------ |
| nsg-application-gateway | ApplicationGatewaySubnet |
| nsg-app                 | Subnet Aplicação         |
| nsg-private-endpoints   | Subnet Private Endpoints |

Regras conceituais recomendadas:

| Origem                  | Destino             | Porta/Protocolo | Ação     | Justificativa                          |
| ----------------------- | ------------------- | --------------- | -------- | -------------------------------------- |
| AzureFrontDoor.Backend (service tag) | Application Gateway | HTTPS/443 | Permitir | Entrada regional; validar header X-Azure-FDID |
| Application Gateway     | VM Scale Set        | HTTP/HTTPS      | Permitir | Balanceamento para camada de aplicação |
| VM Scale Set            | Private Endpoint    | TCP/1433        | Permitir | Acesso privado ao Azure SQL Database   |
| Internet                | VM Scale Set        | SSH/22          | Negar    | VMs não devem expor SSH público        |
| Subnets não autorizadas | Subnet Aplicação    | Qualquer        | Negar    | Redução de tráfego lateral indevido    |

As regras exatas devem ser refinadas conforme endereços, portas e necessidades reais da aplicação.

---

## 8. Azure Firewall

Criar o Azure Firewall na subnet dedicada:

```text
AzureFirewallSubnet — 10.0.5.0/26
```

Responsabilidades:

* Controlar tráfego de saída.
* Centralizar políticas de rede.
* Inspecionar tráfego quando aplicável.
* Restringir comunicação com destinos externos.
* Apoiar a estratégia de segurança em profundidade.

Configurações recomendadas:

* Criar Azure Firewall Policy.
* Definir regras de aplicação para destinos permitidos.
* Definir regras de rede apenas quando necessário.
* Enviar logs para o Log Analytics Workspace.
* Aplicar rotas quando for necessário forçar tráfego pelo firewall.

---

## 9. Azure Bastion

Criar o Azure Bastion na subnet dedicada:

```text
AzureBastionSubnet — 10.0.4.0/27
```

Responsabilidades:

* Permitir acesso administrativo seguro às VMs.
* Evitar exposição de SSH público.
* Permitir acesso às VMs pelo Portal Azure.
* Reduzir a superfície de ataque da camada de aplicação.

Validações esperadas:

* VMs sem IP público.
* Porta SSH não exposta para Internet.
* Acesso administrativo realizado via Azure Bastion.

---

## 10. Observabilidade Base

Criar os recursos centrais de observabilidade antes da implantação dos serviços principais.

Recursos:

* Log Analytics Workspace.
* Application Insights.
* Azure Monitor.
* Configurações de diagnóstico.

Sugestão de nomes:

```text
law-ecommerce-prod
appi-ecommerce-prod
```

Fontes de logs e métricas previstas:

* Azure Front Door.
* Application Gateway.
* VM Scale Set Linux.
* Azure SQL Database.
* Azure Firewall.
* Azure Bastion.
* Aplicação.

---

## 11. Azure Key Vault

Criar um Azure Key Vault para armazenamento seguro de segredos, certificados e informações sensíveis.

Sugestão de nome:

```text
kv-ecommerce-prod
```

Itens que podem ser armazenados:

* Certificados TLS.
* Segredos de aplicação.
* Connection strings, quando necessárias.
* Chaves criptográficas.

Boas práticas:

* Permitir acesso ao Key Vault por Managed Identity.
* Evitar segredos hardcoded na aplicação.
* Habilitar logs de auditoria.
* Aplicar princípio do menor privilégio.
* Restringir acesso administrativo.

---

## 12. Azure SQL Database

Criar o banco de dados gerenciado como serviço PaaS.

Sugestão de nome:

```text
sqldb-ecommerce-prod
```

Características recomendadas:

* Alta disponibilidade gerenciada.
* Backups automáticos.
* Point-in-Time Restore.
* Criptografia em repouso.
* Autenticação integrada ao Microsoft Entra ID.
* Acesso privado via Private Endpoint.
* Geo-replicação para região secundária.

O Azure SQL Database deve ser tratado como serviço PaaS externo à VNet. A comunicação privada com a VNet será feita por Private Endpoint.

---

## 13. Private Endpoint e Private DNS Zone

Criar um Private Endpoint para o Azure SQL Database na subnet:

```text
Subnet Private Endpoints — 10.0.3.0/24
```

Também deve ser criada e associada uma Private DNS Zone para resolução privada do banco.

Objetivos:

* Permitir que a aplicação acesse o banco por IP privado.
* Evitar exposição pública desnecessária do banco.
* Garantir que a resolução DNS do banco aponte para o Private Endpoint.
* Manter a comunicação entre aplicação e banco por caminho privado.

Fluxo esperado:

```text
VM Scale Set Linux
    ↓
Private Endpoint
    ↓
Private Link
    ↓
Azure SQL Database
```

Validações esperadas:

* A VM resolve o FQDN do banco para IP privado.
* A aplicação conecta ao banco via Private Link.
* O endpoint público do banco fica restrito ou desabilitado, conforme política adotada.

---

## 14. Managed Identity

Criar ou habilitar Managed Identity para o VM Scale Set Linux.

Objetivo:

* Permitir acesso seguro ao Azure SQL Database.
* Permitir acesso seguro ao Azure Key Vault.
* Evitar credenciais fixas na aplicação.
* Integrar autenticação com Microsoft Entra ID.

Modelo de acesso ao banco:

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

As permissões de leitura e escrita devem ser controladas dentro do Azure SQL Database por meio de roles de banco, e não apenas por RBAC genérico.

---

## 15. VM Scale Set Linux

Criar o VM Scale Set responsável pela camada de aplicação.

Configuração proposta:

| Item                 | Valor                          |
| -------------------- | ------------------------------ |
| Sistema operacional  | Linux                          |
| Mínimo de instâncias | 3                              |
| Máximo de instâncias | 6                              |
| Distribuição         | Availability Zones 1, 2 e 3    |
| IP público nas VMs   | Não                            |
| Administração        | Azure Bastion                  |
| Identidade           | Managed Identity               |
| Subnet               | Subnet Aplicação — 10.0.2.0/24 |

O VM Scale Set deve ser associado ao backend pool do Application Gateway.

Distribuição esperada:

| Availability Zone | Instância mínima |
| ----------------- | ---------------- |
| AZ 1              | 1 VM Linux       |
| AZ 2              | 1 VM Linux       |
| AZ 3              | 1 VM Linux       |

Essa distribuição garante que a aplicação permaneça disponível mesmo diante de falha em uma zona de disponibilidade.

---

## 16. Autoscaling

Configurar autoscaling do VM Scale Set.

Política sugerida:

| Condição                               | Ação                  |
| -------------------------------------- | --------------------- |
| CPU média acima de 70% por 10 minutos  | Adicionar 1 instância |
| CPU média abaixo de 30% por 15 minutos | Remover 1 instância   |
| Mínimo de instâncias                   | 3                     |
| Máximo de instâncias                   | 6                     |

Métricas adicionais que podem ser utilizadas:

* Requisições por segundo.
* Latência média.
* Uso de memória.
* Métricas customizadas da aplicação.

O limite mínimo de 3 instâncias preserva a distribuição entre Availability Zones. O limite máximo de 6 instâncias evita crescimento descontrolado de custo.

---

## 17. Application Gateway

Criar o Application Gateway na subnet dedicada:

```text
ApplicationGatewaySubnet — 10.0.1.0/24
```

Responsabilidades:

* Receber tráfego encaminhado pelo Azure Front Door.
* Atuar como origin regional.
* Realizar balanceamento L7.
* Encaminhar requisições para o VM Scale Set.
* Executar health probes nas instâncias.

Configurações recomendadas:

* Listener HTTPS.
* Backend pool apontando para o VM Scale Set.
* Health probes.
* Regras de roteamento.
* Diagnóstico enviado para Log Analytics Workspace.

Fluxo esperado:

```text
Azure Front Door + WAF Policy
    ↓
Application Gateway
    ↓
VM Scale Set Linux
```

---

## 18. Azure Front Door + WAF Policy

Criar o Azure Front Door como camada global de entrada da aplicação.

Responsabilidades:

* Entrada global HTTPS.
* Roteamento na borda.
* Proteção com WAF Policy.
* Encaminhamento para o Application Gateway como origin regional.
* Melhoria de disponibilidade e performance percebida pelos usuários.

Configurações recomendadas:

* Endpoint público global.
* Origin configurado para o Application Gateway.
* WAF Policy associada.
* TLS habilitado.
* Logs enviados para Log Analytics Workspace.

Fluxo esperado:

```text
Usuários da Internet
    ↓
Azure Front Door + WAF Policy
    ↓
HTTPS para origin regional
    ↓
Application Gateway
```

---

## 19. Disaster Recovery

Configurar a estratégia de recuperação de desastres para o banco de dados.

Regiões:

| Região                 | Função            |
| ---------------------- | ----------------- |
| Azure Brazil South     | Região primária   |
| Azure Brazil Southeast | Região secundária |

Componentes de DR:

* Azure SQL Geo-replica.
* Auto-Failover Group.
* Backups automáticos.
* Point-in-Time Restore.

Objetivos:

* Reduzir impacto de falha regional.
* Permitir failover do banco para região secundária.
* Manter capacidade de recuperação pontual dos dados.
* Aumentar a continuidade operacional da aplicação.

Fluxo esperado:

```text
Azure SQL Database — Brazil South
    ↓
Geo-replicação / Failover
    ↓
Azure SQL Geo-replica — Brazil Southeast
```

---

## 20. Monitoramento e Alertas

Configurar monitoramento centralizado para os principais componentes.

Serviços utilizados:

* Azure Monitor.
* Log Analytics Workspace.
* Application Insights.

Fontes monitoradas:

* Azure Front Door.
* Application Gateway.
* VM Scale Set Linux.
* Azure SQL Database.
* Azure Firewall.
* Aplicação.

Alertas recomendados:

| Alerta                       | Objetivo                                         |
| ---------------------------- | ------------------------------------------------ |
| CPU elevada nas VMs          | Identificar necessidade de escala                |
| Falha de health probe        | Detectar instâncias indisponíveis                |
| Erros HTTP 5xx               | Identificar falhas da aplicação                  |
| Latência elevada             | Detectar degradação de performance               |
| Falha de conexão com banco   | Identificar indisponibilidade da camada de dados |
| Consumo elevado do Azure SQL | Detectar gargalos no banco                       |
| Eventos de failover          | Monitorar DR                                     |
| Tráfego bloqueado pelo WAF   | Identificar tentativas de ataque                 |

---

## 21. Validações Finais da Implantação

Após a implantação, devem ser realizadas as seguintes validações:

### Rede

* VNet criada corretamente.
* Subnets criadas com os CIDRs planejados.
* NSGs associados às subnets.
* Azure Firewall implantado em subnet dedicada.
* Azure Bastion implantado em subnet dedicada.

### Aplicação

* VM Scale Set criado com imagem Linux.
* Mínimo de 3 instâncias configurado.
* Máximo de 6 instâncias configurado.
* Instâncias distribuídas entre Availability Zones.
* VMs sem IP público.
* Acesso administrativo disponível via Azure Bastion.

### Entrada e Balanceamento

* Azure Front Door recebendo tráfego HTTPS.
* WAF Policy associada ao Front Door.
* Application Gateway configurado como origin regional.
* Backend pool apontando para o VM Scale Set.
* Health probes funcionando.

### Banco de Dados

* Azure SQL Database criado.
* Private Endpoint configurado.
* Private DNS Zone associada.
* Conectividade privada funcionando.
* Backups automáticos habilitados.
* Geo-replica configurada.
* Auto-Failover Group configurado.

### Identidade e Segurança

* Managed Identity habilitada no VM Scale Set.
* Acesso ao Key Vault configurado.
* Permissões no Azure SQL atribuídas por roles de banco.
* SSH público bloqueado.
* Acesso ao banco realizado via Private Link.

### Observabilidade

* Logs enviados ao Log Analytics Workspace.
* Application Insights recebendo telemetria.
* Métricas disponíveis no Azure Monitor.
* Alertas principais configurados.

---

## 22. Considerações Sobre Custos

Alguns serviços da arquitetura podem gerar custos relevantes, principalmente:

* Azure Front Door.
* Application Gateway.
* Azure Firewall.
* VM Scale Set.
* Azure SQL Database.
* Log Analytics Workspace.
* Application Insights.
* Geo-replicação do banco.

Em uma implantação real, recomenda-se:

* Definir SKUs adequados ao ambiente.
* Aplicar autoscaling com limites.
* Monitorar custos com Azure Cost Management.
* Utilizar tags para rastreamento financeiro.
* Avaliar ambientes separados para desenvolvimento, homologação e produção.

---

## 23. Possível Evolução com Infraestrutura como Código

A arquitetura pode ser automatizada futuramente utilizando:

* Terraform.
* Azure Bicep.
* Azure Resource Manager Templates.
* Azure CLI.
* Azure DevOps Pipelines.
* GitHub Actions.

A automação permitiria:

* Reprodutibilidade.
* Versionamento da infraestrutura.
* Redução de erros manuais.
* Padronização de ambientes.
* Auditoria de mudanças.

---

## 24. Conclusão

Este plano de implantação apresenta uma sequência estruturada para provisionar a arquitetura de e-commerce 24/7 em Microsoft Azure.

A proposta contempla recursos de alta disponibilidade, escalabilidade automática, segurança de rede, identidade gerenciada, banco de dados PaaS, observabilidade e recuperação de desastres.

A execução prática da implantação deve seguir boas práticas de governança, segurança, controle de custos e validação operacional.
