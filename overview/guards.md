# Guardas

Um guarda é uma classe anotada com o decorador `@Injectable()`, que implementa a interface `CanActivate`.

![image](https://user-images.githubusercontent.com/22455192/222290632-54fa84b2-c153-4435-baf4-b6a6dc27e480.png)

Guardas têm uma única responsabilidade. Eles determinam se uma determinada solicitação será tratada pelo manipulador de rota ou não, dependendo de certas condições (como permissões, funções, ACLs, etc.) presentes em tempo de execução. Isso geralmente é chamado de autorização. Autorização (e seu primo, autenticação, com o qual geralmente colabora) foi normalmente tratado por [middleware](/overview/middleware.md) em aplicações tradicionais com o Express. O middleware é uma ótima opção para autenticação, pois coisas como validação de token e anexação de propriedades ao request o objeto não está fortemente conectado a um contexto de rota específico (e seus metadados).

Mas o middleware, por sua natureza, é burro. Não sabe qual manipulador será executado depois de chamar a função `next()`. Por outro lado, Guardas ter acesso a instância `ExecutionContext` e portanto, saiba exatamente o que será executado a seguir. Eles são projetados, assim como filtros de exceção, pipes e interceptores, para permitir que você interponha a lógica de processamento exatamente no ponto certo no ciclo de solicitação/resposta e faça-o declarativamente. Isso ajuda a manter seu código DRY e declarativo.

> **DICA**
> 
> Os Guardas são executados depois de todos os middleware, ou antes qualquer interceptador ou pipes.

## Guardas de autorização
Como mencionado, autorização é um ótimo caso de uso para guardas, porque rotas específicas devem estar disponíveis apenas quando o invocador (geralmente um usuário autenticado específico) tiver permissões suficientes. O `AuthGuard` que construiremos agora assume um usuário autenticado (e que, portanto, um token é anexado aos cabeçalhos de solicitação). Ele extrairá e validará o token e usará as informações extraídas para determinar se a solicitação pode prosseguir ou não.

```ts
// auth.guard.ts

import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

> **DICA**
> 
> Se você está procurando um exemplo do mundo real sobre como implementar um mecanismo de autenticação em seu aplicativo, [visite este capítulo](/security/authentication.md). Da mesma forma, para um exemplo de autorização mais sofisticado, verifique esta [página](/security/authorization.md).

A lógica dentro da função `validateRequest()` pode ser tão simples ou sofisticada quanto necessário. O ponto principal deste exemplo é mostrar como os guardas se encaixam no ciclo de solicitação/resposta.

Todo guarda deve implementar uma função `canActivate()`. Esta função deve retornar um booleano, indicando se a solicitação atual é permitida ou não. Ele pode retornar a resposta de forma síncrona ou assíncrona (por meio de um `Promise` ou `Observable`). Nest usa o valor de retorno para controlar a próxima ação:

* se voltar `true`, a solicitação será processada.
* se voltar `false`, Nest negará o pedido.

## Contexto de execução
A função `canActivate()` pega um único argumento, a instância `ExecutionContext`. O `ExecutionContext` herda de `ArgumentsHost`. Vimos `ArgumentsHost` anteriormente no capítulo de filtros de exceção. Na amostra acima, estamos apenas usando os mesmos métodos auxiliares definidos em `ArgumentsHost` que usamos anteriormente, para obter uma referência ao objeto `Request`. Você pode consultar o capítulo novamente na seção **Host de argumentos** do filtros de exceção para mais informações sobre este tópico.

Estendendo `ArgumentsHost`, `ExecutionContext` também adiciona vários novos métodos auxiliares que fornecem detalhes adicionais sobre o processo de execução atual. Esses detalhes podem ser úteis na construção de proteções mais genéricas que podem funcionar em um amplo conjunto de controladores, métodos e contextos de execução. Saiba mais sobre `ExecutionContext` [aqui](/fundamentals/execution-context.md).

## Autenticação baseada em função
Vamos criar uma proteção mais funcional que permita o acesso apenas a usuários com uma função específica. Começaremos com um modelo básico de guarda e o desenvolveremos nas próximas seções. Por enquanto, permitimos que todas as solicitações prossigam:

```ts
// roles.guard.ts

import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
```

## Guardas de ligação
Como pipes e filtros de exceção, os guardas podem ser com escopo de controlador, com escopo de método ou com escopo global. Abaixo, configuramos um guarda com escopo de controlador usando o decorador `@UseGuards()`. Esse decorador pode usar um único argumento ou uma lista separada por vírgula de argumentos. Isso permite aplicar facilmente o conjunto apropriado de guardas com uma declaração.

```ts
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

> **DICA**
> 
> O decorador `@UseGuards()` é importado do pacote `@nestjs/common`.

Acima, passamos pela classe `RolesGuard` (em vez de uma instância), deixando a responsabilidade pela instanciação à estrutura e permitindo a injeção de dependência. Como nos pipes e filtros de exceção, também podemos passar por uma instância no local:

```ts
@Controller('cats')
@UseGuards(new RolesGuard())
export class CatsController {}
```

A construção acima anexa a proteção a todos os manipuladores declarados por este controlador. Se desejamos que o guarda se aplique apenas a um único método, aplicamos o decorador `@UseGuards()` no nível do método.

Para configurar uma proteção global, use o método `useGlobalGuards()` da instância do aplicativo Nest:

```ts
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

> **AVISO**
> 
> No caso de aplicativos híbridos, o método `useGlobalGuards()` não configura guardas para gateways e microservices por padrão (consulte [Aplicação híbrida](/faq/hybrid-application.md) para informações sobre como alterar esse comportamento). Para aplicativos de microsserviços "padrão" (não híbridos), `useGlobalGuards()` para montar os guardas globalmente.

Guardas globais são usados em toda a aplicação, para todos os controladores e manipuladores de rotas. Em termos de injeção de dependência, guardas globais registrados de fora de qualquer módulo (com `useGlobalGuards()` como no exemplo acima) não pode injetar dependências, pois isso é feito fora do contexto de qualquer módulo. Para resolver esse problema, você pode configurar um guarda diretamente de qualquer módulo usando a seguinte construção:

```ts
// app.module.ts

import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

> **DICA**
> 
> Ao usar essa abordagem para executar a injeção de dependência para o guarda, observe que, independentemente do módulo onde essa construção é empregada, o guarda é de fato, global. Onde isso deve ser feito? Escolha o módulo onde o guarda (`RolesGuard` no exemplo acima) é definido. Além disso, `useClass` não é a única maneira de lidar com registro de provedor personalizado. Saiba mais [aqui](/fundamentals/custom-providers.md).

## Definindo funções por manipulador
Nosso `RolesGuard` está funcionando, mas ainda não é muito inteligente. Ainda não estamos aproveitando o recurso de guarda mais importante - o contexto de execução. Ainda não se sabe sobre os papéis ou quais papéis são permitidos para cada manipulador. O `CatsController`, por exemplo, poderia ter diferentes esquemas de permissão para diferentes rotas. Alguns podem estar disponíveis apenas para um usuário administrador e outros podem estar abertos para todos. Como podemos combinar papéis com rotas de maneira flexível e reutilizável?

É aqui que os **metadados personalizados** entra em jogo (aprenda mais [aqui](/fundamentals/execution-context.md)). Nest fornece a capacidade de anexar personalizado metadados para rotear manipuladores através do decorador `@SetMetadata()`. Esses metadados fornecem a nossa ausência de `role`, que um guarda inteligente precisa para tomar decisões. Vamos dar uma olhada no uso de `@SetMetadata()`:

```ts
// cats.controller.ts

@Post()
@SetMetadata('roles', ['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

> **DICA**
> 
> O decorador `@SetMetadata()` é importado do pacote `@nestjs/common`.

Com a construção acima, anexamos o metadados `roles` (`roles` é uma chave, enquanto `['admin']` é um valor específico) para o método `create()`. Enquanto isso funciona, não é uma boa prática usar `@SetMetadata()` diretamente em suas rotas. Em vez disso, crie seus próprios decoradores, como mostrado abaixo:

```ts
// roles.decorator.ts

import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

Essa abordagem é muito mais limpa e legível e é fortemente tipada. Agora que temos um decorador `@Roles()` customizado, podemos usá-lo para decorar o método `create()`.

```ts
// cats.controller.ts

@Post()
@Roles('admin')
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

## Juntando tudo
Vamos agora voltar e amarrar isso junto com o nosso `RolesGuard`. Atualmente, ele simplesmente retorna `true` em todos os casos, permitindo que cada solicitação prossiga. Queremos condicionar o valor de retorno com base na comparação das funções atribuídas ao usuário atual às funções reais exigidas pela rota atual que está sendo processada. Para acessar o papel da(s) rota(s) (metadados personalizados), usaremos a classe auxiliar `Reflector` auxiliar, que é fornecida de maneira _out of the box_ (fora da caixa) pela estrutura e exposta a partir do pacote `@nestjs/core`.

```ts
// roles.guard.ts

import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

> **DICA**
> 
> No mundo `node.js`, é prática comum anexar o usuário autorizado ao objeto `request`. Assim, em nosso código de amostra acima, estamos assumindo que `request.user` contém a instância do usuário e as funções permitidas. No seu aplicativo, você provavelmente fará essa associação no seu guarda customizado de autenticação (ou middleware). Verifique [este capítulo](/security/authentication.md) para mais informações sobre este tópico.

> **AVISO**
> 
> A lógica dentro da função `matchRoles()` pode ser tão simples ou sofisticada quanto necessário. O ponto principal deste exemplo é mostrar como os guardas se encaixam no ciclo de solicitação/resposta.

Consulte a seção [Reflexão e metadados](/fundamentals/execution-context.md) do capítulo sobre Contexto de execução para mais detalhes sobre a utilização do `Reflector` de uma maneira sensível ao contexto.

Quando um usuário com privilégios insuficientes solicita um ponto final, o Nest retorna automaticamente a seguinte resposta:

```json
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

Observe que nos bastidores, quando um guarda retorna `false`, a estrutura lança um `ForbiddenException`. Se você deseja retornar uma resposta de erro diferente, deve lançar sua própria exceção específica. Por exemplo:

```ts
throw new UnauthorizedException();
```

Qualquer exceção lançada por um guarda será tratada pela [camada de exceções](/overview/exception-filters.md) (e quaisquer filtros de exceções aplicados ao contexto atual).

> **DICA**
> 
> Se você está procurando um exemplo do mundo real sobre como implementar a autorização, [verifique este capítulo](/security/authorization.md).

