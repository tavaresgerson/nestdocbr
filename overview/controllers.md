# Controllers

Os controladores são responsáveis por lidar com as entradas de requisições e retornando respostas para o cliente.

![image](https://user-images.githubusercontent.com/22455192/219870456-7ae0f9a0-f4ff-4458-b7fe-446f0003bb29.png)

O objetivo de um controlador é receber solicitações específicas para o aplicativo. O mecanismo de controle **roteamento** que direciona quais controller irão receber as solicitações. Freqüentemente, cada controlador possui mais de uma rota e rotas diferentes podem executar ações diferentes.

Para criar um controlador básico, usamos classes e decoradores. Os **decoradores** associam as classes aos metadados necessários e permitem que o Nest crie um mapa de roteamento (para os controladores correspondentes).

> **DICA**
>
> Para criar rapidamente um controlador CRUD com a validação embutida, você pode usar as CLI's Gerador CRUD: nest g resource [name].

## Roteamento
No exemplo a seguir, usaremos o decorador `@Controller()`, que é requerido para definir um controlador básico. Especificaremos um prefixo opcional do caminho da rota de `cats`. Usando um prefixo de caminho em um decorador `@Controller()` nos permite agrupar facilmente um conjunto de rotas relacionadas e minimizar o código repetitivo. Por exemplo, podemos optar por agrupar um conjunto de rotas que gerenciam interações com uma entidade de `cat` sob a rota `/cats`. Nesse caso, poderíamos especificar o prefixo do caminho `cats` no decorador `@Controller()` para que não tenhamos que repetir essa parte do caminho para cada rota no arquivo.

```ts
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

> **DICA**
> 
> Para criar um controlador usando a CLI, basta executar o comando `$ nest g controller cats`.

O decorador `@Get()` do método de solicitação HTTP, que está escrito antes do método `findAll()` informa ao Nest para criar um manipulador para um endpoint específico para solicitações HTTP. O endpoint corresponde ao método de solicitação HTTP (GET, neste caso) e ao caminho da rota. Qual é o caminho da rota? O caminho da rota para um manipulador é determinado concatenando o prefixo (opcional) declarado para o controlador e qualquer caminho especificado no decorador do método. Desde que declaramos um prefixo para cada rota (`cats`) e não adicionou nenhuma informação de caminho no decorador, o Nest mapeará requisições `GET /cats` para este manipulador. Como mencionado, o caminho inclui o prefixo opcional do caminho do controlador e qualquer sequência de caminho declarada no decorador do método de solicitação. Por exemplo, um prefixo de caminho de `cats` combinado com o decorador `@Get('breed')` produziria um mapeamento de rota para solicitações como `GET /cats/breed`.

Em nosso exemplo acima, quando uma solicitação GET é feita para esse endpoint, a Nest encaminha a solicitação para o método `findAll()` definido pelo usuário. Observe que o nome do método que escolhemos aqui é completamente arbitrário. Obviamente, devemos declarar um método para vincular a rota, mas Nest não atribui nenhum significado ao nome do método escolhido.

Esse método retornará um código de status 200 e a resposta associada, que neste caso é apenas uma sequência. Por que isso acontece? Para explicar, primeiro apresentaremos o conceito de que a Nest emprega duas diferentes opções para manipular respostas:

<table>
  <tr>
    <td>
      <b>Padrão (recomendado)</b>
    </td>
    <td>
      Usando esse método embutido, quando um manipulador de solicitações retorna um objeto ou matriz JavaScript, ele retornará automaticamente ser serializado para JSON. Quando retorna um tipo primitivo de JavaScript (por exemplo. <code>string</code>, <code>number</code>, <code>boolean</code>), no entanto, o Nest enviará apenas o valor sem tentar serializá-lo. Isso simplifica o tratamento da resposta: basta retornar o valor e o Nest cuida do resto.
  
  Além disso, a resposta código de status é sempre 200 por padrão, exceto para solicitações POST que usam 201. Podemos mudar facilmente esse comportamento adicionando o decorador <code>@HttpCode(...)</code> no nível do manipulador (ver à seção Códigos de status).
    </td>
  </tr>
  <tr>
    <td>
      <b>Específico da biblioteca</b>
    <td>
    <td>
      Podemos usar o (específico da biblioteca, por exemplo, Express) objeto de resposta, que pode ser injetado usando o decorador <code>@Res()</code> na assinatura do manipulador de métodos (por exemplo <code>findAll(@Res() response)</code>). Com essa abordagem, você pode usar os métodos nativos de tratamento de respostas expostos por esse objeto. No caso com o Express, você pode construir respostas usando código como <code>response.status(200).send()</code>.
    </td>
  </tr>
</table>

> **AVISO**
> O Nest detecta quando o manipulador está usando `@Res()` ou `@Next()`, indicando que você escolheu a opção específica da biblioteca. Se ambas as abordagens forem usadas ao mesmo tempo, a abordagem padrão será desativada automaticamente para esta rota única e não funcionará mais como esperado. Para usar as duas abordagens ao mesmo tempo (por exemplo, injetando o objeto de resposta apenas para definir cookies/cabeçalhos, mas ainda deixando o restante na estrutura), você deve definir a opção `passthrough` para `true` no decorador `@Res({ passthrough: true })`.

## Objeto de solicitação
Os manipuladores geralmente precisam acessar os detalhes da requisição do cliente. Nest fornece acesso ao objeto de solicitação da plataforma subjacente (Express por padrão). Podemos acessar o objeto de solicitação instruindo o Nest a injetá-lo adicionando o decorador `@Req()` para a assinatura do manipulador.

```ts
// cats.controller.ts

import { Controller, Get, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(@Req() request: Request): string {
    return 'This action returns all cats';
  }
}
```

> **DICA**
> Para tirar proveito dos types do `express` (como `request: Request` no exemplo acima), instale o pacote `@types/express`.

O objeto de solicitação representa a solicitação HTTP e possui propriedades para a sequência de querystrings, parâmetros, cabeçalhos HTTP e corpo (leia mais [aqui](https://expressjs.com/en/api.html#req)). Na maioria dos casos, não é necessário pegar essas propriedades manualmente. Em vez disso, podemos usar decoradores dedicados, como `@Body()` ou `@Query()`, que estão disponíveis fora da caixa. Abaixo está uma lista dos decoradores fornecidos e dos objetos simples específicos da plataforma que eles representam.

|                           |                                   |
|---------------------------|-----------------------------------|
| `@Request()`, `@Req()`    |	`req`                             |
| `@Response()`, `@Res()*`  | `res`                             |
| `@Next()`                 | `next`                            |
| `@Session()`              |	`req.session`                     |
| `@Param(key?: string)`    | `req.params`/`req.params[key]`    |
| `@Body(key?: string)`     | `req.body`/`req.body[key]`        |
| `@Query(key?: string)`    |	`req.query`/`req.query[key]`      |
| `@Headers(name?: string)`	| `req.headers`/`req.headers[name]` |
| `@Ip()`	                  | `req.ip`                          |
| `@HostParam()`            |	`req.hosts`                       |


Para compatibilidade com tipagens nas plataformas HTTP subjacentes (por exemplo, Express e Fastify), o Nest fornece os decoradores `@Res()` e `@Response()`. `@Res()` é simplesmente um apelido para `@Response()`. Ambos expõem diretamente a interface do objeto response nativo. Ao usá-los, você também deve importar as tipagens para a biblioteca subjacente (por exemplo `@types/express`) para tirar o máximo proveito. Observe que quando você injeta `@Res()` ou `@Response()` em um manipulador de métodos, você coloca o Nest em **Modo específico da biblioteca** para esse manipulador e você se torna responsável por gerenciar a resposta. Ao fazer isso, você deve emitir algum tipo de resposta retornando para o objeto `response` (por exemplo `res.json(...)` ou `res.send(...)`) ou o servidor HTTP será suspenso.

> **DICA**
> 
> Para aprender a criar seus próprios decoradores personalizados, visite [este](/overview/custom-decorators.md) capítulo.

## Recursos
Anteriormente, definimos um ponto final para buscar o recurso de _cats_ (GET rota). Normalmente, também queremos fornecer um endpoint que crie novos registros. Para isso, vamos criar o manipulador de POST:

```ts
// cats.controller.tsJS

import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create(): string {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

É simples assim. Nest fornece decoradores para todos os métodos HTTP padrão: `@Get()`, `@Post()`, `@Put()`, `@Delete()`, `@Patch()`, `@Options()`, e `@Head()`. Além disso, `@All()` define um endpoint que lida com todos eles.

## Wildcards de Rotas
Rotas baseadas em padrões também são suportadas. Por exemplo, o asterisco é usado como um curinga e corresponde a qualquer combinação de caracteres.

```ts
@Get('ab*cd')
findAll() {
  return 'This route uses a wildcard';
}
```

O caminho de rota `'ab*cd'` corresponderá a `abcd`, `ab_cd`, `abecd`, e assim por diante. Os caracteres `?`, `+`, `*` e `()` podem ser usado em um caminho de rota e são subconjuntos de suas contrapartes de expressão regular. O hífen (`-`) e o ponto (`.`) são interpretados literalmente por caminhos baseados em string.

## Código de status
Como mencionado, a resposta de código de status é sempre 200 por padrão, exceto para solicitações POST que são 201. Podemos mudar facilmente esse comportamento adicionando o decorador `@HttpCode(...)` no nível de um manipulador.

```ts
@Post()
@HttpCode(204)
create() {
  return 'This action adds a new cat';
}
```

> **DICA**
> Importe `HttpCode` do pacote `@nestjs/common`.

Muitas vezes, seu código de status não é estático, mas depende de vários fatores. Nesse caso, você pode usar uma resposta da biblioteca específica (injetar usando o objeto `@Res()`) (ou, em caso de erro, lança uma exceção).

## Cabeçalhos
Para especificar um cabeçalho de resposta personalizado, você pode usar um decorador `@Header()` ou um objeto de resposta específico da biblioteca (invocando a chamada `res.header()` diretamente).

```ts
@Post()
@Header('Cache-Control', 'none')
create() {
  return 'This action adds a new cat';
}
```

> DICA
> Importe `Header` do pacote `@nestjs/common`.

## Redirecionamento
Para redirecionar uma resposta para um URL específico, você pode usar um decorador `@Redirect()` ou um objeto de resposta específico da biblioteca (invocando o método `res.redirect()` diretamente).

`@Redirect()` leva dois argumentos: `url` e `statusCode`, ambos são opcionais. O valor padrão de statusCode é `302 (Found)` se omitido.

```ts
@Get()
@Redirect('https://nestjs.com', 301)
```

Às vezes, convém determinar o código de status HTTP ou o URL de redirecionamento dinamicamente. Faça isso retornando um objeto do método de manipulador de rota com a forma:

```
{
  "url": string,
  "statusCode": number
}
```

Os valores retornados substituirão todos os argumentos passados para o decorador `@Redirect()`. Por exemplo:

```ts
@Get('docs')
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version) {
  if (version && version === '5') {
    return { url: 'https://docs.nestjs.com/v5/' };
  }
}
```

## Parâmetros de rota
Rotas com caminhos estáticos não funcionarão quando você precisar aceitar dados dinâmicos como parte da solicitação (por exemplo `GET /cats/1` para obter cat com id `1`). Para definir rotas com parâmetros, podemos adicionar parâmetros de rota como **tokens** no caminho da rota para capturar o valor dinâmico nessa posição no URL da solicitação. O parâmetro de _token_ no decorador `@Get()` será exemplificado abaixo. Os parâmetros de rota declarados dessa maneira podem ser acessados usando o decorador `@Param()`, que deve ser adicionado à assinatura do método.

> **DICA**
> Rotas com parâmetros devem ser declaradas após quaisquer caminhos estáticos. Isso impede que os caminhos parametrizados interceptem o tráfego destinado aos caminhos estáticos.

```ts
@Get(':id')
findOne(@Param() params): string {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
```

`@Param()` é usado para decorar um parâmetro de método (`params` no exemplo acima), e faz os parâmetros da rota disponíveis como propriedades desse método decorado dentro do corpo do método. Como visto no código acima, podemos acessar o parâmetro `id` referenciando `params.id`. Você também pode passar um parâmetro específico para o decorador e, em seguida, fazer referência ao parâmetro route diretamente pelo nome no corpo do método.

> **DICA**
> Importe `Param` do pacote `@nestjs/common`.

```ts
@Get(':id')
findOne(@Param('id') id: string): string {
  return `This action returns a #${id} cat`;
}
```

## Roteamento de sub-domínio
O decorador `@Controller` pode levar uma opção `host` para exigir que o host HTTP das solicitações recebidas corresponda a algum valor específico.

```ts
@Controller({ host: 'admin.example.com' })
export class AdminController {
  @Get()
  index(): string {
    return 'Admin page';
  }
}
```

> **AVISO**
> Fastify não possui suporte para roteadores aninhados; ao usar o roteamento de subdomínio, o adaptador do Express padrão deve ser usado.

Semelhante a um caminho de rota, A opção `hosts` pode usar tokens para capturar o valor dinâmico nessa posição no nome do host. O parâmetro com token no `@Controller()` será exemplificado abaixo. Os parâmetros do host declarados dessa maneira podem ser acessados usando o decorador `@HostParam()`, que deve ser adicionado à assinatura do método.

```ts
@Controller({ host: ':account.example.com' })
export class AccountController {
  @Get()
  getInfo(@HostParam('account') account: string) {
    return account;
  }
}
```

## Escopos
Para pessoas provenientes de diferentes contextos de linguagem de programação, pode ser inesperado saber que, no Nest, quase tudo é compartilhado entre as solicitações recebidas. Temos um pool de conexões com o banco de dados, serviços singleton com estado global etc. Lembre-se de que o Node.js não segue o modelo sem estado multiencadeado de solicitação/resposta no qual cada solicitação é processada por um encadeamento separado. Portanto, o uso de instâncias singleton é totalmente seguro para nossas aplicações.

No entanto, existem casos de borda quando a vida útil do controlador baseada em solicitação pode ser o comportamento desejado, por exemplo, cache por solicitação em aplicativos GraphQL, rastreamento de solicitação ou multitenização. Aprenda a controlar escopos [aqui](/fundamentals/injection-scopes.md).

## Asincronicidade
Adoramos o JavaScript moderno e sabemos que a extração de dados é principalmente assíncrono. É por isso que o Nest suporta e funciona bem com funções `async`.

> **DICA**
> Saiba mais sobre async/await [aqui](https://kamilmysliwiec.com/typescript-2-1-introduction-async-await)

Toda função async precisa retornar um `Promise`. Isso significa que você pode retornar um valor diferido que a Nest poderá resolver sozinha. Vamos ver um exemplo disso:

```ts
// cats.controller.ts

@Get()
async findAll(): Promise<any[]> {
  return [];
}
```

O código acima é totalmente válido. Além disso, os manipuladores de rota Nest são ainda mais poderosos ao poder retornar o RxJS [streams observáveis](http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html). O Nest se inscreve automaticamente na fonte abaixo e assume o último valor emitido (depois que o stream for concluído).

```ts
// cats.controller.ts

@Get()
findAll(): Observable<any[]> {
  return of([]);
}
```

Ambas as abordagens acima funcionam e você pode usar o que for adequado às suas necessidades.

