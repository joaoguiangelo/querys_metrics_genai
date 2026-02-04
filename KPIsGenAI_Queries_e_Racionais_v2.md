# KPIs GenAI - Queries e Racionais
## Documentação Técnica e de Negócio

---

## Índice de Fontes de Dados

| Código | Nome da Fonte | Tipo | Descrição |
|--------|---------------|------|-----------|
| **FONTE-01** | Data Warehouse - Atendimento | Trino/SQL | Tabela `customer_service.customer_service.historic_service` - Histórico de todas as interações de atendimento |
| **FONTE-02** | Data Lake - Eventos de Cobrança | Hive/SQL | Tabela `hive.neonpottencial.dbo_analyticsevento` - Eventos de analytics do funil de cobrança |
| **FONTE-03** | Workspace - Produção Cobrança | Trino/SQL | Tabela `workspace.cobranca.BancoProducao` - Dados de pagamentos e recuperação de crédito |
| **FONTE-04** | Grafana/Mimir | PromQL | Métricas de infraestrutura Kubernetes para monitoramento de disponibilidade |
| **FONTE-05** | Fatura OpenAI | Manual | Fatura mensal do fornecedor de LLM (extração manual) |
| **FONTE-06** | Workspace - Prevenção à Fraude | Trino/SQL | Tabela `workspace.prevfraude.superset_orquestrador` - Base de clientes bloqueados |
| **FONTE-07** | MarTech - Tracking | Trino/SQL | Tabela `martech.martech.client_tracking_inf` - Mapeamento de identificadores de clientes |

---

## 1. ENGAJAMENTO

### 1.1 Conversas Qualificadas 🆗

#### Descrição Detalhada
Este indicador mede o **volume de conversas que tiveram interação real e significativa** entre o cliente e o assistente de IA. O objetivo é filtrar conversas "vazias" (onde o cliente abriu o chat mas não interagiu) e focar apenas em atendimentos onde houve engajamento efetivo.

**O que é considerado uma "Conversa Qualificada"?**
- **Para Atendimento (WhatsApp/Chat):** Uma conversa onde a IA enviou pelo menos 2 mensagens E o cliente respondeu pelo menos 1 vez. Isso elimina: aberturas acidentais, timeouts por inatividade e interações onde o cliente desistiu imediatamente.
- **Para Cobrança (WhatsAppCobranca):** Uma conversa onde o cliente informou seu CPF. Informar o CPF demonstra intenção de prosseguir com a negociação.

**Por que esse filtro?**
Sem esse filtro, contaríamos milhares de "conversas" onde nada aconteceu (ex: cliente clicou sem querer, fechou a janela). Esse indicador mostra quantos clientes **realmente** interagiram com a IA.

#### Fórmula

**Atendimento (WhatsApp e Chat):**
```
Conversas Qualificadas = COUNT DISTINCT (Protocolos)
ONDE:
  - Mensagens enviadas pela IA ≥ 2
  - Mensagens enviadas pelo Cliente ≥ 1
```

**Cobrança (WhatsAppCobranca):**
```
Conversas Qualificadas = COUNT DISTINCT (ID do Cliente)
ONDE:
  - Ação = 'get-client' (cliente informou CPF)
```

**Total Mensal:**
```
Total = Conversas Atendimento + Conversas Cobrança
```

#### Fontes de Dados
| Componente | Fonte | Tabela/Sistema |
|------------|-------|----------------|
| Atendimento (WhatsApp/Chat) | **FONTE-01** | `customer_service.customer_service.historic_service` |
| Cobrança (WhatsAppCobranca) | **FONTE-02** | `hive.neonpottencial.dbo_analyticsevento` |

#### Query SQL
```sql
WITH metricas_atendimento AS (
    -- ATENDIMENTO: Chat e WhatsApp padrão
    SELECT
        DATE_TRUNC('month', created_at_dt) AS mes,
        partner_channel_ds AS canal,
        COUNT(DISTINCT protocol_nm) AS conversas_qualificadas
    FROM customer_service.customer_service.historic_service
    WHERE 
        partner_integration_origem_nm = 'AiAssistant'
        AND partner_channel_ds IN ('WhatsApp', 'Chat')
        AND created_at_dt >= DATE '2025-01-01'
        -- Regra de qualificação: interação bidirecional real
        AND CARDINALITY(FILTER(service_history, x -> x.author_tp = 'Customer')) >= 1 
        AND CARDINALITY(FILTER(service_history, x -> x.author_tp = 'GenAI')) >= 2
    GROUP BY 1, 2

    UNION ALL

    -- COBRANÇA: WhatsApp GenAI (CPF informado = engajamento mínimo)
    SELECT 
        DATE_TRUNC('month', DataCriacao) AS mes,
        'WhatsAppCobranca' AS canal,
        COUNT(DISTINCT idcliente) AS conversas_qualificadas
    FROM hive.neonpottencial.dbo_analyticsevento
    WHERE 
        area = 'cobranca.neon.whats.gen-ai'
        AND descricao = 'CLICK'
        AND acao = 'get-client'  -- CPF informado = qualificado
        AND DATE(DataCriacao) >= DATE '2025-01-01'
    GROUP BY 1, 2
)

SELECT
    mes,
    canal,
    conversas_qualificadas,
    SUM(conversas_qualificadas) OVER(PARTITION BY mes) AS total_mes
FROM metricas_atendimento
ORDER BY mes DESC, canal;
```

---

### 1.2 % Uptime (Disponibilidade Técnica) 🆗

#### Descrição Detalhada
Este indicador mede **quanto tempo o assistente de IA ficou disponível e funcionando** durante o período. É uma métrica de infraestrutura que monitora se os servidores (chamados de "pods" no Kubernetes) estão operacionais.

**O que significa na prática?**
- **100% Uptime:** O assistente esteve disponível o tempo todo, sem quedas.
- **99% Uptime:** O assistente ficou fora do ar por aproximadamente 7 horas no mês.
- **95% Uptime:** O assistente ficou indisponível por aproximadamente 36 horas no mês.

**O que é monitorado?**
O sistema monitora os containers do serviço `gugelmin-primary` (nome interno do assistente) no cluster Kubernetes. Se pelo menos 1 instância está rodando, o serviço está UP.

#### Fórmula

```
% Uptime = (Minutos com pelo menos 1 pod ativo / Total de minutos do período) × 100
```

**Simplificando:**
```
% Uptime = Média do status do serviço ao longo do tempo × 100

Onde:
  - Status = 1 se pelo menos 1 pod está rodando
  - Status = 0 se nenhum pod está rodando
```

#### Fontes de Dados
| Componente | Fonte | Sistema |
|------------|-------|---------|
| Métricas de containers | **FONTE-04** | Grafana/Mimir (Kubernetes metrics) |

#### Query PromQL (Grafana)
```promql
avg_over_time(
  (
    clamp_max(
      sum(
        kube_pod_container_status_running{namespace="neon", container="gugelmin"} 
        * on (namespace, pod) group_left() 
        namespace_workload_pod:kube_pod_owner:relabel{namespace="neon", workload=~"gugelmin-primary"}
      ) or vector(0), 1
    )
  )[$__range:1m]
) * 100
```

---

### 1.3 WCU Agregado (Média Mensal de Usuários Semanais) 🆗

#### Descrição Detalhada
**WCU = Weekly Conversational Users** (Usuários Conversacionais Semanais)

Este indicador mede a **constância de uso do assistente** ao longo do mês. Em vez de contar quantas pessoas usaram no mês todo (que seria o MAU - Monthly Active Users), ele calcula quantas pessoas usam **em uma semana típica**.

**Como funciona o cálculo?**
1. Primeiro, conta-se quantos usuários únicos usaram o assistente em cada semana do mês
2. Depois, tira-se a média dessas semanas

**Por que usar WCU em vez de MAU?**
- **MAU** mostra o alcance total (quantas pessoas diferentes usaram)
- **WCU Médio** mostra o engajamento recorrente (quantas pessoas usam regularmente)

**Exemplo prático:**
- Mês com MAU = 10.000 e WCU Médio = 2.500 → Cada usuário usa em média 1x por mês
- Mês com MAU = 10.000 e WCU Médio = 5.000 → Usuários estão voltando mais vezes

#### Fórmula

**Etapa 1 - Calcular WCU de cada semana:**
```
WCU_Semana = COUNT DISTINCT (Usuários únicos na semana)
```

**Etapa 2 - Calcular média mensal:**
```
WCU_Médio_Mensal = MÉDIA (WCU_Semana1, WCU_Semana2, WCU_Semana3, WCU_Semana4)
```

**Em notação matemática:**
$$\text{WCU Médio} = \frac{\sum_{i=1}^{n} \text{WCU}_i}{n}$$

Onde *n* = número de semanas no mês

#### Fontes de Dados
| Componente | Fonte | Tabela |
|------------|-------|--------|
| Histórico de atendimentos | **FONTE-01** | `customer_service.customer_service.historic_service` |

#### Query SQL
```sql
WITH weekly_metrics AS (
    -- Etapa 1: Calcular WCU de cada semana individualmente
    SELECT
        DATE_TRUNC('week', created_at_dt) AS semana_referencia,
        COUNT(DISTINCT person_uuid) AS wcu_semanal
    FROM
        customer_service.customer_service.historic_service
    WHERE
        partner_channel_ds IN ('WhatsApp', 'Chat', 'WhatsAppCobranca')
        AND partner_integration_origem_nm = 'AiAssistant'
        AND created_at_dt >= DATE '2025-12-01'
    GROUP BY
        1
)
-- Etapa 2: Tirar a MÉDIA desses valores agrupando pelo mês
SELECT
    DATE_TRUNC('month', semana_referencia) AS mes_referencia,
    AVG(wcu_semanal) AS media_wcu_mensal -- Alterado de SUM para AVG
FROM
    weekly_metrics
GROUP BY
    1
ORDER BY
    1 DESC;
```

---

## 2. IMPACTO NO NEGÓCIO

### 2.1 % Conversão (Acordo) 🆗

#### Descrição Detalhada
Este indicador mede a **efetividade do funil de negociação de dívidas** no canal de cobrança via IA. Ele acompanha cada etapa que o cliente percorre, desde o momento que informa seu CPF até a geração do boleto para pagamento.

**Etapas do Funil (em ordem):**
| Etapa | Código | Significado |
|-------|--------|-------------|
| 0 | `get-client` | Cliente informou seu CPF |
| 1 | `get-invoice-debt` | Cliente visualizou sua dívida |
| 2 | `simulate-agreements` | Cliente simulou opções de acordo |
| 3 | `confirm-agreement` | Cliente confirmou um acordo |
| 4 | `paymentslip-generate` | Boleto foi gerado |

**Para que serve?**
Permite identificar **onde os clientes estão desistindo**. Por exemplo:
- Se muitos param na etapa 1, o problema pode ser o valor da dívida apresentado
- Se muitos param na etapa 2, as opções de acordo podem não ser atrativas
- Se muitos param na etapa 3, pode haver fricção no processo de confirmação

#### Fórmula

**Taxa de Conversão entre Etapas:**
```
% Conversão (Etapa A → B) = (Clientes na Etapa B / Clientes na Etapa A) × 100
```

**Exemplo:**
```
% Conversão (CPF → Boleto) = (Clientes que geraram boleto / Clientes que informaram CPF) × 100
```

#### Fontes de Dados
| Componente | Fonte | Tabela |
|------------|-------|--------|
| Eventos do funil de cobrança | **FONTE-02** | `hive.neonpottencial.dbo_analyticsevento` |

#### Query SQL
```sql
select
    date_trunc('month', DataCriacao) as mes,
    'Neon AI' as canal,
    case
        when a.acao = 'get-client' then '0. Informou CPF'
        when a.acao = 'get-invoice-debt' then '1. Visualizou a divida'
        when a.acao = 'simulate-agreements' then '2. Simulacao'
        when a.acao = 'confirm-agreement' then '3. Confirmacao'
        when a.acao = 'paymentslip-generate' then '4. Geracao boleto'
        else '5. Outros'
    end as Funil,
    acao,
    descricao,
    count(distinct idcliente) as total
from hive.neonpottencial.dbo_analyticsevento a
where
    a.area = 'cobranca.neon.whats.gen-ai'
    and a.descricao = 'CLICK'
    -- Alterado para capturar desde o início de Outubro/2025
    and date(DataCriacao) >= date('2025-10-01') 
    and date(DataCriacao) <= current_date
group by 1, 2, 3, 4, 5 
order by 1 desc, 3 asc
```

---

### 2.2 Recuperação de Crédito (Amount Recovered) 🆗

#### Descrição Detalhada
Este indicador mede o **volume financeiro de dívidas recuperadas** através dos canais de cobrança, separando dois tipos de entrada de caixa:

**New Cash (Caixa Novo):**
- Pagamentos da **1ª parcela** de acordos
- Representa **novos acordos sendo fechados**
- Indica efetividade na conversão de devedores em pagadores

**Old Cash (Caixa Recorrente):**
- Pagamentos da **2ª parcela em diante**
- Representa **acordos sendo mantidos**
- Indica sustentabilidade dos acordos (clientes continuando a pagar)

**Por que separar?**
- Alto New Cash + Baixo Old Cash = Muitos acordos novos, mas clientes não mantêm
- Baixo New Cash + Alto Old Cash = Poucos acordos novos, mas carteira saudável
- O ideal é ter ambos crescendo

**Canais comparados:**
| Canal na Query | Nome Comercial | Descrição |
|----------------|----------------|-----------|
| `Whatsapp GenAI` | Neon AI | Assistente de IA para cobrança |
| `WhatsApp` | Coddera | Parceiro de cobrança via WhatsApp |
| `Monest` | Monest | Parceiro de cobrança |
| `1Digital` | 1Digital | Parceiro de cobrança |

#### Fórmula

**New Cash:**
```
Valor New Cash = SOMA (Valor das parcelas pagas)
ONDE: Número da parcela = 1

Qtde New Cash = COUNT (Pagamentos)
ONDE: Número da parcela = 1
```

**Old Cash:**
```
Valor Old Cash = SOMA (Valor das parcelas pagas)
ONDE: Número da parcela > 1

Qtde Old Cash = COUNT (Pagamentos)
ONDE: Número da parcela > 1
```

**Total Recuperado:**
```
Valor Total = New Cash + Old Cash
```

#### Fontes de Dados
| Componente | Fonte | Tabela |
|------------|-------|--------|
| Pagamentos de acordos | **FONTE-03** | `workspace.cobranca.BancoProducao` |

#### Query SQL
```sql
select
    date_trunc('month', dtpagamento) as mes
    ,case 
        when dscanal ='WhatsApp' then 'Coddera'
        when dscanal ='Whatsapp GenAI' then 'Neon AI'
        else dscanal
    end as canal
    ,'6. Pagamentos' as Funil 
    ,'Pgtos' as acao
    ,'NA' as descricao
    -- Métricas de New Cash (nrparcela = 1)
    ,count(case when nrparcela = 1 then clientid end) as qtde_new_cash
    ,sum(case when nrparcela = 1 then vlparcelapaga else 0 end) as valor_new_cash
    -- Métricas de Old Cash (nrparcela > 1)
    ,count(case when nrparcela > 1 then clientid end) as qtde_old_cash
    ,sum(case when nrparcela > 1 then vlparcelapaga else 0 end) as valor_old_cash
    -- Totais Gerais do Mês
    ,count(clientid) as total_pgtos
    ,sum(vlparcelapaga) as valor_recuperado_total
from workspace.cobranca.BancoProducao
where
    dscanal in ('Monest','1Digital','WhatsApp','Whatsapp GenAI')
    and dtcadastronegociacao >= date('2024-01-01')
    and dstipopessoa = 'F'
    and dtpagamento >= date('2025-10-01')
group by 1, 2
order by mes desc, canal;
```

---

### 2.3 Custo por Conversa (Proxy Financeiro) 🆗

#### Descrição Detalhada
Este indicador mede **quanto custa, em média, cada conversa** processada pelo assistente de IA. É um proxy (aproximação) do custo real até que tenhamos dados mais granulares.

**Por que é um "proxy"?**
O custo real por conversa depende de vários fatores: tamanho da conversa, modelo usado, tokens processados. Atualmente, não temos essa granularidade, então usamos uma aproximação:
- **Numerador:** Valor total da fatura da OpenAI no mês
- **Denominador:** Quantidade total de conversas no mês

**Limitações atuais:**
- Inclui conversas de teste e desenvolvimento no custo
- Não diferencia conversas longas (mais caras) de curtas (mais baratas)
- Mistura diferentes modelos que têm custos diferentes

**Evolução planejada:**
Quando o Langfuse/LiteLLM estiver com 100% de cobertura, teremos custo real por conversa baseado em tokens consumidos.

#### Fórmula

$$\text{Custo por Conversa} = \frac{\text{Valor Total da Fatura OpenAI (R\$)}}{\text{Quantidade de Conversas Brutas}}$$

**Componentes:**
```
Numerador = Fatura mensal OpenAI em R$ (extração manual)

Denominador = COUNT (partner_integration_uuid)
ONDE: Canal IN ('Chat', 'WhatsApp', 'WhatsAppCobranca')
```

#### Fontes de Dados
| Componente | Fonte | Origem |
|------------|-------|--------|
| Custo total (numerador) | **FONTE-05** | Fatura OpenAI (extração manual) |
| Volume de conversas (denominador) | **FONTE-01** | `customer_service.customer_service.historic_service` |

#### Query SQL (Denominador)
```sql
SELECT
  COUNT(partner_integration_uuid) AS qtd_conversas_brutas
FROM customer_service.customer_service.historic_service
WHERE
  created_at_dt >= DATE '2025-01-01' -- Ajustar data inicio do mês
  AND created_at_dt < DATE '2025-02-01' -- Ajustar data inicio do mês seguinte
  AND partner_channel_ds IN ('Chat', 'WhatsApp', 'WhatsAppCobranca')
  -- Filtro amplo para capturar todo volume que gera custo de processamento
```

---

## 3. QUALIDADE

### 3.1 CSAT & Retenção - Agentes Conversacionais (Consolidada) 🆗

#### Descrição Detalhada
Este indicador mede **dois aspectos da qualidade do atendimento** da IA:

**CSAT (Customer Satisfaction Score):**
Mede a satisfação do cliente com o atendimento. Usa a metodologia **Top-2-Box**:
- Notas 4 e 5 = Cliente Satisfeito ✅
- Notas 1, 2 e 3 = Cliente Insatisfeito ❌

**Retenção:**
Mede o percentual de atendimentos que a **IA conseguiu resolver sozinha**, sem precisar transferir para um atendente humano (transbordo).
- Retenção Alta = IA está resolvendo bem
- Retenção Baixa = IA está transferindo muito para humanos

**Filtros aplicados:**

1. **Exclui transbordo humano:** Quando a conversa é transferida para um atendente humano, a avaliação de CSAT não conta para a IA (pois foi o humano que finalizou)

2. **Exclui clientes bloqueados:** Clientes bloqueados por fraude, PLD, etc. têm experiência comprometida por fatores externos. Sua insatisfação não reflete a qualidade da IA.

**Por que dois cálculos (com e sem bloqueados)?**
- **Com bloqueados:** Visão real do que o cliente experienciou
- **Sem bloqueados:** Visão justa da performance da IA (sem fatores externos)
- **Delta:** Mostra o impacto dos bloqueios na percepção de qualidade

#### Fórmula

**CSAT (Top-2-Box):**
```
% CSAT = (Votos com nota 4 ou 5 / Total de votos) × 100
```

**Retenção:**
```
% Retenção = (Atendimentos SEM transbordo / Total de atendimentos) × 100
```

**Variações calculadas:**
| Métrica | Inclui Bloqueados? | Inclui Transbordo? |
|---------|-------------------|-------------------|
| CSAT Geral | Sim | Não |
| CSAT Sem Bloqueados | Não | Não |
| Retenção Geral | Sim | N/A |
| Retenção Sem Bloqueados | Não | N/A |

**Delta (impacto dos bloqueios):**
```
Delta CSAT = CSAT Sem Bloqueados - CSAT Geral
Delta Retenção = Retenção Sem Bloqueados - Retenção Geral
```

#### Fontes de Dados
| Componente | Fonte | Tabela |
|------------|-------|--------|
| Histórico de atendimentos | **FONTE-01** | `customer_service.customer_service.historic_service` |
| Base de clientes bloqueados | **FONTE-06** | `workspace.prevfraude.superset_orquestrador` |
| Mapeamento de clientes | **FONTE-07** | `martech.martech.client_tracking_inf` |

#### Query SQL
```sql
WITH bloqueados AS (
   SELECT
       clientid AS client_id,
       MIN(bloqueio) AS data_bloqueio  -- Primeiro bloqueio do cliente
   FROM workspace.prevfraude.superset_orquestrador
   WHERE lock_reason_detail_ds IN (
       'PLD', 'Terrorismo', 'Fraude ideológica', 'Subscrição', 'Fraude boleto',
       'Fraude Whatsapp', 'Fraude Ted', 'Fraude PIX', 'Bureau Único', 'Facematch',
       'Relacionamento com fraudador', 'Multiplicidade CPF', 'Fraude Gap',
       'Estorno Base 2', 'Mecanismo de devolução', 'Erro Operacional (Mecanismo de devolução)',
       'Desenquadramento da MEI', 'Ordem Judicial', 'Bloqueio Preventivo',
       'Vedação Judicial Crédito', 'Ausência de Documento Fiscal', 'Conta Laranja'
   )
   AND bloqueio <= CURRENT_DATE
   GROUP BY 1
),
base AS (
   SELECT
       hs.protocol_nm,
       hs.partner_channel_ds AS canal,
       DATE_TRUNC('month', hs.created_at_dt) AS mes,
       TRY_CAST(hs.partner_csat_cd AS INTEGER) AS csat,
       -- TRANSBORDO: humanTakeOver presente e TRUE = transbordo, ausente ou FALSE = IA resolveu
       ANY_MATCH(hs.meta_data, x -> x.KEY = 'humanTakeOver' AND UPPER(CAST(x.VALUE AS VARCHAR)) = 'TRUE') AS transbordo,
       (b.client_id IS NOT NULL AND b.data_bloqueio <= hs.created_at_dt) AS bloqueado
   FROM customer_service.customer_service.historic_service hs
   LEFT JOIN martech.martech.client_tracking_inf ct ON ct.person_id = LOWER(hs.person_uuid)
   LEFT JOIN bloqueados b ON b.client_id = ct.client_id
   WHERE hs.created_at_dt >= TIMESTAMP '2025-01-01'
     AND hs.created_at_dt < CURRENT_DATE + INTERVAL '1' DAY
     AND hs.partner_channel_ds IN ('WhatsApp', 'Chat')
     AND hs.partner_integration_origem_nm = 'AiAssistant'
),
metricas AS (
   SELECT
       mes,
       canal,
       -- Volumetria
       COUNT(DISTINCT protocol_nm) AS total_atendimentos,
       COUNT(DISTINCT CASE WHEN bloqueado THEN protocol_nm END) AS atend_bloqueados,
       COUNT(DISTINCT CASE WHEN NOT bloqueado THEN protocol_nm END) AS atend_sem_bloqueio,
       -- Retenção (geral - com bloqueados)
       COUNT(DISTINCT CASE WHEN NOT transbordo THEN protocol_nm END) AS retido_geral,
       -- Retenção (sem bloqueados)
       COUNT(DISTINCT CASE WHEN NOT bloqueado AND NOT transbordo THEN protocol_nm END) AS retido_sem_bloq,
       -- CSAT (geral - com bloqueados)
       COUNT(*) FILTER (WHERE NOT transbordo AND csat IS NOT NULL) AS votos_geral,
       COUNT(*) FILTER (WHERE NOT transbordo AND csat >= 4) AS votos_positivos_geral,
       -- CSAT (sem bloqueados)
       COUNT(*) FILTER (WHERE NOT bloqueado AND NOT transbordo AND csat IS NOT NULL) AS votos_sem_bloq,
       COUNT(*) FILTER (WHERE NOT bloqueado AND NOT transbordo AND csat >= 4) AS votos_positivos_sem_bloq
   FROM base
   GROUP BY mes, canal
)
SELECT
   mes,
   canal,
   -- Volumetria
   total_atendimentos,
   atend_bloqueados,
   atend_sem_bloqueio,
   -- Retenção geral (com bloqueados)
   ROUND(retido_geral * 100.0 / NULLIF(total_atendimentos, 0), 2) AS retencao_geral_pct,
   -- Retenção sem bloqueados (por canal)
   ROUND(retido_sem_bloq * 100.0 / NULLIF(atend_sem_bloqueio, 0), 2) AS retencao_sem_bloq_pct,
   -- CSAT geral (com bloqueados)
   votos_geral,
   ROUND(votos_positivos_geral * 100.0 / NULLIF(votos_geral, 0), 2) AS csat_geral_pct,
   -- CSAT sem bloqueados (por canal)
   votos_sem_bloq,
   ROUND(votos_positivos_sem_bloq * 100.0 / NULLIF(votos_sem_bloq, 0), 2) AS csat_sem_bloq_pct,
   -- Delta (diferença com/sem bloqueados)
   ROUND(
       (retido_sem_bloq * 100.0 / NULLIF(atend_sem_bloqueio, 0)) -
       (retido_geral * 100.0 / NULLIF(total_atendimentos, 0)),
   2) AS delta_retencao_pp,
   ROUND(
       (votos_positivos_sem_bloq * 100.0 / NULLIF(votos_sem_bloq, 0)) -
       (votos_positivos_geral * 100.0 / NULLIF(votos_geral, 0)),
   2) AS delta_csat_pp,
   -- Consolidado do mês (sem bloqueados)
   ROUND(SUM(retido_sem_bloq) OVER(PARTITION BY mes) * 100.0 / NULLIF(SUM(atend_sem_bloqueio) OVER(PARTITION BY mes), 0), 2) AS retencao_consolidada_pct,
   ROUND(SUM(votos_positivos_sem_bloq) OVER(PARTITION BY mes) * 100.0 / NULLIF(SUM(votos_sem_bloq) OVER(PARTITION BY mes), 0), 2) AS csat_consolidado_pct
FROM metricas
ORDER BY mes, canal
```

---

## Glossário de Termos

| Termo | Definição |
|-------|-----------|
| **CSAT** | Customer Satisfaction Score - Índice de satisfação do cliente |
| **Top-2-Box** | Metodologia que considera apenas as 2 notas mais altas (4 e 5) como "satisfeito" |
| **Transbordo** | Quando a IA transfere o atendimento para um humano |
| **Retenção** | Percentual de atendimentos resolvidos pela IA sem transbordo |
| **WCU** | Weekly Conversational Users - Usuários únicos por semana |
| **MAU** | Monthly Active Users - Usuários únicos por mês |
| **New Cash** | Pagamentos de 1ª parcela de acordos (novos acordos) |
| **Old Cash** | Pagamentos de parcelas recorrentes (acordos mantidos) |
| **Pod** | Unidade de execução no Kubernetes (container) |
| **Uptime** | Tempo em que o sistema está disponível e funcionando |
| **GenAI** | Generative AI - Inteligência Artificial Generativa |
| **LLM** | Large Language Model - Modelo de linguagem que alimenta a IA |

---

*Documento atualizado em: Fevereiro/2025*
*Versão: 2.0*
