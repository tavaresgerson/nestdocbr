# Introdução

Nest (NestJS) é uma estrutura para a construção de um ambiente eficiente e escalável para aplicativos Node.js no lado do servidor. Ele usa JavaScript progressivo, é construído e suporta totalmente TypeScript (ainda permite que os desenvolvedores codifiquem em JavaScript puro) e combina elementos de OOP (Programação Orientada a Objetos), FP (Programação Funcional) e FRP (Programação Reativa Funcional).

Sob o capô, a Nest utiliza estruturas robustas de servidor HTTP, como [Express](https://expressjs.com/) (por padrão) e opcionalmente pode ser configurado para uso com [Fastify](https://github.com/fastify/fastify) também!

O Nest fornece um nível de abstração acima dessas estruturas comuns do Node.js (Express/Fastify), mas também expõe suas APIs diretamente ao desenvolvedor. Isso fornece a liberdade de usar uma infinidade de módulos de terceiros disponíveis para a plataforma subjacente.

## Filosofia
Nos últimos anos, graças ao Node.js, o JavaScript se tornou a “língua franca” da web para aplicativos de front e back-end. Isso deu origem a projetos impressionantes, como Angular, React e Vue, que melhoram a produtividade do desenvolvedor e permitem a criação de aplicativos frontend rápidos, testáveis e extensíveis. No entanto, embora existam muitas bibliotecas, auxiliares e ferramentas excelentes para o Node (e JavaScript no servidor), nenhum deles resolve efetivamente o principal problema - Arquitetura.

O Nest fornece uma arquitetura de aplicativo pronta para uso, que permite que desenvolvedores e equipes criem aplicativos altamente testáveis, escaláveis, pouco acoplados e facilmente sustentáveis. A arquitetura é fortemente inspirada no Angular.

## Instalação
Para começar, você pode usar o scaffold do projeto com o [Nest CLI](https://docs.nestjs.com/cli/overview), ou clone um projeto inicial (ambos produzirão o mesmo resultado).

Para alternar o projeto com a Nest CLI, execute os comandos a seguir. Isso criará um novo diretório de projeto e preencherá o diretório com os arquivos principais iniciais do Nest e os módulos de suporte, criando uma estrutura básica convencional para o seu projeto. Criar um novo projeto com o Nest CLI é recomendado para usuários iniciantes. Continuaremos com essa abordagem em [Primeiros passos](/overview/first-steps.md).

```bash
$ npm i -g @nestjs/cli
$ nest new project-name
```

> DICA
> Para criar um novo projeto com o TypeScript no [modo strict](https://www.typescriptlang.org/tsconfig#strict) ativado, passe a flag `--strict` para o comando `nest new`.

## Alternativas
Como alternativa, para instalar o projeto inicial do TypeScript com Git:

```bash
$ git clone https://github.com/nestjs/typescript-starter.git project
$ cd project
$ npm install
$ npm run start
```

> DICA
> Se você deseja clonar o repositório sem o histórico git, pode usar [degit](https://github.com/Rich-Harris/degit).

Abra o navegador e navegue para http://localhost:3000/.

Para instalar o JavaScript do projeto inicial, use `javascript-starter.git` na sequência de comandos acima.

Você também pode criar manualmente um novo projeto do zero, instalando os arquivos principais e de suporte com *npm* (ou *yarn*). Nesse caso, é claro, você será responsável por criar os arquivos de boilerplates do projeto.

```bash
$ npm i --save @nestjs/core @nestjs/common rxjs reflect-metadata
```
