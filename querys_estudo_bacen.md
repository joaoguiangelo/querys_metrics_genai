## 1. Qualidade da Retenção GenAI (Visão Bacen)
**Racional**
Esta query isola os atendimentos onde a IA atuou e não houve transferência para humano (seja por resolução bem-sucedida ou abandono do fluxo pelo cliente).

Lógica de Filtro: Utilizamos uma exclusão (NOT ANY_MATCH) na flag humanTakeOver. Se a flag de transbordo não existe ou é falsa, consideramos que o atendimento foi retido pela IA.

Janela de Atribuição: Cruzamos com a base do Bacen verificando se houve abertura de reclamação entre a data do atendimento e 30 dias depois.

KPIs Gerados:

Base_Clientes_Retidos_IA: Volume total de retenção.

Qtd_Clientes_que_Reclamaram: Volume de ofensores.

Taxa_Insatisfacao_Bacen_Pct: Percentual de risco regulatório.

```sql
-- ============================================================================
-- RELATÓRIO EXECUTIVO: Qualidade da Retenção GenAI vs. Bacen
-- ============================================================================

-- -----------------------------------------------------------------------------
-- ETAPA 1: Buscar atendimentos 100% retidos pela IA (Lógica Corrigida)
-- -----------------------------------------------------------------------------
WITH atendimentos_genai AS (
    SELECT
        hs.protocol_nm,
        hs.person_uuid,
        hs.created_at_dt AS dt_atendimento_genai,
        DATE_TRUNC('month', hs.created_at_dt) AS mes_atendimento,
        hs.partner_channel_ds AS canal_genai,
        ct.client_id
    FROM customer_service.customer_service.historic_service hs
    INNER JOIN martech.martech.client_tracking_inf ct 
        ON ct.person_id = LOWER(hs.person_uuid)
    WHERE hs.created_at_dt >= TIMESTAMP '2025-01-01'
      AND hs.created_at_dt < CURRENT_DATE + INTERVAL '1' DAY
      AND hs.partner_channel_ds IN ('WhatsApp', 'Chat')
      AND hs.partner_integration_origem_nm = 'AiAssistant'
      
      -- FILTRO DE RETENÇÃO (IA resolveu sozinha)
      -- Lógica: Excluímos quem teve transbordo (TRUE). O resto é retenção.
      AND NOT ANY_MATCH(hs.meta_data, x -> x.KEY = 'humanTakeOver' AND UPPER(CAST(x.VALUE AS VARCHAR)) = 'TRUE')
),

-- -----------------------------------------------------------------------------
-- ETAPA 2: Buscar Base de Reclamações Bacen
-- -----------------------------------------------------------------------------
reclamacoes_bacen AS (
    SELECT
        client_id,
        dt_abertura,
        protocolo_id_canal
    FROM workspace.gec.base_atualizacao_bacen_qs
    WHERE nm_canal = 'BACEN NEON'
      AND dt_abertura >= DATE '2025-01-01'
),

-- -----------------------------------------------------------------------------
-- ETAPA 3: Cruzamento (Quem da IA foi para o Bacen?)
-- -----------------------------------------------------------------------------
genai_para_bacen AS (
    SELECT
        g.protocol_nm,
        g.client_id,
        g.mes_atendimento,
        g.canal_genai,
        g.dt_atendimento_genai,
        b.dt_abertura
    FROM atendimentos_genai g
    INNER JOIN reclamacoes_bacen b 
        ON CAST(g.client_id AS VARCHAR) = CAST(b.client_id AS VARCHAR)
        -- Reclamação ocorreu DEPOIS do atendimento e em até 30 dias
        AND b.dt_abertura >= CAST(g.dt_atendimento_genai AS DATE)
        AND b.dt_abertura <= CAST(g.dt_atendimento_genai AS DATE) + INTERVAL '30' DAY
),

-- -----------------------------------------------------------------------------
-- ETAPA 4: Agrupamento de Dados
-- -----------------------------------------------------------------------------
metricas AS (
    SELECT
        g.mes_atendimento,
        g.canal_genai,
        -- Volumetria
        COUNT(DISTINCT g.client_id) AS total_clientes,
        COUNT(DISTINCT gb.client_id) AS clientes_reclamaram
    FROM atendimentos_genai g
    LEFT JOIN genai_para_bacen gb 
        ON CAST(g.client_id AS VARCHAR) = CAST(gb.client_id AS VARCHAR)
        AND g.protocol_nm = gb.protocol_nm
    GROUP BY g.mes_atendimento, g.canal_genai
)

-- -----------------------------------------------------------------------------
-- ETAPA 5: Apresentação Final (Nomes Amigáveis)
-- -----------------------------------------------------------------------------
SELECT
    -- 1. TEMPO E CANAL
    CAST(mes_atendimento AS DATE) AS Mes_Referencia,
    canal_genai AS Canal,
    
    -- 2. VOLUMETRIA (Quantas pessoas a IA atendeu?)
    total_clientes AS Base_Clientes_Retidos_IA,
    
    -- 3. RESULTADO NEGATIVO (Quantos reclamaram?)
    clientes_reclamaram AS Qtd_Clientes_que_Reclamaram,
    
    -- 4. INDICADORES DE QUALIDADE (Percentuais)
    ROUND(clientes_reclamaram * 100.0 / NULLIF(total_clientes, 0), 4) AS Taxa_Insatisfacao_Bacen_Pct,
    
    -- 5. CONTEXTO GERAL (Média do mês para comparar)
    ROUND(
        SUM(clientes_reclamaram) OVER(PARTITION BY mes_atendimento) * 100.0 / 
        NULLIF(SUM(total_clientes) OVER(PARTITION BY mes_atendimento), 0), 
    4) AS Taxa_Media_Mensal_Global_Pct

FROM metricas
ORDER BY Mes_Referencia DESC, Canal;
```

---

### 2. Qualidade do Transbordo Humano vs. Bacen (Benchmark)
**Racional**
Esta query serve como Linha de Base (Benchmark) para comparação. Ela isola os atendimentos que iniciaram na IA mas foram transferidos para um humano.

Lógica de Filtro: Utilizamos a inclusão (ANY_MATCH) buscando explicitamente a flag humanTakeOver = 'TRUE'.

Objetivo: Comparar se a taxa de reclamação de quem fala com o humano é maior ou menor do que quem fica retido na IA. Isso valida se a IA está "segurando" problemas ou resolvendo de fato.

```sql
-- ============================================================================
-- RELATÓRIO EXECUTIVO: Qualidade do Transbordo Humano vs. Bacen (BENCHMARK)
-- ============================================================================

-- -----------------------------------------------------------------------------
-- ETAPA 1: Buscar atendimentos com TRANSBORDO para Humano
-- -----------------------------------------------------------------------------
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
    WHERE hs.created_at_dt >= TIMESTAMP '2025-01-01'
      AND hs.created_at_dt < CURRENT_DATE + INTERVAL '1' DAY
      AND hs.partner_channel_ds IN ('WhatsApp', 'Chat')
      AND hs.partner_integration_origem_nm = 'AiAssistant'
      
      -- FILTRO DE TRANSBORDO (Foi para o Humano)
      -- Lógica: Buscamos explicitamente a flag TRUE.
      AND ANY_MATCH(hs.meta_data, x -> x.KEY = 'humanTakeOver' AND UPPER(CAST(x.VALUE AS VARCHAR)) = 'TRUE')
),

-- -----------------------------------------------------------------------------
-- ETAPA 2: Buscar Base de Reclamações Bacen
-- -----------------------------------------------------------------------------
reclamacoes_bacen AS (
    SELECT
        client_id,
        dt_abertura,
        protocolo_id_canal
    FROM workspace.gec.base_atualizacao_bacen_qs
    WHERE nm_canal = 'BACEN NEON'
      AND dt_abertura >= DATE '2025-01-01'
),

-- -----------------------------------------------------------------------------
-- ETAPA 3: Cruzamento (Quem do Humano foi para o Bacen?)
-- -----------------------------------------------------------------------------
humano_para_bacen AS (
    SELECT
        h.protocol_nm,
        h.client_id,
        h.mes_atendimento,
        h.canal,
        h.dt_atendimento,
        b.dt_abertura
    FROM atendimentos_humano h
    INNER JOIN reclamacoes_bacen b 
        ON CAST(h.client_id AS VARCHAR) = CAST(b.client_id AS VARCHAR)
        -- Reclamação ocorreu DEPOIS do atendimento e em até 30 dias
        AND b.dt_abertura >= CAST(h.dt_atendimento AS DATE)
        AND b.dt_abertura <= CAST(h.dt_atendimento AS DATE) + INTERVAL '30' DAY
),

-- -----------------------------------------------------------------------------
-- ETAPA 4: Agrupamento de Dados
-- -----------------------------------------------------------------------------
metricas AS (
    SELECT
        h.mes_atendimento,
        h.canal,
        -- Volumetria
        COUNT(DISTINCT h.client_id) AS total_clientes,
        COUNT(DISTINCT hb.client_id) AS clientes_reclamaram
    FROM atendimentos_humano h
    LEFT JOIN humano_para_bacen hb 
        ON CAST(h.client_id AS VARCHAR) = CAST(hb.client_id AS VARCHAR)
        AND h.protocol_nm = hb.protocol_nm
    GROUP BY h.mes_atendimento, h.canal
)

-- -----------------------------------------------------------------------------
-- ETAPA 5: Apresentação Final (Igual ao Modelo GenAI)
-- -----------------------------------------------------------------------------
SELECT
    -- 1. TEMPO E CANAL
    CAST(mes_atendimento AS DATE) AS Mes_Referencia,
    canal AS Canal,
    
    -- 2. VOLUMETRIA (Diferença no nome apenas para identificar o contexto)
    total_clientes AS Base_Clientes_Transbordo,
    
    -- 3. RESULTADO NEGATIVO 
    clientes_reclamaram AS Qtd_Clientes_que_Reclamaram,
    
    -- 4. INDICADORES DE QUALIDADE (Percentuais)
    ROUND(clientes_reclamaram * 100.0 / NULLIF(total_clientes, 0), 4) AS Taxa_Insatisfacao_Bacen_Pct,
    
    -- 5. CONTEXTO GERAL
    ROUND(
        SUM(clientes_reclamaram) OVER(PARTITION BY mes_atendimento) * 100.0 / 
        NULLIF(SUM(total_clientes) OVER(PARTITION BY mes_atendimento), 0), 
    4) AS Taxa_Media_Mensal_Global_Pct

FROM metricas
ORDER BY Mes_Referencia DESC, Canal;