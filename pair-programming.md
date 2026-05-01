# Entrevista de Pair Programming: Split de Pagamentos (Marketplace)

## Contexto
No ecossistema do Grupo Boticário (como a Beleza na Web), um único pedido pode conter produtos próprios (Boticário) e produtos de lojistas parceiros (Sellers de Marketplace).
Ao aprovar um pagamento, precisamos calcular o "Split de Pagamentos", ou seja, dividir o valor pago entre o Gateway (taxa de processamento), a Plataforma (comissão do marketplace) e os Lojistas (valor líquido).

## O Desafio
Implemente uma classe `PaymentSplitter` (em Java ou Kotlin) contendo a lógica matemática para realizar essa divisão financeira.

### Entradas
- **Order (Pedido)**: Contém o valor total pago e uma lista de `Item`s.
- **Item**: Contém `sellerId`, `price`, e `category`.
- **Gateway Fee**: Taxa cobrada pelo processamento do cartão. Consiste em um valor fixo `F` (ex: R$ 1,50) + um percentual `P` (ex: 2,9%) sobre o valor total do pedido.
- **Marketplace Commission**: Comissão cobrada pelo Boticário dos sellers terceiros. É um percentual sobre o `price` do item que varia por categoria (ex: Perfumaria = 10%, Maquiagem = 15%). O Boticário (`sellerId = "BOTICARIO"`) é isento dessa comissão em seus próprios itens.

### Regras de Negócio
1. A **Taxa de Gateway** é calculada sobre o total do pedido. O custo dessa taxa deve ser distribuído proporcionalmente entre todos os itens do pedido com base no valor de cada item.
2. A **Comissão do Marketplace** incide sobre o valor bruto do item (apenas para parceiros).
3. O valor líquido de cada lojista é a soma dos valores de seus itens, subtraída a sua parte proporcional na Taxa de Gateway e a Comissão do Marketplace.
4. **Precisão Matemática**: Cálculos financeiros devem evitar erros de ponto flutuante (use tipos corretos para moeda, como `BigDecimal`). Arredondamentos devem ser de 2 casas decimais usando `HALF_EVEN` (Arredondamento Bancário).
5. **Regra de Ouro**: A soma total dos repasses (Gateway + Plataforma Boticário + Repasses aos Sellers) DEVE ser exatamente igual ao valor inicial do Pedido. Qualquer diferença de centavos causada por arredondamentos deve ser creditada ou debitada da Plataforma ("BOTICARIO").

### Saída Esperada
O método principal deve retornar uma lista ou mapa de extratos (`Payout`), indicando exatamente quanto cada participante deve receber:
- ID do recebedor (ex: `"GATEWAY"`, `"BOTICARIO"`, `"SELLER_A"`)
- Valor final a receber

### Exemplo de Teste de Mesa
Pedido Total: R$ 100,00
- Item 1: R$ 40,00 | Seller "A" | Perfumaria (10% comissão)
- Item 2: R$ 60,00 | Seller "B" | Maquiagem (15% comissão)
* Gateway Fee: R$ 1,00 fixo + 2% do total. (Total da taxa gateway = R$ 3,00)

Sua solução deve calcular o rateio correto para cada seller e alocar os centavos residuais adequadamente para o Boticário.

### Critérios de Avaliação
- Domínio de tipagem e cálculos financeiros (ex: uso correto de `BigDecimal` em Java).
- Código limpo, modular, com boa abstração (Clean Code e SOLID).
- Lógica matemática sólida e tratamento de edge cases (ex: itens muito baratos).
- Escrita de testes unitários cobrindo o cenário de sucesso e edge cases.
