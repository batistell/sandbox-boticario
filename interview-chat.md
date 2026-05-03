# Entrevista: Software Engineer - Gateway de Pagamentos (Grupo Boticário)

**Recrutador (Tech Lead / Arquiteto):**
Olá! Muito prazer em falar com você hoje. Obrigado por dedicar seu tempo para esta etapa técnica do nosso processo seletivo para o time de Produtos Digitais Financeiros aqui no Grupo Boticário.

Como você deve saber, nosso time atua como uma fintech interna, sendo responsável pelo processamento de pagamentos de diversas frentes do grupo (E-commerce da Beleza na Web, milhares de lojas físicas de O Boticário, Venda Direta com Eudora e Vult, etc.). Nosso foco hoje é falar sobre **System Design** e **Arquitetura**.

Nós não buscamos uma única "resposta correta", mas queremos entender a sua linha de raciocínio, sua forma de avaliar trade-offs técnicos e como você constrói sistemas distribuídos e altamente escaláveis em Java/Kotlin.

---

### O Cenário
Estamos modernizando nossa arquitetura financeira. Você é o engenheiro ou engenheira encarregada de desenhar o novo **Gateway de Pagamentos Interno** do Grupo Boticário. 

Aqui estão nossos desafios (Requisitos Não-Funcionais):
1. **Volumetria:** Média de 5.000 transações por segundo (TPS), com picos de até 25.000 TPS na Black Friday.
2. **Disponibilidade:** 99.99%. Não pode haver Single Point of Failure. O sistema cair significa perder dinheiro na hora.
3. **Latência de Aceite:** O sistema cliente (um PDV ou o Checkout Web) precisa de uma resposta síncrona de "Transação Aceita" em **menos de 200ms**.
4. **Processamento Real:** A autorização final no adquirente (Cielo, Rede, Stone) demora de 2 a 5 segundos e precisa ser feita de forma desacoplada (assíncrona).
5. **Idempotência:** A rede oscila muito, especialmente em lojas físicas de interior. Precisamos garantir 100% que o cliente não seja cobrado duas vezes (duplo faturamento).

### Pergunta 1: Arquitetura de Alto Nível
Considerando esse cenário, **como você desenharia a arquitetura de alto nível que recebe essa requisição do cliente (Checkout/PDV)?** 

Descreva quais seriam os primeiros componentes (APIs, Load Balancers, Bancos, Filas, etc.) envolvidos para garantir que a gente receba o request, salve a intenção de pagamento de forma segura e devolva o status 202 (Accepted) em menos de 200ms. 

Qual banco de dados e solução de mensageria você escolheria para essa primeira etapa e por quê?

---
**Candidato:**

