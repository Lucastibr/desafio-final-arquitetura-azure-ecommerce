# Segurança e IAM

## 1. Objetivo

Este documento descreve a estratégia de segurança e controle de acesso da arquitetura de e-commerce 24/7 proposta para Microsoft Azure.

O objetivo é demonstrar como a solução protege a aplicação, os dados, a rede e os acessos administrativos utilizando recursos nativos do Azure, como Microsoft Entra ID, Managed Identity, Azure Key Vault, Network Security Groups, Azure Firewall, Azure Bastion, Private Endpoint e WAF Policy.

A estratégia segue princípios de segurança em profundidade, menor privilégio, isolamento de rede e redução de exposição pública.

---

## 2. Escopo

A estratégia de segurança contempla:

* Controle de identidade e autenticação.
* Acesso seguro das VMs ao banco de dados.
* Eliminação de credenciais fixas na aplicação.
* Proteção de segredos e certificados.
* Segmentação de rede.
* Controle de tráfego entre subnets.
* Proteção da entrada pública da aplicação.
* Administração segura das VMs Linux.
* Acesso privado ao banco de dados.
* Auditoria, logs e rastreabilidade.

---

## 3. Princípios de Segurança Aplicados

A arquitetura foi desenhada com base nos seguintes princípios:

| Princípio                    | Aplicação na Arquitetura                         |
| ---------------------------- | ------------------------------------------------ |
| Menor privilégio             | Permissões concedidas apenas para o necessário   |
| Defesa em profundidade       | Controles de segurança em múltiplas camadas      |
| Zero secrets hardcoded       | Uso de Managed Identity e Key Vault              |
| Isolamento de rede           | Separação por subnets e NSGs                     |
| Acesso privado a dados       | Azure SQL acessado via Private Endpoint          |
| Administração segura         | VMs acessadas via Azure Bastion, sem SSH público |
| Monitoramento contínuo       | Logs e métricas enviados para Log Analytics      |
| Redução de exposição pública | Banco e VMs sem exposição direta à Internet      |

---

## 4. Visão Geral da Segurança

A segurança da arquitetura está distribuída em camadas:

1. **Borda / Entrada Global**

   * Azure Front Door.
   * WAF Policy.
   * HTTPS/TLS.

2. **Entrada Regional**

   * Application Gateway.
   * Health probes.
   * Balanceamento L7.

3. **Rede**

   * Virtual Network.
   * Subnets dedicadas.
   * Network Security Groups.
   * Azure Firewall.
   * Private Endpoint.

4. **Identidade**

   * Microsoft Entra ID.
   * Managed Identity.
   * Autenticação integrada ao Azure SQL.

5. **Segredos**

   * Azure Key Vault.
   * Acesso controlado por identidade.

6. **Administração**

   * Azure Bastion.
   * Ausência de SSH público.

7. **Auditoria e Monitoramento**

   * Azure Monitor.
   * Log Analytics Workspace.
   * Application Insights.

---

## 5. Microsoft Entra ID

O Microsoft Entra ID é utilizado como provedor central de identidade da solução.

Funções na arquitetura:

* Gerenciar identidades.
* Autenticar recursos e usuários administrativos.
* Integrar autenticação com Managed Identity.
* Apoiar autenticação no Azure SQL Database.
* Controlar acesso a recursos Azure.
* Permitir rastreabilidade de acessos.

Na arquitetura proposta, o Microsoft Entra ID é utilizado principalmente para permitir que a aplicação, executando no VM Scale Set Linux, acesse serviços Azure sem uso de credenciais estáticas.

---

## 6. Managed Identity

A arquitetura utiliza Managed Identity para permitir que o VM Scale Set Linux acesse recursos Azure de forma segura.

Objetivos:

* Evitar usuário e senha hardcoded na aplicação.
* Evitar armazenamento de credenciais em arquivos de configuração.
* Reduzir risco de vazamento de segredos.
* Permitir autenticação controlada pelo Microsoft Entra ID.
* Melhorar rastreabilidade e governança de acessos.

Fluxo conceitual:

```text
VM Scale Set Linux
    ↓
Managed Identity
    ↓
Microsoft Entra ID
    ↓
Azure SQL Database / Azure Key Vault
```

A Managed Identity deve receber apenas as permissões necessárias para execução da aplicação.

---

## 7. Acesso ao Azure SQL Database

O acesso das VMs ao Azure SQL Database deve ocorrer usando:

* Managed Identity.
* Microsoft Entra ID Authentication.
* Roles dentro do banco de dados.
* Comunicação privada via Private Endpoint.

Fluxo de acesso:

```text
Aplicação nas VMs Linux
    ↓
Managed Identity
    ↓
Microsoft Entra ID Auth
    ↓
Database Roles
    ↓
Azure SQL Database
```

As permissões de leitura e escrita devem ser tratadas no nível do banco de dados, não apenas por RBAC genérico do Azure.

Exemplos de permissões conceituais:

| Permissão                              | Finalidade                                   |
| -------------------------------------- | -------------------------------------------- |
| Leitura                                | Consultar dados necessários para a aplicação |
| Escrita                                | Inserir e atualizar dados transacionais      |
| Execução                               | Executar procedures ou funções autorizadas   |
| Negação de privilégios administrativos | Evitar acesso excessivo ao banco             |

Em uma implantação real, recomenda-se criar roles customizadas no banco, concedendo apenas as permissões necessárias para a aplicação.

---

## 8. Azure Key Vault

O Azure Key Vault é utilizado para armazenamento seguro de informações sensíveis.

Itens que podem ser armazenados:

* Certificados TLS.
* Segredos de aplicação.
* Chaves criptográficas.
* Connection strings, quando necessárias.
* Configurações sensíveis.

Modelo de acesso recomendado:

```text
VM Scale Set Linux
    ↓
Managed Identity
    ↓
Azure Key Vault
```

Boas práticas aplicadas:

* Evitar segredos hardcoded no código-fonte.
* Restringir acesso ao Key Vault por identidade.
* Habilitar logs de auditoria.
* Aplicar princípio do menor privilégio.
* Controlar permissões de leitura, escrita e administração.
* Separar responsabilidades entre administradores e aplicações.

---

## 9. Segurança da Entrada Pública

A entrada pública da aplicação ocorre pelo Azure Front Door com WAF Policy.

Responsabilidades do Azure Front Door + WAF Policy:

* Receber tráfego HTTPS dos usuários.
* Aplicar proteção WAF na borda.
* Encaminhar tráfego para o Application Gateway como origin regional.
* Reduzir exposição direta da camada regional.
* Proteger contra ameaças comuns de camada web.

A WAF Policy contribui para mitigar riscos como:

* Requisições maliciosas.
* Tentativas de exploração de vulnerabilidades web.
* Padrões suspeitos de tráfego.
* Ataques comuns em HTTP/HTTPS.

Fluxo de entrada:

```text
Usuários da Internet
    ↓
Azure Front Door + WAF Policy
    ↓
Application Gateway
    ↓
VM Scale Set Linux
```

---

## 10. Application Gateway

O Application Gateway atua como camada regional L7.

Responsabilidades de segurança:

* Receber tráfego apenas da camada esperada.
* Encaminhar requisições para o backend pool do VM Scale Set.
* Executar health probes.
* Evitar envio de tráfego para instâncias não saudáveis.
* Centralizar regras regionais de roteamento HTTP/HTTPS.

O Application Gateway está localizado em subnet dedicada:

```text
ApplicationGatewaySubnet — 10.0.1.0/24
```

Essa separação permite aplicar regras específicas de rede e reduzir acoplamento com a camada de aplicação.

---

## 11. Segmentação de Rede

A arquitetura utiliza uma Virtual Network principal segmentada por subnets.

VNet:

```text
vnet-ecommerce-prod — 10.0.0.0/16
```

Subnets:

| Subnet                   |        CIDR | Finalidade                             |
| ------------------------ | ----------: | -------------------------------------- |
| ApplicationGatewaySubnet | 10.0.1.0/24 | Subnet dedicada ao Application Gateway |
| Subnet Aplicação         | 10.0.2.0/24 | Subnet do VM Scale Set Linux           |
| Subnet Private Endpoints | 10.0.3.0/24 | Subnet dedicada aos Private Endpoints  |
| AzureBastionSubnet       | 10.0.4.0/27 | Subnet dedicada ao Azure Bastion       |
| AzureFirewallSubnet      | 10.0.5.0/26 | Subnet dedicada ao Azure Firewall      |

Benefícios da segmentação:

* Isolamento lógico dos recursos.
* Aplicação de regras específicas por função.
* Redução do tráfego lateral indevido.
* Melhor governança de rede.
* Maior clareza arquitetural.

---

## 12. Network Security Groups

Network Security Groups são utilizados para controlar tráfego nas subnets.

NSGs sugeridos:

| NSG                     | Associação               |
| ----------------------- | ------------------------ |
| nsg-application-gateway | ApplicationGatewaySubnet |
| nsg-app                 | Subnet Aplicação         |
| nsg-private-endpoints   | Subnet Private Endpoints |

Regras conceituais recomendadas:

| Origem                  | Destino             | Porta/Protocolo | Ação     | Justificativa                                |
| ----------------------- | ------------------- | --------------- | -------- | -------------------------------------------- |
| Azure Front Door        | Application Gateway | HTTPS/443       | Permitir | Entrada regional controlada                  |
| Application Gateway     | VM Scale Set        | HTTP/HTTPS      | Permitir | Balanceamento para aplicação                 |
| VM Scale Set            | Private Endpoint    | TCP/1433        | Permitir | Acesso ao Azure SQL                          |
| Internet                | VM Scale Set        | SSH/22          | Negar    | VMs não devem expor SSH público              |
| Internet                | Azure SQL Database  | Qualquer        | Negar    | Banco deve ser acessado via Private Endpoint |
| Subnets não autorizadas | Subnet Aplicação    | Qualquer        | Negar    | Redução de movimento lateral                 |

As regras exatas devem ser refinadas conforme portas reais da aplicação, endereços e requisitos operacionais.

---

## 13. Azure Firewall

O Azure Firewall é utilizado para controle e inspeção de tráfego.

Localização:

```text
AzureFirewallSubnet — 10.0.5.0/26
```

Responsabilidades:

* Controlar tráfego de saída.
* Centralizar políticas de rede.
* Inspecionar tráfego norte-sul quando aplicável.
* Restringir comunicação com destinos externos.
* Registrar logs de tráfego e bloqueios.

Configurações recomendadas:

* Azure Firewall Policy.
* Regras de aplicação para destinos permitidos.
* Regras de rede somente quando necessárias.
* Logs enviados ao Log Analytics Workspace.
* Rotas configuradas quando for necessário forçar egress pelo firewall.

Exemplos de controles:

| Controle                             | Objetivo                              |
| ------------------------------------ | ------------------------------------- |
| Permitir apenas destinos necessários | Reduzir risco de comunicação indevida |
| Bloquear tráfego desconhecido        | Aumentar segurança da saída           |
| Registrar tráfego negado             | Apoiar auditoria e investigação       |
| Aplicar políticas centralizadas      | Melhorar governança                   |

---

## 14. Azure Bastion

O Azure Bastion é utilizado para acesso administrativo seguro às VMs Linux.

Localização:

```text
AzureBastionSubnet — 10.0.4.0/27
```

Objetivos:

* Permitir administração das VMs sem IP público.
* Evitar exposição de SSH para Internet.
* Reduzir superfície de ataque.
* Centralizar o acesso administrativo pelo Portal Azure.

Modelo de administração:

```text
Administrador autorizado
    ↓
Azure Portal
    ↓
Azure Bastion
    ↓
VM Scale Set Linux
```

Validações esperadas:

* VMs sem IP público.
* SSH público bloqueado.
* Acesso administrativo feito por Bastion.
* Acesso restrito a usuários autorizados.

---

## 15. Private Endpoint e Private Link

O Azure SQL Database é acessado por meio de Private Endpoint e Private Link.

Private Endpoint localizado em:

```text
Subnet Private Endpoints — 10.0.3.0/24
```

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

* Acesso privado ao banco de dados.
* Redução da exposição pública.
* Tráfego por caminho privado.
* Melhor controle de rede.
* Redução da superfície de ataque.

A utilização de Private Endpoint é uma decisão importante para proteger a camada de dados.

---

## 16. Private DNS Zone

A Private DNS Zone é utilizada para resolver o endereço do Azure SQL Database para o IP privado do Private Endpoint.

Objetivo:

* Garantir resolução privada do banco.
* Evitar uso do endpoint público.
* Facilitar conexão transparente da aplicação ao Azure SQL.
* Integrar resolução DNS com a VNet.

Validações esperadas:

* O FQDN do banco resolve para IP privado.
* As VMs conseguem acessar o banco pelo nome.
* O tráfego utiliza Private Link.

---

## 17. Segurança das VMs Linux

As VMs Linux são executadas dentro do VM Scale Set.

Controles aplicados:

* Sem IP público nas VMs.
* Acesso administrativo via Azure Bastion.
* Managed Identity habilitada.
* NSG protegendo a subnet de aplicação.
* Comunicação com banco via Private Endpoint.
* Logs enviados para observabilidade centralizada.
* Escala automática com limites definidos.

Recomendações adicionais para uma implantação real:

* Hardening do sistema operacional.
* Atualizações e patches regulares.
* Uso de imagens base confiáveis.
* Restrição de pacotes e serviços desnecessários.
* Monitoramento de integridade.
* Antimalware ou solução de proteção de workload, conforme política da organização.

---

## 18. Segurança do Banco de Dados

O Azure SQL Database é utilizado como serviço PaaS gerenciado.

Controles considerados:

* Acesso privado via Private Endpoint.
* Autenticação com Microsoft Entra ID.
* Permissões por roles de banco.
* Backups automáticos.
* Point-in-Time Restore.
* Criptografia em repouso.
* Geo-replicação para região secundária.
* Logs de auditoria e diagnóstico.

Boas práticas recomendadas:

* Restringir ou desabilitar acesso público.
* Utilizar usuários e roles com menor privilégio.
* Separar permissões administrativas de permissões da aplicação.
* Habilitar auditoria.
* Monitorar tentativas de login malsucedidas.
* Controlar acesso a dados sensíveis.

---

## 19. Controle de Acesso por Função

A arquitetura deve adotar separação de responsabilidades.

Papéis conceituais:

| Papel                  | Responsabilidade                               |
| ---------------------- | ---------------------------------------------- |
| Administrador Cloud    | Gerenciar recursos Azure                       |
| Administrador de Rede  | Gerenciar VNet, NSG, Firewall e Bastion        |
| Administrador de Banco | Gerenciar Azure SQL e permissões de dados      |
| Aplicação              | Acessar banco e Key Vault com Managed Identity |
| Operação               | Monitorar logs, métricas e alertas             |
| Segurança              | Auditar acessos, eventos e políticas           |

Essa separação reduz riscos de privilégio excessivo e melhora a governança.

---

## 20. Logs e Auditoria de Segurança

A arquitetura deve registrar eventos relevantes de segurança.

Fontes de logs:

* Azure Front Door.
* WAF Policy.
* Application Gateway.
* Azure Firewall.
* Azure Bastion.
* VM Scale Set Linux.
* Azure SQL Database.
* Key Vault.
* Microsoft Entra ID.
* Network Security Groups.

Eventos importantes:

| Evento                        | Finalidade                                |
| ----------------------------- | ----------------------------------------- |
| Tráfego bloqueado pelo WAF    | Identificar ataques web                   |
| Tentativas de acesso negadas  | Detectar comportamento suspeito           |
| Falhas de autenticação        | Identificar risco de credenciais ou abuso |
| Acesso ao Key Vault           | Auditar uso de segredos                   |
| Acesso ao banco               | Auditar operações sensíveis               |
| Tráfego bloqueado no firewall | Investigar comunicação indevida           |
| Eventos administrativos       | Rastrear alterações críticas              |

Os logs devem ser centralizados no Log Analytics Workspace para consulta, correlação e alertas.

---

## 21. Alertas de Segurança Recomendados

Alertas recomendados:

| Alerta                                           | Objetivo                                        |
| ------------------------------------------------ | ----------------------------------------------- |
| Alto volume de bloqueios no WAF                  | Detectar ataque ou varredura                    |
| Tentativas repetidas de autenticação malsucedida | Detectar tentativa de acesso indevido           |
| Acesso incomum ao Key Vault                      | Identificar possível abuso de segredos          |
| Acesso administrativo fora do padrão             | Detectar comportamento suspeito                 |
| Tráfego negado pelo Azure Firewall               | Investigar comunicação indevida                 |
| Alteração em NSG ou Firewall Policy              | Detectar mudança crítica de segurança           |
| Tentativa de conexão pública ao banco            | Identificar exposição ou configuração incorreta |
| Falha de autenticação no Azure SQL               | Investigar acesso não autorizado                |

---

## 22. Matriz de Controles de Segurança

| Risco                               | Controle Aplicado                                |
| ----------------------------------- | ------------------------------------------------ |
| Exposição direta das VMs à Internet | VMs sem IP público + Azure Bastion               |
| Acesso indevido ao banco            | Private Endpoint + Entra ID Auth + DB Roles      |
| Vazamento de credenciais            | Managed Identity + Key Vault                     |
| Ataques web                         | Azure Front Door + WAF Policy                    |
| Movimento lateral na rede           | Segmentação por subnets + NSGs                   |
| Saída indevida para Internet        | Azure Firewall                                   |
| Falha de rastreabilidade            | Logs centralizados no Log Analytics              |
| Privilégio excessivo                | Menor privilégio e separação de funções          |
| Falha regional no banco             | Geo-replica + Auto-Failover Group + Backups/PITR |

---

## 23. Validações Recomendadas

Após a implantação, as seguintes validações devem ser realizadas:

### Identidade

* Managed Identity habilitada no VM Scale Set.
* Managed Identity com acesso somente aos recursos necessários.
* Acesso ao Key Vault validado por identidade.
* Acesso ao Azure SQL validado com Entra ID Auth.

### Banco de Dados

* Conexão ao Azure SQL funcionando via Private Endpoint.
* Endpoint público restrito ou desabilitado, conforme política.
* Roles de banco atribuídas corretamente.
* Aplicação sem uso de credenciais fixas.

### Rede

* VMs sem IP público.
* SSH público bloqueado.
* Azure Bastion funcionando.
* NSGs associados às subnets.
* Azure Firewall registrando logs.
* Private DNS Zone resolvendo o banco para IP privado.

### Entrada Pública

* Front Door recebendo tráfego HTTPS.
* WAF Policy associada.
* Application Gateway recebendo tráfego do Front Door.
* Backend pool encaminhando para o VM Scale Set.

### Auditoria

* Logs chegando ao Log Analytics Workspace.
* Logs do Key Vault habilitados.
* Logs do WAF habilitados.
* Logs do Azure SQL habilitados.
* Alertas principais configurados.

---

## 24. Considerações Sobre Menor Privilégio

A arquitetura deve evitar permissões amplas.

Recomendações:

* Não conceder permissões administrativas à Managed Identity da aplicação.
* Criar roles específicas no banco para leitura e escrita.
* Separar permissões de operação, segurança e administração.
* Evitar uso de contas compartilhadas.
* Revisar permissões periodicamente.
* Remover acessos não utilizados.
* Registrar alterações críticas.

---

## 25. Considerações Sobre Segredos

A aplicação não deve armazenar segredos diretamente no código-fonte.

Boas práticas:

* Utilizar Azure Key Vault.
* Acessar segredos por Managed Identity.
* Rotacionar segredos quando aplicável.
* Auditar acessos ao Key Vault.
* Evitar connection strings com usuário e senha fixos.
* Preferir autenticação baseada em identidade sempre que possível.

---

## 26. Limitações e Melhorias Futuras

Esta documentação descreve a estratégia conceitual de segurança da arquitetura.

Em uma implantação real, ainda seria necessário detalhar:

* Regras específicas de NSG.
* Regras específicas do Azure Firewall.
* Políticas detalhadas do WAF.
* Configuração de Microsoft Defender for Cloud.
* Políticas de Azure Policy.
* Classificação de dados sensíveis.
* Estratégia de criptografia avançada.
* Gestão de chaves.
* Auditoria formal de permissões.
* Testes de segurança.
* Testes de resposta a incidentes.
* Plano de hardening das VMs Linux.

Esses pontos podem ser evoluídos conforme maturidade, orçamento e criticidade do ambiente.

---

## 27. Conclusão

A arquitetura proposta utiliza controles de segurança em múltiplas camadas para proteger a aplicação, a rede, os dados e os acessos administrativos.

A combinação de Azure Front Door com WAF Policy, Application Gateway, segmentação de rede, NSGs, Azure Firewall, Azure Bastion, Private Endpoint, Managed Identity, Microsoft Entra ID e Azure Key Vault fornece uma base sólida de segurança para um ambiente de e-commerce em nuvem.

A estratégia reduz exposição pública, evita credenciais fixas, protege o banco de dados, melhora rastreabilidade e aplica o princípio do menor privilégio.
