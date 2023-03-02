# Interceptores

Um interceptador é uma classe anotada com o decorador `@Injectable()` e implementa a interface `NestInterceptor`.

![image](https://user-images.githubusercontent.com/22455192/222297825-65cba56e-1267-4e99-8d79-aef29d08366c.png)

Os interceptores têm um conjunto de recursos úteis inspirados em [Programação Orientada a Aspectos Técnica (AOP)](https://en.wikipedia.org/wiki/Aspect-oriented_programming). Eles possibilitam:

- vincular lógica extra antes/depois da execução do método
- transformar o resultado retornado de uma função
- transformar a exceção lançada de uma função
- estender o comportamento básico da função
- substituir completamente uma função, dependendo de condições específicas (por exemplo, para fins de cache)

## Noções básicas
Cada interceptador implementa o método `intercept()`, que leva dois argumentos. O primeiro é a instância `ExecutionContext` (exatamente o mesmo objeto que para [guardas](/overview/guards.md)). O `ExecutionContext` herda de `ArgumentsHost`. Vimos `ArgumentsHost` antes no capítulo de filtros de exceção. Lá, vimos que é um invólucro em torno dos argumentos que foram passados para o manipulador original e contém diferentes matrizes de argumentos com base no tipo de aplicativo. Você pode consultar novamente o [filtros de exceção](/overview/exception-filters.md) para mais sobre este tópico.

## Contexto de execução
Estendendo `ArgumentsHost`, `ExecutionContext` também adiciona vários novos métodos auxiliares que fornecem detalhes adicionais sobre o processo de execução atual. Esses detalhes podem ser úteis na construção de interceptores mais genéricos que podem funcionar em um amplo conjunto de controladores, métodos e contextos de execução. Saiba mais sobre `ExecutionContext` [aqui](/fundamentals/execution-context.md).

## Manipulador de chamadas
O segundo argumento é um `CallHandler`. A interface `CallHandler` implementa o método `handle()`, que você pode usar para invocar o método do manipulador de rota em algum momento do seu interceptador. Se você não ligar para o método `handle()` na sua implementação do método `intercept()`, o método do manipulador de rota não será executado.

Essa abordagem significa que o método `intercept()` eficaz envolve o fluxo de solicitação/resposta. Como resultado, você pode implementar lógica personalizada antes e depois a execução do manipulador final da rota. É claro que você pode escrever código no seu método `intercept()` que executa antes chamando `handle()`, mas como você afeta o que acontece depois? Porque o método `handle()` retorna um `Observable`, podemos usar os poderosos operadores de [RxJS](https://github.com/ReactiveX/rxjs) para manipular ainda mais a resposta. Usando a terminologia de programação orientada a aspectos, a invocação do manipulador de rotas (ou seja, chamando `handle()`) é chamado de [Pointcut](https://en.wikipedia.org/wiki/Pointcut), indicando que é o ponto em que nossa lógica adicional é inserida.

Considere, por exemplo, uma entrada de solicitação `POST /cats`. Este pedido é destinado ao manipulador `create()` definido dentro do `CatsController`. Se um interceptador que não chamar de método `handle()` é chamado em qualquer lugar ao longo do caminho, o método `create()` não será executado. Uma vez que `handle()` é chamado (e seu `Observable` foi retornado), o manipulador `create()` será acionado. E uma vez que o fluxo de resposta seja recebido através do `Observable`, operações adicionais podem ser executadas no fluxo e um resultado final retornado ao invocador.

## Interceptação de aspectos
O primeiro caso de uso que examinaremos é usar um interceptador para registrar a interação do usuário (por exemplo, armazenando chamadas do usuário, despachando eventos de forma assíncrona ou calculando um timestamp). Mostramos um simples `LoggingInterceptor` abaixo:

```ts
// logging.interceptor.ts

import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
```

> **DICA**
> 
> O `NestInterceptor<T, R>` é uma interface genérica na qual `T` indica o tipo de `Observable<T>` (suportando o fluxo de resposta) e `R` é o tipo do valor agrupado por `Observable<R>`.

> **AVISO**
> 
> Interceptores, como controladores, provedores, guardas e assim por diante, podem injetar dependências através de seus constructor.

Desde que `handle()` retorne um RxJS `Observable`, temos uma grande variedade de operadores que podemos usar para manipular o fluxo. No exemplo acima, usamos o operador `tap()`, que invoca nossa função de registro anônima após o término gracioso ou excepcional do fluxo observável, mas de outra forma não interfere no ciclo de resposta.

## Interceptores de ligação
Para configurar o interceptador, usamos o decorador `@UseInterceptors()` importado do pacote `@nestjs/common`. Como [pipes](/overview/pipes.md) e [guardas](/overview/guards.md), os interceptores podem estar em escopo de controlador, escopo de método ou escopo global.

```ts
// cats.controller.ts

@UseInterceptors(LoggingInterceptor)
export class CatsController {}
```

> **DICA**
> 
> O decorador `@UseInterceptors()` é importado do pacote `@nestjs/common`.

Usando a construção acima, cada manipulador de rota definido em `CatsController` vai usar `LoggingInterceptor`. Quando alguém liga para o endpoint `GET /cats`, você verá a seguinte saída na sua saída padrão:

```
Before...
After... 1ms
```

Observe que passamos o tipo `LoggingInterceptor` (em vez de uma instância), deixando a responsabilidade pela instanciação à estrutura e permitindo a injeção de dependência. Assim como nos pipes, guardas e filtros de exceção, também podemos passar por uma instância no local:

```ts
// cats.controller.ts

@UseInterceptors(new LoggingInterceptor())
export class CatsController {}
```

Como mencionado, a construção acima anexa o interceptador a todos os manipuladores declarados por este controlador. Se queremos restringir o escopo do interceptador a um único método, simplesmente aplicamos o decorador no nível do método.

Para configurar um interceptador global, usamos o método `useGlobalInterceptors()` da instância do aplicativo Nest:

```ts
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new LoggingInterceptor());
```

Os interceptores globais são usados em todo o aplicativo, para todos os controladores e manipuladores de rotas. Em termos de injeção de dependência, interceptores globais registrados de fora de qualquer módulo (com `useGlobalInterceptors()`, como no exemplo acima) não pode injetar dependências, pois isso é feito fora do contexto de qualquer módulo. Para resolver esse problema, você pode configurar um interceptador diretamente de qualquer módulo usando a seguinte construção:

```ts
// app.module.ts

import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

> **DICA**
> 
> Ao usar essa abordagem para executar a injeção de dependência para o interceptador, observe que, independentemente da módulo onde essa construção é empregada, o interceptador é, de fato, global. Onde isso deve ser feito? Escolha o módulo onde o interceptador (`LoggingInterceptor` no exemplo acima) é definido. Além disso, `useClass` não é a única maneira de lidar com o registro personalizado do provedor. Saiba mais [aqui](/fundamentals/custom-providers.md).

## Mapeamento de resposta
Já sabemos que `handle()` retorna um `Observable`. O fluxo contém o valor retornou do manipulador de rotas e, portanto, podemos mutá-lo facilmente usando o operador `map()` do RxJS.

> **AVISO**
> 
> O recurso de mapeamento de respostas não funciona com a estratégia de resposta específica da biblioteca (usando o objeto `@Res()` diretamente é proibido).

Vamos criar o `TransformInterceptor`, que modificará cada resposta de maneira trivial para demonstrar o processo. Usará o operador `map()` de RxJS para atribuir o objeto de resposta a data da propriedade de um objeto recém-criado, retornando o novo objeto ao cliente.

```ts
// transform.interceptor.ts

import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

> **DICA**
> 
> Os interceptores do nest funcionam com `intercept()` síncrono e assíncrono. Você pode simplesmente mudar o método para `async` se necessário.

Com a construção acima, quando alguém acessar o endpoint `GET /cats`, a resposta se pareceria com o seguinte (assumindo que o manipulador de rota retorne uma matriz vazia `[]`):

```json
{
  "data": []
}
```

Os interceptores têm grande valor na criação de soluções reutilizáveis para os requisitos que ocorrem em toda uma aplicação. Por exemplo, imagine que precisamos transformar cada ocorrência de um valor `null` para uma sequência vazia `''`. Podemos fazê-lo usando uma linha de código e ligar o interceptador globalmente para que ele seja usado automaticamente por cada manipulador registrado.

```ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
```

## Mapeamento de exceção
Outro caso de uso interessante é aproveitar o operador `catchError()` de RxJS para substituir exceções lançadas:

```ts
// errors.interceptor.ts

import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  BadGatewayException,
  CallHandler,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
```

## Substituição de fluxo
Existem várias razões pelas quais às vezes podemos impedir completamente de ligar para o manipulador e retornar um valor diferente. Um exemplo óbvio é implementar um cache para melhorar o tempo de resposta. Vamos dar uma olhada em um simples interceptor de cache que retorna sua resposta de um cache. Em um exemplo realista, gostaríamos de considerar outros fatores como TTL, invalidação de cache, tamanho de cache etc., mas isso está além do escopo desta discussão. Aqui forneceremos um exemplo básico que demonstra o conceito principal.

```ts
// cache.interceptor.ts

import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```

Nosso `CacheInterceptor` tem código rígido: uma variável `isCached` e também uma resposta codificada `[]`. O ponto principal a ser observado é que retornamos um novo fluxo aqui, criado pelo operador `of()` de RxJS, portanto, o manipulador de rotas não será chamado em absoluto. Quando alguém chama um endpoint que faz uso de `CacheInterceptor`, a resposta (uma matriz vazia e codificada) será retornada imediatamente. Para criar uma solução genérica, você pode aproveitar o `Reflector` e criar um decorador personalizado. O `Reflector` é bem descrito no capítulo [guardas](/overview/guards.md).

## Mais operadores
A possibilidade de manipular o fluxo usando operadores RxJS nos oferece muitos recursos. Vamos considerar outro caso de uso comum. Imagine que você gostaria de lidar com tempos perdidos de rota. Quando o seu ponto final não retornar nada após um período de tempo, você deseja finalizar com uma resposta de erro. A seguinte construção permite isso:

```ts
// timeout.interceptor.ts

import { Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
};
```

Após 5 segundos, o processamento da solicitação será cancelado. Você também pode adicionar lógica personalizada antes de lançar uma `RequestTimeoutException` (por exemplo recursos de liberação).
