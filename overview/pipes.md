# Pipes

Um pipe é uma classe anotada com o decorador `@Injectable()`, que implementa a interface `PipeTransform`.

![image](https://user-images.githubusercontent.com/22455192/221988250-da34facf-d160-4789-892b-4d3e1812b50b.png)

Os pipes têm dois casos de uso típicos:

* **transformação**: transforme os dados de entrada na forma desejada (por exemplo, de string para inteiro)
* **validação**: avaliar dados de entrada e, se válido, simplesmente deixa passar inalterados; caso contrário, faça uma exceção

Nos dois casos, os pipes operam no `arguments` sendo processado por um manipulador de rota do controlador. O Nest interpõe um pipe imediatamente antes de um método ser chamado, e o pipe recebe os argumentos destinados ao método e opera sobre eles. Qualquer operação de transformação ou validação ocorre naquele momento, após o qual o manipulador de rota é invocado com quaisquer argumentos transformados (potencialmente).

O Nest vem com vários pipes embutidos que você pode usar imediatamente. Você também pode construir seus próprios pipes personalizados. Neste capítulo, apresentaremos os pipes embutidos e mostraremos como ligá-los aos manipuladores de rota. Examinaremos vários pipes personalizados para mostrar como você pode construir um do zero.

> **DICA**
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
> Ao usar `ParseUUIDPipe()` você está analisando o UUID nas versões 3, 4 ou 5; se você precisar apenas de uma versão específica do UUID, poderá passar uma versão nas opções do pipe.

Acima, vimos exemplos de ligação aos vários pipes `Parse*` embutidos. Vincular pipes de validação é um pouco diferente; discutiremos isso na seção a seguir.

> **DICA**
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
