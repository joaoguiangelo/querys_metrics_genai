# KPIs GenAI - Queries e Racionais

---

## 1. ENGAJAMENTO

### 1.1 Conversas Qualificadas 🆗

**Racional:** Conta conversas onde houve interação real bidirecional. O filtro de "2 mensagens GenAI + 1 resposta Customer" elimina aberturas acidentais, timeouts e interações não-engajadas. Consolida atendimento (WhatsApp/Chat) e cobrança (WhatsAppCobranca) em uma única métrica.

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

**Racional:** Mede a disponibilidade dos pods do bot via métricas de infraestrutura (Mimir/Grafana). Monitora se os containers essenciais (`gugelmin-primary`) estão rodando. Substitui a lógica de gaps por monitoramento real.
**Fonte:** Grafana (Mimir) - Extração via API ou Consulta Manual.

**Query PromQL (Grafana):**
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

## 2. IMPACTO NO NEGÓCIO

### 2.1 % Conversão (Acordo) 🆗

**Racional:** Mede efetividade do funil. Calcula taxa de conversão em cada etapa, partindo do CPF informado (lead qualificado) até acordo confirmado. Permite identificar gargalos no funil.

```sql
WITH funil_genai AS (
    SELECT 
        DATE_TRUNC('month', DataCriacao) AS mes,
        CASE
            WHEN acao = 'get-client' THEN '1_cpf_informado'
            WHEN acao = 'get-invoice-debt' THEN '2_visualizou_divida'
            WHEN acao = 'simulate-agreements' THEN '3_simulacao'
            WHEN acao = 'confirm-agreement' THEN '4_acordo_confirmado'
            WHEN acao = 'paymentslip-generate' THEN '5_boleto_gerado'
        END AS etapa,
        COUNT(DISTINCT idcliente) AS clientes
    FROM hive.neonpottencial.dbo_analyticsevento
    WHERE 
        area = 'cobranca.neon.whats.gen-ai'
        AND descricao = 'CLICK'
        AND acao IN ('get-client', 'get-invoice-debt', 'simulate-agreements', 
                     'confirm-agreement', 'paymentslip-generate')
        AND DATE(DataCriacao) >= DATE '2025-01-01'
    GROUP BY 1, 2
),

funil_pivot AS (
    SELECT
        mes,
        MAX(CASE WHEN etapa = '1_cpf_informado' THEN clientes END) AS cpf_informado,
        MAX(CASE WHEN etapa = '2_visualizou_divida' THEN clientes END) AS visualizou_divida,
        MAX(CASE WHEN etapa = '3_simulacao' THEN clientes END) AS simulacao,
        MAX(CASE WHEN etapa = '4_acordo_confirmado' THEN clientes END) AS acordo_confirmado,
        MAX(CASE WHEN etapa = '5_boleto_gerado' THEN clientes END) AS boleto_gerado
    FROM funil_genai
    GROUP BY 1
)

SELECT
    mes,
    cpf_informado,
    visualizou_divida,
    simulacao,
    acordo_confirmado,
    boleto_gerado,
    
    -- Conversões etapa a etapa
    ROUND(visualizou_divida * 100.0 / NULLIF(cpf_informado, 0), 2) AS conv_cpf_para_divida_pct,
    ROUND(simulacao * 100.0 / NULLIF(visualizou_divida, 0), 2) AS conv_divida_para_simul_pct,
    ROUND(acordo_confirmado * 100.0 / NULLIF(simulacao, 0), 2) AS conv_simul_para_acordo_pct,
    ROUND(boleto_gerado * 100.0 / NULLIF(acordo_confirmado, 0), 2) AS conv_acordo_para_boleto_pct,
    
    -- KPI Principal: CPF → Acordo
    ROUND(acordo_confirmado * 100.0 / NULLIF(cpf_informado, 0), 2) AS conversao_total_pct

FROM funil_pivot
ORDER BY mes DESC;
```

### 2.2 Recuperação de Crédito (New Cash e Old Cash) 🆗

**Racional:** Mede o volume de pagamentos efetivos segregando a entrada de caixa novo (**New Cash** - 1ª parcela) da sustentabilidade da carteira (**Old Cash** - parcelas recorrentes > 1). A visão unificada permite acompanhar o fluxo de caixa total gerado pelo canal em uma única execução.

```sql
SELECT
    date(dtpagamento) AS dia,
    CASE 
        WHEN dscanal ='WhatsApp' THEN 'Coddera'
        WHEN dscanal ='Whatsapp GenAI' THEN 'Neon AI'
        ELSE dscanal
    END AS canal,
    '6. Pagamentos' AS Funil,
    'Pgtos' AS acao,
    'NA' AS descricao,
    
    -- Coluna New Cash (apenas parcela 1)
    COUNT(CASE WHEN nrparcela = 1 THEN clientid END) AS total_new_cash,
    
    -- Coluna Old Cash (apenas parcelas > 1)
    COUNT(CASE WHEN nrparcela > 1 THEN clientid END) AS total_old_cash,
    
    -- Coluna Total (Soma de ambos, opcional para conferência)
    COUNT(clientid) AS total_geral

FROM workspace.cobranca.BancoProducao
WHERE
    dscanal IN ('Monest','1Digital','WhatsApp','Whatsapp GenAI')
    AND dtcadastronegociacao BETWEEN DATE('2024-01-01') AND CURRENT_DATE
    AND dstipopessoa = 'F'
    AND dtpagamento IS NOT NULL
    -- Removemos o filtro de nrparcela do WHERE para permitir que ambos os tipos entrem no cálculo
GROUP BY 1, 2
ORDER BY date(dtpagamento) DESC, canal;
```

---

### 2.3 % Retenção (Resolução Autônoma) 🆗

**Racional:** Mede maturidade da automação. Considera apenas conversas onde o cliente não foi transferido para humano (`transbordo_humano = FALSE`). Exclui clientes bloqueados por fraude/PLD pois esses casos têm transferência obrigatória.

```sql
WITH client_ids_bloqueados AS (
    SELECT DISTINCT 
        clientid AS client_id,
        bloqueio
    FROM workspace.prevfraude.superset_orquestrador
    WHERE lock_reason_detail_ds IN (
        'PLD', 'Terrorismo', 'Fraude ideológica', 'Subscrição',
        'Fraude boleto', 'Fraude Whatsapp', 'Fraude Ted', 'Fraude PIX',
        'Bureau Único', 'Facematch', 'Relacionamento com fraudador',
        'Multiplicidade CPF', 'Fraude Gap', 'Estorno Base 2',
        'Mecanismo de devolução', 'Erro Operacional (Mecanismo de devolução)',
        'Desenquadramento da MEI', 'Ordem Judicial', 'Bloqueio Preventivo',
        'Vedação Judicial Crédito', 'Ausência de Documento Fiscal', 'Conta Laranja'
      )
),
atendimentos AS (
    SELECT 
        DATE_TRUNC('month', hs.created_at_dt) AS mes,
        hs.protocol_nm,
        ANY_MATCH(
            hs.meta_data, 
            x -> x.KEY = 'chatThroughputEvalHumanTakeOver' 
                 AND UPPER(CAST(x.VALUE AS VARCHAR)) = 'TRUE'
        ) AS transbordo_humano,
        CASE WHEN b.client_id IS NOT NULL AND b.bloqueio <= hs.created_at_dt THEN TRUE ELSE FALSE END AS is_blocked
    FROM customer_service.customer_service.historic_service hs
    LEFT JOIN martech.martech.client_tracking_inf ct
        ON ct.person_id = LOWER(hs.person_uuid)
    LEFT JOIN client_ids_bloqueados b 
        ON b.client_id = ct.client_id
    WHERE 
        hs.created_at_dt >= TIMESTAMP '2025-01-01 00:00:00'
        AND hs.partner_channel_ds IN ('WhatsApp', 'Chat')
        AND hs.partner_integration_origem_nm = 'AiAssistant'
)
SELECT 
    mes,
    
    -- Volumetria
    COUNT(DISTINCT protocol_nm) AS total_atendimentos,
    COUNT(DISTINCT CASE WHEN is_blocked = FALSE THEN protocol_nm END) AS atend_elegivel,
    
    -- Retenção
    COUNT(DISTINCT CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE 
          THEN protocol_nm END) AS resolvido_ia,
    
    ROUND(
        COUNT(DISTINCT CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE 
              THEN protocol_nm END) * 100.0 /
        NULLIF(COUNT(DISTINCT CASE WHEN is_blocked = FALSE THEN protocol_nm END), 0),
    2) AS retencao_pct
FROM atendimentos
GROUP BY 1
ORDER BY mes DESC;
```

---

### 2.4 Custo por Conversa (Proxy Financeiro) 🆗

**Racional:** Cálculo temporário (Jan/Fev) dividindo o custo total faturado (OpenAI) pelo volume de conversas do período. Mantém-se este método até que a ingestão do Langfuse/LiteLLM tenha 100% de cobertura e paridade de valores validada.
**Fonte:** Fatura OpenAI (Numerador) + Query SQL (Denominador).

**Fórmula:** $$\text{Custo por Conversa} = \frac{\text{Valor Fatura OpenAI (R\$)}}{\text{Qtd. Conversas Brutas}}$$

**Query para Denominador (Volumetria Bruta):**
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

## 4. QUALIDADE

### 4.1 CSAT - Agentes Conversacionais 🆗

**Racional:** Usa metodologia Top-2-Box (notas 4 e 5 = satisfeito). Exclui transbordo humano (avalia só a IA) e clientes bloqueados (experiência comprometida por fatores externos). Calcula tanto por canal quanto consolidado.

```sql
WITH client_ids_bloqueados AS (
    SELECT DISTINCT 
        clientid AS client_id,
        bloqueio
    FROM workspace.prevfraude.superset_orquestrador
    WHERE lock_reason_detail_ds IN (
        'PLD', 'Terrorismo', 'Fraude ideológica', 'Subscrição',
        'Fraude boleto', 'Fraude Whatsapp', 'Fraude Ted', 'Fraude PIX',
        'Bureau Único', 'Facematch', 'Relacionamento com fraudador',
        'Multiplicidade CPF', 'Fraude Gap', 'Estorno Base 2',
        'Mecanismo de devolução', 'Erro Operacional (Mecanismo de devolução)',
        'Desenquadramento da MEI', 'Ordem Judicial', 'Bloqueio Preventivo',
        'Vedação Judicial Crédito', 'Ausência de Documento Fiscal', 'Conta Laranja'
      )
),
atendimentos AS (
    SELECT 
        DATE_TRUNC('month', hs.created_at_dt) AS mes,
        hs.partner_channel_ds AS canal,
        hs.protocol_nm,
        TRY_CAST(hs.partner_csat_cd AS INTEGER) AS csat,
        ANY_MATCH(
            hs.meta_data, 
            x -> x.KEY = 'chatThroughputEvalHumanTakeOver' 
                 AND UPPER(CAST(x.VALUE AS VARCHAR)) = 'TRUE'
        ) AS transbordo_humano,
        CASE WHEN b.client_id IS NOT NULL AND b.bloqueio <= hs.created_at_dt THEN TRUE ELSE FALSE END AS is_blocked
    FROM customer_service.customer_service.historic_service hs
    LEFT JOIN martech.martech.client_tracking_inf ct
        ON ct.person_id = LOWER(hs.person_uuid)
    LEFT JOIN client_ids_bloqueados b 
        ON b.client_id = ct.client_id
    WHERE 
        hs.created_at_dt >= TIMESTAMP '2025-01-01 00:00:00'
        AND hs.partner_channel_ds IN ('WhatsApp', 'Chat')
        AND hs.partner_integration_origem_nm = 'AiAssistant'
)
SELECT 
    mes,
    canal,
    
    -- Votos elegíveis (sem transbordo, sem bloqueio, com nota)
    SUM(CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE 
             AND csat IS NOT NULL THEN 1 ELSE 0 END) AS votos_elegiveis,
    
    -- Votos positivos (nota >= 4)
    SUM(CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE 
             AND csat >= 4 THEN 1 ELSE 0 END) AS votos_positivos,
    
    -- CSAT por canal
    ROUND(
        SUM(CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE 
                 AND csat >= 4 THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(SUM(CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE 
                        AND csat IS NOT NULL THEN 1 ELSE 0 END), 0),
    2) AS csat_pct,
    
    -- CSAT consolidado do mês (Window Function)
    ROUND(
        SUM(SUM(CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE 
                     AND csat >= 4 THEN 1 ELSE 0 END)) OVER(PARTITION BY mes) * 100.0 /
        NULLIF(SUM(SUM(CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE 
                            AND csat IS NOT NULL THEN 1 ELSE 0 END)) OVER(PARTITION BY mes), 0),
    2) AS csat_consolidado_mes_pct
FROM atendimentos
GROUP BY 1, 2
ORDER BY mes DESC, canal;
```

---
### 4.2 CSAT & Retenção - Agentes Conversacionais (Consolidada) 🆗

**Racional:** Usa metodologia Top-2-Box (notas 4 e 5 = satisfeito). Exclui transbordo humano (avalia só a IA) e clientes bloqueados (experiência comprometida por fatores externos). Calcula tanto por canal quanto consolidado.

```sql
WITH client_ids_bloqueados AS (
    SELECT DISTINCT 
        clientid AS client_id,
        bloqueio
    FROM workspace.prevfraude.superset_orquestrador
    WHERE lock_reason_detail_ds IN (
        'PLD', 'Terrorismo', 'Fraude ideológica', 'Subscrição',
        'Fraude boleto', 'Fraude Whatsapp', 'Fraude Ted', 'Fraude PIX',
        'Bureau Único', 'Facematch', 'Relacionamento com fraudador',
        'Multiplicidade CPF', 'Fraude Gap', 'Estorno Base 2',
        'Mecanismo de devolução', 'Erro Operacional (Mecanismo de devolução)',
        'Desenquadramento da MEI', 'Ordem Judicial', 'Bloqueio Preventivo',
        'Vedação Judicial Crédito', 'Ausência de Documento Fiscal', 'Conta Laranja'
      )
),
atendimentos AS (
    SELECT 
        hs.protocol_nm,
        hs.partner_channel_ds AS canal,
        DATE_TRUNC('month', hs.created_at_dt) AS mes,
        TRY_CAST(hs.partner_csat_cd AS INTEGER) AS csat,
        ANY_MATCH(
            hs.meta_data, 
            x -> x.KEY = 'chatThroughputEvalHumanTakeOver' 
                 AND UPPER(CAST(x.VALUE AS VARCHAR)) = 'TRUE'
        ) AS transbordo_humano,
        ct.client_id,
        CASE WHEN b.client_id IS NOT NULL AND b.bloqueio <= hs.created_at_dt THEN TRUE ELSE FALSE END AS is_blocked
    FROM customer_service.customer_service.historic_service hs
    LEFT JOIN martech.martech.client_tracking_inf ct 
        ON ct.person_id = LOWER(hs.person_uuid)
    LEFT JOIN client_ids_bloqueados b 
        ON b.client_id = ct.client_id
    WHERE 
        hs.created_at_dt >= TIMESTAMP '2025-01-01 00:00:00'
        AND hs.partner_channel_ds IN ('WhatsApp', 'Chat')
        AND hs.partner_integration_origem_nm = 'AiAssistant'
)
SELECT 
    mes,
    canal,
    COUNT(DISTINCT protocol_nm) AS total_atendimentos,
    COUNT(DISTINCT CASE WHEN is_blocked = TRUE THEN protocol_nm END) AS atend_bloqueados,
    COUNT(DISTINCT CASE WHEN is_blocked = FALSE THEN protocol_nm END) AS atend_sem_bloqueio,
    ROUND(COUNT(DISTINCT CASE WHEN transbordo_humano = FALSE THEN protocol_nm END) * 100.0 / NULLIF(COUNT(DISTINCT protocol_nm), 0), 2) AS retencao_geral_pct,
    ROUND(COUNT(DISTINCT CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE THEN protocol_nm END) * 100.0 / NULLIF(COUNT(DISTINCT CASE WHEN is_blocked = FALSE THEN protocol_nm END), 0), 2) AS retencao_sem_bloq_pct,
    SUM(CASE WHEN transbordo_humano = FALSE AND csat IS NOT NULL THEN 1 ELSE 0 END) AS votos_geral,
    ROUND(SUM(CASE WHEN transbordo_humano = FALSE AND csat >= 4 THEN 1 ELSE 0 END) * 100.0 / NULLIF(SUM(CASE WHEN transbordo_humano = FALSE AND csat IS NOT NULL THEN 1 ELSE 0 END), 0), 2) AS csat_geral_pct,
    SUM(CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE AND csat IS NOT NULL THEN 1 ELSE 0 END) AS votos_sem_bloq,
    ROUND(SUM(CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE AND csat >= 4 THEN 1 ELSE 0 END) * 100.0 / NULLIF(SUM(CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE AND csat IS NOT NULL THEN 1 ELSE 0 END), 0), 2) AS csat_sem_bloq_pct,
    ROUND((COUNT(DISTINCT CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE THEN protocol_nm END) * 100.0 / NULLIF(COUNT(DISTINCT CASE WHEN is_blocked = FALSE THEN protocol_nm END), 0)) - (COUNT(DISTINCT CASE WHEN transbordo_humano = FALSE THEN protocol_nm END) * 100.0 / NULLIF(COUNT(DISTINCT protocol_nm), 0)), 2) AS delta_retencao_pp,
    ROUND((SUM(CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE AND csat >= 4 THEN 1 ELSE 0 END) * 100.0 / NULLIF(SUM(CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE AND csat IS NOT NULL THEN 1 ELSE 0 END), 0)) - (SUM(CASE WHEN transbordo_humano = FALSE AND csat >= 4 THEN 1 ELSE 0 END) * 100.0 / NULLIF(SUM(CASE WHEN transbordo_humano = FALSE AND csat IS NOT NULL THEN 1 ELSE 0 END), 0)), 2) AS delta_csat_pp,
    ROUND(SUM(COUNT(DISTINCT CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE THEN protocol_nm END)) OVER(PARTITION BY mes) * 100.0 / NULLIF(SUM(COUNT(DISTINCT CASE WHEN is_blocked = FALSE THEN protocol_nm END)) OVER(PARTITION BY mes), 0), 2) AS retencao_total_mes_sem_bloq_pct,
    ROUND(SUM(SUM(CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE AND csat >= 4 THEN 1 ELSE 0 END)) OVER(PARTITION BY mes) * 100.0 / NULLIF(SUM(SUM(CASE WHEN is_blocked = FALSE AND transbordo_humano = FALSE AND csat IS NOT NULL THEN 1 ELSE 0 END)) OVER(PARTITION BY mes), 0), 2) AS csat_total_mes_sem_bloq_pct
FROM atendimentos
GROUP BY 1, 2
ORDER BY 1, 2;
```
