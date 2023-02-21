# Providers

Os providers são um conceito fundamental no Nest. Muitas das classes básicas de Nest podem ser tratadas como um fornecedor de serviços, repositórios, fábricas, auxiliares e assim por diante. A principal idéia de um provider é que ele pode ser injetado como uma dependência; isso significa que os objetos podem criar vários relacionamentos entre si, e a função de "ligar" instâncias de objetos pode ser amplamente delegada ao sistema de tempo de execução do Nest.

![image](https://user-images.githubusercontent.com/22455192/220456115-4fe42025-5dc8-4aba-a569-362c5913170b.png)

No capítulo anterior, construímos um simples `CatsController`. Os controladores devem lidar com solicitações HTTP e delegar tarefas mais complexas para provedores. Os fornecedores são classes JavaScript simples que são declaradas como providers em um [módulo](/overview/modules.md).

> **DICA**
> Como o Nest permite a possibilidade de projetar e organizar dependências de uma maneira mais OO, é altamente recomendável seguir os princípios [SOLID](https://en.wikipedia.org/wiki/SOLID).

## Serviços
Vamos começar criando um simples `CatsService`. Este serviço será responsável pelo armazenamento e recuperação de dados e foi projetado para ser usado pelo `CatsController`, portanto, é um bom candidato a ser definido como um provedor.

```ts
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];
DICAe
  create(cat: Cat) {
    this.cats.push(cat);
  }

  findAll(): Cat[] {
    return this.cats;
  }
}
```

> **DICA**
> Para criar um serviço usando a CLI, basta executar o comando `$ nest g service cats`.

Nosso `CatsService` é uma classe básica com uma propriedade e dois métodos. O único recurso novo é que ele usa o decorador `@Injectable()`. O decorador `@Injectable()` anexa metadados, o que declara que `CatsService` é uma classe que pode ser gerenciada pelo container Nest [IoC](https://en.wikipedia.org/wiki/Inversion_of_control). A propósito, este exemplo também usa uma interface `Cat`, que provavelmente se parece com isso:

```ts
// interfaces/cat.interface.ts

export interface Cat {
  name: string;
  age: number;
  breed: string;
}
```

Agora que temos uma classe de serviço para recuperar cats, vamos usá-lo dentro do `CatsController`:

```ts
cats.controller.tsJS

import { Controller, Get, Post, Body } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

O `CatsService` é injetado através do construtor de classes. Observe o uso da sintaxe `private`. Essa abreviação nos permite declarar e inicializar o membro `catsService` imediatamente no mesmo local.

## Injeção de dependência
O nest é construído em torno do forte padrão de design comumente conhecido como Injeção de dependência. Recomendamos a leitura de um ótimo artigo sobre esse conceito na documentação do [Angular](https://angular.io/guide/dependency-injection).

No Nest, graças aos recursos do TypeScript, é extremamente fácil gerenciar dependências porque elas são resolvidas apenas por tipo. No exemplo abaixo, Nest resolverá o `catsService` criando e retornando uma instância de `CatsService` (ou, no caso normal de um singleton, retornando a instância existente se já tiver sido solicitada em outro lugar). Essa dependência é resolvida e passada para o construtor do seu controlador (ou atribuída à propriedade indicada):

```ts
constructor(private catsService: CatsService) {}
```

## Escopos
Os providers normalmente têm um escopo vitalício ("") sincronizado com o ciclo de vida do aplicativo. Quando o aplicativo é inicializado, toda dependência deve ser resolvida e, portanto, todo provedor deve ser instanciado. Da mesma forma, quando o aplicativo for encerrado, cada provedor será destruído. No entanto, existem maneiras de tornar a vida útil do seu provedor com escopo de solicitação também. Você pode ler mais sobre essas técnicas [aqui](/fundamentals/injection-scopes.md).

## Providers personalizados
O Nest possui uma inversão interna do contêiner de controle ("IoC") que resolve as relações entre os providers. Esse recurso está subjacente ao recurso de injeção de dependência descrito acima, mas na verdade é muito mais poderoso do que o que descrevemos até agora. Existem várias maneiras de definir um provider: você pode usar valores simples, classes e fábricas assíncronas ou síncronas. Mais exemplos são fornecidos [aqui](/fundamentals/dependency-injection.md).

## Providers opcionais
Ocasionalmente, você pode ter dependências que não precisam necessariamente ser resolvidas. Por exemplo, sua classe pode depender de um objeto de configuração, mas se nenhum for passado, os valores padrão deverão ser usados. Nesse caso, a dependência se torna opcional, porque a falta do provedor de configuração não levaria a erros.

Para indicar que um provedor é opcional, use o decorador `@Optional()` na assinatura do construtor.

```ts
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  constructor(@Optional() @Inject('HTTP_OPTIONS') private httpClient: T) {}
}
```

Observe que, no exemplo acima, estamos usando um provider personalizado, razão pela qual incluímos o token personalizado `HTTP_OPTIONS`. Exemplos anteriores mostraram injeção baseada em construtor, indicando uma dependência através de uma classe no construtor. Leia mais sobre provedores personalizados e seus tokens associados [aqui](/fundamentals/custom-providers.md).

## Injeção baseada em propriedade
A técnica que usamos até agora é chamada de injeção baseada em construtor, pois os provedores são injetados pelo método construtor. Em alguns casos muito específicos, injeção baseada em propriedades pode ser útil. Por exemplo, se sua classe de nível superior depender de um ou vários provedores, repassá-los até o fim chamando `super()` nas subclasses do construtor pode ser muito tedioso. Para evitar isso, você pode usar o decorador `@Inject()` no nível da propriedade.

```ts
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}
```

> **AVISO**
> Se sua classe não estender outro provider, você sempre deve preferir usar baseado em construtor injeção.

## Registro do provider
Agora que definimos um provider (`CatsService`), e temos um consumidor desse serviço (`CatsController`), precisamos registrar o serviço no Nest para que ele possa executar a injeção. Fazemos isso editando nosso arquivo de módulo (`app.module.ts`) e adicionando o serviço ao providers matriz do decorador `@Module()`.

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

Nest agora poderá resolver as dependências da classe `CatsController`.

É assim que nossa estrutura de diretórios deve ficar agora:

![Screenshot from 2023-02-21 19-39-39](https://user-images.githubusercontent.com/22455192/220475179-77883ba2-e352-4128-a0ca-28861e97978d.png)

## Instanciação manual
Até o momento, discutimos como o Nest lida automaticamente com a maioria dos detalhes da resolução de dependências. Em certas circunstâncias, pode ser necessário sair do sistema de injeção de dependência integrado e recuperar ou instanciar manualmente os provedores. Discutimos brevemente dois desses tópicos abaixo.

Para obter instâncias existentes ou instanciar provedores dinamicamente, você pode usar a [Referência do módulo](/fundamentals/module-ref.md).

Para obter provedores dentro da função `bootstrap()` (por exemplo, para aplicativos independentes sem controladores ou para utilizar um serviço de configuração durante o inicialização), consulte [Aplicativos independentes](/overview/standalone-applications.md).
