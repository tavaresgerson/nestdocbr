# Módulos Dinâmicos

O Capítulo de [módulos](/overview/modules.md) abrange o básico dos módulos Nest e inclui uma breve introdução a módulos dinâmicos. Este capítulo se expande sobre o assunto de módulos dinâmicos. Após a conclusão, você deve ter uma boa compreensão do que são, como e quando usá-los.

## Introdução
A maioria dos exemplos de código de aplicativo na seção **Visão geral** da documentação utiliza módulos regulares ou estáticos. Módulos definem grupos de componentes como provedores e controladores que se encaixam como parte modular de uma aplicação geral. Eles fornecem um contexto de execução, ou escopo, para esses componentes. Por exemplo, os provedores definidos em um módulo são visíveis para outros membros do módulo sem a necessidade de exportá-los. Quando um provedor precisa estar visível fora de um módulo, ele é exportado primeiro do módulo host e depois importado para o módulo de consumo.

Vamos passar por um exemplo familiar.

Primeiro, definiremos um `UsersModule` para provedor e exportar um `UsersService`. `UsersModule` é o módulo anfitrião para `UsersService`.

```ts
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

Em seguida, definiremos um `AuthModule`, que importa `UsersModule`, fazendo os provedores de `UsersModule` exportados disponíveis no interior de `AuthModule`:

```ts
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

Essas construções nos permitem injetar `UsersService` em, por exemplo o `AuthService` que está hospedado em `AuthModule`:

```ts
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}
  /*
    Implementação que utiliza this.usersService
  */
}
```

Vamos nos referir a isso como ligação estática ao módulo. Todas as informações que o Nest precisa para conectar os módulos já foram declaradas no host e nos módulos de consumo. Vamos explicar o que está acontecendo durante esse processo. Nest faz `UsersService` disponível dentro de `AuthModule` por:

1. Instanciando `UsersModule`, incluindo a importação transitória de outros módulos que `UsersModule` consome e resolve transitivamente quaisquer dependências (veja Provedores personalizados).
2. Instanciando `AuthModule`, e fazendo os provedores de `UsersModule` exportados disponíveis para componentes em `AuthModule` (como se tivessem sido declarados em `AuthModule`).
3. Injetando uma instância de `UsersService` em `AuthService`.

## Caso de uso com módulo dinâmico
Com a ligação estática do módulo, não há oportunidade para o módulo consumidor influenciar como os fornecedores do módulo host estão configurados. Por que isso importa? Considere um caso em que temos um módulo de uso geral que precisa se comportar de maneira diferente em diferentes casos de uso. Isso é análogo ao conceito de "plug-in" em muitos sistemas, onde uma instalação genérica requer alguma configuração antes de poder ser usada por um consumidor.

Um bom exemplo com Nest é um módulo de configuração. Muitos aplicativos acham útil externalizar os detalhes da configuração usando um módulo de configuração. Isso facilita a alteração dinâmica das configurações do aplicativo em diferentes implantações: por exemplo, um banco de dados de desenvolvimento para programadores, um banco de dados de preparação para o ambiente de stage/teste, etc. Ao delegar o gerenciamento dos parâmetros de configuração a um módulo de configuração, o código fonte do aplicativo permanece independente dos parâmetros de configuração.

O desafio é que o próprio módulo de configuração, por ser genérico (semelhante a um "plugin"), precisa ser personalizado pelo módulo de consumo. É aqui que os módulos dinâmicos entra em jogo. Usando recursos dinâmicos do módulo, podemos criar nosso módulo de configuração dinâmico para que o módulo consumidor possa usar uma API para controlar como o módulo de configuração é personalizado no momento em que é importado.

Em outras palavras, os módulos dinâmicos fornecem uma API para importar um módulo para outro e personalizar as propriedades e o comportamento desse módulo quando ele é importado, em vez de usar as ligações estáticas que vimos até agora.

## Exemplo de configuração do módulo
Usaremos a versão básica do código de exemplo do [capítulo de configuração](/techniques/configuration.md) para esta seção. A versão concluída no final deste capítulo está disponível como uma amostra de exemplo [aqui](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules).

Nosso requisito é fazer `ConfigModule` aceitar um objeto `options` para personalizá-lo. Aqui está o recurso que queremos oferecer suporte: a localização do arquivo `.env` para estar na pasta raiz do projeto. Vamos supor que queremos tornar isso configurável, para que você possa gerenciar seu arquivo `.env` em qualquer pasta de sua escolha. Por exemplo, imagine que você deseja armazenar seus vários arquivos `.env` em uma pasta sob a raiz do projeto chamada config (ou seja, uma pasta de semelhante para `src`). Você gostaria de poder escolher pastas diferentes ao usar o `ConfigModule` em diferentes projetos.

Os módulos dinâmicos nos dão a capacidade de passar parâmetros para o módulo que está sendo importado, para que possamos mudar seu comportamento. Vamos ver como isso funciona. É útil se começarmos com o objetivo final de como isso pode parecer da perspectiva do módulo consumidor e depois trabalharmos para trás. Primeiro, vamos revisar rapidamente o exemplo de estaticamente importando o `ConfigModule` (ou seja, uma abordagem que não tem capacidade de influenciar o comportamento do módulo importado). Preste muita atenção aos imports matriz no decorador `@Module()`:

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Vamos considerar o que é importar _módulo dinâmico_, onde estamos passando um objeto de configuração, e o que pode parecer. Compare a diferença no imports matriz entre esses dois exemplos:

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Vamos ver o que está acontecendo no exemplo dinâmico acima. Quais são as partes móveis?

1. `ConfigModule` é uma classe normal, então podemos inferir que ela deve ter método estático chamado `register()`. Sabemos que é estático porque estamos chamando na classe `ConfigModule`, não em uma instância da classe. Nota: esse método, que criaremos em breve, pode ter qualquer nome arbitrário, mas, por convenção, devemos chamá-lo `forRoot()` ou `register()`.
2. O método `register()` é definido por nós, para que possamos aceitar quaisquer argumentos de entrada que desejarmos. Nesse caso, vamos aceitar um simples objeto `options` com propriedades adequadas, que é o caso típico.
3. Podemos inferir que o método `register()` deve retornar algo como um `module` como seu valor de retorno aparece no `imports`. A lista que vimos até agora inclui uma lista de módulos.

De fato, o que é nosso método `register()` retornará é um `DynamicModule`. Um módulo dinâmico nada mais é do que um módulo criado em tempo de execução, com as mesmas propriedades exatas que um módulo estático, além de uma propriedade adicional chamada `module`. Vamos revisar rapidamente uma amostra de declaração estática do módulo, prestando muita atenção às opções do módulo transmitidas ao decorador:

```ts
@Module({
  imports: [DogsModule],
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
``` 

Os módulos dinâmicos devem retornar um objeto com exatamente a mesma interface, além de uma propriedade adicional chamada `module`. A propriedade `module` serve como o nome do módulo e deve ser igual ao nome da classe do módulo, conforme mostrado no exemplo abaixo.

> **DICA**
> 
> Para um módulo dinâmico, todas as propriedades do objeto de opções do módulo são opcionais exceto `module`.

E o método estático `register()`? Agora podemos ver que seu trabalho é devolver um objeto que possui a interface `DynamicModule`. Quando o chamamos, estamos efetivamente fornecendo um módulo para a lista `imports`, semelhante à maneira como faríamos isso no caso estático, listando um nome de classe do módulo. Em outras palavras, a API do módulo dinâmico simplesmente retorna um módulo, mas em vez de corrigir as propriedades no decorador `@Module`, nós os especificamos programaticamente.

Ainda existem alguns detalhes a serem abordados para ajudar a completar a imagem:

1. Agora podemos afirmar que o decorador `@Module` a propriedade `imports` pode levar não apenas um nome de classe do módulo (por exemplo: `imports: [UsersModule]`), mas também uma função retornando um módulo dinâmico (por exemplo: `imports: [ConfigModule.register(...)]`).

2. Um módulo dinâmico pode importar outros módulos. Não o faremos neste exemplo, mas se o módulo dinâmico depender de provedores de outros módulos, você os importará usando a propriedade opcional `imports`. Novamente, isso é exatamente análogo à maneira como você declararia metadados para um módulo estático usando o decorador `@Module()`.

Armado com esse entendimento, agora podemos ver como a nossa declaração dinâmica `ConfigModule` deve se parecer. Vamos dar uma olhada nisso.

```ts
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(): DynamicModule {
    return {
      module: ConfigModule,
      providers: [ConfigService],
      exports: [ConfigService],
    };
  }
}
```

Agora deve ficar claro como as peças se unem. Chamando `ConfigModule.register(...)` retorna um objeto `DynamicModule` com propriedades que são essencialmente as mesmas que até agora, fornecemos como metadados através do decorador `@Module()`.

> **DICA**
> 
> Importe `DynamicModule` de `@nestjs/common`.

## Configuração do módulo
A solução óbvia para personalizar o comportamento do `ConfigModule` é passar um objeto `options` no método estático `register()`, como adivinhamos acima. Vejamos mais uma vez o nosso módulo de consumo da propriedade `imports`:

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Isso lida bem com a passagem de um objeto `options` para o nosso módulo dinâmico. Como usamos um objeto `options` no `ConfigModule`? Vamos considerar isso por um minuto. Sabemos que nossos `ConfigModule` é basicamente um host para fornecer e exportar um serviço injetável - o `ConfigService` - para uso por outros provedores. Na verdade, é nosso `ConfigService` que precisa ler o objetos `options` para personalizar seu comportamento. Vamos supor por um momento que sabemos como obter de alguma forma o options do método `register()` para o `ConfigService`. Com essa suposição, podemos fazer algumas alterações no serviço para personalizar seu comportamento com base nas propriedades do objeto `options`. (**Nota**: por enquanto, desde que não tenha realmente determinado como transmiti-lo, vamos apenas codificar `options`. Vamos consertar isso em um minuto).

```ts
import { Injectable } from '@nestjs/common';
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor() {
    const options = { folder: './config' };

    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

Agora nossa `ConfigService` sabe como encontrar o arquivo `.env` na pasta em que especificamos `options`.

Nossa tarefa restante é injetar de alguma forma o objeto `options` do `register()` em nosso `ConfigService`. E, claro, vamos usar injeção de dependência para fazer isso. Este é um ponto-chave, portanto, certifique-se de entendê-lo. Nosso `ConfigModule` está fornecendo `ConfigService`. `ConfigService` por sua vez depende do objeto `options` fornecido apenas em tempo de execução. Então, em tempo de execução, precisaremos primeiro vincular o `options` opor-se ao contêiner Nest IoC e depois pedir ao Nest que o injete no nosso `ConfigService`. Lembre-se do capítulo [**Provedores personalizados**](/fundamentals/custom-providers.md) que os provedores podem incluir qualquer valor não apenas serviços, por isso estamos bem usando injeção de dependência para lidar com um simples objeto `options`.

Vamos abordar a ligação do objeto de opções ao contêiner de IoC primeiro. Fazemos isso em nosso método estático `register()`. Lembre-se de que estamos construindo dinamicamente um módulo, e uma das propriedades de um módulo é sua lista de fornecedores. Portanto, o que precisamos fazer é definir nosso objeto de opções como um provedor. Isso tornará injetável no `ConfigService`, do qual aproveitaremos no próximo passo. No código abaixo, preste atenção a matriz `providers`:

```ts
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(options: Record<string, any>): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options,
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }
}
```

Agora podemos concluir o processo injetando o provedor `'CONFIG_OPTIONS'` no `ConfigService`. Lembre-se de que, quando definimos um provedor usando um token não classificado, precisamos usar o decorador `@Inject()` conforme descrito [aqui](/fundamentals/custom-providers.md).

```ts
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';
import { Injectable, Inject } from '@nestjs/common';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor(@Inject('CONFIG_OPTIONS') private options: Record<string, any>) {
    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

Uma nota final: por simplicidade, usamos um token de injeção baseado em strings (`'CONFIG_OPTIONS'`) acima, mas a melhor prática é defini-lo como uma constante (ou Symbol) em um arquivo separado e importe esse arquivo. Por exemplo:

```ts
export const CONFIG_OPTIONS = 'CONFIG_OPTIONS';
```

## Exemplo
Um exemplo completo do código neste capítulo pode ser encontrado [aqui](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules).

## Orientações comunitárias
Você pode ter visto o uso de métodos como `forRoot`, `register`, e `forFeature` em torno de alguns dos pacotes `@nestjs/` e pode estar se perguntando qual é a diferença para todos esses métodos. Não há regra difícil sobre isso, mas os pacotes `@nestjs/` tentam seguir estas diretrizes:

Ao criar um módulo com:

* `register`, você espera configurar um módulo dinâmico com uma configuração específica para uso somente pelo módulo de chamada. Por exemplo, com o pacote Nest `@nestjs/axios`: `HttpModule.register({ baseUrl: 'someUrl' })`. Se, em outro módulo, você usar `HttpModule.register({ baseUrl: 'somewhere else' })`, terá a configuração diferente. Você pode fazer isso para quantos módulos quiser.
* `forRoot`, você espera configurar um módulo dinâmico uma vez e reutilizar essa configuração em vários locais (embora possivelmente sem saber, pois ela se abstraiu). É por isso que você tem um `GraphQLModule.forRoot()`, um `TypeOrmModule.forRoot()`, etc.
* `forFeature`, você espera usar a configuração de um módulo dinâmico `forRoot` mas precisa modificar alguma configuração específica para as necessidades do módulo de chamada (ou seja, a qual repositório esse módulo deve ter acesso ou o contexto que um registrador deve usar).

Todos estes, geralmente, têm suas contrapartes `async` também: `registerAsync`, `forRootAsync`, e `forFeatureAsync`, isso significa a mesma coisa, mas use a Injeção de Dependência do Nest para a configuração também.

## Construtor de módulos configurável
Criando manualmente módulos dinâmicos altamente configuráveis que expõem métodos `async` (`registerAsync`, `forRootAsync`, etc.) é bastante complicado, especialmente para os recém-chegados, Nest expõe a classe `ConfigurableModuleBuilder` que facilita esse processo e permite construir um módulo "planejado" em apenas algumas linhas de código.

Por exemplo, vamos dar o exemplo que usamos acima (`ConfigModule`) e converta-o para usar o `ConfigurableModuleBuilder`. Antes de começarmos, vamos criar uma interface dedicada que represente quais opções nossos `ConfigModule` aceita.

```ts
export interface ConfigModuleOptions {
  folder: string;
}
```

Com isso, crie um novo arquivo dedicado (além do arquivo `config.module.ts` existente) e nomeie-o para `config.module-definition.ts`. Neste arquivo, vamos utilizar o `ConfigurableModuleBuilder` para construir a definição de `ConfigModule`.

```ts
// config.module-definition.ts

import { ConfigurableModuleBuilder } from '@nestjs/common';
import { ConfigModuleOptions } from './interfaces/config-module-options.interface';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
```

Agora vamos abrir o arquivo `config.module.ts` e modificar sua implementação para alavancar o `ConfigurableModuleClass` gerado automaticamente:

```ts
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { ConfigurableModuleClass } from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {}
```

Estendendo o `ConfigurableModuleClass` significa que `ConfigModule` fornece agora não apenas o método `register` (como anteriormente com a implementação personalizada), mas também o método `registerAsync` que permite que os consumidores configurem assincronamente esse módulo, por exemplo, fornecendo fábricas de assíncronas:

```ts
@Module({
  imports: [
    ConfigModule.register({ folder: './config' }),
    // or alternatively:
    // ConfigModule.registerAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...any extra dependencies...]
    // }),
  ],
})
export class AppModule {}
```

Por fim, vamos atualizar a classe `ConfigService` para injetar o provedor de opções de módulo gerado em vez de `'CONFIG_OPTIONS'P  que usamos até agora.

```ts
@Injectable()
export class ConfigService {
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions) { ... }
}
```

## Chave de método personalizado
`ConfigurableModuleClass` por padrão fornece o `register` e sua contraparte `registerAsync`. Para usar um nome de método diferente, use o método `ConfigurableModuleBuilder#setClassMethodName`, como se segue:

```ts
// config.module-definition.ts

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setClassMethodName('forRoot').build();
```

Esta construção irá instruir `ConfigurableModuleBuilder` para gerar uma classe que expõe `forRoot` e `forRootAsync` em vez disso. Exemplo:

```ts
@Module({
  imports: [
    ConfigModule.forRoot({ folder: './config' }), // <-- Observe o uso de "forRoot" em vez de "register"
    // or alternatively:
    // ConfigModule.forRootAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...quaisquer dependência extra...]
    // }),
  ],
})
export class AppModule {}
```

## Classe de fábrica de opções personalizadas
Desde o método `registerAsync` (ou `forRootAsync`, ou qualquer outro nome, dependendo da configuração), permite que o consumidor passe na definição de provedor que resolva a configuração do módulo, um consumidor de biblioteca poderia fornecer uma classe para ser usada para construir o objeto de configuração.

```ts
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory,
    }),
  ],
})
export class AppModule {}
```

Essa classe, por padrão, deve fornecer o método `create()` que retorna um objeto de configuração do módulo. No entanto, se sua biblioteca seguir uma convenção de nomenclatura diferente, você poderá alterar esse comportamento e instruir `ConfigurableModuleBuilder` esperar um método diferente, por exemplo, `createConfigOptions`, usando o método `ConfigurableModuleBuilder#setFactoryMethodName`:

```ts
// config.module-definition.ts

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setFactoryMethodName('createConfigOptions').build();
```

Agora, a classe `ConfigModuleOptionsFactory` deve expor o método `createConfigOptions` (em vez de `create`):

```ts
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory, // <-- Esta classe deve fornecer o método "CreateConfigOptions"
    }),
  ],
})
export class AppModule {}
```

## Opções extras
Existem casos de string em que seu módulo pode precisar de opções extras que determinam como ele deve se comportar (um bom exemplo dessa opção é o sinalizador `isGlobal` - ou apenas `global`) que, ao mesmo tempo, não deve ser incluído no provedor `MODULE_OPTIONS_TOKEN` (por serem irrelevantes para serviços/provedores registrados nesse módulo, por exemplo, `ConfigService` não precisa saber se seu módulo host está registrado como um módulo global).

Nesses casos, o método `ConfigurableModuleBuilder#setExtras` pode ser usado. Veja o seguinte exemplo:

```ts
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>()
    .setExtras(
      {
        isGlobal: true,
      },
      (definition, extras) => ({
        ...definition,
        global: extras.isGlobal,
      }),
    )
    .build();
```

No exemplo acima, o primeiro argumento passou para o método `setExtras` é um objeto que contém valores padrão para as propriedades "extra". O segundo argumento é uma função que usa definições de módulo geradas automaticamente (com `provider`, `exports`, etc.) e objeto `extras` que representa propriedades extras (especificadas pelo consumidor ou padrões). O valor retornado desta função é uma definição de módulo modificada. Neste exemplo específico, estamos tomando o `extras.isGlobal' e atribuindo-o à propriedade `global` da definição do módulo (que, por sua vez, determina se um módulo é global ou não, leia mais [aqui](/overview/modules.md)).

Agora, ao consumir este módulo, o sinalizador adicional `isGlobal` pode ser passado, da seguinte maneira:

```ts
@Module({
  imports: [
    ConfigModule.register({
      isGlobal: true,
      folder: './config',
    }),
  ],
})
export class AppModule {}
```

No entanto, desde `isGlobal` é declarado como uma propriedade "extra", não estará disponível no provedor `MODULE_OPTIONS_TOKEN`:

```ts
@Injectable()
export class ConfigService {
  constructor(
    @Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions,
  ) {
    // "options" O objeto não terá a propriedade "isglobal"
    // ...
  }
}
```

## Estendendo métodos gerados automaticamente
Os métodos estáticos gerados automaticamente (`register`, `registerAsync`, etc.) pode ser estendido, se necessário, da seguinte forma:

```ts
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import {
  ConfigurableModuleClass,
  ASYNC_OPTIONS_TYPE,
  OPTIONS_TYPE,
} from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {
  static register(options: typeof OPTIONS_TYPE): DynamicModule {
    return {
      // sua lógica customizada aqui
      ...super.register(options),
    };
  }

  static registerAsync(options: typeof ASYNC_OPTIONS_TYPE): DynamicModule {
    return {
      // sua lógica customizada aqui
      ...super.registerAsync(options),
    };
  }
}
```

Observe o uso dos tipos `OPTIONS_TYPE` e `ASYNC_OPTIONS_TYPE` que devem ser exportados do arquivo de definição do módulo:

```ts
export const {
  ConfigurableModuleClass,
  MODULE_OPTIONS_TOKEN,
  OPTIONS_TYPE,
  ASYNC_OPTIONS_TYPE,
} = new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
```
