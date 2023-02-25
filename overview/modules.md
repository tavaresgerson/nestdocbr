# Módulos

Um módulo é uma classe anotada com um decorador `@Module()`. O `@Module()` fornece metadados que Nest faz uso para organizar a estrutura do aplicativo.

![image](https://user-images.githubusercontent.com/22455192/221326749-575d6a5a-0dff-495c-a272-a5cf617835c5.png)

Cada aplicativo possui pelo menos um módulo, um módulo raiz. O módulo raiz é o ponto de partida que o Nest usa para criar o gráfico de aplicação - a estrutura de dados interna que o Nest usa para resolver relacionamentos e dependências de módulos e provedores. Embora aplicativos muito pequenos possam teoricamente ter apenas o módulo raiz, esse não é o caso típico. Queremos enfatizar que os módulos são fortemente recomendado como uma maneira eficaz de organizar seus componentes. Assim, para a maioria dos aplicativos, a arquitetura resultante empregará vários módulos, cada um encapsulando um conjunto de recursos.

O `@Module()` pega um único objeto cujas propriedades descrevem o módulo:

|               |         |
|---------------|---------|
| `providers`	  | os provedores que serão instanciados pelo injetor Nest e que podem ser compartilhados ao menos neste módulo |
| `controllers` | o conjunto de controladores definido neste módulo que deve ser instanciado |
| `imports`     | a lista de módulos importados que exportam os provedores necessários neste módulo |
| `exports`     | o subconjunto de `providers` que são fornecidos por este módulo e devem estar disponíveis em outros módulos que importam este módulo. Você pode usar o próprio provedor ou apenas seu token (valor `provide`) |

O módulo encapsula provedores por padrão. Isso significa que é impossível injetar provedores que não fazem parte diretamente do módulo atual nem são exportados dos módulos importados. Assim, você pode considerar os provedores exportados de um módulo como a interface pública do módulo ou API.

## Módulos de recursos
O `CatsController` e `CatsService` pertencem ao mesmo domínio de aplicativo. Como eles estão intimamente relacionados, faz sentido movê-los para um módulo de recurso. Um módulo de recurso simplesmente organiza o código relevante para um recurso específico, mantendo o código organizado e estabelecendo limites claros. Isso nos ajuda a gerenciar a complexidade e desenvolver com princípios SOLID, especialmente à medida que o tamanho do aplicativo e/ou equipe cresce.

Para demonstrar isso, criaremos o `CatsModule`.

```ts
// cats/cats.module.ts

import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

> **DICA**
> Para criar um módulo usando a CLI, basta executar o comando `$ nest g module cats`.

Acima, definimos o `CatsModule` no arquivo `cats.module.ts` e movemos tudo relacionado a este módulo para o diretório `cats`. A última coisa que precisamos fazer é importar este módulo para o módulo raiz (`AppModule`, definido no arquivo `app.module.ts`).

```ts
// app.module.ts

import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```

Aqui está como nossa estrutura de diretórios se parece agora:

