
# O que são consultas projetadas no EF Core? Exemplos avançados

No início da minha carreira, percebi um padrão comum nas aplicações em que trabalhava: as APIs frequentemente retornavam toda a entidade que representava a tabela no banco de dados. Era comum buscar a tabela inteira e usar um mapper para transformar os dados, seja para renderizar uma página Razor ou retornar o resultado de uma API. Por convenção, acabava seguindo esse padrão nas features entregues pelos outros desenvolvedores da equipe, onde a maioria dos campos da entidade estavam presentes na classe de retorno, mesmo que a tela utilizasse apenas uma fração dessas propriedades.

Trabalhei dessa forma por um tempo até me deparar com um problema inevitável: **os requests para a API começaram a demorar muito para finalizar**. 

---

## Um Caso Real

Uma das telas que desenvolvi era uma lista com vários registros, exibindo uma grid com filtros e campos básicos como nome, descrição e mais cinco informações relevantes de cada registro. A classe de retorno (chamada `Output` na época) tinha cerca de **40 propriedades**, mas a grid utilizava apenas **7 dessas propriedades**.

A grid era paginada, exibindo pelo menos 10 registros por página. Isso significava que cada request enviava 400 campos para o front-end, sendo que apenas 70 eram realmente utilizados. Durante o desenvolvimento local, não percebi o problema, pois o ambiente localhost mascarava a latência. No entanto, quando o sistema foi testado pelo QA, a bomba explodiu: o cliente tinha o hábito de selecionar **200 registros por página**, o que resultava em **8000 campos carregados em uma única requisição**. Isso, obviamente, era inaceitável.

A solução inicial foi **reduzir os campos retornados**. Em vez de 40 propriedades, passei a retornar apenas as 7 necessárias. Essa mudança simples reduziu o tamanho do JSON de **10 MB para menos de 1 MB**.

Apesar dessa melhora, percebi outro problema: mesmo enviando menos dados para o front-end, a API ainda estava carregando a entidade completa na memória antes de aplicar a transformação. Isso me fez refletir sobre como criar queries mais eficientes e diretas. Foi aí que descobri as **consultas projetadas**.

---

## O que são Consultas Projetadas?

Consultas projetadas são queries que retornam **somente os dados necessários**, sem carregar a entidade completa na memória. Isso é feito por meio de **projeções**, onde os dados são mapeados diretamente para um DTO, uma classe anônima ou até mesmo uma estrutura personalizada.

No .NET, utilizamos o método `Select` do LINQ (Language Integrated Query) para realizar essas projeções.

---

## Exemplos Práticos de Consultas Projetadas

### 1. Projeção Básica

```csharp
var resultado = await _dbContext.Usuarios
    .Where(u => u.Ativo)
    .Select(u => new UsuarioDto
    {
        Nome = u.Nome,
        Email = u.Email
    })
    .ToListAsync();
```

Neste exemplo:
- Filtramos apenas os usuários ativos.
- A projeção retorna apenas as propriedades `Nome` e `Email`, em vez de carregar a entidade `Usuario` inteira.

---

### 2. Projeções com Classes Aninhadas

Cenários reais muitas vezes envolvem estruturas mais complexas, como classes aninhadas:

```csharp
var pedidos = await _dbContext.Pedidos
    .Select(p => new PedidoDto
    {
        Id = p.Id,
        Data = p.Data,
        Cliente = new ClienteDto
        {
            Nome = p.Cliente.Nome,
            Telefone = p.Cliente.Telefone
        }
    })
    .ToListAsync();
```

Aqui, o DTO `PedidoDto` inclui um sub-objeto `ClienteDto`. Apenas os dados necessários são carregados.

---

### 3. Projeções com Agrupamento

Você pode usar projeções em conjunto com `GroupBy` para calcular dados agregados:

```csharp
var resumoVendas = await _dbContext.Vendas
    .GroupBy(v => v.ProdutoId)
    .Select(g => new ResumoVendasDto
    {
        ProdutoId = g.Key,
        QuantidadeVendida = g.Sum(v => v.Quantidade),
        TotalFaturado = g.Sum(v => v.Preco * v.Quantidade)
    })
    .ToListAsync();
```

No exemplo:
- Agrupamos vendas por `ProdutoId`.
- Calculamos a quantidade total vendida e o valor faturado para cada produto.

> **Nota:** Alguns bancos de dados, como Oracle, podem gerar queries mais eficientes com versões mais recentes do EF Core, como a partir do .NET 6.

---

### 4. Consultas Projetadas com Subqueries

Subqueries permitem obter informações adicionais sem carregar entidades completas. Por exemplo:

```csharp
var pedidos = await _dbContext.Pedidos
    .Select(p => new PedidoDto
    {
        Id = p.Id,
        Data = p.Data,
        ClienteNome = p.Cliente.Nome,
        TotalPedidosCliente = _dbContext.Pedidos
            .Where(pedido => pedido.ClienteId == p.ClienteId)
            .Count() // Subquery para contar pedidos do cliente
    })
    .ToListAsync();
```

Neste caso:
- A propriedade `TotalPedidosCliente` utiliza uma subquery para contar o número total de pedidos de cada cliente.

---

## Cuidados ao Usar Consultas Projetadas

1. **Revise o SQL gerado:** Sempre inspecione o SQL gerado para garantir que ele seja eficiente. Queries desnecessariamente complexas podem impactar negativamente a performance.
2. **Use `LogTo`:** Configure o EF Core para exibir o SQL gerado no console ou em um arquivo de log. Exemplo:
   ```csharp
   optionsBuilder
       .UseSqlServer("SuaConnectionStringAqui")
       .LogTo(Console.WriteLine, Microsoft.Extensions.Logging.LogLevel.Information);
   ```
3. **Evite projeções excessivamente complexas:** Consultas projetadas devem ser claras e objetivas. Projeções muito complicadas podem dificultar a manutenção do código e aí o colega que pegar o código depois vai te xingar.

---

## Conclusão

Consultas projetadas são uma ferramenta poderosa para melhorar a performance e reduzir o consumo de memória em suas aplicações. Ao retornar apenas os dados necessários, você otimiza o tráfego de rede e reduz a sobrecarga no front-end e no back-end.

Seja projetando dados básicos, agrupamentos ou utilizando subqueries, o importante é SEMPRE REVISAR o SQL gerado e garantir que está performático e não tem um sql maluco. Com essas práticas, você terá uma aplicação mais eficiente e preparada para escalar.
