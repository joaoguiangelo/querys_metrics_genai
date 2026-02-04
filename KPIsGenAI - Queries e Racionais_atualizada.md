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

### 1.3 WCU Agregado (Média Mensal de Usuários Semanais) 🆗
**Racional:** Indica a constância de uso do assistente. Calcula-se primeiramente os usuários únicos de cada semana (WCU) e, posteriormente, **extrai-se a média aritmética desses WCUs semanais** para representar o mês. Diferente do MAU (que mostra alcance total), a Média de WCU demonstra o volume típico de engajamento semanal. Se a Média WCU sobe e o MAU se mantém, significa que os mesmos usuários estão voltando mais vezes (maior retenção).
**Fonte:** Trino / Data Lake / Data Warehouse (Tabela: `customer_service.customer_service.historic_service`)

```
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

## 2. IMPACTO NO NEGÓCIO

### 2.1 % Conversão (Acordo) 🆗

**Racional:** Mede efetividade do funil. Calcula taxa de conversão em cada etapa, partindo do CPF informado (lead qualificado) até acordo confirmado. Permite identificar gargalos no funil.

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

### 2.2 Recuperação de Crédito (Amount Recovered) 🆗

**Racional:** Mede o volume de pagamentos efetivos segregando a entrada de caixa novo (**New Cash** - 1ª parcela) da sustentabilidade da carteira (**Old Cash** - parcelas recorrentes > 1). A visão unificada permite acompanhar o fluxo de caixa total gerado pelo canal em uma única execução.

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

## 3. QUALIDADE

---

### 3.1 CSAT & Retenção - Agentes Conversacionais (Consolidada) 🆗

**Racional:** Usa metodologia Top-2-Box (notas 4 e 5 = satisfeito). Exclui transbordo humano (avalia só a IA) e clientes bloqueados (experiência comprometida por fatores externos). Calcula tanto por canal quanto consolidado.

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








