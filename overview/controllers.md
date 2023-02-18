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
