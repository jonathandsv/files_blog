# Chaves alternativas com EFCore (Principal Keys)

Quando se trabalha com ORM é necessário um bom conhecimento das funcionalidades para conseguir mapear bem o banco de dados e criar relacionamentos corretamente para aproveitar todos os benefícios que isso nos trás.

Durante o mapeamento de uma banco legado tive a necessidade de buscar como criar chaves alternativas para indicar relacionamentos entre tabelas. Com isso cheguei a uma funcionalidade do EF Core chamada Principal Keys.

## O que é uma Principal Key?

No EFCore, os relacionamentos entre entidades são geralmente definidos com base em uma chave primária e uma chave estrangeira. A PrincipalKey é a chave principal que é referenciada por uma entidade “dependente”. Uma PrincipalKey é uma propriedade ou combinação de propriedades em uma entidade que serve como a chave principal em um relacionamento. Por padrão, o EF Core usa a propriedade marcada como [Key] ou a propriedade chamada Id como a chave principal de uma entidade. No entanto, o EF Core permite que você defina uma chave principal diferente para o relacionamento, utilizando o método HasPrincipalKey.

### Exemplo:

Imagine que você tenha duas entidades, Pedido e Cliente, onde um Pedido está relacionado a um Cliente. Por padrão, o EF Core mapeia o relacionamento com base na chave primária de Cliente (por exemplo, ClienteId). No entanto, suponha que queremos que o relacionamento seja baseado em outra chave única, como o CodigoCliente de Cliente.

```c#
public class Cliente
{
    public int ClienteId { get; set; }
    public string CodigoCliente { get; set; } // Chave alternativa
    public string Nome { get; set; }

    public ICollection<Pedido> Pedidos { get; set; }
}

public class Pedido
{
    public int PedidoId { get; set; }
    public string CodigoCliente { get; set; } // Chave estrangeira
    public DateTime DataPedido { get; set; }

    public Cliente Cliente { get; set; }
}

```

Então na configuração do contexto usamos o HasPrincipalKey:

```c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Cliente>()
        .HasMany(c => c.Pedidos)
        .WithOne(p => p.Cliente)
        .HasForeignKey(p => p.CodigoCliente)
        .HasPrincipalKey(c => c.CodigoCliente);
}

```

### Explicação do Código:

1. **Definição de Chave Alternativa (PrincipalKey)**: Ao usar o método HasPrincipalKey, definimos que o relacionamento entre Pedido e Cliente deve ser baseado na propriedade CodigoCliente de Cliente em vez da chave primária padrão (ClienteId).

2. **Chave Estrangeira**: A chave estrangeira CodigoCliente na entidade Pedido referencia a PrincipalKey (CodigoCliente) da entidade Cliente.

## Conclusão

O uso de PrincipalKey no EFCore é uma poderosa ferramenta para definir e gerenciar relacionamentos entre entidades, permitindo o uso de chaves compostas ou alternativas como chaves principais. No meu caso me ajudou bastante a trabalhar com banco de dados legados onde não poderia alterar a estrutura do banco, mas o uso de PrincipalKeys também ajuda na flexibilidade no design de banco de dados e facilita a adaptação a requisitos de negócios complexos.

Espero que isso ajude a criar estruturas de dados mais robustas e alinhadas com os requisitos de sua aplicação.
