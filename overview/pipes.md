# Pipes

Um pipe é uma classe anotada com o decorador `@Injectable()`, que implementa a interface `PipeTransform`.

![image](https://user-images.githubusercontent.com/22455192/221988250-da34facf-d160-4789-892b-4d3e1812b50b.png)

Os pipes têm dois casos de uso típicos:

* **transformação**: transforme os dados de entrada na forma desejada (por exemplo, de string para inteiro)
* **validação**: avaliar dados de entrada e, se válido, simplesmente deixa passar inalterados; caso contrário, faça uma exceção

Nos dois casos, os pipes operam no `arguments` sendo processado por um manipulador de rota do controlador. O Nest interpõe um pipe imediatamente antes de um método ser chamado, e o pipe recebe os argumentos destinados ao método e opera sobre eles. Qualquer operação de transformação ou validação ocorre naquele momento, após o qual o manipulador de rota é invocado com quaisquer argumentos transformados (potencialmente).

O Nest vem com vários pipes embutidos que você pode usar imediatamente. Você também pode construir seus próprios pipes personalizados. Neste capítulo, apresentaremos os pipes embutidos e mostraremos como ligá-los aos manipuladores de rota. Examinaremos vários pipes personalizados para mostrar como você pode construir um do zero.

> **DICA**
> 
> Pipes correm dentro da zona de exceções. Isso significa que, quando um pipe lança uma exceção, ele é tratado pela camada de exceções (filtro de exceções globais e qualquer [filtros de exceções](/overview/exception-filters.md) que são aplicados ao contexto atual). Dado o exposto, deve ficar claro que, quando uma exceção é lançada em um pipe, nenhum método de controlador é executado posteriormente. Isso fornece uma técnica de prática recomendada para validar dados que entram no aplicativo de fontes externas no limite do sistema.

## Pipes embutidos
O Nest vem com nove pipes disponíveis imediatamente:

* `ValidationPipe`
* `ParseIntPipe`
* `ParseFloatPipe`
* `ParseBoolPipe`
* `ParseArrayPipe`
* `ParseUUIDPipe`
* `ParseEnumPipe`
* `DefaultValuePipe`
* `ParseFilePipe`

Eles são exportados do pacote `@nestjs/common`.

Vamos dar uma olhada rápida no uso de `ParseIntPipe`. Este é um exemplo do transformação caso de uso, em que o pipe garante que um parâmetro manipulador de método seja convertido em um número inteiro JavaScript (ou lança uma exceção se a conversão falhar). Mais tarde neste capítulo, mostraremos uma implementação personalizada simples para um `ParseIntPipe`. As técnicas de exemplo abaixo também se aplicam aos outros pipes de transformação embutidos (`ParseBoolPipe`, `ParseFloatPipe`, `ParseEnumPipe`, `ParseArrayPipe` e `ParseUUIDPipe`, que chamaremos de pipes `Parse*` neste capítulo).

## Pipes de ligação
Para usar um pipe, precisamos vincular uma instância da classe de pipe ao contexto apropriado. Em nossa `ParseIntPipe` por exemplo, queremos associar o pipe a um método específico de manipulador de rota e garantir que ele funcione antes que o método seja chamado. Fazemos isso com a seguinte construção, à qual nos referiremos como vinculando ao pipe no nível do parâmetro do método:

```ts
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

Isso garante que uma das duas condições a seguir seja verdadeira: o parâmetro que recebemos no método `findOne()` é um número (conforme esperado em nossa chamada para `this.catsService.findOne()`), ou uma exceção é lançada antes que o manipulador de rota seja chamado.

Por exemplo, suponha que a rota seja chamada como:

```
GET localhost:3000/abc
```

Nest lançará uma exceção como esta:

```json
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}
```

A exceção impedirá o corpo do método `findOne()` de execução.

No exemplo acima, passamos uma classe (`ParseIntPipe`), não uma instância, deixando a responsabilidade pela instanciação na estrutura e permitindo a injeção de dependência. Como nos pipes e guardas, podemos passar por uma instância local. Passar uma instância local é útil se queremos personalizar o comportamento do pipe embutido, passando pelas opções:

```ts
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number,
) {
  return this.catsService.findOne(id);
}
```

Ligando os outros pipes de transformação (todos os pipes `Parse*`) funcionam da mesma forma. Todos esses pipes funcionam no contexto da validação de parâmetros de rota, parâmetros de cadeia de consultas e valores do corpo de solicitação.

Por exemplo, com um parâmetro de sequência de consulta:

```ts
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

Aqui está um exemplo de uso do `ParseUUIDPipe` para analisar um parâmetro de sequência e validar se é um UUID.

```ts
@Get(':uuid')
async findOne(@Param('uuid', new ParseUUIDPipe()) uuid: string) {
  return this.catsService.findOne(uuid);
}
```

> **DICA**
> 
> Ao usar `ParseUUIDPipe()` você está analisando o UUID nas versões 3, 4 ou 5; se você precisar apenas de uma versão específica do UUID, poderá passar uma versão nas opções do pipe.

Acima, vimos exemplos de ligação aos vários pipes `Parse*` embutidos. Vincular pipes de validação é um pouco diferente; discutiremos isso na seção a seguir.

> **DICA**
> 
> Além disso, veja [Técnicas de validação](/techniques/validation.md) para exemplos de pipes de validação.

## Pipes personalizados
Como mencionado, você pode construir seus próprios pipes personalizados. Enquanto o Nest fornece um robusto `ParseIntPipe` e `ValidationPipe`, iremos criar versões personalizadas simples de cada um do zero para ver como os pipes personalizados são construídos.

Começamos com um simples `ValidationPipe`. Inicialmente, simplesmente pegaremos um valor de entrada e retornaremos imediatamente o mesmo valor, comportando-se como uma função de identidade.

```ts
validation.pipe.tsJS

import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
```

> **DICA**
> 
> `PipeTransform<T, R>` é uma interface genérica que deve ser implementada por qualquer pipe. A interface genérica usa `T` para indicar o tipo de entrada `value`, e `R` para indicar o tipo de retorno do método `transform()`.

Todo pipe deve implementar o método `transform()` para cumprir o contrato de interface `PipeTransform`. Este método possui dois parâmetros:

* `value`
* `metadata`

O parâmetro `value` é o argumento do método atualmente processado (antes de ser recebido pelo método de manuseio de rota), e `metadata` é o metadado do argumento do método atualmente processado. O objeto de metadados possui estas propriedades:

```ts
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string;
}
```

Essas propriedades descrevem o argumento atualmente processado.

|         |         |
|---------|---------|
| `type`  | Indica se o argumento é um corpo `@Body()`, consulta `@Query()`, param `@Param()`, ou um parâmetro personalizado (leia mais aqui). |
| `metatype` | Fornece o metatipo do argumento, por exemplo, `String`. Nota: o valor é `undefined` se você omitir uma declaração de tipo na assinatura do método de manipulador de rota ou usar o JavaScript vanilla. |
| `data`  | A `string` passou para o decorador, por exemplo `@Body('string')`. Está `undefined` se você deixar o parêntese do decorador vazio. |

> **AVISO**
> 
> As interfaces TypeScript desaparecem durante a transpilação. Portanto, se o tipo de parâmetro de método for declarado como uma interface em vez de uma classe, o valor `metatype` será `Object`.

## Validação baseada em esquema
Vamos tornar nosso pipe de validação um pouco mais útil. Dê uma olhada mais de perto no método `create()` do `CatsController`, onde provavelmente gostaríamos de garantir que o objeto post body seja válido antes de tentar executar nosso método de serviço.

```ts
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Vamos nos concentrar no parâmetro do corpo de `createCatDto`. Seu tipo é `CreateCatDto`:

```ts
// create-cat.dto.ts

export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Queremos garantir que qualquer solicitação recebida para o método `create` contenha um corpo válido. Então temos que validar os três membros do objeto `createCatDto`. Poderíamos fazer isso dentro do método do manipulador de rotas, mas isso não é o ideal, pois quebraria o **regra de responsabilidade única** (SRP).

Outra abordagem poderia ser criar um classe validadora e delegar a tarefa lá. Isso tem a desvantagem que teríamos que lembrar de chamar esse validador no início de cada método.

Que tal criar middleware de validação? Isso pode funcionar, mas infelizmente não é possível criar middleware genérico que pode ser usado em todos os contextos em todo o aplicativo. Isso ocorre porque o middleware não tem conhecimento do contexto de execução, incluindo o manipulador que será chamado e qualquer um de seus parâmetros.

É claro que esse é exatamente o caso de uso para o qual os pipes são projetados. Então, vamos em frente e refine nosso pipe de validação.

## Validação de esquema de objetos
Existem várias abordagens disponíveis para validar objetos de maneira limpa, seguindo o padrão [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself). Uma abordagem comum é usar baseado em esquema de validação. Vamos em frente e tentar essa abordagem.

A biblioteca [Joi](https://github.com/sideway/joi) permite criar esquemas de maneira direta, com uma API legível. Vamos construir um pipe de validação que faça uso dos esquemas baseados em Joi.

Comece instalando o pacote necessário:

```bash
$ npm install --save joi
```

Na amostra de código abaixo, criamos uma classe simples que usa um argumento no `constructor`. Em seguida, aplicamos o método `schema.validate()`, que valida nosso argumento recebido contra o esquema fornecido.

Como observado anteriormente, um pipe de validação retorna o valor inalterado ou lança uma exceção.

Na próxima seção, você verá como fornecemos o esquema apropriado para um determinado método de controlador usando o decorador `@UsePipes()`. Fazer isso torna nosso pipe de validação reutilizável em contextos, exatamente como nos propusemos a fazer.

```ts
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { ObjectSchema } from 'joi';

@Injectable()
export class JoiValidationPipe implements PipeTransform {
  constructor(private schema: ObjectSchema) {}

  transform(value: any, metadata: ArgumentMetadata) {
    const { error } = this.schema.validate(value);
    if (error) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }
}
```

## Pipes de validação de ligação
Anteriormente, vimos como ligar pipes de transformação (como `ParseIntPipe` e o resto dos pipes `Parse*`).

A ligação de pipes de validação também é muito direta.

Nesse caso, queremos ligar o pipe no nível da chamada do método. Em nosso exemplo atual, precisamos fazer o seguinte para usar o `JoiValidationPipe`:

* Crie uma instância do `JoiValidationPipe`
* Passe o esquema Joi específico do contexto no construtor de classes do pipe
* Amarre o pipe ao método

Exemplo de esquema de Joi:

```ts
const createCatSchema = Joi.object({
  name: Joi.string().required(),
  age: Joi.number().required(),
  breed: Joi.string().required(),
})

export interface CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Fazemos isso usando o decorador `@UsePipes()` como mostrado abaixo:

```ts
@Post()
@UsePipes(new JoiValidationPipe(createCatSchema))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

> **DICA**
>
> O decorador `@UsePipes()` é importado do pacote `@nestjs/common`.

## Validador de classe

> **AVISO**
> 
> As técnicas nesta seção requerem TypeScript e não estão disponíveis se o seu aplicativo for gravado usando o JavaScript vanilla.

Vejamos uma implementação alternativa para nossa técnica de validação.

Nest funciona bem com o biblioteca [validador de classe](https://github.com/typestack/class-validator). Esta poderosa biblioteca permite que você use a validação baseada em decorador. A validação baseada em decorador é extremamente poderosa, especialmente quando combinada com os recursos de pipes do Nest, pois temos acesso ao `metatype` da propriedade processada. Antes de começar, precisamos instalar os pacotes necessários:

```bash
$ npm i --save class-validator class-transformer
```

Depois de instalados, podemos adicionar alguns decoradores a classe `CreateCatDto`. Aqui vemos uma vantagem significativa dessa técnica: A classe `CreateCatDto` continua sendo a única fonte de verdade para o nosso objeto Post body (em vez de ter que criar uma classe de validação separada).

```ts
// create-cat.dto.ts

import { IsString, IsInt } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;

  @IsString()
  breed: string;
}
```

> **DICA**
> 
> Leia mais sobre os decoradores de classe validadora [aqui](https://github.com/typestack/class-validator#usage).

Agora podemos criar uma classe `ValidationPipe` que usa essas anotações.

```ts
// validation.pipe.ts

import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

> **AVISO**
> 
> Acima, usamos a biblioteca [class transform](https://github.com/typestack/class-transformer). É feito pelo mesmo autor da biblioteca **validador de classe** e, como resultado, eles tocam muito bem juntos.

Vamos passar por esse código. Primeiro, observe que o método `transform()` é marcado como `async`. Isso é possível porque o Nest suporta pipes síncrono e assíncrono. Nós fazemos esse método `async` porque algumas das validações do [validador de classe pode ser assíncrono](https://github.com/typestack/class-validator#custom-validation-classes) (utilize `Promises`).

Em seguida, observe que estamos usando a desestruturação para extrair o campo do metatipo (extraindo apenas esse membro de um `ArgumentMetadata`) em nosso parâmetro `metatype`. Isso é apenas uma abreviação para obter o total de `ArgumentMetadata` e depois ter uma instrução adicional para atribuir a variável `metatype`.

Em seguida, observe a função auxiliar `toValidate()`. É responsável por ignorar a etapa de validação quando o argumento atual que está sendo processado é um tipo de JavaScript nativo (que não pode ter decoradores de validação anexados, portanto, não há razão para executá-los na etapa de validação).

Em seguida, usamos a função de transformador de classe `plainToInstance()` para transformar nosso objeto de argumento JavaScript simples em um objeto typescript para que possamos aplicar a validação. A razão pela qual devemos fazer isso é que o objeto de corpo de postagem recebido, quando desserializado da solicitação de rede, não possui nenhuma informação de tipo (é assim que a plataforma subjacente, como o Express, funciona). O validador de classe precisa usar os decoradores de validação que definimos para o nosso DTO anteriormente, por isso precisamos executar essa transformação para tratar o corpo recebido como um objeto decorado adequadamente, não apenas um objeto simples de javascript vanilla.

Finalmente, como observado anteriormente, uma vez que este é um pipe de validação, deve retorna o valor inalterado ou lançar uma exceção.

O último passo é vincular a `ValidationPipe`. Os pipes podem ser com escopo de parâmetro, escopo de método, escopo de controlador ou escopo global. Anteriormente, com nosso pipe de validação baseado em Joi, vimos um exemplo de ligação do pipe no nível do método. No exemplo abaixo, vincularemos a instância do pipe ao manipulador de rota no decorador `@Body()` para que nosso pipe seja chamado para validar o corpo.

```ts
// cats.controller.ts

@Post()
async create(
  @Body(new ValidationPipe()) createCatDto: CreateCatDto,
) {
  this.catsService.create(createCatDto);
}
```

Os pipes com escopo de parâmetro são úteis quando a lógica de validação se refere a apenas um parâmetro especificado.

## Pipes com escopo global
Desde que o `ValidationPipe` foi criado para ser o mais genérico possível, podemos perceber que é um utilitário completo, configurando-o como um pipe com escopo global para que seja aplicado a todos os manipuladores de rota em toda a aplicação.

```ts
main.tsJS

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

> **AVISO**
> 
> No caso de [aplicativos híbridos](/faq/hybrid-application.md) o método `useGlobalPipes()` não configura pipes para gateways e microservices. Para aplicativos de microsservices "padrão" (não híbridos), o `useGlobalPipes()` monta pipes globalmente.

Pipes globais são usados em toda a aplicação, para todos os controladores e manipuladores de rotas.

Observe que, em termos de injeção de dependência, os pipes globais são registrados de fora de qualquer módulo (com `useGlobalPipes()`, como no exemplo acima) não pode injetar dependências, pois a ligação foi feita fora do contexto de qualquer módulo. Para resolver esse problema, você pode configurar um pipe global diretamente de qualquer módulo usando a seguinte construção:

```ts
// app.module.ts

import { Module } from '@nestjs/common';
import { APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}
```

> **DICA**
> 
> Ao usar essa abordagem para executar a injeção de dependência do pipe, observe que, independentemente do módulo em que essa construção é empregada, o pipe é, de fato, global. Onde isso deve ser feito? Escolha o módulo em que o pipe (`ValidationPipe` no exemplo acima) é definido. Além disso, `useClass` não é a única maneira de lidar com o registro personalizado do provedor. Saiba mais [aqui](/fundamentals/custom-providers.md).

## O ValidationPipe embutido
Como lembrete, você não precisa criar um tubo de validação genérico por conta própria, pois o `ValidationPipe` é fornecido pelo Nest e está pronto para uso. O embutido `ValidationPipe` oferece mais opções do que a amostra que construímos neste capítulo, que foi mantida básica para ilustrar a mecânica de um pipe personalizado. Você pode encontrar detalhes completos, juntamente com muitos exemplos aqui.

## Caso de uso da transformação
A validação não é o único caso de uso para tubos personalizados. No início deste capítulo, mencionamos que um tubo também pode transformar os dados de entrada para o formato desejado. Isso é possível porque o valor retornado da função `transform` substitui completamente o valor anterior do argumento.

Quando isso é útil? Considere que, às vezes, os dados passados do cliente precisam sofrer algumas alterações - por exemplo, converter uma sequência em um número inteiro - antes que possam ser manipulados adequadamente pelo método do manipulador de rotas. Além disso, alguns campos de dados necessários podem estar ausentes e gostaríamos de aplicar valores padrão. Pipes de transformação podem executar essas funções interpondo uma função de processamento entre a solicitação do cliente e o manipulador de solicitações.

Aqui está um simples `ParseIntPipe` responsável por analisar uma sequência em um valor inteiro. (Como observado acima, o Nest possui um `ParseIntPipe` embutido isso é mais sofisticado; incluímos isso como um exemplo simples de um pipe de transformação personalizado).

```ts
// parse-int.pipe.ts

import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```

Podemos então vincular esse pipe ao parâmetro selecionado, como mostrado abaixo:

```ts
@Get(':id')
async findOne(@Param('id', new ParseIntPipe()) id) {
  return this.catsService.findOne(id);
}
```

Outro caso de transformação útil seria selecionar um usuário existente como entidade do banco de dados usando um ID fornecido na solicitação:

```ts
@Get(':id')
findOne(@Param('id', UserByIdPipe) userEntity: UserEntity) {
  return userEntity;
}
```

Deixamos a implementação deste pipe para o leitor, mas observe que como todos os outros pipes de transformação, ele recebe um valor de entrada (um `id`) e retorna um valor de saída (um objeto `UserEntity`). Isso pode tornar seu código mais declarativo e DRY abstraindo o código do boilerplate do seu manipulador e em um pipe comum.

## Fornecendo padrões
Os pipes `Parse*` esperam que o valor de um parâmetro seja definido. Eles lançam uma exceção ao receber valores `null` ou `undefined`. Para permitir que um terminal lide com valores de parâmetros ausentes da consulta, precisamos fornecer um valor padrão a ser injetado antes do pipe `Parse*` para que operem nesses valores. O `DefaultValuePipe` serve a esse propósito. Simplesmente instanciar um `DefaultValuePipe` no decorador `@Query()` antes do pipe relevante, como mostrado abaixo:

```ts
@Get()
async findAll(
  @Query('activeOnly', new DefaultValuePipe(false), ParseBoolPipe) activeOnly: boolean,
  @Query('page', new DefaultValuePipe(0), ParseIntPipe) page: number,
) {
  return this.catsService.findAll({ activeOnly, page });
}
```
