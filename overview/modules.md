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

![Screenshot from 2023-02-24 22-05-09](https://user-images.githubusercontent.com/22455192/221327493-1e73f0cf-5b2f-44e6-916d-41e65dd6c134.png)

## Módulos compartilhados
No Nest, os módulos são **singletons** por padrão, e assim você pode compartilhar a mesma instância de qualquer provedor entre vários módulos sem esforço.

![image](https://user-images.githubusercontent.com/22455192/221327540-a20c14ae-b70c-43e6-93d1-68aa5f21551f.png)

Todo módulo é automaticamente um módulo compartilhado. Uma vez criado, ele pode ser reutilizado por qualquer módulo. Vamos imaginar que queremos compartilhar uma instância do `CatsService` entre vários outros módulos. Para fazer isso, primeiro precisamos exportação o provedor `CatsService` adicionando-o ao módulo `exports` matriz, como mostrado abaixo:

```ts
// cats.module.ts

import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
```

Agora, qualquer módulo que importe o `CatsModule` tem acesso ao `CatsService` e compartilhará a mesma instância com todos os outros módulos que a importam também.

## Reexportação de módulos
Como visto acima, os Módulos podem exportar seus provedores internos. Além disso, eles podem reexportar módulos que importam. No exemplo abaixo, o `CommonModule` ambos importados e exportado do `CoreModule`, disponibilizando-o para outros módulos que importam este.

```
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```

## Injeção de dependência
Uma classe de módulo pode **injetar** provedores também (por exemplo, para fins de configuração):

```ts
// cats.module.ts

import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private catsService: CatsService) {}
}
```

No entanto, as próprias classes de módulos não podem ser injetadas como provedores devido a [dependência circular](/fundamentals/circular-dependency.md).

## Módulos globais
Se você precisar importar o mesmo conjunto de módulos em qualquer lugar, ele poderá ficar entediante. Ao contrário do Nest, Angularproviders estão registrados no escopo global. Uma vez definidos, eles estão disponíveis em todos os lugares. Nest, no entanto, encapsula provedores dentro do escopo do módulo. Você não pode usar os provedores de um módulo em outro lugar sem primeiro importar o módulo de encapsulamento.

Quando você deseja fornecer um conjunto de provedores que devem estar disponíveis em qualquer lugar pronto para uso (por exemplo, auxiliares, conexões de banco de dados etc.), faça o módulo global com o decorador `@Global()`.

```ts
import { Module, Global } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

O decorador `@Global()` faz o módulo com escopo global. Módulos globais devem ser registrados apenas uma vez, geralmente pelo módulo raiz ou núcleo. No exemplo acima, o provedor `CatsService` será onipresente e os módulos que desejam injetar o serviço não precisarão importar o `CatsModule` em sua matriz de importações.

> **DICA**
> Tornar tudo global não é uma boa decisão de design. Módulos globais estão disponíveis para reduzir a quantidade de código boilerplate necessário. O array `imports` é geralmente a maneira preferida de disponibilizar a API do módulo aos consumidores.

## Módulos dinâmicos
O sistema de módulos Nest inclui um recurso poderoso chamado módulos dinâmicos. Esse recurso permite criar facilmente módulos personalizáveis que podem registrar e configurar provedores dinamicamente. Os módulos dinâmicos são amplamente abordados aqui. Neste capítulo, forneceremos uma breve visão geral para concluir a introdução aos módulos.

A seguir, é apresentado um exemplo de uma definição dinâmica de módulo para um `DatabaseModule`:

```ts
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

> **DICA**
> O método `forRoot()` pode retornar um módulo dinâmico de forma síncrona ou assíncrona (ou seja, através de um `Promise`).

Este módulo define o provedor `Connection` por padrão (no metadados do decorador `@Module()`), mas adicionalmente - dependendo do entities e options objetos passados para o método `forRoot()` - expõe uma coleção de provedores, por exemplo, repositórios. Observe que as propriedades retornadas pelo módulo dinâmico estendem (em vez de substituir) os metadados do módulo base definidos no decorador `@Module()`. Foi assim que ambos declararam estaticamente o provedor `Connection` e os provedores de repositório gerados dinamicamente são exportados do módulo.

Se você deseja registrar um módulo dinâmico no escopo global, defina a propriedade `global` para `true`.

```json
{
  global: true,
  module: DatabaseModule,
  providers: providers,
  exports: providers,
}
```

> **AVISO**
> Como mencionado acima, tornar tudo global **não é uma boa decisão de design**.

O `DatabaseModule` pode ser importado e configurado da seguinte maneira:

```ts
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
})
export class AppModule {}
```

Se você deseja reexportar um módulo dinâmico, pode omitir o método `forRoot()` de chamada na matriz de exportações:

```ts
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
  exports: [DatabaseModule],
})
export class AppModule {}
```

O capítulo dos [Módulos dinâmicos](/fundamentals/dynamic-modules.md) aborda esse tópico com mais detalhes e inclui um exemplo de trabalho.
