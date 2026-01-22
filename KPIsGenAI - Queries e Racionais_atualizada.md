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

### 1.2 % Uptime dos Agentes Conversacionais ⌛ (a ser criada)

**Racional:** Mede disponibilidade técnica. Como não existe tabela de health-check nas queries fornecidas, proponho uma **proxy baseada em gaps de atendimento**: se há períodos sem nenhuma conversa durante horário comercial, indica possível indisponibilidade. 

**Opção A - Proxy via gaps (com dados existentes):**

```sql
-- Detecta gaps de mais de 1h sem conversas durante horário comercial
-- Proxy para identificar possíveis indisponibilidades

WITH conversas_por_hora AS (
    SELECT 
        DATE_TRUNC('hour', created_at_dt) AS hora,
        COUNT(DISTINCT protocol_nm) AS qtd_conversas
    FROM customer_service.customer_service.historic_service
    WHERE 
        partner_integration_origem_nm = 'AiAssistant'
        AND partner_channel_ds IN ('WhatsApp', 'Chat')
        AND created_at_dt >= DATE '2025-01-01'
    GROUP BY 1
),

horas_esperadas AS (
    -- Gera todas as horas do período (horário comercial: 6h-23h)
    SELECT hora_gerada
    FROM UNNEST(SEQUENCE(
        TIMESTAMP '2025-01-01 06:00:00',
        CURRENT_TIMESTAMP,
        INTERVAL '1' HOUR
    )) AS t(hora_gerada)
    WHERE HOUR(hora_gerada) BETWEEN 6 AND 23
),

disponibilidade AS (
    SELECT
        DATE_TRUNC('month', he.hora_gerada) AS mes,
        COUNT(*) AS horas_esperadas,
        COUNT(cph.hora) AS horas_com_conversas,
        COUNT(*) - COUNT(cph.hora) AS horas_sem_conversas
    FROM horas_esperadas he
    LEFT JOIN conversas_por_hora cph ON cph.hora = he.hora_gerada
    GROUP BY 1
)

SELECT
    mes,
    horas_esperadas,
    horas_com_conversas,
    horas_sem_conversas,
    ROUND(horas_com_conversas * 100.0 / horas_esperadas, 2) AS uptime_proxy_pct
FROM disponibilidade
ORDER BY mes DESC;
```

**Opção B - Query para tabela de monitoramento ⌛ (a ser criada):**

```sql
-- Requer criação de tabela: workspace.genai.agent_health_checks
-- Populada por job que faz ping nos endpoints a cada 5 min

SELECT
    DATE_TRUNC('month', check_timestamp) AS mes,
    agent_name,
    COUNT(*) AS total_checks,
    SUM(CASE WHEN status = 'healthy' THEN 1 ELSE 0 END) AS checks_ok,
    ROUND(
        SUM(CASE WHEN status = 'healthy' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 
        2
    ) AS uptime_pct
FROM workspace.genai.agent_health_checks
WHERE check_timestamp >= DATE '2025-01-01'
GROUP BY 1, 2
ORDER BY mes DESC, agent_name;
```

---

## 2. IMPACTO NO NEGÓCIO

### 2.1 % Conversão (Acordo) ⌛ (a confirmar)

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
    SELECT DISTINCT clientid AS client_id
    FROM workspace.prevfraude.superset_orquestrador
    WHERE unlock_reason_ds IS NULL
      AND lock_reason_detail_ds IN (
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
        CASE WHEN b.client_id IS NOT NULL THEN TRUE ELSE FALSE END AS is_blocked
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

### 2.4 Custo por Conversa (Langfuse Integration) ⌛ (a ser criada)

**Racional:** Monitora a eficiência econômica cruzando o custo de LLM (ingestão via API Langfuse) com o volume de conversas qualificadas.
**Dependência:** Tabela `workspace.genai.custos_langfuse` (a ser criada com dados da API).

**Sugestão** ❗

```sql
WITH custos_langfuse AS (
    -- Custos agregados por mês vindos da API
    SELECT
        DATE_TRUNC('month', data_referencia) AS mes,
        SUM(total_cost_usd) AS custo_total_usd
    FROM workspace.genai.custos_langfuse
    WHERE 
        project_name IN ('cobranca-bot', 'atendimento-bot') -- Ajustar nomes conforme Langfuse
        AND data_referencia >= DATE '2025-01-01'
    GROUP BY 1
),

volumetria AS (
    -- Total de conversas qualificadas (Soma das queries do item 1.1)
    SELECT
        mes,
        SUM(conversas_qualificadas) AS total_conversas
    FROM (
        -- ... Inserir lógica da CTE metricas_atendimento do item 1.1 aqui ...
        -- Para simplificar o documento, pode-se referenciar a tabela final de KPIs se ela existir
    ) 
    GROUP BY 1
)

SELECT
    v.mes,
    c.custo_total_usd,
    v.total_conversas,
    ROUND(c.custo_total_usd / NULLIF(v.total_conversas, 0), 4) AS cpc_usd
FROM volumetria v
LEFT JOIN custos_langfuse c ON c.mes = v.mes
ORDER BY v.mes DESC;
```

---

## 4. QUALIDADE

### 4.1 CSAT - Agentes Conversacionais 🆗

**Racional:** Usa metodologia Top-2-Box (notas 4 e 5 = satisfeito). Exclui transbordo humano (avalia só a IA) e clientes bloqueados (experiência comprometida por fatores externos). Calcula tanto por canal quanto consolidado.

```sql
WITH client_ids_bloqueados AS (
    SELECT DISTINCT clientid AS client_id
    FROM workspace.prevfraude.superset_orquestrador
    WHERE unlock_reason_ds IS NULL
      AND lock_reason_detail_ds IN (
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
        CASE WHEN b.client_id IS NOT NULL THEN TRUE ELSE FALSE END AS is_blocked
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