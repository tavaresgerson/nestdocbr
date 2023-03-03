# Decoradores Customizados

O nest é construído em torno de um recurso de linguagem chamado decoradores. Os decoradores são um conceito bem conhecido em muitas linguagens de programação comumente usadas, mas no mundo do JavaScript ainda são relativamente novos. Para entender melhor como os decoradores funcionam, recomendamos a leitura este [artigo](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841). Aqui está uma definição simples:

> Um decorador ES2016 é uma expressão que retorna uma função e pode usar um descritor de destino, nome e propriedade como argumentos. Você o aplica prefixando o decorador com um caractere `@` e colocando isso no topo do que você está tentando decorar. Decoradores podem ser definidos para uma classe, um método ou uma propriedade.

## Parâmetros de Decoradores
Nest fornece um conjunto de parâmetros de decoradores que você pode usar junto com os manipuladores de rota HTTP. Abaixo está uma lista dos decoradores fornecidos e dos objetos Express (ou Fastify) que eles representam

|                         |               |
|-------------------------|---------------|
| `@Request()`, `@Req()`  | `req`         |
| `@Response()`, `@Res()` | `res`         |
| `@Next()`               | `next`        |
| `@Session()`            | `req.session` |
| `@Param(param?: string)` | `req.params`/`req.params[param]`|
| `@Body(param?: string)`  | `req.body`/`req.body[param]` |
| `@Query(param?: string)`	| `req.query`/`req.query[param]` |
| `@Headers(param?: string)`  | `req.headers`/`req.headers[param]` |
| `@Ip()`                     | `req.ip`    |
| `@HostParam()`              | `req.hosts` |

Além disso, você pode criar o seu próprio decoradores personalizados. Por que isso é útil?

No mundo node.js, é prática comum anexar propriedades ao pedido objeto. Em seguida, você os extrai manualmente em cada manipulador de rota, usando código como o seguinte:

```ts
const user = req.user;
```

Para tornar seu código mais legível e transparente, você pode criar um decorador `@User()` e reutilize-o em todos os seus controladores.

```ts
// user.decorator.ts

import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

Em seguida, você pode simplesmente usá-lo onde quer que atenda às suas necessidades.

```ts
@Get()
async findOne(@User() user: UserEntity) {
  console.log(user);
}
```

## Passando dados
Quando o comportamento do seu decorador depende de algumas condições, você pode usar o parâmetro `data` para passar um argumento para a função de fábrica do decorador. Um caso de uso para isso é um decorador personalizado que extrai propriedades do objeto de solicitação por chave. Vamos supor, por exemplo, que nossa [camada de autenticação](/techniques/authentication.md) valida solicitações e anexa uma entidade do usuário ao objeto de solicitação. A entidade do usuário para uma solicitação autenticada pode se parecer com:

```json
{
  "id": 101,
  "firstName": "Alan",
  "lastName": "Turing",
  "email": "alan@email.com",
  "roles": ["admin"]
}
```

Vamos definir um decorador que usa um nome de propriedade como chave e retorna o valor associado se existir (ou indefinido se não existir, ou se o objeto `user` não foi criado).

```ts
// user.decorator.ts

import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);
```

Veja como você pode acessar uma propriedade específica por meio do decorador `@User()` no controlador:

```ts
@Get()
async findOne(@User('firstName') firstName: string) {
  console.log(`Hello ${firstName}`);
}
```

Você pode usar esse mesmo decorador com teclas diferentes para acessar propriedades diferentes. Se o objeto `user` é profundo ou complexo, isso pode facilitar a legibilidade das implementações do manipulador de solicitações.

> **DICA**
> 
> Para usuários do TypeScript, observe que `createParamDecorator<T>()` é genérico. Isso significa que você pode impor explicitamente a segurança do tipo, por exemplo `createParamDecorator<string>((data, ctx) => ...)`. Como alternativa, especifique um tipo de parâmetro na função de fábrica, por exemplo `createParamDecorator((data: string, ctx) => ...)`. Se você omitir os dois, o tipo para `data` será `any`.

## Trabalhando com pipes
Nest trata decoradores personalizados da mesma maneira que os pipes (`@Body()`, `@Param()` e `@Query()`). Isso significa que os pipes são executados para os parâmetros anotados personalizados, bem como (em nossos exemplos, o argumento `user`). Além disso, você pode aplicar o pipe diretamente ao decorador personalizado:

```ts
@Get()
async findOne(
  @User(new ValidationPipe({ validateCustomDecorators: true }))
  user: UserEntity,
) {
  console.log(user);
}
```

> **DICA**
> 
> Observe que a opção `validateCustomDecorators` deve ser definida como `true`. `ValidationPipe` não valida argumentos anotados com os decoradores personalizados por padrão.

## Composição do decorador
Nest fornece um método auxiliar para compor vários decoradores. Por exemplo, suponha que você queira combinar todos os decoradores relacionados à autenticação em um único decorador. Isso pode ser feito com a seguinte construção:

```ts
// auth.decorator.ts

import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
```

Você pode usar esse decorador customizado `@Auth()` da seguinte forma:

```ts
@Get('users')
@Auth('admin')
findAllUsers() {}
```

Isso tem o efeito de aplicar todos os quatro decoradores com uma única declaração.

> **AVISO**
> 
> O decorador `@ApiHideProperty()` do pacote `@nestjs/swagger` não é composível e não funcionará corretamente com a função `applyDecorators`.

