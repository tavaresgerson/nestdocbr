# Provedores Assíncronos

Às vezes, o início do aplicativo deve ser adiado até uma ou mais tarefas assíncronas estarem completas. Por exemplo, talvez você não queira começar a aceitar solicitações até que a conexão com o banco de dados seja estabelecida. Você pode conseguir isso usando provedores assíncronos.

A sintaxe para isso é usar `async`/`await` com a sintaxe `useFactory`. A fábrica retorna um `Promise`, e a função de fábrica pode esperar (`await`) tarefas assíncronas. O Nest aguardará a resolução da promessa antes de instanciar qualquer classe que dependa de (injeções) desse provedor.

```ts
{
  provide: 'ASYNC_CONNECTION',
  useFactory: async () => {
    const connection = await createConnection(options);
    return connection;
  },
}
```

> **DICA**
> 
> Saiba mais sobre a sintaxe do provedor personalizado [aqui](/fundamentals/custom-providers.md).

## Exemplo
A [receita TypeORM](/recipes/sql-typeorm.md) tem um exemplo mais substancial de um provedor assíncrono.
