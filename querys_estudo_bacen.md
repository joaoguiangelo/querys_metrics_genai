-- ============================================================================
-- QUERIES ESTUDO BACEN - CORRIGIDAS
-- Contagem por ATENDIMENTO (protocol_nm) para consistência com métricas de retenção
-- ============================================================================


-- ============================================================================
-- QUERY 1: VOLUME TOTAL DE ENTRADAS NO BACEN (Por Mês)
-- ============================================================================
-- Racional: Total geral de reclamações Bacen, independente de atendimento prévio
-- Nota: Esta query permanece por client_id pois é o total GERAL do Bacen

SELECT
    DATE_TRUNC('month', dt_abertura) AS mes_referencia,
    COUNT(DISTINCT protocolo_id_canal) AS total_reclamacoes_bacen,
    COUNT(DISTINCT client_id) AS total_clientes_unicos
FROM workspace.gec.base_atualizacao_bacen_qs
WHERE nm_canal = 'BACEN NEON'
  AND dt_abertura >= DATE '2025-09-01'
  AND dt_abertura < CURRENT_DATE
GROUP BY DATE_TRUNC('month', dt_abertura)
ORDER BY mes_referencia DESC;


-- ============================================================================
-- QUERY 2: VOLUME BACEN APÓS ATENDIMENTO GENAI (Retenção)
-- ============================================================================
-- Racional: Atendimentos retidos pela IA que geraram reclamação no Bacen
-- Alteração: Contagem por protocol_nm (atendimentos) em vez de client_id

WITH atendimentos_genai AS (
    SELECT
        hs.protocol_nm,
        hs.person_uuid,
        hs.created_at_dt AS dt_atendimento,
        DATE_TRUNC('month', hs.created_at_dt) AS mes_atendimento,
        hs.partner_channel_ds AS canal,
        ct.client_id
    FROM customer_service.customer_service.historic_service hs
    INNER JOIN martech.martech.client_tracking_inf ct 
        ON ct.person_id = LOWER(hs.person_uuid)
    WHERE hs.created_at_dt >= TIMESTAMP '2025-09-01'
      AND hs.created_at_dt < CURRENT_DATE
      AND hs.partner_channel_ds IN ('WhatsApp', 'Chat')
      AND hs.partner_integration_origem_nm = 'AiAssistant'
      -- Retenção: NÃO houve transbordo (consistente com query de retenção)
      AND NOT ANY_MATCH(hs.meta_data, x -> x.KEY = 'humanTakeOver' AND UPPER(CAST(x.VALUE AS VARCHAR)) = 'TRUE')
),

reclamacoes_bacen AS (
    SELECT
        client_id,
        dt_abertura,
        protocolo_id_canal
    FROM workspace.gec.base_atualizacao_bacen_qs
    WHERE nm_canal = 'BACEN NEON'
      AND dt_abertura >= DATE '2025-09-01'
),

-- Cruzamento: Atendimentos que geraram reclamação
genai_para_bacen AS (
    SELECT DISTINCT
        g.protocol_nm,  -- Chave primária = atendimento
        g.mes_atendimento,
        g.canal,
        g.client_id,
        b.protocolo_id_canal AS protocolo_bacen
    FROM atendimentos_genai g
    INNER JOIN reclamacoes_bacen b 
        ON CAST(g.client_id AS VARCHAR) = CAST(b.client_id AS VARCHAR)
        AND b.dt_abertura >= CAST(g.dt_atendimento AS DATE)
        AND b.dt_abertura <= CAST(g.dt_atendimento AS DATE) + INTERVAL '30' DAY
),

metricas AS (
    SELECT
        g.mes_atendimento,
        g.canal,
        -- Base: Total de ATENDIMENTOS retidos pela IA
        COUNT(DISTINCT g.protocol_nm) AS total_atendimentos,
        -- Reclamações: ATENDIMENTOS que geraram ida ao Bacen
        COUNT(DISTINCT gb.protocol_nm) AS atendimentos_reclamaram
    FROM atendimentos_genai g
    LEFT JOIN genai_para_bacen gb 
        ON g.protocol_nm = gb.protocol_nm
    GROUP BY g.mes_atendimento, g.canal
)

SELECT
    CAST(mes_atendimento AS DATE) AS Mes_Referencia,
    canal AS Canal,
    total_atendimentos AS Base_Atendimentos_Retidos_IA,
    atendimentos_reclamaram AS Qtd_Atendimentos_Reclamaram,
    ROUND(atendimentos_reclamaram * 100.0 / NULLIF(total_atendimentos, 0), 4) AS Taxa_Insatisfacao_Bacen_Pct,
    -- Contexto: Taxa média do mês (todos os canais)
    ROUND(
        SUM(atendimentos_reclamaram) OVER(PARTITION BY mes_atendimento) * 100.0 / 
        NULLIF(SUM(total_atendimentos) OVER(PARTITION BY mes_atendimento), 0), 
    4) AS Taxa_Media_Mensal_Pct
FROM metricas
ORDER BY Mes_Referencia DESC, Canal;


-- ============================================================================
-- QUERY 3: VOLUME BACEN APÓS ATENDIMENTO HUMANO (Transbordo/Geek)
-- ============================================================================
-- Racional: Atendimentos transbordados para humano que geraram reclamação
-- Alteração: Contagem por protocol_nm (atendimentos) em vez de client_id

WITH atendimentos_humano AS (
    SELECT
        hs.protocol_nm,
        hs.person_uuid,
        hs.created_at_dt AS dt_atendimento,
        DATE_TRUNC('month', hs.created_at_dt) AS mes_atendimento,
        hs.partner_channel_ds AS canal,
        ct.client_id
    FROM customer_service.customer_service.historic_service hs
    INNER JOIN martech.martech.client_tracking_inf ct 
        ON ct.person_id = LOWER(hs.person_uuid)
    WHERE hs.created_at_dt >= TIMESTAMP '2025-09-01'
      AND hs.created_at_dt < CURRENT_DATE
      AND hs.partner_channel_ds IN ('WhatsApp', 'Chat')
      AND hs.partner_integration_origem_nm = 'AiAssistant'
      -- Transbordo: HOUVE transferência para humano
      AND ANY_MATCH(hs.meta_data, x -> x.KEY = 'humanTakeOver' AND UPPER(CAST(x.VALUE AS VARCHAR)) = 'TRUE')
),

reclamacoes_bacen AS (
    SELECT
        client_id,
        dt_abertura,
        protocolo_id_canal
    FROM workspace.gec.base_atualizacao_bacen_qs
    WHERE nm_canal = 'BACEN NEON'
      AND dt_abertura >= DATE '2025-09-01'
),

-- Cruzamento: Atendimentos que geraram reclamação
humano_para_bacen AS (
    SELECT DISTINCT
        h.protocol_nm,  -- Chave primária = atendimento
        h.mes_atendimento,
        h.canal,
        h.client_id,
        b.protocolo_id_canal AS protocolo_bacen
    FROM atendimentos_humano h
    INNER JOIN reclamacoes_bacen b 
        ON CAST(h.client_id AS VARCHAR) = CAST(b.client_id AS VARCHAR)
        AND b.dt_abertura >= CAST(h.dt_atendimento AS DATE)
        AND b.dt_abertura <= CAST(h.dt_atendimento AS DATE) + INTERVAL '30' DAY
),

metricas AS (
    SELECT
        h.mes_atendimento,
        h.canal,
        -- Base: Total de ATENDIMENTOS transbordados
        COUNT(DISTINCT h.protocol_nm) AS total_atendimentos,
        -- Reclamações: ATENDIMENTOS que geraram ida ao Bacen
        COUNT(DISTINCT hb.protocol_nm) AS atendimentos_reclamaram
    FROM atendimentos_humano h
    LEFT JOIN humano_para_bacen hb 
        ON h.protocol_nm = hb.protocol_nm
    GROUP BY h.mes_atendimento, h.canal
)

SELECT
    CAST(mes_atendimento AS DATE) AS Mes_Referencia,
    canal AS Canal,
    total_atendimentos AS Base_Atendimentos_Transbordo,
    atendimentos_reclamaram AS Qtd_Atendimentos_Reclamaram,
    ROUND(atendimentos_reclamaram * 100.0 / NULLIF(total_atendimentos, 0), 4) AS Taxa_Insatisfacao_Bacen_Pct,
    -- Contexto: Taxa média do mês (todos os canais)
    ROUND(
        SUM(atendimentos_reclamaram) OVER(PARTITION BY mes_atendimento) * 100.0 / 
        NULLIF(SUM(total_atendimentos) OVER(PARTITION BY mes_atendimento), 0), 
    4) AS Taxa_Media_Mensal_Pct
FROM metricas
ORDER BY Mes_Referencia DESC, Canal;


-- ============================================================================
-- QUERY 4: PERCENTUAL DE RECLAMAÇÕES POR CANAL (Chat vs WhatsApp)
-- ============================================================================
-- Racional: Distribuição das reclamações Bacen por canal de origem
-- Alteração: Contagem por protocol_nm (atendimentos)

WITH atendimentos_todos AS (
    SELECT
        hs.protocol_nm,
        hs.created_at_dt AS dt_atendimento,
        DATE_TRUNC('month', hs.created_at_dt) AS mes_atendimento,
        hs.partner_channel_ds AS canal,
        ct.client_id,
        CASE 
            WHEN ANY_MATCH(hs.meta_data, x -> x.KEY = 'humanTakeOver' AND UPPER(CAST(x.VALUE AS VARCHAR)) = 'TRUE')
            THEN 'Humano'
            ELSE 'GenAI'
        END AS tipo_atendimento
    FROM customer_service.customer_service.historic_service hs
    INNER JOIN martech.martech.client_tracking_inf ct 
        ON ct.person_id = LOWER(hs.person_uuid)
    WHERE hs.created_at_dt >= TIMESTAMP '2025-09-01'
      AND hs.created_at_dt < CURRENT_DATE
      AND hs.partner_channel_ds IN ('WhatsApp', 'Chat')
      AND hs.partner_integration_origem_nm = 'AiAssistant'
),

reclamacoes_bacen AS (
    SELECT client_id, dt_abertura, protocolo_id_canal
    FROM workspace.gec.base_atualizacao_bacen_qs
    WHERE nm_canal = 'BACEN NEON'
      AND dt_abertura >= DATE '2025-09-01'
),

atendimento_para_bacen AS (
    SELECT DISTINCT
        a.protocol_nm,  -- Chave = atendimento
        a.mes_atendimento,
        a.canal,
        a.tipo_atendimento
    FROM atendimentos_todos a
    INNER JOIN reclamacoes_bacen b 
        ON CAST(a.client_id AS VARCHAR) = CAST(b.client_id AS VARCHAR)
        AND b.dt_abertura >= CAST(a.dt_atendimento AS DATE)
        AND b.dt_abertura <= CAST(a.dt_atendimento AS DATE) + INTERVAL '30' DAY
),

metricas_canal AS (
    SELECT
        mes_atendimento,
        canal,
        COUNT(DISTINCT protocol_nm) AS total_atendimentos_reclamaram
    FROM atendimento_para_bacen
    GROUP BY mes_atendimento, canal
)

SELECT
    CAST(mes_atendimento AS DATE) AS mes_referencia,
    canal,
    total_atendimentos_reclamaram,
    ROUND(
        total_atendimentos_reclamaram * 100.0 / SUM(total_atendimentos_reclamaram) OVER(PARTITION BY mes_atendimento),
        2
    ) AS percentual_canal
FROM metricas_canal
ORDER BY mes_referencia DESC, canal;


-- ============================================================================
-- QUERY 5: PERCENTUAL DE RECLAMAÇÕES POR TIPO (GenAI vs Humano/Geek)
-- ============================================================================
-- Racional: Comparação direta entre retenção IA e transbordo humano
-- Alteração: Contagem por protocol_nm (atendimentos)

WITH atendimentos_todos AS (
    SELECT
        hs.protocol_nm,
        hs.created_at_dt AS dt_atendimento,
        DATE_TRUNC('month', hs.created_at_dt) AS mes_atendimento,
        hs.partner_channel_ds AS canal,
        ct.client_id,
        CASE 
            WHEN ANY_MATCH(hs.meta_data, x -> x.KEY = 'humanTakeOver' AND UPPER(CAST(x.VALUE AS VARCHAR)) = 'TRUE')
            THEN 'Humano (Geek)'
            ELSE 'GenAI (Retenção)'
        END AS tipo_atendimento
    FROM customer_service.customer_service.historic_service hs
    INNER JOIN martech.martech.client_tracking_inf ct 
        ON ct.person_id = LOWER(hs.person_uuid)
    WHERE hs.created_at_dt >= TIMESTAMP '2025-09-01'
      AND hs.created_at_dt < CURRENT_DATE
      AND hs.partner_channel_ds IN ('WhatsApp', 'Chat')
      AND hs.partner_integration_origem_nm = 'AiAssistant'
),

reclamacoes_bacen AS (
    SELECT client_id, dt_abertura, protocolo_id_canal
    FROM workspace.gec.base_atualizacao_bacen_qs
    WHERE nm_canal = 'BACEN NEON'
      AND dt_abertura >= DATE '2025-09-01'
),

atendimento_para_bacen AS (
    SELECT DISTINCT
        a.protocol_nm,  -- Chave = atendimento
        a.mes_atendimento,
        a.canal,
        a.tipo_atendimento
    FROM atendimentos_todos a
    INNER JOIN reclamacoes_bacen b 
        ON CAST(a.client_id AS VARCHAR) = CAST(b.client_id AS VARCHAR)
        AND b.dt_abertura >= CAST(a.dt_atendimento AS DATE)
        AND b.dt_abertura <= CAST(a.dt_atendimento AS DATE) + INTERVAL '30' DAY
),

metricas_tipo AS (
    SELECT
        mes_atendimento,
        tipo_atendimento,
        COUNT(DISTINCT protocol_nm) AS total_atendimentos_reclamaram
    FROM atendimento_para_bacen
    GROUP BY mes_atendimento, tipo_atendimento
)

SELECT
    CAST(mes_atendimento AS DATE) AS mes_referencia,
    tipo_atendimento,
    total_atendimentos_reclamaram,
    ROUND(
        total_atendimentos_reclamaram * 100.0 / SUM(total_atendimentos_reclamaram) OVER(PARTITION BY mes_atendimento),
        2
    ) AS percentual_tipo
FROM metricas_tipo
ORDER BY mes_referencia DESC, tipo_atendimento;


-- ============================================================================
-- QUERY CONSOLIDADA: VISÃO EXECUTIVA COMPLETA (por Atendimento)
-- ============================================================================
-- Racional: Uma única query que traz todos os dados para o dashboard
-- Todas as métricas contadas por protocol_nm (atendimentos)

WITH atendimentos_base AS (
    SELECT
        hs.protocol_nm,
        hs.created_at_dt AS dt_atendimento,
        DATE_TRUNC('month', hs.created_at_dt) AS mes_atendimento,
        hs.partner_channel_ds AS canal,
        ct.client_id,
        CASE 
            WHEN ANY_MATCH(hs.meta_data, x -> x.KEY = 'humanTakeOver' AND UPPER(CAST(x.VALUE AS VARCHAR)) = 'TRUE')
            THEN 'Humano'
            ELSE 'GenAI'
        END AS tipo_atendimento
    FROM customer_service.customer_service.historic_service hs
    INNER JOIN martech.martech.client_tracking_inf ct 
        ON ct.person_id = LOWER(hs.person_uuid)
    WHERE hs.created_at_dt >= TIMESTAMP '2025-09-01'
      AND hs.created_at_dt < CURRENT_DATE
      AND hs.partner_channel_ds IN ('WhatsApp', 'Chat')
      AND hs.partner_integration_origem_nm = 'AiAssistant'
),

reclamacoes_bacen AS (
    SELECT 
        client_id, 
        dt_abertura, 
        protocolo_id_canal,
        DATE_TRUNC('month', dt_abertura) AS mes_reclamacao
    FROM workspace.gec.base_atualizacao_bacen_qs
    WHERE nm_canal = 'BACEN NEON'
      AND dt_abertura >= DATE '2025-09-01'
),

-- Total Bacen por mês (Query 1) - Este permanece por protocolo Bacen
total_bacen AS (
    SELECT
        mes_reclamacao,
        COUNT(DISTINCT protocolo_id_canal) AS total_geral_bacen
    FROM reclamacoes_bacen
    GROUP BY mes_reclamacao
),

-- Cruzamento atendimento -> Bacen
atendimento_para_bacen AS (
    SELECT DISTINCT
        a.protocol_nm,  -- Chave = atendimento
        a.mes_atendimento,
        a.canal,
        a.tipo_atendimento
    FROM atendimentos_base a
    INNER JOIN reclamacoes_bacen b 
        ON CAST(a.client_id AS VARCHAR) = CAST(b.client_id AS VARCHAR)
        AND b.dt_abertura >= CAST(a.dt_atendimento AS DATE)
        AND b.dt_abertura <= CAST(a.dt_atendimento AS DATE) + INTERVAL '30' DAY
),

-- Base total de atendimentos (para calcular taxa)
base_atendimentos AS (
    SELECT
        mes_atendimento,
        canal,
        tipo_atendimento,
        COUNT(DISTINCT protocol_nm) AS total_atendimentos
    FROM atendimentos_base
    GROUP BY mes_atendimento, canal, tipo_atendimento
),

-- Atendimentos que geraram reclamação
metricas_reclamacoes AS (
    SELECT
        mes_atendimento,
        canal,
        tipo_atendimento,
        COUNT(DISTINCT protocol_nm) AS atendimentos_reclamaram
    FROM atendimento_para_bacen
    GROUP BY mes_atendimento, canal, tipo_atendimento
)

SELECT
    CAST(b.mes_atendimento AS DATE) AS mes_referencia,
    t.total_geral_bacen,
    b.canal,
    b.tipo_atendimento,
    b.total_atendimentos AS base_atendimentos,
    COALESCE(m.atendimentos_reclamaram, 0) AS atendimentos_reclamaram,
    -- Taxa de insatisfação por segmento
    ROUND(
        COALESCE(m.atendimentos_reclamaram, 0) * 100.0 / NULLIF(b.total_atendimentos, 0),
        4
    ) AS taxa_insatisfacao_pct,
    -- Percentual do total de reclamações atribuídas
    ROUND(
        COALESCE(m.atendimentos_reclamaram, 0) * 100.0 / 
        NULLIF(SUM(COALESCE(m.atendimentos_reclamaram, 0)) OVER(PARTITION BY b.mes_atendimento), 0),
        2
    ) AS pct_do_total_atribuido
FROM base_atendimentos b
LEFT JOIN metricas_reclamacoes m 
    ON b.mes_atendimento = m.mes_atendimento 
    AND b.canal = m.canal 
    AND b.tipo_atendimento = m.tipo_atendimento
LEFT JOIN total_bacen t 
    ON b.mes_atendimento = t.mes_reclamacao
ORDER BY mes_referencia DESC, canal, tipo_atendimento;
