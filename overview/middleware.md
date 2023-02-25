# Middleware

Middleware é uma função chamada antes do manipulador de rotas. As funções de middleware têm acesso aos objetos `req`, `res` e `next()` na função de middleware no ciclo de solicitação-resposta do aplicativo. A próxima função middleware é geralmente indicada por uma variável chamada `next`.

![image](https://user-images.githubusercontent.com/22455192/221360374-ea1e4936-6550-4c70-963e-dacec6d1a155.png)

O middleware do nest é, por padrão, equivalente ao middleware do [Express](https://expressjs.com/en/guide/using-middleware.html). A seguinte descrição da documentação oficial do Express descreve os recursos do middleware:

As funções de middleware podem executar as seguintes tarefas:
* execute qualquer código.
* faça alterações nos objetos de solicitação e resposta.
* encerre o ciclo de solicitação-resposta.
* chame a próxima função de middleware na pilha.
* se a função de middleware atual não encerrar o ciclo de solicitação-resposta, ela deverá chamar `next()` para passar o controle para a próxima função de middleware. Caso contrário, o pedido será deixado em espera.

Você implementa um middleware personalizado no Nest em uma função ou em uma classe com um decorador `@Injectable()`. A classe deve implementar a interface `NestMiddleware`, enquanto a função não possui requisitos especiais. Vamos começar implementando um recurso simples de middleware usando o método de classe.

> AVISO
> `Express` e `fastify` manipula o middleware de maneira diferente e fornece assinaturas de métodos diferentes, [leia mais aqui](/techniques/performance).

```ts
// logger.middleware.ts

import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('Request...');
    next();
  }
}
```

## Injeção de dependência
O middleware do nest suporta totalmente a Injeção de Dependência. Assim como nos provedores e controladores, eles são capazes de injetar dependências disponíveis no mesmo módulo. Como sempre, isso é feito através do `constructor`.

## Aplicando middleware
Não há lugar para middleware no decorador `@Module()`. Em vez disso, configuramos eles usando o método `configure()` da classe do módulo. Módulos que incluem middleware precisam implementar a interface `NestModule`. Vamos configurar o `LoggerMiddleware` no nível do `AppModule`.

```ts
// app.module.ts

import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
```

No exemplo acima, configuramos o `LoggerMiddleware` para o manipulador de rota `/cats` que foram definidos anteriormente dentro do `CatsController`. Também podemos restringir ainda mais um middleware a um método de solicitação específico, passando um objeto que contém o caminho de rota e solicitar `method` para o método `forRoutes()` ao configurar o middleware. No exemplo abaixo, observe que importamos o enum `RequestMethod` para fazer referência ao tipo de método de solicitação desejado.

```ts
// app.module.ts

import { Module, NestModule, RequestMethod, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
```

> **DICA**
> O método `configure()` pode ser feito assíncrono usando `async/await` (por exemplo, você pode aguardar à conclusão de uma operação assíncrona dentro do corpo do método `configure()`).

> **AVISO**
> Ao usar o adaptador do Express, o aplicativo NestJS irá registrar `json` e `urlencoded` do pacote `body-parser` por padrão. Isso significa que se você deseja personalizar esse middleware através do `MiddlewareConsumer`, você precisa desativar o middleware global configurando o `bodyParser` para `false` ao criar o aplicativo com `NestFactory.create()`.

## Wildcards de rota
Rotas baseadas em padrões também são suportadas. Por exemplo, o asterisco é usado como um curinga, e corresponderá a qualquer combinação de caracteres:

```ts
forRoutes({ path: 'ab*cd', method: RequestMethod.ALL });
```

O `'ab*cd'` caminho da rota corresponderá a `abcd`, `ab_cd`, `abecd`, e assim por diante. Os caracteres `?`, `+`, `*`, e `()` pode ser usado em um caminho de rota e são subconjuntos de suas contrapartes de expressão regular. O hífen (`-`) e o ponto (`.`) são interpretados literalmente por caminhos baseados em string.

> **AVISO**
> O pacote `fastify` usa a versão mais recente do pacote `path-to-regexp`, que não suporta mais asteriscos curinga `*`. Em vez disso, você deve usar parâmetros ( por exemplo `(.*)`, `:splat*`).

## Consumidor de middleware
O `MiddlewareConsumer` é uma classe de ajudante. Ele fornece vários métodos internos para gerenciar middleware. Todos eles podem ser simplesmente encadeados no [estilo fluente](https://en.wikipedia.org/wiki/Fluent_interface). O método `forRoutes()` pode levar uma única sequência, várias strings, um objeto `RouteInfo`, uma classe de controlador e até várias classes de controlador. Na maioria dos casos, você provavelmente passará uma lista de controladores separado por vírgulas. Abaixo está um exemplo com um único controlador:

```ts
// app.module.ts

import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes(CatsController);
  }
}
```

> **DICA**
> O método `apply()` pode usar um único middleware ou vários argumentos para especificar vários middlewares.

## Excluindo rotas
Às vezes queremos excluir certas rotas da aplicação do middleware. Podemos facilmente excluir determinadas rotas com o método `exclude()`. Este método pode levar uma única sequência, várias cadeias ou um objeto `RouteInfo` que identifica rotas a serem excluídas, como mostrado abaixo:

```ts
consumer
  .apply(LoggerMiddleware)
  .exclude(
    { path: 'cats', method: RequestMethod.GET },
    { path: 'cats', method: RequestMethod.POST },
    'cats/(.*)',
  )
  .forRoutes(CatsController);
```

> **DICA**
> O método `exclude()` suporta parâmetros curinga usando caminho para o pacote `regexp`.

Com o exemplo acima, LoggerMiddleware será vinculado a todas as rotas definidas dentro CatsControllerexceto os três passaram para o exclude() método.

## Middleware funcional
O LoggerMiddleware a classe que usamos é bastante simples. Não possui membros, métodos adicionais e dependências. Por que não podemos simplesmente defini-lo em uma função simples em vez de em uma classe? De fato, nós podemos. Esse tipo de middleware é chamado middleware funcional. Vamos transformar o middleware do logger de middleware baseado em classe em middleware funcional para ilustrar a diferença:

```ts
// logger.middleware.ts

import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`Request...`);
  next();
};
```

E use-o dentro do `AppModule`:

```ts
// app.module.ts

consumer
  .apply(logger)
  .forRoutes(CatsController);
```

> **DICA**
> Considere usar o mais simples middleware funcional como alternativa e construa seus métodos de forma que não precisem de dependências.

## Múltiplos middlewares
Como mencionado acima, para vincular vários middleware executados sequencialmente, basta fornecer uma lista separada por vírgula dentro do método `apply()`:

```ts
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);
```

## Middleware global
Se quisermos vincular middleware a todas as rotas registradas de uma só vez, podemos usar o método `use()` fornecido pela instância `INestApplication`:

```ts
// main.tsJS

const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(3000);
```

> **DICA**
> Não é possível acessar o contêiner DI em um middleware global. Você pode usar um middleware funcional ao invés disso, ao usar `app.use()`. Como alternativa, você pode usar um middleware de classe e consumi-lo com `.forRoutes('*')` dentro do `AppModule` (ou qualquer outro módulo).

