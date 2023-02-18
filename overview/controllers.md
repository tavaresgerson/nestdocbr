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
