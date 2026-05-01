# Entrevista de System Design: Gateway de Pagamentos do Grupo Boticário

## Contexto
O Grupo Boticário está modernizando sua arquitetura financeira para unificar o processamento de pagamentos de diversas frentes de negócio (E-commerce Beleza na Web, Lojas Físicas de O Boticário, Venda Direta com Eudora e Vult). Você foi encarregado de desenhar a arquitetura do novo Gateway de Pagamentos Interno, em um ecossistema focado em alta disponibilidade e performance.

## Requisitos do Sistema
1. **Volume e Escala**: O sistema deve suportar uma média de 5.000 transações por segundo (TPS), com picos de até 25.000 TPS durante eventos críticos como a Black Friday e Beauty Week.
2. **Alta Disponibilidade (99.99%)**: Não pode haver ponto único de falha (Single Point of Failure). O tempo de inatividade resulta em perda direta de receita para toda a rede de franquias e e-commerces.
3. **Baixa Latência de Resposta**: O cliente (PDV da loja ou Carrinho do E-commerce) deve receber um retorno síncrono em menos de 200ms informando se a transação foi aceita para processamento.
4. **Processamento Assíncrono**: A autorização real junto aos adquirentes externos (Cielo, Rede, Stone, etc) pode demorar de 2 a 5 segundos e deve acontecer de forma desacoplada.
5. **Idempotência e Retentativas**: Falhas de rede em integrações são comuns. O sistema deve garantir de forma determinística que o cliente não seja cobrado duas vezes, mesmo sob instabilidade.
6. **Rastreabilidade e Log**: Toda transação deve gerar uma trilha de auditoria e os logs devem facilitar o "root cause analysis" rápido por parte do time de engenharia financeira.

## O que deve ser modelado e discutido durante a entrevista
- **Arquitetura de Alto Nível**: Desenhe os componentes principais (APIs REST, mensageria/Kafka, bancos de dados, workers de integração).
- **Fluxo da Transação (Event-Driven)**: Como o dado entra no sistema, onde é persistido, como é enfileirado e como o chamador é notificado sobre o status final.
- **Escolha de Tecnologias (Data Storage)**: Qual tipo de banco de dados (Relacional vs NoSQL) utilizar para o core ledger de transações e por quê?
- **Estratégias de Resiliência**: Como você implementaria Circuit Breakers para os parceiros externos? Como lidar com "Rate Limiting" do adquirente?
- **Idempotência em Escala**: Como garantir que uma mesma requisição (mesmo `transaction_id`) não passe do estágio de validação inicial gerando faturamento duplicado?

## Critérios de Avaliação
- Capacidade de decompor um problema complexo de alta volumetria focando em tradeoffs.
- Proficiência com microsserviços, tecnologias do ecossistema moderno (Spring Boot, Kafka) e padrões assíncronos.
- Preocupação explícita com segurança de dados, escalabilidade, monitoramento e qualidade de engenharia de software de missão crítica.
