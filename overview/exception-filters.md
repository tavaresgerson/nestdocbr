# Filtros de Exceção

Nest vem embutido com uma **camada de exceções** responsável pelo processamento de todas as exceções não tratadas em um aplicativo. Quando uma exceção não é tratada pelo código do aplicativo, ela é capturada por essa camada, que envia automaticamente uma resposta apropriada para o usuário.

![image](https://user-images.githubusercontent.com/22455192/221690627-a93097ea-fd66-4dcf-9f4b-7b97e4c59b56.png)

Fora da caixa, essa ação é realizada por um embutido filtro de exceção global, que lida com exceções do tipo `HttpException` (e subclasses dele). Quando uma exceção não é reconhecida (não é `HttpException` nem uma classe que herda de `HttpException`), o filtro de exceção embutido gera a seguinte resposta JSON padrão:

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

> **DICA**
> O filtro de exceção global suporta parcialmente a biblioteca `http-errors`. Basicamente, qualquer exceção lançada contendo as propriedades `statusCode` e `message` serão preenchidas adequadamente e enviadas de volta como resposta (em vez do padrão `InternalServerErrorException` para exceções não reconhecidas).

## Exibindo exceções padrão
O Nest fornece uma classe `HttpException` embutida, exposta do pacote `@nestjs/common`. Para aplicativos típicos baseados em API HTTP REST/GraphQL, é a melhor prática enviar objetos de resposta HTTP padrão quando ocorrem certas condições de erro.

Por exemplo, no `CatsController`, nós temos um método `findAll()` (um `GET` no manipulador de rota). Vamos supor que este manipulador de rota lança uma exceção por algum motivo. Para demonstrar isso, codificaremos da seguinte maneira:

```ts
// cats.controller.ts

@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```

> **DICA**
> Usamos o `HttpStatus` aqui. Este é um enum auxiliar importado do pacote `@nestjs/common`.

Quando o cliente chama esse ponto final, a resposta fica assim:

```json
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

O construtor do `HttpException` pega dois argumentos necessários que determinam o resposta:

* O argumento `response` define o corpo de resposta JSON. Pode ser um `string` ou um `object` conforme descrito abaixo.
* O argumento `status` define o Código de status HTTP.

Por padrão, o corpo de resposta JSON contém duas propriedades:

* `statusCode`: padrões para o código de status HTTP fornecido no argumento `status`
* `message`: uma breve descrição do erro HTTP com base no `status`

Para substituir apenas a parte da mensagem do corpo de resposta JSON, forneça uma sequência no argumento `response`. Para substituir todo o corpo de resposta JSON, passe um objeto no argumento `response`. Nest será serializado e retornará como o corpo de resposta JSON.

O segundo argumento do construtor - `status` - deve ser um código de status HTTP válido. A melhor prática é usar o `HttpStatus` enum importado de `@nestjs/common`.

Há um terceiro argumento do construtor ( opcional ) - options - que pode ser usado para fornecer um erro causa. Este cause o objeto não é serializado no objeto de resposta, mas pode ser útil para fins de registro, fornecendo informações valiosas sobre o erro interno que causou o HttpException para ser jogado.

Aqui está um exemplo que substitui todo o corpo da resposta e fornece uma causa de erro:

```ts
// cats.controller.ts

@Get()
async findAll() {
  try {
    await this.service.findAll()
  } catch (error) { 
    throw new HttpException({
      status: HttpStatus.FORBIDDEN,
      error: 'This is a custom message',
    }, HttpStatus.FORBIDDEN, {
      cause: error
    });
  }
}
```

Usando o exemplo acima, é assim que a resposta seria:

```json
{
  "status": 403,
  "error": "This is a custom message"
}
```

## Exceções personalizadas
Em muitos casos, você não precisará escrever exceções personalizadas e pode usar a exceção HTTP Nest incorporada, conforme descrito na próxima seção. Se você precisar criar exceções personalizadas, é uma boa prática criar suas próprias hierarquia de exceções, onde suas exceções personalizadas herdam da classe base `HttpException`. Com essa abordagem, o Nest reconhecerá suas exceções e cuidará automaticamente das respostas de erro. Vamos implementar uma exceção personalizada:

```ts
// proibed.exception.ts

export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

`ForbiddenException` estende a classe base `HttpException`, que funcionará perfeitamente com o manipulador de exceções embutido e, portanto, podemos usá-lo dentro do método `findAll()`.

```ts
// cats.controller.ts

@Get()
async findAll() {
  throw new ForbiddenException();
}
```

## Exceções HTTP integradas
O Nest fornece um conjunto de exceções padrão que herdam da classe base `HttpException`. Estes são expostos a partir do pacote `@nestjs/common` e representa muitas das exceções HTTP mais comuns:

* `BadRequestException`
* `UnauthorizedException`
* `NotFoundException`
* `ForbiddenException`
* `NotAcceptableException`
* `RequestTimeoutException`
* `ConflictException`
* `GoneException`
* `HttpVersionNotSupportedException`
* `PayloadTooLargeException`
* `UnsupportedMediaTypeException`
* `UnprocessableEntityException`
* `InternalServerErrorException`
* `NotImplementedException`
* `ImATeapotException`
* `MethodNotAllowedException`
* `BadGatewayException`
* `ServiceUnavailableException`
* `GatewayTimeoutException`
* `PreconditionFailedException`

Todas as exceções internas também podem fornecer um `cause` e uma descrição do erro usando o parâmetro `options`:

```ts
throw new BadRequestException('Something bad happened', { cause: new Error(), description: 'Some error description' })
```

Usando o exemplo acima, é assim que a resposta seria:

```json
{
  "message": "Something bad happened",
  "error": "Some error description",
  "statusCode": 400,
}
```

## Filtros de exceção
Embora o filtro de exceção básico (incorporado) possa lidar automaticamente com muitos casos para você, convém controle total sobre a camada de exceções. Por exemplo, convém adicionar log ou usar um esquema JSON diferente com base em alguns fatores dinâmicos. Filtros de exceção são projetados para exatamente esse fim. Eles permitem controlar o fluxo exato de controle e o conteúdo da resposta enviada de volta ao cliente.

Vamos criar um filtro de exceção responsável por capturar exceções que são uma instância da classe `HttpException` e implementando a lógica de resposta personalizada para eles. Para fazer isso, precisaremos acessar os objetos `Request` e `Response` da plataforma subjacente. Vamos acessar o objeto `Request` para que possamos retirar a url original e inclua isso nas informações de registro. Vamos usar o objeto `Response` para assumir o controle direto da resposta enviada, usando o método `response.json()`.

```ts
// http-exception.filter.ts

import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
```

> **DICA**
> Todos os filtros de exceção devem implementar a interface o genérica `ExceptionFilter<T>`. Isso requer que você forneça o método `catch(exception: T, host: ArgumentsHost)` com a assinatura indicada. `T` indica o tipo da exceção.

> **AVISO**
> Se você está usando `@nestjs/platform-fastify` você pode usar `response.send()` em vez de `response.json()`. Não se esqueça de importar os tipos corretos de `fastify`.

O decorador `@Catch(HttpException)` vincula os metadados necessários ao filtro de exceção, informando à Nest que esse filtro específico está procurando exceções do tipo HttpException e nada mais. O decorador `@Catch()` pode usar um único parâmetro ou uma lista separada por vírgula. Isso permite configurar o filtro para vários tipos de exceções ao mesmo tempo.

## Host de argumentos
Vejamos os parâmetros do método `catch()`. O parâmetro `exception` é o objeto de exceção atualmente sendo processado. O host parâmetro é um objeto `ArgumentsHost`. `ArgumentsHost` é um poderoso objeto utilitário que examinaremos mais adiante no capítulo do [contexto de execução*](/fundamentals/execution-context.md). Nesta amostra de código, usamos-a para obter uma referência aos objetos `Request` e `Response` que estão sendo passados para o manipulador de solicitação original (no controlador em que a exceção se origina). Nesta amostra de código, usamos alguns métodos auxiliares em `ArgumentsHost` para obter os objetos `Request` e `Response` desejados. Saiba mais sobre `ArgumentsHost` [aqui](/fundamentals/execution-context.md).

> * A razão para esse nível de abstração é que `ArgumentsHost` funciona em todos os contextos (por exemplo, no contexto do servidor HTTP com o qual estamos trabalhando agora, mas também nos microsserviços e websockets). No capítulo do contexto de execução, veremos como podemos acessar os [argumentos subjacentes](/fundamentals/execution-context.md) para qualquer contexto de execução com o poder de `ArgumentsHost` e suas funções auxiliares. Isso nos permitirá escrever filtros de exceção genéricos que operam em todos os contextos.

## Filtros de ligação
Vamos amarrar nosso novo `HttpExceptionFilter` para o método `create()` de `CatsController`.

```ts
// cats.controller.ts

@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

> **DICA**
> O decorador `@UseFilters()` que é importado do pacote `@nestjs/common`.

Nós usamos o decorador `@UseFilters()` aqui. Semelhante ao decorador `@Catch()`, ele pode usar uma única instância de filtro ou uma lista separada por vírgula de instâncias de filtro. Aqui, criamos a instância de `HttpExceptionFilter` no lugar. Como alternativa, você pode passar a classe (em vez de uma instância), deixando a responsabilidade pela instanciação à estrutura e ativando a injeção de dependência.

```ts
// cats.controller.ts

@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

> **DICA**
> Prefira aplicar filtros usando classes em vez de instâncias, quando possível. Isso reduz o uso de memória, com o Nest você pode facilmente reutilizar instâncias da mesma classe em todo o módulo.

No exemplo acima, o `HttpExceptionFilter` é aplicado apenas ao único manipulador de rota `create()`, tornando-o com escopo de método. Os filtros de exceção podem ser limpos em diferentes níveis: com escopo de método, com escopo de controlador ou com escopo global. Por exemplo, para configurar um filtro como com escopo de controlador, você faria o seguinte:

```ts
// cats.controller.ts

@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```

Esta construção configura o `HttpExceptionFilter` para cada manipulador de rota definido dentro do `CatsController`.

Para criar um filtro com escopo global, você faria o seguinte:

```ts
main.tsJS

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(3000);
}

bootstrap();
```

> **AVISO**
> O método `useGlobalFilters()` não configura filtros para gateways ou aplicativos híbridos.

Os filtros com escopo global são usados em todo o aplicativo, para todos os controladores e manipuladores de rotas. Em termos de injeção de dependência, filtros globais registrados de fora de qualquer módulo (com `useGlobalFilters()`, como no exemplo acima) não pode injetar dependências, pois isso é feito fora do contexto de qualquer módulo. Para resolver esse problema, você pode registrar um filtro com escopo global diretamente de qualquer módulo usando a seguinte construção:

```ts
// app.module.ts

import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

> **DICA**
> Ao usar essa abordagem para executar a injeção de dependência do filtro, observe que, independentemente do módulo em que essa construção é empregada, o filtro é, de fato, global. Onde isso deve ser feito? Escolha o módulo em que o filtro (`HttpExceptionFilter` no exemplo acima) é definido. Além disso, `useClass` não é a única maneira de lidar com o registro personalizado do provedor. Saiba mais [aqui](/fundamentals/custom-providers.md).

Você pode adicionar quantos filtros com esta técnica forem necessários; basta adicionar cada um à matriz de provedores.

## Capturar tudo
Para pegar toda exceção não tratada (independentemente do tipo de exceção), deixe a lista de parâmetros do decorador `@Catch()` vazia, por exemplo: `@Catch()`.

No exemplo abaixo, temos um código que é de plataforma agnóstica porque usa o [Adaptador HTTP](/faq/http-adapter.md) para fornecer a resposta e não usa nenhum dos objetos específicos da plataforma (`Request` e `Response`) diretamente:

```ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // Em certas situações `httpAdapter` pode não estar disponível no
    // método construtor, portanto devemos resolvê-lo aqui.
    const { httpAdapter } = this.httpAdapterHost;

    const ctx = host.switchToHttp();

    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };

    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

> **AVISO**
> Ao combinar um filtro de exceção que captura tudo com um filtro vinculado a um tipo específico, o filtro "Captura qualquer coisa" deve ser declarado primeiro para permitir que o filtro específico lide corretamente com o tipo vinculado.

## Herança
Normalmente, você cria filtros de exceção totalmente personalizados criados para atender aos requisitos de aplicação. No entanto, pode haver casos de uso quando você deseja simplesmente estender o padrão interno filtro de exceção global, e substitua o comportamento com base em certos fatores.

Para delegar o processamento de exceções ao filtro base, você precisa estender `BaseExceptionFilter` e chame o método herdado `catch()`.

```ts
// all-exceptions.filter.ts

import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
```

> **AVISO**
> Filtros com escopo de método e com escopo de controlador que estendem a `BaseExceptionFilter` não deve ser instanciado com `new`. Em vez disso, deixe a estrutura instanciá-los automaticamente.

A implementação acima é apenas um shell demonstrando a abordagem. Sua implementação do filtro de exceção estendido incluiria sua lógica de negócio (por exemplo, lidando com várias condições).

Filtros globais pode estender o filtro base. Isso pode ser feito de duas maneiras.

O primeiro método é injetar a referência do `HttpAdapter` ao instanciar o filtro global personalizado:

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionsFilter(httpAdapter));

  await app.listen(3000);
}
bootstrap();
```

O segundo método é usar o token `APP_FILTER` como mostrado [aqui](/overview/exception-filters.md#filtros-de-ligação).

