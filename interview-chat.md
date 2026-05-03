# Entrevista: Software Engineer - Gateway de Pagamentos (Grupo Boticário)

**Recrutador (Tech Lead / Arquiteto):**
Olá! Muito prazer em falar com você hoje. Obrigado por dedicar seu tempo para esta etapa técnica do nosso processo seletivo para o time de Produtos Digitais Financeiros aqui no Grupo Boticário.

Como você deve saber, nosso time atua como uma fintech interna, sendo responsável pelo processamento de pagamentos de diversas frentes do grupo (E-commerce da Beleza na Web, milhares de lojas físicas de O Boticário, Venda Direta com Eudora e Vult, etc.). Nosso foco hoje é falar sobre **System Design** e **Arquitetura**.

Nós não buscamos uma única "resposta correta", mas queremos entender a sua linha de raciocínio, sua forma de avaliar trade-offs técnicos e como você constrói sistemas distribuídos e altamente escaláveis em Java/Kotlin.

---

### O Cenário
Estamos modernizando nossa arquitetura financeira. Você é o engenheiro ou engenheira encarregada de desenhar o novo **Gateway de Pagamentos Interno** do Grupo Boticário. 

### O Desafio
O sistema receberá requisições de pagamento de diversas fontes do nosso ecossistema (E-commerces, milhares de Lojas Físicas, Venda Direta) e precisará processá-las e roteá-las para os adquirentes externos (Cielo, Rede, Stone, etc.).

Como você começaria a abordar o design dessa solução? Fique à vontade para fazer perguntas sobre o negócio, volumetria ou qualquer outro requisito que você considere importante antes de começar a desenhar os componentes.

---
**Candidato:**

Beleza vamos lá, primeiramente gostaria de entender melhor o contexto do negócio e os requisitos não funcionais e funcionais do sistema. 
Podemos começar pelos requisitos funcionais? (seja breve na resposta agente! rs)

---

**Recrutador:**
Claro, serei direto! Em termos de **Requisitos Funcionais**, nosso MVP precisa focar no caminho feliz e no processamento do pagamento em si:
1. **Receber Pagamentos:** O gateway deve expor uma API para receber requisições de pagamento (cartão de crédito) de qualquer canal interno (PDV, E-commerce, App).
2. **Roteamento e Autorização:** O sistema deve bater no adquirente correto (Cielo, Stone, Rede) para processar e autorizar a transação.
3. **Notificação de Status:** O canal que originou a venda precisa ser notificado do status final (Aprovado ou Negado).

Apenas isso por enquanto, vamos deixar de fora fluxos de estorno e conciliação para simplificar. 

Quer aproveitar para investigar e alinhar os **requisitos não-funcionais** (volumetria, SLAs, tempo de resposta) antes de irmos para os componentes de arquitetura?

---
**Candidato:**
Claro, vamos falar dos não funcionais.
Qual o volume de transações por segundo esperado para o sistema?

---

**Recrutador:**
Ótima pergunta. Nosso pico histórico (Black Friday) atinge cerca de **2.000 transações por segundo (TPS)**.
Para o MVP, projete com base em **1.000 TPS**, mas garantindo que o sistema seja elástico para escalar a **2.000 TPS** em picos.

---
**Candidato:**
Entendido vamos anotar então, 
Volumetria: 1000 - 2000tps. 

E sobre os adquirentes que voces pretentem utilizar, tem alguma ideia?


---

**Recrutador:**
Excelente questão. **Integração com múltiplos adquirentes** é um requisito mandatório. 
Hoje trabalhamos com Cielo, Stone e Rede.

Mas temos uma particularidade: **Nossa saída principal de rede não é a melhor da Cielo**. Precisamos garantir que, se a Cielo estiver com problemas (timeout, erro), o sistema consiga rotear automaticamente para Stone ou Rede (fallback). 

---
**Candidato:**
Nossa isso aumenta um pouco a complexidade, mas é necessário pra garantir a integridade dos pagamentos. 

Outra questão importante, em relação ao tempo de resposta, qual o SLA máximo para autorização? Precisa ser síncrono ou assíncrono?

---

**Recrutador:**
Esse é o ponto mais crítico do nosso design! 
O cliente na ponta (o PDV da loja ou o carrinho do E-commerce) não pode ficar travado esperando, porque as APIs dos adquirentes (Cielo, Stone) podem levar de 2 a 5 segundos para processar a autorização real. 

Portanto, **nossa resposta inicial para o cliente deve ser síncrona e extremamente rápida (menos de 200ms)**. Nós apenas recebemos a intenção de pagamento, salvamos com segurança e devolvemos um status de "Aceito para processamento" (ex: HTTP 202). 
A autorização real junto ao adquirente externo deve ser feita de forma **assíncrona**.

Aproveitando, nossa meta de disponibilidade é de **99.99%**. Se nós cairmos, as lojas não vendem, o que significa perda financeira imediata para os franqueados.

Com todos esses requisitos na mesa (Assíncrono, < 200ms na entrada, 99.99% de SLA e 2.000 TPS), a bola agora é sua:

**Como você desenharia a arquitetura (Bancos, Filas, APIs) que recebe esse request inicial do cliente de forma síncrona, garante que não vamos perder o dado, e responde em menos de 200ms?**

---
**Candidato:**

Beleza vamos anotar os requisitos funcionais então


1.Receber pagamento (gateway) 
2. Roteamento e autorização (adquirentes cielo, stone e rede) 
3. Notificação de status para o canal que originou a venda.


E agora os não funcionais:

1. Volumetria: 1000 - 2000tps.
2. Múltiplos adquirentes: Cielo, Stone e Rede.
3. SLA Máximo: 2 segundos (para não impactar a experiência do caixa na loja física).
4. Fluxo: Síncrono para o cliente, mas assíncrono no backoffice para processamento e notificações.
5. Tempo de resposta: 200ms < p99
6. Disponibilidade: 99.99%

Legal, vocês utilizam algum tipo de message broker hoje? Kafka, RabbitMQ, AWS SQS?

---
***Recrutador:***
Sim, utilizamos **AWS SQS** para filas de processamento interno.

Com base nesse cenário, apresente como você desenharia o sistema de Gateway de Pagamentos.

---
