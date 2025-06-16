-----

# Desafio DIO: Monitoramento de Máquinas Virtuais no Azure

-----

Este repositório documenta a experiência prática na configuração e gerenciamento do monitoramento de recursos no Microsoft Azure, com foco em Máquinas Virtuais (VMs). O objetivo principal é demonstrar como manter **visibilidade, controle e resposta proativa** sobre eventos críticos no ambiente de nuvem, como a exclusão inesperada de uma VM.

Este material serve como um resumo de estudos, anotações e dicas sobre o uso de serviços de monitoramento do Azure, visando auxiliar em futuras implementações e aprofundar o conhecimento adquirido no bootcamp da Digital Innovation One (DIO).

-----

## Status de Execução do Desafio

Este desafio prático foi **concluído com sucesso** no ambiente Microsoft Azure. As configurações de monitoramento foram aplicadas a uma máquina virtual Linux, e o alerta para a exclusão da VM foi testado e validado.

  * **Data de Execução:** ***16 de junho de 2025***
  * **Recursos Monitorados:** ***'vm-monitoramento-teste'***
  * **Alerta Testado:** ***'Alerta VM Excluída - vm-monitoramento-teste'***
  * **Resultado do Teste:** O alerta disparou conforme esperado, e a notificação de e-mail foi recebida com sucesso após a simulação da exclusão da VM. O evento foi devidamente registrado no Activity Log, confirmando a rastreabilidade da operação.

A seguir, a documentação detalhada dos conceitos e dos passos práticos realizados.

-----

## 1\. Introdução

No cenário atual de cloud computing, monitorar a saúde e o desempenho dos recursos é fundamental. O **Azure Monitor** fornece um conjunto robusto de ferramentas para coletar, analisar e agir sobre dados de telemetria de todo o seu ambiente Azure. Este projeto focou em como utilizar esses recursos para garantir que eventos críticos, como a remoção acidental ou não autorizada de uma VM, sejam prontamente detectados e notificados, garantindo a integridade e a disponibilidade dos seus recursos.

-----

## 2\. Conceitos Chave de Monitoramento no Azure

Para este projeto, os seguintes conceitos e serviços do Azure Monitor foram explorados e são fundamentais para o gerenciamento eficaz de recursos na nuvem:

### 2.1. Azure Monitor

É o serviço abrangente do Azure para **coletar, analisar e agir sobre dados de telemetria** de ambientes Azure e on-premises. Atua como a solução central para monitoramento de saúde, desempenho e disponibilidade dos seus recursos.

### 2.2. Log Analytics Workspace

Um **recurso do Azure e um contêiner central onde os dados de monitoramento são coletados, agregados, analisados e apresentados**. Ele serve como o repositório principal para logs e métricas de diversos recursos, permitindo consultas avançadas usando a linguagem KQL. É a base para a maioria das funcionalidades de log do Azure Monitor.

### 2.3. Métricas vs. Logs

  * **Métricas:** São **dados quantitativos**, como valores percentuais de uso (ex: % de CPU, memória disponível, latência de rede). São ideais para gráficos de desempenho, tendências e alertas baseados em limiares.
  * **Logs:** São **mensagens descritivas sobre eventos** que ocorreram (ex: "VM iniciada", "erro de aplicação", "login falhou"). São essenciais para diagnósticos detalhados, auditoria e investigação de causas-raiz.

### 2.4. Regras de Alerta

Componentes cruciais para a proatividade no monitoramento. Uma regra de alerta define:

  * **Recurso:** O alvo do monitoramento (ex: uma VM, um Grupo de Recursos, etc.).
  * **Condição:** A lógica que, se satisfeita pelos dados (métricas ou logs), **dispara o alerta** (ex: "Uso de CPU \> 90% por 5 minutos" ou "Evento de 'exclusão de VM' no Log de Atividades").
  * **Ações:** O que acontece quando o alerta é disparado (definido em um Grupo de Ação).
  * **Detalhes de Alerta:** Incluem a **severidade do alerta** (a gravidade do problema, ex: Crítico, Erro, Aviso), que ajuda a priorizar a resposta.

### 2.5. Grupos de Ação

São **destinatários ou ações configuradas para serem executadas quando um alerta é disparado**. Eles centralizam as preferências de notificação (e-mail, SMS, push para aplicativo móvel) e automação (webhooks, Azure Functions, Logic Apps) para serem reutilizadas em múltiplas regras de alerta, garantindo uma resposta rápida e consistente.

### 2.6. Activity Log (Registro de Atividades)

É um log que registra **operações de controle e gerenciamento** (quem criou, modificou ou excluiu o quê) em sua assinatura do Azure. É vital para auditoria, pois mostra **quem** realizou **o quê**, **quando** e em **qual recurso**. Para identificar quem excluiu um recurso (como uma VM), a categoria de evento **"Administrativo"** no Activity Log é a mais relevante.

### 2.7. Azure Service Health

Recurso que permite acompanhar a **saúde dos serviços da Microsoft** e receber notificações sobre indisponibilidade, manutenções planejadas ou avisos que possam afetar seus recursos. É essencial para entender o impacto de incidentes na plataforma Azure em sua própria infraestrutura.

### 2.8. Azure Network Watcher

Um conjunto de ferramentas para monitoramento e diagnóstico de rede virtual. Inclui funcionalidades como:

  * **Topologia:** Para obter uma representação visual dos elementos da rede virtual.
  * **IP Flow Verify:** Para verificar rapidamente se uma regra de segurança (NSG) está bloqueando o tráfego de ou para uma VM.
  * **Extensão VM do Agente Observador de Rede:** Necessária na VM para capturar o tráfego de rede para análise aprofundada.

### 2.9. Microsoft Defender for Cloud

Ferramenta que pode ser integrada para **tratar alertas de segurança no Azure** relacionados a recursos monitorados, oferecendo proteção unificada e gerenciamento de postura de segurança em ambientes multi-cloud e híbridos.

-----

## 3\. Cenário Prático: Monitoramento de Exclusão de VM

Neste laboratório, o foco foi configurar o monitoramento para garantir a visibilidade e resposta a um evento crítico: a exclusão de uma Máquina Virtual. Este cenário simula a necessidade de auditoria e notificação imediata em caso de operações críticas nos recursos.

### 3.1. Configuração do Ambiente no Azure

#### 3.1.1. Criação do Grupo de Recursos

O primeiro passo foi criar um Grupo de Recursos dedicado para organizar todos os recursos do laboratório. Isso facilita a gestão e a exclusão posterior.

  * **Nome do Grupo de Recursos:** ***`rg-monitoramento-lab-dio`***
  * **Região:** ***`Brazil South`***

#### 3.1.2. Implantação da Máquina Virtual (VM)

Em seguida, uma máquina virtual foi implantada. Esta VM serviu como o recurso a ser monitorado e o alvo para o teste de exclusão.

  * **Nome da VM:** ***`vm-monitoramento-teste`***
  * **Sistema Operacional:** ***`Ubuntu Server 22.04 LTS`***
  * **Tamanho:** ***`Standard_B1ls`***
  * **Grupo de Recursos:** ***`rg-monitoramento-lab-dio`***
  * **Região:** ***`Brazil South`***
  * Foi configurado acesso via SSH para eventuais validações internas.

#### 3.1.3. Criação e Configuração do Log Analytics Workspace

Um Log Analytics Workspace foi criado para centralizar a coleta de logs e métricas. A VM foi conectada a este workspace para que seus dados de monitoramento fossem ingeridos.

  * **Nome do Workspace:** ***`law-monitoramento-dio`***
  * **Região:** ***`Brazil South`*** (mesma da VM)
  * A conexão da VM ao Workspace foi realizada através das **Configurações de Diagnóstico** da VM, garantindo que os logs do sistema operacional e as métricas de desempenho fossem encaminhados.

### 3.2. Exploração Inicial de Logs (Opcional, mas Recomendado)

Após a conexão da VM ao Log Analytics Workspace, acessamos a seção "Logs" no Azure Monitor para verificar o fluxo de dados e a saúde da VM. Utilizamos Kusto Query Language (KQL) para isso.

*Exemplo de consulta para verificar o heartbeat da VM:*

```kusto
Heartbeat
| where Computer == "vm-monitoramento-teste" // Substitua pelo nome exato da sua VM
| summarize LastHeartbeat=max(TimeGenerated) by Computer
| project Computer, LastHeartbeat
| render table
```

Esta consulta confirmou que a VM estava enviando batimentos cardíacos regularmente ao Log Analytics Workspace, indicando que o agente de monitoramento estava funcionando corretamente e os dados estavam sendo coletados.

### 3.3. Configuração da Regra de Alerta para Exclusão de VM

O ponto central do desafio foi configurar um alerta para detectar a exclusão da VM. Isso envolveu a criação de um Grupo de Ação e a Regra de Alerta propriamente dita.

#### 3.3.1. Criação do Grupo de Ação

Um Grupo de Ação foi criado para definir quem seria notificado quando o alerta de exclusão de VM fosse disparado.

  * **Nome do Grupo de Ação:** ***`ag-notificacao-admin`***
  * **Ação Configurada:** Envio de **e-mail** para ***`seu.email@example.com`***. Isso garante que a equipe responsável seja prontamente avisada.

#### 3.3.2. Criação da Regra de Alerta

A regra de alerta foi configurada para monitorar o Log de Atividades e acionar o Grupo de Ação.

  * **Navegação:** Azure Monitor -\> Alertas -\> Regras de Alerta -\> Criar.
  * **Escopo:** A regra foi definida para monitorar o **Grupo de Recursos** ***`rg-monitoramento-lab-dio`***, abrangendo a VM de teste.
  * **Condição:** O tipo de sinal escolhido foi **"Activity Log"**. A condição específica foi a operação **"Excluir Máquina Virtual" (`Microsoft.Compute/virtualMachines/delete`)**.
  * **Ações:** O **Grupo de Ação** ***`ag-notificacao-admin`*** criado anteriormente foi selecionado para ser acionado.
  * **Detalhes:**
      * **Nome do Alerta:** ***`Alerta VM Excluída - vm-monitoramento-teste`***
      * **Descrição:** Alerta disparado quando a VM de teste é excluída.
      * **Severidade:** **Crítico (S0)**, devido à natureza destrutiva e crítica da operação de exclusão de uma VM.

### 3.4. Teste e Validação do Alerta

A etapa final e crucial foi testar se o alerta funcionava conforme o esperado, simulando a exclusão da VM.

#### 3.4.1. Executando a Ação de Teste

Para testar o alerta, a Máquina Virtual ***`vm-monitoramento-teste`*** foi deliberadamente excluída através do portal do Azure. Esta ação simulou um evento crítico real.

#### 3.4.2. Verificação da Notificação

Momentos após a exclusão da VM, uma notificação por e-mail foi recebida no endereço configurado no Grupo de Ação. O e-mail continha todos os detalhes do alerta, como o nome do alerta, a severidade, o recurso afetado e um link direto para o Azure Monitor para investigação.

#### 3.4.3. Análise no Registro de Atividades

Para auditar a exclusão da VM e entender quem realizou a operação, o Registro de Atividades (Activity Log) no Azure Monitor foi consultado.

  * Acessamos o **Azure Monitor \> Registro de Atividades**.
  * Filtramos por categoria **"Administrativo"** e pela operação **"Excluir Máquina Virtual"**.
  * O log mostrou o evento de exclusão, incluindo o usuário que realizou a operação, o carimbo de data/hora e o status de sucesso da operação. Isso valida a capacidade de auditoria e rastreabilidade que o Azure Monitor oferece.

-----

## 4\. Conclusão e Aprendizados

Este laboratório prático demonstrou a importância crítica do monitoramento proativo em ambientes de nuvem. Através do **Azure Monitor**, fomos capazes de configurar um sistema robusto para detectar e notificar sobre eventos críticos, como a exclusão de uma VM, que poderia ter impactos significativos na operação.

Aprendi que a integração entre o **Log Analytics Workspace** (como repositório de logs), o **Registro de Atividades** (para eventos de controle), as **Regras de Alerta** (para detecção de condições) e os **Grupos de Ação** (para notificações) forma um ecossistema poderoso para garantir a visibilidade e o controle. A **Linguagem de Consulta Kusto (KQL)** se mostrou uma ferramenta indispensável para explorar e analisar os logs coletados, permitindo uma investigação aprofundada de qualquer incidente.

Este conhecimento é fundamental para qualquer profissional que atue com Azure, capacitando-o a manter a estabilidade, segurança e conformidade da infraestrutura em nuvem no dia a dia corporativo.

-----

## 5\. Dicas e Anotações Adicionais

  * **Dica 1:** **Sempre verifique se o Log Analytics Workspace está na mesma região que os recursos que você deseja monitorar.** Isso otimiza o desempenho na ingestão e consulta de logs e pode reduzir custos com transferências de dados entre regiões.
  * **Dica 2:** **Explore as "consultas pré-definidas" ou "exemplos de consulta"** disponíveis no Log Analytics. Elas são um excelente ponto de partida para análises de auditoria, segurança, desempenho e podem ser facilmente adaptadas às suas necessidades específicas.
  * **Dica 3:** Para depurar rapidamente problemas de conectividade de rede entre VMs, o **"IP Flow Verify" do Azure Network Watcher** é uma ferramenta muito eficaz para testar rapidamente se uma regra de Grupo de Segurança de Rede (NSG) está bloqueando o tráfego esperado.
  * **Dica 4:** O **"Registro de Atividades" (Activity Log)** é sua primeira parada para investigar **quem fez o quê** na sua assinatura do Azure. Ele é crucial para auditoria, conformidade e para entender mudanças no control plane.
  * **Dica 5:** Considere usar **Grupos de Ação com Webhooks** para integrar seus alertas do Azure Monitor com ferramentas de comunicação internas (como Microsoft Teams ou Slack) ou com sistemas de gerenciamento de tickets de TI (ITSM), automatizando ainda mais o processo de resposta a incidentes.

-----

## 6\. Recursos Úteis

  * [Configurar o monitoramento de máquinas virtuais - Artigo no Microsoft Learn](https://learn.microsoft.com/pt-br/azure/azure-monitor/vm/monitor-vm)
  * [Documentação Oficial do Azure Monitor](https://learn.microsoft.com/pt-br/azure/azure-monitor/)
  * [Documentação da Linguagem de Consulta Kusto (KQL)](https://learn.microsoft.com/pt-br/azure/data-explorer/kusto/query/)
  * [GitHub Quick Start - Repositório com Link para Aulas de Git e GitHub](https://github.com/digitalinnovationone/github-quickstart)
  * [Documentação do GitHub - Guia completo para uso do GitHub](https://docs.github.com/pt)
  * [GitHub Markdown - Guia específico para Markdown no GitHub](https://docs.github.com/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)

-----

**Autor:** [**Vinícius Medeiros**]
**LinkedIn:** [Vinícius Medeiros](https://www.linkedin.com/in/vinicius-medeiros-1b192b270/))]
**GitHub:** [[GitHub de Vinícius Medeiros](https://github.com/viinicius-medeiross)]
