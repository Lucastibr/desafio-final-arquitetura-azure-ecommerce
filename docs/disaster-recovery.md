# Disaster Recovery

## 1. Objetivo

Este documento descreve a estratégia de Disaster Recovery da arquitetura de e-commerce 24/7 proposta para Microsoft Azure.

O objetivo é apresentar como a solução lida com falhas relevantes na camada de dados e com indisponibilidade parcial ou regional, utilizando recursos de replicação, failover e recuperação pontual.

A estratégia proposta contempla:

* Região primária para operação normal.
* Região secundária para recuperação de desastres.
* Azure SQL Database com geo-replica.
* Auto-Failover Group.
* Backups automáticos.
* Point-in-Time Restore.
* Monitoramento e alertas para eventos críticos.

---

## 2. Escopo

A estratégia de Disaster Recovery cobre principalmente a camada de dados da aplicação, pois o banco de dados é o componente mais crítico para continuidade e integridade da operação.

O escopo considera:

* Azure SQL Database na região primária.
* Geo-replica do Azure SQL Database na região secundária.
* Auto-Failover Group para failover entre regiões.
* Backups automáticos.
* Point-in-Time Restore.
* Monitoramento de disponibilidade.
* Procedimento conceitual de failover.
* Procedimento conceitual de retorno à operação normal.

Fora do escopo deste documento:

* Implementação real da infraestrutura.
* Testes práticos de failover.
* Definição final de SLA contratual.
* Replicação completa da camada de aplicação para a região secundária.
* Plano detalhado de continuidade de negócios.

---

## 3. Regiões Utilizadas

A arquitetura considera duas regiões Azure:

| Região                 | Função                                 |
| ---------------------- | -------------------------------------- |
| Azure Brazil South     | Região primária                        |
| Azure Brazil Southeast | Região secundária de Disaster Recovery |

A região primária concentra a operação normal da aplicação.

A região secundária é utilizada como destino de recuperação em caso de falha relevante na região primária, principalmente para a camada de dados.

---

## 4. Componentes de Disaster Recovery

A estratégia utiliza os seguintes componentes:

| Componente              | Função                                                      |
| ----------------------- | ----------------------------------------------------------- |
| Azure SQL Database      | Banco de dados PaaS principal                               |
| Azure SQL Geo-replica   | Réplica do banco em região secundária                       |
| Auto-Failover Group     | Gerenciamento de failover entre banco primário e secundário |
| Backups Automáticos     | Proteção contra perda de dados                              |
| Point-in-Time Restore   | Recuperação para um ponto específico no tempo               |
| Azure Monitor           | Monitoramento de métricas e disponibilidade                 |
| Log Analytics Workspace | Centralização de logs                                       |
| Application Insights    | Telemetria da aplicação                                     |

---

## 5. Estratégia Geral

Durante a operação normal, a aplicação utiliza o Azure SQL Database localizado na região primária:

```text
Azure Brazil South
```

O banco de dados principal mantém uma geo-replica na região secundária:

```text
Azure Brazil Southeast
```

Fluxo conceitual:

```text
Azure SQL Database — Brazil South
    ↓
Geo-replicação
    ↓
Azure SQL Geo-replica — Brazil Southeast
```

Em caso de falha relevante na região primária, o Auto-Failover Group pode ser utilizado para promover a réplica secundária, permitindo continuidade da operação com menor impacto.

---

## 6. Operação Normal

Em cenário normal, o fluxo da aplicação ocorre da seguinte forma:

```text
Usuários
    ↓
Azure Front Door + WAF Policy
    ↓
Application Gateway
    ↓
VM Scale Set Linux
    ↓
Private Endpoint
    ↓
Azure SQL Database — Brazil South
```

A região primária processa as requisições da aplicação, enquanto a região secundária mantém uma cópia replicada do banco de dados.

Durante esse período:

* O banco primário recebe leituras e escritas.
* A geo-replica recebe os dados replicados.
* Backups automáticos permanecem habilitados.
* Métricas e logs são enviados para observabilidade.
* O ambiente é monitorado para identificar falhas ou degradação.

---

## 7. Cenários de Falha Considerados

A estratégia de DR considera diferentes tipos de falha.

### 7.1 Falha de Instância da Aplicação

Exemplo:

* Uma VM Linux fica indisponível.
* Uma instância do VM Scale Set falha.
* Uma zona de disponibilidade apresenta falha parcial.

Mitigação:

* Application Gateway remove instâncias não saudáveis do balanceamento.
* VM Scale Set recompõe instâncias conforme necessário.
* Distribuição em múltiplas Availability Zones reduz impacto.

Esse tipo de falha não exige failover regional do banco.

---

### 7.2 Falha de Zona de Disponibilidade

Exemplo:

* Uma Availability Zone fica indisponível.

Mitigação:

* VM Scale Set permanece ativo nas demais zonas.
* Application Gateway encaminha tráfego para instâncias saudáveis.
* O mínimo de 3 instâncias permite manter distribuição entre zonas.

Esse tipo de falha pode ser absorvido pela alta disponibilidade regional.

---

### 7.3 Falha do Banco Primário

Exemplo:

* Azure SQL Database primário apresenta indisponibilidade.
* Há degradação severa na camada de dados.
* O banco principal não responde dentro dos limites aceitáveis.

Mitigação:

* Auto-Failover Group pode promover a geo-replica.
* A aplicação passa a utilizar o endpoint do failover group.
* Backups/PITR continuam disponíveis como mecanismo adicional de recuperação.

---

### 7.4 Falha Regional

Exemplo:

* A região Azure Brazil South fica indisponível.
* Recursos críticos da região primária deixam de operar.

Mitigação:

* Azure SQL Geo-replica em Brazil Southeast pode ser promovida.
* Auto-Failover Group reduz o esforço de redirecionamento do banco.
* Azure Front Door pode ser usado em estratégia futura para redirecionar tráfego para uma implantação secundária da aplicação, caso essa camada também seja replicada.

Observação: no desenho atual, a estratégia de DR está focada principalmente na camada de dados.

---

## 8. Auto-Failover Group

O Auto-Failover Group é utilizado para gerenciar o failover entre o Azure SQL Database primário e sua réplica secundária.

Funções principais:

* Agrupar banco primário e secundário.
* Fornecer endpoint lógico para conexão.
* Permitir failover entre regiões.
* Reduzir impacto de indisponibilidade do banco.
* Simplificar redirecionamento da aplicação.

Fluxo conceitual:

```text
Aplicação
    ↓
Endpoint do Auto-Failover Group
    ↓
Azure SQL Database primário
    ↓
Failover, se necessário
    ↓
Azure SQL Geo-replica secundária
```

O uso de endpoint lógico evita que a aplicação dependa diretamente do endereço físico de um banco específico.

---

## 9. Backups Automáticos

O Azure SQL Database possui recursos de backup automático que apoiam a recuperação de dados.

Objetivos dos backups:

* Recuperar dados após erro operacional.
* Proteger contra exclusão acidental.
* Apoiar recuperação em caso de corrupção lógica.
* Permitir restauração para momento anterior.

Backups não substituem geo-replicação. Eles complementam a estratégia.

Diferença conceitual:

| Recurso               | Finalidade                                 |
| --------------------- | ------------------------------------------ |
| Geo-replica           | Continuidade operacional em outra região   |
| Auto-Failover Group   | Failover entre banco primário e secundário |
| Backup automático     | Recuperação de dados                       |
| Point-in-Time Restore | Restauração para ponto específico no tempo |

---

## 10. Point-in-Time Restore

O Point-in-Time Restore permite restaurar o banco para um ponto anterior no tempo.

Esse recurso é útil em cenários como:

* Exclusão acidental de dados.
* Atualização incorreta.
* Corrupção lógica.
* Erro humano.
* Implantação com impacto negativo nos dados.

Fluxo conceitual:

```text
Identificação do problema
    ↓
Definição do ponto de restauração
    ↓
Restauração do banco
    ↓
Validação dos dados
    ↓
Redirecionamento ou correção da aplicação
```

O PITR é uma estratégia de recuperação de dados, não necessariamente de continuidade imediata.

---

## 11. RTO e RPO

Para fins arquiteturais, a solução considera os seguintes conceitos:

| Indicador | Significado                                      |
| --------- | ------------------------------------------------ |
| RTO       | Tempo máximo aceitável para restaurar a operação |
| RPO       | Quantidade máxima aceitável de perda de dados    |

Estimativa conceitual:

| Indicador | Estimativa       |
| --------- | ---------------- |
| RTO       | Baixo a moderado |
| RPO       | Baixo            |

Justificativa:

* O RTO tende a ser reduzido com uso de Auto-Failover Group.
* O RPO tende a ser reduzido pela geo-replicação do Azure SQL.
* O tempo real depende de configuração, volume de dados, testes de failover e requisitos de negócio.

Em um projeto real, RTO e RPO devem ser definidos formalmente com as áreas de negócio.

---

## 12. Procedimento Conceitual de Failover

Em caso de falha crítica na camada de dados, o procedimento conceitual de failover seria:

1. Detectar falha ou indisponibilidade no banco primário.
2. Validar impacto na aplicação.
3. Confirmar que a geo-replica está íntegra.
4. Acionar failover pelo Auto-Failover Group.
5. Promover a réplica secundária.
6. Validar conectividade da aplicação.
7. Validar integridade dos dados.
8. Monitorar métricas após o failover.
9. Registrar incidente e ações realizadas.

Fluxo:

```text
Falha no banco primário
    ↓
Detecção por monitoramento
    ↓
Acionamento do Auto-Failover Group
    ↓
Promoção da geo-replica
    ↓
Aplicação conecta ao banco secundário
    ↓
Validação operacional
```

---

## 13. Procedimento Conceitual de Retorno

Após normalização da região primária, o retorno deve ser planejado com cuidado.

Etapas conceituais:

1. Validar estabilidade da região primária.
2. Confirmar integridade do banco atualmente ativo.
3. Avaliar necessidade de sincronização.
4. Planejar janela de retorno, se necessário.
5. Executar failback de forma controlada.
6. Validar aplicação após o retorno.
7. Monitorar logs, métricas e erros.
8. Atualizar documentação do incidente.

O failback não deve ser feito automaticamente sem validação, pois pode gerar risco adicional para a aplicação.

---

## 14. Monitoramento de DR

O ambiente deve monitorar eventos relacionados a disponibilidade e recuperação.

Serviços utilizados:

* Azure Monitor.
* Log Analytics Workspace.
* Application Insights.

Métricas e sinais relevantes:

| Item Monitorado              | Objetivo                                            |
| ---------------------------- | --------------------------------------------------- |
| Disponibilidade do Azure SQL | Detectar falha na camada de dados                   |
| Latência de conexão ao banco | Identificar degradação                              |
| Erros de conexão             | Detectar indisponibilidade ou falha de autenticação |
| Eventos de failover          | Acompanhar transição entre regiões                  |
| Status da geo-replica        | Validar integridade da replicação                   |
| Erros HTTP 5xx               | Identificar impacto na aplicação                    |
| Health probes                | Detectar falhas na camada de aplicação              |

---

## 15. Alertas Recomendados

Alertas recomendados para DR:

| Alerta                               | Severidade | Objetivo                              |
| ------------------------------------ | ---------- | ------------------------------------- |
| Banco primário indisponível          | Crítica    | Acionar análise de failover           |
| Falha de conexão com Azure SQL       | Crítica    | Identificar impacto na aplicação      |
| Latência elevada no banco            | Alta       | Detectar degradação                   |
| Geo-replica atrasada ou indisponível | Alta       | Identificar risco no DR               |
| Evento de failover iniciado          | Alta       | Acompanhar transição                  |
| Erros HTTP 5xx elevados              | Alta       | Detectar impacto para usuários        |
| Falha de health probe                | Média/Alta | Detectar indisponibilidade de backend |
| Consumo elevado do banco             | Média      | Antecipar degradação                  |

---

## 16. Testes de DR

A estratégia de DR só é confiável se for testada periodicamente.

Testes recomendados:

| Teste                               | Objetivo                                      |
| ----------------------------------- | --------------------------------------------- |
| Teste de failover planejado         | Validar promoção da réplica                   |
| Teste de conectividade pós-failover | Garantir que aplicação conecta ao banco ativo |
| Teste de restauração PITR           | Validar recuperação pontual                   |
| Teste de backup                     | Verificar se backups são restauráveis         |
| Teste de alerta                     | Validar acionamento operacional               |
| Teste de documentação               | Garantir que o procedimento é executável      |

Frequência sugerida:

| Tipo de Teste               | Frequência |
| --------------------------- | ---------- |
| Validação de backups        | Mensal     |
| Teste de PITR               | Trimestral |
| Teste de failover planejado | Semestral  |
| Revisão do plano de DR      | Semestral  |

---

## 17. Responsabilidades

Papéis conceituais envolvidos em DR:

| Papel                  | Responsabilidade                                 |
| ---------------------- | ------------------------------------------------ |
| Arquiteto de Soluções  | Definir estratégia e validar desenho             |
| Equipe de Operações    | Monitorar ambiente e executar runbooks           |
| Administrador de Banco | Validar integridade e recuperação dos dados      |
| Segurança              | Avaliar riscos e acessos durante incidente       |
| Desenvolvimento        | Validar funcionamento da aplicação após failover |
| Gestão do Negócio      | Definir RTO/RPO e priorização                    |

---

## 18. Runbook Resumido de Incidente

Runbook conceitual para indisponibilidade do banco:

1. Confirmar alerta crítico.
2. Verificar status do Azure SQL Database.
3. Verificar impacto na aplicação.
4. Validar status da geo-replica.
5. Acionar responsáveis técnicos.
6. Decidir entre aguardar recuperação ou realizar failover.
7. Executar failover pelo Auto-Failover Group.
8. Validar conectividade da aplicação.
9. Validar integridade dos dados.
10. Monitorar estabilidade.
11. Registrar incidente.
12. Planejar correção definitiva ou failback.

---

## 19. Limitações da Estratégia Atual

A estratégia proposta é adequada para o escopo do desafio, mas possui limitações importantes.

Limitações:

* A camada de aplicação não está completamente replicada na região secundária.
* O DR está focado principalmente no Azure SQL Database.
* Não há pipeline de implantação multi-região detalhado.
* Não há definição formal de RTO/RPO aprovada pelo negócio.
* Não há plano completo de continuidade operacional.
* Não há testes práticos executados neste escopo.
* Não há automação de infraestrutura como código descrita em detalhe.

Em uma arquitetura de produção real, seria recomendável replicar também a camada de aplicação e infraestrutura regional crítica na região secundária.

---

## 20. Melhorias Futuras

Evoluções possíveis:

* Implantar VM Scale Set também na região secundária.
* Configurar Application Gateway secundário.
* Configurar origin secundário no Azure Front Door.
* Automatizar infraestrutura com Terraform ou Bicep.
* Criar pipelines de CI/CD multi-região.
* Definir RTO/RPO formal com o negócio.
* Criar runbooks operacionais detalhados.
* Executar testes regulares de failover.
* Implementar Microsoft Defender for Cloud.
* Criar Azure Policy para governança.
* Monitorar custos de replicação e DR.

---

## 21. Conclusão

A estratégia de Disaster Recovery proposta utiliza recursos gerenciados do Azure para reduzir impacto em caso de falha crítica na camada de dados.

A combinação de Azure SQL Geo-replica, Auto-Failover Group, Backups Automáticos e Point-in-Time Restore fornece uma base adequada para continuidade e recuperação de dados.

Para o escopo do desafio, a arquitetura demonstra preocupação com resiliência, continuidade operacional e recuperação de desastres. Em um ambiente produtivo real, a estratégia poderia ser evoluída com replicação completa da camada de aplicação, automação de failover e testes periódicos formais.
