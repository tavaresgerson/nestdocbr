# Provedores Customizados

Nos capítulos anteriores, abordamos vários aspectos de Injeção de dependência (DI) e como é usado no Nest. Um exemplo disso é o [baseado em construtor](/overview/providers.md) injeção de dependência usada para injetar instâncias (frequentemente prestadores de serviços) em classes. Você não ficará surpreso ao saber que a Injeção de Dependência é incorporada ao núcleo do Nest de maneira fundamental. Até agora, exploramos apenas um padrão principal. À medida que sua aplicação se torna mais complexa, pode ser necessário aproveitar todos os recursos do sistema de DI, portanto, vamos explorá-los com mais detalhes.

## Fundamentos de DI
Injeção de dependência é uma inversão de controle (IoC), técnica em que você delega a instanciação de dependências no contêiner de IoC (no nosso caso, o sistema de tempo de execução NestJS), em vez de fazê-lo em seu próprio código imperativamente. Vamos examinar o que está acontecendo neste exemplo a partir do [capítulo de providers](/overview/providers.md).

Primeiro, definimos um provedor. O decorador `@Injectable()` marca a classe `CatsService` como provedor.

```ts
// cats.service.ts

import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  findAll(): Cat[] {
    return this.cats;
  }
}
```

Em seguida, solicitamos que o Nest injete o provedor em nossa classe de controlador:

```ts
// cats.controller.ts

import { Controller, Get } from '@nestjs/common';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

Por fim, registramos o provedor no contêiner Nest IoC:

```ts
// app.module.ts

import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})

export class AppModule {}
```

O que exatamente está acontecendo sob o capô para fazer isso funcionar? Existem três etapas principais no processo:

1. No `cats.service.ts`, o decorador `@Injectable()` declara a classe `CatsService` como uma classe que pode ser gerenciada pelo contêiner Nest IoC.
2. No `cats.controller.ts`, `CatsController` declara uma dependência do token `CatsService` com injeção de construtor:

```ts
constructor(private catsService: CatsService)
```

3. No `app.module.ts`, associamos o token `CatsService` com a classe `CatsService` do arquivo `cats.service.ts`. Veja abaixo exatamente como essa associação (também _registrado_) ocorre.

Quando o contêiner Nest IoC instancia um `CatsController`, primeiro procura quaisquer dependências*. Quando encontra a dependência `CatsService`, ele realiza uma pesquisa no token `CatsService`, que retorna a classe `CatsService`, de acordo com a etapa de registro (Item 3 acima). Assumindo `SINGLETON` escopo (o comportamento padrão), o Nest criará uma instância de `CatsService`, armazena em cache e devolve ou, se já estiver em cache, retorne a instância existente.

*Esta explicação é um pouco simplificada para ilustrar. Uma área importante que encobrimos é que o processo de análise do código para dependências é muito sofisticado e ocorre durante o inicialização do aplicativo. Uma característica principal é que a análise de dependência (ou "criando o gráfico de dependência") é transitivo. No exemplo acima, o `CatsService` tinha dependências, essas também seriam resolvidas. O gráfico de dependência garante que as dependências sejam resolvidas na ordem correta - essencialmente "de baixo para cima". Esse mecanismo evita que o desenvolvedor precise gerenciar gráficos de dependência complexos.*

## Provedores padrão
Vamos dar uma olhada mais de perto no decorador `@Module()`. No `app.module`, declaramos:

```ts
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
```

A propriedade `providers` leva uma matriz de providers. Até agora, fornecemos esses provedores por meio de uma lista de nomes de classes. De fato, a sintaxe `providers: [CatsService]` é uma mão curta para a sintaxe mais completa:

```ts
providers: [
  {
    provide: CatsService,
    useClass: CatsService,
  },
];
```

Agora que vemos essa construção explícita, podemos entender o processo de registro. Aqui, estamos claramente associando o token CatsService com a classe CatsService. A notação de mão curta é apenas uma conveniência para simplificar o caso de uso mais comum, onde o token é usado para solicitar uma instância de uma classe com o mesmo nome.

## Provedores personalizados
O que acontece quando seus requisitos vão além daqueles oferecidos por _Provedores padrão_? Aqui estão alguns exemplos:

* Você deseja criar uma instância personalizada em vez de instanciar o Nest (ou retornar uma instância em cache de) uma classe
* Você deseja reutilizar uma classe existente em uma segunda dependência
* Você deseja substituir uma classe por uma versão simulada para testar

O Nest permite definir provedores personalizados para lidar com esses casos. Ele fornece várias maneiras de definir provedores personalizados. Vamos passar por eles.

> **DICA**
> 
> Se você estiver tendo problemas com a resolução de dependência, poderá definir a variável de ambiente `NEST_DEBUG` para `true` e obtenha logs de resolução de dependência extra durante a inicialização.

## Provedores de valor: `useValue`
A sintaxe de `useValue` é útil para injetar um valor constante, colocar uma biblioteca externa no contêiner Nest ou substituir uma implementação real por um objeto simulado. Digamos que você gostaria de forçar o Nest a usar um mock `CatsService` para fins de teste.

```ts
import { CatsService } from './cats.service';

const mockCatsService = {
  /* implementação do mock
  ...
  */
};

@Module({
  imports: [CatsModule],
  providers: [
    {
      provide: CatsService,
      useValue: mockCatsService,
    },
  ],
})
export class AppModule {}
```

Neste exemplo, o `CatsService` vai resolver o `mockCatsService` para o objeto simulado. `useValue` requer um valor - nesse caso, um objeto literal que tenha a mesma interface que a classe `CatsService` que está substituindo. Por causa da tipagem estrutural do TypeScript, você pode usar qualquer objeto que possua uma interface compatível, incluindo um objeto literal ou uma instância de classe instanciada com `new`.

## Tokens de provedores não baseados em classe
Até agora, usamos nomes de classe como nosso provedor de tokens (o valor da propriedade `provide` em um provedor listado na matriz `providers`). Isso é correspondido pelo padrão usado com [injeção baseada em construtor](/overview/providers.md), onde o token também é um nome de classe. (Consulte novamente [fundamentos DI](/fundamentals/custom-providers.md) para uma atualização sobre tokens se esse conceito não estiver totalmente claro). Às vezes, podemos querer flexibilidade para usar strings ou símbolos como o token DI. Por exemplo:

```ts
import { connection } from './connection';

@Module({
  providers: [
    {
      provide: 'CONNECTION',
      useValue: connection,
    },
  ],
})
export class AppModule {}
```

Neste exemplo, estamos associando um token com valor de sequência (`'CONNECTION'`) com um objeto `connection` pré-existente que importamos de um arquivo externo.

> **AVISO**
> 
> Além de usar strings como valores de token, você também pode usar o [Symbols JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) ou [enums TypeScript](https://www.typescriptlang.org/docs/handbook/enums.html).

Já vimos como injetar um provedor usando o padrão injeção baseada em construtor padrão. Esse padrão requer que a dependência seja declarada com um nome de classe. O provedor personalizado `'CONNECTION'` usa um token com valor de string. Vamos ver como injetar esse provedor. Para fazer isso, usamos o decorador `@Inject()`. Este decorador leva um único argumento - o token.

```ts
@Injectable()
export class CatsRepository {
  constructor(@Inject('CONNECTION') connection: Connection) {}
}
```

> **DICA**
> 
> O decorador `@Inject()` é importado do pacote `@nestjs/common`.

Enquanto usamos diretamente a string `'CONNECTION'` nos exemplos acima, para fins ilustrativos, para organização de código limpo, é melhor definir tokens em um arquivo separado, como `constants.ts`. Trate-os da maneira que desejar, símbolos ou enums definidos em seu próprio arquivo e importados quando necessário.

## Provedores de classe: `useClass`
A sintaxe `useClass` permite determinar dinamicamente uma classe que um token deve resolver. Por exemplo, suponha que tenhamos uma classe abstract (ou padrão) `ConfigService`. Dependendo do ambiente atual, queremos que o Nest forneça uma implementação diferente do serviço de configuração. O código a seguir implementa essa estratégia.

```ts
const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};

@Module({
  providers: [configServiceProvider],
})
export class AppModule {}
```

Vejamos alguns detalhes nesta amostra de código. Você notará que definimos `configServiceProvider` primeiro com um objeto literal e depois passamos no decorador do módulo providers a propriedade. Isso é apenas um pouco de organização do código, mas é funcionalmente equivalente aos exemplos que usamos até agora neste capítulo.

Além disso, usamos o nome da classe `ConfigService` como nosso símbolo. Para qualquer classe que dependa de `ConfigService`, Nest injetará uma instância da classe fornecida (`DevelopmentConfigService` ou `ProductionConfigService`) substituindo qualquer implementação padrão que possa ter sido declarada em outro lugar (por exemplo, um `ConfigService` declarado com um decorador `@Injectable()`).

## Provedores de fábrica: `useFactory`
A sintaxe `useFactory` permite criar provedores dinamicamente. O provedor real será fornecido pelo valor retornado de uma função de fábrica. A função de fábrica pode ser tão simples ou complexa quanto necessário. Uma fábrica simples pode não depender de outros fornecedores. Uma fábrica mais complexa pode injetar outros fornecedores necessários para calcular seu resultado. Neste último caso, a sintaxe do provedor de fábrica possui um par de mecanismos relacionados:

1. A função de fábrica pode aceitar argumentos (opcionais).
2. A propriedade `inject` (opcional) aceita uma matriz de provedores que o Nest resolverá e passará como argumentos para a função de fábrica durante o processo de instanciação. Além disso, esses provedores podem ser marcados como opcionais. As duas listas devem estar correlacionadas: Nest passará instâncias do `inject` listando argumentos para a função de fábrica na mesma ordem. O exemplo abaixo demonstra isso.

```ts
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider, optionalProvider?: string) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \_____________/            \__________________/
  //        Este provider              O provider com este
  //        é obrigatório.            token pode ser `undefined`.
};

@Module({
  providers: [
    connectionProvider,
    OptionsProvider,
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
```

## Fornecedores de alias: `useExisting`
A sintaxe de `useExisting` permite criar aliases para provedores existentes. Isso cria duas maneiras de acessar o mesmo provedor. No exemplo abaixo, o token (baseado em string) `'AliasedLoggerService'` é um alias para o token (baseado em classe) `LoggerService`. Suponha que temos duas dependências diferentes, uma para `'AliasedLoggerService'` e um para `LoggerService`. Se ambas as dependências forem especificadas com escopo `SINGLETON`, ambos resolverão para a mesma instância.

```ts
@Injectable()
class LoggerService {
  /* detalhes da implementação */
}

const loggerAliasProvider = {
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
};

@Module({
  providers: [LoggerService, loggerAliasProvider],
})
export class AppModule {}
```

## Provedores não baseados em serviços
Embora os provedores geralmente forneçam serviços, eles não se limitam a esse uso. Um provedor pode fornecer qualquer valor. Por exemplo, um provedor pode fornecer uma matriz de objetos de configuração com base no ambiente atual, como mostrado abaixo:

```ts
const configFactory = {
  provide: 'CONFIG',
  useFactory: () => {
    return process.env.NODE_ENV === 'development' ? devConfig : prodConfig;
  },
};

@Module({
  providers: [configFactory],
})
export class AppModule {}
```

## Exportar provedor personalizado
Como qualquer provedor, um provedor personalizado é escalonado para o módulo de declaração. Para torná-lo visível para outros módulos, ele deve ser exportado. Para exportar um provedor personalizado, podemos usar seu token ou o objeto provedor completo.

O exemplo a seguir mostra a exportação usando o token:

```ts
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
``` 

Como alternativa, exporte com o objeto completo do provedor:

```ts
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
```
