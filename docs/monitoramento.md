# Monitoramento e Observabilidade

## 1. Objetivo

Este documento descreve a estratégia de monitoramento e observabilidade da arquitetura de e-commerce 24/7 proposta para Microsoft Azure.

O objetivo é demonstrar como a solução pode acompanhar a saúde da aplicação, infraestrutura, banco de dados, rede, segurança e mecanismos de recuperação de desastres utilizando serviços nativos do Azure.

A estratégia contempla:

* Coleta de métricas.
* Centralização de logs.
* Telemetria da aplicação.
* Alertas operacionais.
* Monitoramento de disponibilidade.
* Monitoramento de performance.
* Monitoramento de segurança.
* Monitoramento de banco de dados.
* Apoio à análise de incidentes.

---

## 2. Escopo

A estratégia de monitoramento considera os seguintes componentes:

* Azure Front Door.
* WAF Policy.
* Application Gateway.
* VM Scale Set Linux.
* Azure SQL Database.
* Private Endpoint.
* Azure Firewall.
* Azure Bastion.
* Azure Key Vault.
* Managed Identity.
* Azure Monitor.
* Log Analytics Workspace.
* Application Insights.
* Azure SQL Geo-replica.
* Auto-Failover Group.

---

## 3. Serviços Utilizados

A arquitetura utiliza três serviços principais para observabilidade:

| Serviço                 | Finalidade                                                      |
| ----------------------- | --------------------------------------------------------------- |
| Azure Monitor           | Coleta de métricas, alertas e acompanhamento dos recursos Azure |
| Log Analytics Workspace | Centralização, consulta e correlação de logs                    |
| Application Insights    | Telemetria da aplicação, dependências, erros e performance      |

Esses serviços permitem acompanhar tanto a infraestrutura quanto a aplicação.

---

## 4. Visão Geral da Observabilidade

A observabilidade da arquitetura é organizada em três camadas:

1. **Infraestrutura**

   * VM Scale Set.
   * Application Gateway.
   * Azure Firewall.
   * Azure SQL Database.
   * Azure Front Door.

2. **Aplicação**

   * Requisições.
   * Latência.
   * Erros.
   * Dependências.
   * Exceções.
   * Disponibilidade.

3. **Segurança e Auditoria**

   * WAF.
   * NSG.
   * Firewall.
   * Key Vault.
   * Azure SQL.
   * Microsoft Entra ID.

Fluxo conceitual:

```text
Recursos Azure
    ↓
Métricas, logs e diagnósticos
    ↓
Azure Monitor
    ↓
Log Analytics Workspace
    ↓
Dashboards, alertas e análise operacional
```

---

## 5. Log Analytics Workspace

O Log Analytics Workspace é o ponto central para armazenamento e consulta dos logs da arquitetura.

Sugestão de nome:

```text
law-ecommerce-prod
```

Principais fontes de logs:

| Fonte                | Tipo de dado coletado                         |
| -------------------- | --------------------------------------------- |
| Azure Front Door     | Logs de acesso, roteamento e WAF              |
| Application Gateway  | Access logs, performance logs e firewall logs |
| VM Scale Set Linux   | Logs de sistema, métricas de VM e eventos     |
| Azure SQL Database   | Auditoria, métricas e diagnósticos            |
| Azure Firewall       | Regras aplicadas, tráfego permitido e negado  |
| Azure Key Vault      | Acessos a segredos, certificados e chaves     |
| Azure Bastion        | Logs de acesso administrativo                 |
| Application Insights | Telemetria da aplicação                       |

Benefícios:

* Centralização dos dados operacionais.
* Correlação entre eventos de aplicação, rede e banco.
* Consulta por KQL.
* Criação de alertas.
* Apoio à análise de incidentes.

---

## 6. Azure Monitor

O Azure Monitor é utilizado para acompanhar métricas e eventos dos recursos Azure.

Principais responsabilidades:

* Coletar métricas dos serviços.
* Criar alertas operacionais.
* Acompanhar disponibilidade.
* Identificar degradação de performance.
* Apoiar resposta a incidentes.
* Integrar dados com Log Analytics Workspace.

Métricas acompanhadas:

| Recurso             | Métricas relevantes                                           |
| ------------------- | ------------------------------------------------------------- |
| VM Scale Set        | CPU, memória, disco, rede e quantidade de instâncias          |
| Application Gateway | requisições, throughput, health probes, erros e latência      |
| Azure SQL Database  | CPU, DTU/vCore, conexões, deadlocks, armazenamento e latência |
| Azure Firewall      | tráfego permitido, tráfego negado e regras acionadas          |
| Azure Front Door    | requisições, latência, erros e tráfego bloqueado              |
| Key Vault           | operações, acessos negados e volume de chamadas               |

---

## 7. Application Insights

O Application Insights é utilizado para monitorar a camada de aplicação.

Principais dados coletados:

* Requisições HTTP.
* Tempo de resposta.
* Erros e exceções.
* Dependências externas.
* Chamadas ao banco de dados.
* Taxa de falha.
* Disponibilidade.
* Performance percebida pela aplicação.

Benefícios:

* Identificação de gargalos na aplicação.
* Correlação entre erro de aplicação e infraestrutura.
* Visibilidade de dependências, como Azure SQL Database.
* Detecção de aumento de latência.
* Apoio à investigação de incidentes.

Métricas recomendadas:

| Métrica              | Objetivo                                 |
| -------------------- | ---------------------------------------- |
| Server response time | Identificar lentidão na aplicação        |
| Failed requests      | Detectar falhas de processamento         |
| Exceptions           | Identificar erros internos               |
| Dependency duration  | Medir tempo de chamadas externas         |
| Dependency failures  | Detectar falhas na comunicação com banco |
| Availability tests   | Validar disponibilidade da aplicação     |

---

## 8. Monitoramento do Azure Front Door

O Azure Front Door deve ser monitorado por ser a entrada global da aplicação.

Dados recomendados:

* Total de requisições.
* Latência.
* Erros HTTP 4xx.
* Erros HTTP 5xx.
* Tráfego bloqueado pelo WAF.
* Origem indisponível.
* Volume de tráfego por região.

Alertas recomendados:

| Alerta                       | Severidade | Objetivo                              |
| ---------------------------- | ---------- | ------------------------------------- |
| Aumento de HTTP 5xx          | Alta       | Detectar falha na aplicação ou origin |
| Origin indisponível          | Crítica    | Detectar falha no Application Gateway |
| Latência elevada             | Alta       | Detectar degradação para usuários     |
| Alto volume de bloqueios WAF | Média/Alta | Identificar ataque ou varredura       |
| Queda brusca de tráfego      | Média      | Detectar possível indisponibilidade   |

---

## 9. Monitoramento do WAF

A WAF Policy associada ao Azure Front Door deve gerar logs para análise de segurança.

Eventos importantes:

* Requisições bloqueadas.
* Regras WAF acionadas.
* Padrões suspeitos.
* Tentativas de exploração.
* Aumento anormal de tráfego malicioso.

Métricas e logs recomendados:

| Item                      | Objetivo                            |
| ------------------------- | ----------------------------------- |
| Total de bloqueios        | Avaliar volume de tráfego malicioso |
| Regras mais acionadas     | Identificar padrões de ataque       |
| IPs de origem recorrentes | Apoiar análise de segurança         |
| URLs mais atacadas        | Identificar superfícies vulneráveis |
| Falsos positivos          | Ajustar política WAF                |

---

## 10. Monitoramento do Application Gateway

O Application Gateway é um ponto crítico da camada regional.

Dados recomendados:

* Requisições por segundo.
* Latência.
* Throughput.
* Status dos backend pools.
* Resultado dos health probes.
* Erros HTTP 4xx e 5xx.
* Capacidade utilizada.

Alertas recomendados:

| Alerta                | Severidade | Objetivo                                      |
| --------------------- | ---------- | --------------------------------------------- |
| Backend unhealthy     | Crítica    | Identificar falha nas instâncias da aplicação |
| Aumento de HTTP 5xx   | Alta       | Detectar falha de backend                     |
| Latência elevada      | Alta       | Detectar degradação de performance            |
| Throughput muito alto | Média      | Avaliar necessidade de escala                 |
| Falha em health probe | Alta       | Remover instâncias problemáticas do tráfego   |

---

## 11. Monitoramento do VM Scale Set Linux

O VM Scale Set representa a camada de aplicação.

Métricas recomendadas:

| Métrica                  | Objetivo                                       |
| ------------------------ | ---------------------------------------------- |
| CPU média                | Base para autoscaling e identificação de carga |
| Memória                  | Detectar pressão de recursos                   |
| Disco                    | Identificar gargalos de I/O                    |
| Rede                     | Monitorar volume de entrada e saída            |
| Quantidade de instâncias | Validar comportamento do autoscaling           |
| Status das instâncias    | Identificar falhas em VMs                      |
| Logs do sistema          | Diagnosticar problemas operacionais            |

Alertas recomendados:

| Alerta                          | Severidade | Objetivo                                    |
| ------------------------------- | ---------- | ------------------------------------------- |
| CPU acima de 70% por 10 minutos | Média/Alta | Acionar ou validar escala horizontal        |
| CPU acima de 90%                | Crítica    | Indicar saturação da aplicação              |
| Memória elevada                 | Alta       | Detectar risco de falha                     |
| Instância indisponível          | Alta       | Validar recomposição pelo VMSS              |
| Escala frequente                | Média      | Identificar instabilidade de capacidade     |
| Falha de inicialização          | Alta       | Detectar problema de imagem ou configuração |

---

## 12. Monitoramento do Autoscaling

O autoscaling do VM Scale Set deve ser monitorado para validar se está reagindo corretamente à demanda.

Configuração prevista:

| Parâmetro            | Valor                |
| -------------------- | -------------------- |
| Mínimo de instâncias | 3                    |
| Máximo de instâncias | 6                    |
| Sistema operacional  | Linux                |
| Distribuição         | 3 Availability Zones |

Eventos monitorados:

* Scale-out.
* Scale-in.
* Falha ao escalar.
* Tempo de estabilização.
* Frequência de escala.
* Métrica que acionou a escala.

Alertas recomendados:

| Alerta                                            | Objetivo                                         |
| ------------------------------------------------- | ------------------------------------------------ |
| VMSS atingiu 6 instâncias                         | Indicar demanda no limite da capacidade definida |
| VMSS permaneceu em 6 instâncias por longo período | Avaliar necessidade de novo sizing               |
| Falha ao escalar                                  | Detectar problema operacional                    |
| Oscilação frequente de escala                     | Ajustar thresholds de autoscaling                |

---

## 13. Monitoramento do Azure SQL Database

O Azure SQL Database é um componente crítico da solução.

Métricas recomendadas:

| Métrica               | Objetivo                                |
| --------------------- | --------------------------------------- |
| CPU                   | Identificar saturação do banco          |
| DTU/vCore             | Acompanhar consumo de capacidade        |
| Data IO               | Identificar gargalos de leitura/escrita |
| Log IO                | Identificar pressão no log transacional |
| Conexões ativas       | Monitorar volume de conexões            |
| Deadlocks             | Identificar concorrência problemática   |
| Armazenamento         | Evitar falta de espaço                  |
| Latência de consultas | Detectar degradação                     |
| Erros de conexão      | Identificar falhas ou indisponibilidade |

Alertas recomendados:

| Alerta                       | Severidade | Objetivo                                          |
| ---------------------------- | ---------- | ------------------------------------------------- |
| CPU elevada                  | Alta       | Detectar saturação                                |
| Uso de armazenamento elevado | Alta       | Evitar indisponibilidade por espaço               |
| Erros de conexão             | Crítica    | Detectar impacto na aplicação                     |
| Deadlocks frequentes         | Média/Alta | Identificar problema transacional                 |
| Latência elevada             | Alta       | Detectar degradação de performance                |
| Falha de autenticação        | Alta       | Identificar problema de IAM ou tentativa indevida |
| Evento de failover           | Crítica    | Acompanhar DR                                     |

---

## 14. Monitoramento do Private Endpoint e DNS

O acesso ao Azure SQL Database ocorre por Private Endpoint e Private DNS Zone.

Validações monitoradas:

* Resolução DNS privada funcionando.
* Aplicação conectando ao banco por IP privado.
* Falhas de conectividade.
* Alterações na configuração de DNS.
* Alterações no Private Endpoint.

Sinais de problema:

| Sintoma                        | Possível causa                   |
| ------------------------------ | -------------------------------- |
| Aplicação não conecta ao banco | Falha no Private Endpoint ou DNS |
| FQDN resolve para IP público   | Problema na Private DNS Zone     |
| Erros intermitentes            | Falha de rede ou configuração    |
| Aumento de latência            | Degradação de rede ou banco      |

---

## 15. Monitoramento do Azure Firewall

O Azure Firewall deve enviar logs para o Log Analytics Workspace.

Dados recomendados:

* Tráfego permitido.
* Tráfego negado.
* Regras acionadas.
* IPs de origem e destino.
* Portas e protocolos.
* Volume de tráfego.
* Tentativas de comunicação não autorizada.

Alertas recomendados:

| Alerta                                 | Severidade | Objetivo                                    |
| -------------------------------------- | ---------- | ------------------------------------------- |
| Alto volume de tráfego negado          | Média/Alta | Detectar anomalia ou configuração incorreta |
| Comunicação com destino não autorizado | Alta       | Identificar risco de segurança              |
| Alteração em Firewall Policy           | Alta       | Auditar mudança crítica                     |
| Aumento anormal de egress              | Média/Alta | Detectar vazamento ou comportamento anormal |

---

## 16. Monitoramento do Azure Key Vault

O Key Vault deve ser monitorado para segurança e auditoria.

Eventos relevantes:

* Leitura de segredo.
* Falha de acesso.
* Alteração de segredo.
* Exclusão de segredo.
* Alteração de permissões.
* Acesso por identidade não esperada.

Alertas recomendados:

| Alerta                             | Severidade | Objetivo                                             |
| ---------------------------------- | ---------- | ---------------------------------------------------- |
| Acesso negado ao Key Vault         | Alta       | Detectar problema de permissão ou tentativa indevida |
| Alto volume de leitura de segredos | Média/Alta | Identificar comportamento incomum                    |
| Alteração de segredo               | Alta       | Auditar mudança sensível                             |
| Exclusão de segredo                | Crítica    | Detectar risco operacional                           |
| Acesso fora do padrão              | Alta       | Investigar possível abuso                            |

---

## 17. Monitoramento do Azure Bastion

O Azure Bastion deve ser monitorado para rastrear acessos administrativos.

Eventos recomendados:

* Sessões iniciadas.
* Sessões encerradas.
* Usuário responsável pelo acesso.
* VM acessada.
* Horário do acesso.
* Falhas de conexão.

Alertas recomendados:

| Alerta                                | Severidade | Objetivo                       |
| ------------------------------------- | ---------- | ------------------------------ |
| Acesso administrativo fora do horário | Média/Alta | Detectar atividade incomum     |
| Falha repetida de acesso              | Alta       | Identificar tentativa indevida |
| Acesso a VM crítica                   | Média      | Apoiar auditoria               |
| Aumento de sessões administrativas    | Média      | Detectar operação anormal      |

---

## 18. Monitoramento de Disaster Recovery

A estratégia de DR envolve Azure SQL Geo-replica, Auto-Failover Group e Backups/PITR.

Itens monitorados:

| Item                                | Objetivo                            |
| ----------------------------------- | ----------------------------------- |
| Status da geo-replica               | Validar integridade da replicação   |
| Eventos de failover                 | Acompanhar transição entre regiões  |
| Latência de replicação              | Avaliar risco de perda de dados     |
| Disponibilidade do banco primário   | Detectar falhas                     |
| Disponibilidade do banco secundário | Garantir prontidão do DR            |
| Status dos backups                  | Confirmar capacidade de restauração |
| Testes de PITR                      | Validar recuperação pontual         |

Alertas recomendados:

| Alerta                        | Severidade | Objetivo                      |
| ----------------------------- | ---------- | ----------------------------- |
| Geo-replica indisponível      | Alta       | Identificar risco no DR       |
| Evento de failover            | Crítica    | Acionar validação operacional |
| Falha de backup               | Crítica    | Proteger recuperação de dados |
| Falha em restauração de teste | Alta       | Detectar risco de recuperação |
| Latência alta na replicação   | Alta       | Avaliar RPO                   |

---

## 19. Dashboards Recomendados

A arquitetura deve possuir dashboards para acompanhamento operacional.

Dashboards sugeridos:

### Dashboard Executivo

Indicadores:

* Disponibilidade geral.
* Volume de requisições.
* Erros HTTP 5xx.
* Latência média.
* Status do banco.
* Status do DR.
* Eventos críticos.

### Dashboard de Aplicação

Indicadores:

* Requisições por minuto.
* Tempo médio de resposta.
* Erros por endpoint.
* Exceções.
* Dependências.
* Chamadas ao banco.
* Usuários afetados.

### Dashboard de Infraestrutura

Indicadores:

* CPU das VMs.
* Memória das VMs.
* Quantidade de instâncias.
* Eventos de autoscaling.
* Health probes.
* Throughput do Application Gateway.

### Dashboard de Banco de Dados

Indicadores:

* CPU.
* DTU/vCore.
* Conexões.
* Deadlocks.
* Latência.
* Armazenamento.
* Erros de conexão.

### Dashboard de Segurança

Indicadores:

* Bloqueios WAF.
* Tráfego negado no Firewall.
* Acessos ao Key Vault.
* Falhas de autenticação.
* Alterações em regras de rede.
* Acessos via Bastion.

---

## 20. Alertas Prioritários

Alertas considerados prioritários para operação 24/7:

| Alerta                                   | Severidade | Ação esperada                         |
| ---------------------------------------- | ---------- | ------------------------------------- |
| Aplicação indisponível                   | Crítica    | Acionar equipe de operação            |
| HTTP 5xx elevado                         | Alta       | Investigar aplicação/backend          |
| Application Gateway sem backend saudável | Crítica    | Verificar VMSS e health probes        |
| VMSS atingiu máximo de instâncias        | Alta       | Avaliar capacidade                    |
| CPU crítica nas VMs                      | Alta       | Validar autoscaling                   |
| Banco indisponível                       | Crítica    | Acionar plano de DR                   |
| Erros de conexão com banco               | Crítica    | Validar Private Endpoint, DNS e SQL   |
| Geo-replica indisponível                 | Alta       | Avaliar risco de DR                   |
| Falha de backup                          | Crítica    | Corrigir proteção de dados            |
| Alto volume de bloqueios WAF             | Alta       | Investigar possível ataque            |
| Acesso negado ao Key Vault               | Alta       | Verificar permissões e possível abuso |

---

## 21. Severidade dos Alertas

Classificação sugerida:

| Severidade | Critério                                       |
| ---------- | ---------------------------------------------- |
| Crítica    | Impacto direto na disponibilidade ou nos dados |
| Alta       | Degradação relevante ou risco operacional      |
| Média      | Evento anormal que exige análise               |
| Baixa      | Informação operacional ou tendência            |

Exemplos:

| Evento                                | Severidade  |
| ------------------------------------- | ----------- |
| Aplicação fora do ar                  | Crítica     |
| Banco indisponível                    | Crítica     |
| Erros 5xx elevados                    | Alta        |
| CPU elevada                           | Alta        |
| Tráfego WAF bloqueado acima do normal | Média/Alta  |
| Acesso administrativo fora do padrão  | Média/Alta  |
| Crescimento de armazenamento          | Média       |
| Evento informativo de escala          | Baixa/Média |

---

## 22. Retenção de Logs

A retenção dos logs deve considerar requisitos operacionais, segurança e custos.

Sugestão conceitual:

| Tipo de Log                  | Retenção sugerida            |
| ---------------------------- | ---------------------------- |
| Logs operacionais            | 30 a 90 dias                 |
| Logs de segurança            | 90 a 180 dias                |
| Logs de auditoria            | 180 dias ou mais             |
| Logs de aplicação detalhados | Conforme custo e criticidade |
| Logs de incidentes críticos  | Retenção estendida           |

Em um projeto real, a retenção deve ser definida conforme requisitos de negócio, compliance, LGPD, auditoria e orçamento.

---

## 23. Exemplos de Consultas KQL

As consultas abaixo são exemplos conceituais para análise no Log Analytics Workspace.

### Erros HTTP 5xx

```kql
AzureDiagnostics
| where httpStatus_d >= 500
| summarize Total=count() by bin(TimeGenerated, 5m), Resource
| order by TimeGenerated desc
```

### Tráfego bloqueado pelo WAF

```kql
AzureDiagnostics
| where action_s == "Blocked"
| summarize Total=count() by ruleName_s, clientIP_s
| order by Total desc
```

### Eventos do Application Gateway

```kql
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| summarize Total=count() by httpStatus_d, bin(TimeGenerated, 5m)
| order by TimeGenerated desc
```

### Falhas de conexão com banco

```kql
AzureDiagnostics
| where ResourceType has "SQL"
| where Category has "SQLSecurityAuditEvents"
| summarize Total=count() by action_name_s, bin(TimeGenerated, 15m)
| order by TimeGenerated desc
```

### Eventos do Azure Firewall

```kql
AzureDiagnostics
| where ResourceType == "AZUREFIREWALLS"
| summarize Total=count() by msg_s, bin(TimeGenerated, 15m)
| order by Total desc
```

Essas consultas assumem o modo de coleta legado *Azure diagnostics*, em que tudo é enviado para a tabela `AzureDiagnostics`. Com o modo recomendado *Resource-specific*, os logs vão para tabelas dedicadas — por exemplo `AGWAccessLogs` e `AGWFirewallLogs` (Application Gateway), `FrontDoorAccessLog` e `FrontDoorWebApplicationFirewallLog` (Front Door) e `SQLSecurityAuditEvents` (Azure SQL). Ajuste as consultas conforme o modo de diagnóstico e os nomes reais de tabelas, categorias e campos do ambiente.

---

## 24. Runbook Resumido de Incidente

Procedimento conceitual para incidente crítico:

1. Receber alerta no Azure Monitor.
2. Identificar componente impactado.
3. Verificar dashboards operacionais.
4. Consultar logs no Log Analytics Workspace.
5. Correlacionar eventos de aplicação, rede e banco.
6. Verificar impacto para usuários.
7. Acionar equipe responsável.
8. Aplicar ação corretiva.
9. Monitorar estabilização.
10. Registrar causa provável.
11. Documentar ações realizadas.
12. Criar melhoria preventiva.

Exemplo de fluxo:

```text
Alerta crítico
    ↓
Análise no dashboard
    ↓
Consulta de logs
    ↓
Identificação da causa
    ↓
Correção ou mitigação
    ↓
Validação
    ↓
Registro do incidente
```

---

## 25. Indicadores de Saúde da Solução

Indicadores principais:

| Indicador                    | Objetivo                                      |
| ---------------------------- | --------------------------------------------- |
| Disponibilidade da aplicação | Medir se usuários conseguem acessar o serviço |
| Latência média               | Avaliar experiência do usuário                |
| Taxa de erro                 | Detectar falhas funcionais ou técnicas        |
| Consumo de CPU               | Avaliar carga na camada de aplicação          |
| Quantidade de instâncias     | Validar autoscaling                           |
| Saúde dos backends           | Garantir balanceamento adequado               |
| Conectividade com banco      | Validar camada de dados                       |
| Eventos de segurança         | Detectar ameaças ou abuso                     |
| Status de DR                 | Garantir prontidão para falhas                |

---

## 26. SLOs Conceituais

Para uma aplicação de e-commerce 24/7, podem ser considerados os seguintes SLOs conceituais:

| Indicador                        | Meta conceitual                      |
| -------------------------------- | ------------------------------------ |
| Disponibilidade da aplicação     | Alta disponibilidade                 |
| Taxa de erro HTTP 5xx            | Baixa                                |
| Latência média                   | Controlada dentro do limite esperado |
| Tempo de resposta do banco       | Estável                              |
| Tempo de detecção de incidentes  | Baixo                                |
| Tempo de acionamento operacional | Baixo                                |
| Integridade dos backups          | Validada periodicamente              |
| Prontidão do DR                  | Monitorada continuamente             |

Esses SLOs devem ser refinados em um projeto real com base nas necessidades de negócio e no orçamento disponível.

---

## 27. Responsabilidades Operacionais

Papéis conceituais:

| Papel           | Responsabilidade                                    |
| --------------- | --------------------------------------------------- |
| Operações       | Acompanhar alertas, dashboards e incidentes         |
| Desenvolvimento | Corrigir falhas de aplicação e analisar telemetria  |
| Banco de Dados  | Monitorar Azure SQL, backups e performance          |
| Segurança       | Analisar eventos WAF, Firewall, Key Vault e acessos |
| Arquitetura     | Evoluir desenho, padrões e decisões técnicas        |
| Negócio         | Definir criticidade, SLOs, RTO e RPO                |

---

## 28. Boas Práticas Recomendadas

Boas práticas para monitoramento:

* Habilitar diagnostic settings nos recursos críticos.
* Centralizar logs no Log Analytics Workspace.
* Usar Application Insights na aplicação.
* Criar dashboards por público-alvo.
* Configurar alertas para eventos críticos.
* Evitar excesso de alertas sem ação definida.
* Definir severidade e responsável para cada alerta.
* Testar alertas periodicamente.
* Monitorar custo de ingestão de logs.
* Revisar métricas e dashboards continuamente.
* Registrar e revisar incidentes.

---

## 29. Limitações e Melhorias Futuras

Esta estratégia apresenta uma visão conceitual de monitoramento para o desafio.

Em uma implantação real, ainda seria necessário detalhar:

* Dashboards reais no Azure Portal.
* Queries KQL ajustadas aos logs do ambiente.
* Integração com ITSM.
* Integração com Microsoft Teams, e-mail ou PagerDuty.
* Métricas customizadas da aplicação.
* Synthetic tests de disponibilidade.
* Alertas baseados em comportamento anômalo.
* Monitoramento de custos.
* Workbooks personalizados.
* Testes periódicos dos alertas.
* Runbooks automatizados.

---

## 30. Conclusão

A estratégia de monitoramento proposta utiliza Azure Monitor, Log Analytics Workspace e Application Insights para fornecer visibilidade operacional da aplicação, infraestrutura, banco de dados, segurança e Disaster Recovery.

A centralização de logs e métricas permite detectar falhas, investigar incidentes, acompanhar performance e apoiar decisões operacionais.

Para o escopo do desafio, a solução demonstra preocupação com observabilidade, operação 24/7, segurança e continuidade do serviço.
