# Solução de System Design: Gateway de Pagamentos (Grupo Boticário)

Baseado nos requisitos do desafio de engenharia do Grupo Boticário, esta solução detalha o design de um Gateway de Pagamentos interno de alta performance, projetado para suportar operações multicanal (Marketplace, Varejo Físico e Venda Direta).

## Requisitos Atendidos
- **Volume e Escalabilidade:** Suporte a 5.000+ TPS (picos de 25.000 TPS). A solução utiliza arquitetura baseada em microsserviços e filas de mensagens para escalabilidade horizontal.
- **Alta Disponibilidade (99.99%):** Implementação de múltiplas regiões (Multi-AZ), failover automático e redundância de banco de dados (Hot Standby).
- **Baixa Latência:** APIs síncronas otimizadas com cache e processamento assíncrono para operações de longa duração (autorização).
- **Idempotência:** Chaves de idempotência únicas (` idempotency_key`) e controle transacional para prevenir duplicação de cobranças.
- **Rastreabilidade:** Trilha de auditoria completa com logs imutáveis e métricas detalhadas (New Relic).

---

## 1. Arquitetura de Alto Nível

### Componentes Principais

#### 1.1. API Gateway / Edge Layer
- **Endpoint Global:** Ponto único de entrada para todas as requisições.
- **Rate Limiting:** Proteção contra abusos e picos de tráfego.
- **SSL Termination & Basic Auth:** Camada de segurança inicial.

#### 1.2. Services Layer (Microsserviços)
- **Payment Service:** Ponto de entrada para processamento de pagamentos. Responsável por validar transações, aplicar regras de negócio e coordenar o fluxo.
- **Authorization Service:** Orquestra a comunicação com adquirentes externos (Cielo, Stone, Rede). Implementa logic de retry e circuit breaker.
- **Notification Service:** Gerencia notificações assíncronas para clientes e parceiros via Webhooks e Email/SMS.
- **Merchant Service:** Gerencia informações de merchants (parceiros) e suas configurações de payout.
- **Dispute Service:** Processo de gerenciamento de contestações e chargebacks.
- **Billing Service:** Responsável por calcular impostos, comissões e provisões de pagamento.
- **Reporting & Analytics Service:** Agrega dados para relatórios operacionais e BI.

#### 1.3. Data Layer
- **Transactional DB (PostgreSQL/MySQL):** Dados relacionais (Transações, Merchant Data, Users).
- **Analytics DB (Data Warehouse):** Dados agregados para relatórios.
- **Cache (Redis):** Dados voláteis, Rate Limiting, Session State.

#### 1.4. Asynchronous Processing
- **Message Queue (Kafka/RabbitMQ):** Filas para:
  - `AUTH_REQUESTS`: Requisições de autorização para adquirentes.
  - `NOTIFICATIONS`: Envio de notificações.
  - `BILLING_EVENTS`: Eventos de faturamento.
  - `WEBHOOKS`: Disparo de webhooks para parceiros.

### Infraestrutura
- **Kubernetes (EKS/GKE/AKS):** Orquestração de contêineres.
- **Multi-Region Deployment:** Alta disponibilidade e baixa latência regional.
- **Observability:** Prometheus (métricas), Grafana (dashboards), ELK Stack (logs), Sentry (erros), New Relic (APM).

---

## 2. Fluxo da Transação (Passo a Passo)

### 2.1. Iniciação do Pagamento (Síncrono)
1. **Request:** Cliente envia requisição de pagamento com dados do cartão e valor.
2. **Authentication:** API Gateway verifica tokens e rate limits.
3. **Payment Service:**
   - Valida a requisição (schema, formato).
   - Gera um `idempotency_key` único para evitar duplicação.
   - Verifica se o `merchant_id` existe e está ativo.
   - Valida regras de fraude (se aplicável).
   - Registra transação em estado `PENDING`.
   - **Responde imediatamente** ao cliente (status `PROCESSING`) < 200ms.

### 2.2. Autorização (Assíncrono)
1. **Queueing:** Payment Service publica evento em `AUTH_REQUESTS`.
2. **Authorization Worker:**
   - Processa fila de forma ordenada/priorizada.
   - **Idempotency Check:** Verifica se a mesma `idempotency_key` já foi processada.
   - Encripta dados sensíveis do cartão (Tokenização).
   - Envia para Adquirente Externo (via API REST ou Protocolo Binário).
3. **Response Handling:**
   - **Sucesso:** Atualiza transação para `APPROVED`, inicia fluxo de Payout.
   - **Falha:** Atualiza transação para `DECLINED`, inicia fluxo de notificação de falha.
   - **Timeout/Erro:** Marca transação como `PENDING_RETRY`.

### 2.3. Payout e Split de Pagamentos (Assíncrono)
1. **Billing Trigger:** Transação aprovada gera evento para `BILLING_EVENTS`.
2. **Billing Worker:**
   - Calcula impostos e comissões (split de pagamentos).
   - Gera registros de "provisão" para Merchants e para o Boticário.
   - Atualiza saldos na tabela `wallets` ou `accounts`.
3. **Payout:**
   - Worker executa processamento de repasse (ex: via PIX, TED ou Plataforma Interna).
   - Atualiza estado da transação para `COMPLETED`.

### 2.4. Notificações (Assíncrono)
1. **Queueing:** Eventos de status disparam para `NOTIFICATIONS`.
2. **Notification Worker:**
   - Envia Webhooks para Merchants/Clientes.
   - Envia Emails/SMS/Push.
   - **Retry Logic:** Se o webhook falhar, tenta novamente N vezes com backoff.

### 2.5. Chargeback / Disputa (Assíncrono)
1. **Notification:** Recebimento de notificação de chargeback do adquirente.
2. **Dispute Service:**
   - Marca transação como `DISPUTED`.
   - Inicia investigação (envia requisição de evidências para Merchant).
   - Notifica time financeiro.
   - Atualiza status final (`CHARGEBACK_WON` ou `CHARGEBACK_LOST`).

---

## 3. Escolha de Tecnologias (Data Storage)

### 3.1. Primary DB: PostgreSQL (ou MySQL)
**Justificativa:** Transações financeiras exigem alta consistência (ACID) e integridade referencial. O PostgreSQL oferece transações robustas, suporte a JSON (para metadados), performance comprovada e mature tooling.

**Tabelas Essenciais:**
- `transactions`: (id, idempotency_key, merchant_id, amount, currency, status, type, gateway_reference, created_at, updated_at)
- `merchants`: (id, name, commission_rates, bank_details, status)
- `wallets`: (merchant_id, balance, currency, updated_at)
- `payment_events`: Trilha de auditoria de eventos.
- `gateway_credentials`: Chaves de integração com adquirentes.

### 3.2. Cache: Redis
- **Uso:**
  - Rate limiting global e por merchant.
  - Caching de configurações (comissões, taxas).
  - Limites de concorrência (para evitar double spend).
  - Storage de sessões/tokens temporários.

### 3.3. Data Warehouse: Snowflake / BigQuery / Redshift
- **Uso:**
  - Relatórios de faturamento (Daily/Weekly/Monthly).
  - Análise de performance de adquirentes.
  - Dados agregados para BI.

### 3.4. Logs & Métricas
- **Logs:** ELK Stack (Elasticsearch, Logstash, Kibana).
- **Métricas:** Prometheus + Grafana.
- **APM:** New Relic (Requisito explícito).

---

## 4. Estratégias de Resiliência

### 4.1. Circuit Breaker (para Adquirentes Externos)
- **Implementação:** Bibliotecas como Resilience4j (Java) ou Hystrix (legado).
- **Comportamento:**
  - Se o índice de falha para um adquirente específico exceder um threshold (ex: 20%), abre o circuito
- **Merchants:** Lojistas e comerciantes que recebem pagamentos.
- **Consumers:** Consumidores finais que realizam as compras.

## 3. Transaction Types (Tipos de Transação)
- **One-time Payments:** Pagamentos únicos (compras avulsas).
- **Recurring Payments:** Pagamentos recorrentes (assinaturas e mensalidades).
- **Refunds:** Processamento de reembolsos e devoluções.
- **Dispute Resolutions:** Resoluções de disputas e chargebacks (contestações).

## 4. Currencies (Moedas)
- **Multiple Currencies:** Suporte ao processamento de múltiplas moedas.
- **Real-time Exchange Rates:** Conversão de taxas de câmbio em tempo real.

## 5. Volume
- **High Volume:** Capacidade para processar um alto volume de transações de forma sustentável (TPS elevado).

## 6. Regulatory Compliance (Conformidade Regulatória)
- **PCI DSS:** Padrão de segurança de dados para a indústria de cartões de pagamento.
- **KYC (Know Your Customer):** Processos de verificação de identidade e prevenção a fraudes/lavagem de dinheiro.

## 7. Non-Functional Requirements (Requisitos Não-Funcionais)
- **High Reliability:** Alta confiabilidade (resiliência e tolerância a falhas).
- **Scalability:** Escalabilidade para suportar picos de tráfego (ex: Black Friday).
- **Low Latency:** Baixa latência nas respostas síncronas.
- **Top-notch Security:** Segurança de ponta para proteger dados sensíveis.
