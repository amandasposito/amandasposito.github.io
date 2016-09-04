---
layout: post
title:  "Rails Engines e RSpec"
date:   2016-01-19 21:23:37 -0300
categories:
  - rails engines
  - rspec
---
Você decidiu usar Rails Engines na sua aplicação ou herdou um projeto que faz uso delas e você gostaria de entender como o fluxo funciona. O Rails guides possui uma introdução bem completa sobre o que são as tais das Engines [(http://guides.rubyonrails.org/engines.html)](http://guides.rubyonrails.org/engines.html)

Dentre seus usos, a capacidade de isolar uma parte do sistema, encapsulando seu funcionamento, é uma característica que, quando bem utilizada, poderá ser um grande trunfo na evolução de seu sistema.

É importante ressaltar que servem como módulos e deverão ser “plugados” na aplicação principal. Portanto, tome cuidado com a dependência de código. Tente levar sempre em consideração os princípios [SOLID](https://en.wikipedia.org/wiki/SOLID_%28object-oriented_design%29) para abstraçã0 do código.

Dito isto, vamos ao que interessa.

Criando a uma Rails Engine do zero:

```
$ rails plugin new core -T --mountable --full --dummy-path=spec/dummy_app
```

Ao utilizarmos o comando **rails plugin —help** podemos verificar as opções utilizadas na instrução:

```
-T                 # Skip Test::Unit files
--mountable        # Generate mountable isolated application
--full             # Generate a rails engine with bundled Rails
                     application for testing
--dummy-path       # Create dummy application at given path
```

Se por algum motivo, a engine for gerada sem o dummy spec, não tem problema, ele foi feito para não ser ligado a aplicação principal, então você pode criá-lo em outro lugar e depois movê-lo para seu lugar final.

Nesse post, vamos usar o [RSpec](http://rspec.info/) como framework de testes. Para configurarmos ele com nossa engine, iremos adicionar a dependência ao nosso arquivo .gemspec na raiz da engine.

Isso se dá porque, de acordo com [a documentação do Rails](http://guides.rubyonrails.org/engines.html#other-gem-dependencies), a engine pode ser adicionada como uma Gem futuramente e se as dependências estivessem no GemFile, elas poderiam não ser reconhecidas na hora da instalação, o que causaria bugs.

{% gist https://gist.github.com/amandasposito/b181aa6509a00cfb9af6#file-core-gemspec %}

Feito isso, iremos executar o comando **bundle install** no terminal, dentro da pasta **core**.

Na raiz da nossa aplicação, deve existir um arquivo no caminho lib/core/engine.rb. Iremos configurá-lo para incluir o RSpec.

{% gist https://gist.github.com/amandasposito/16be9724ff87db2bddd3#file-engine-rb %}

E logo depois **rails generate rspec:install**, conforme a documentação do rspec, para criar o diretório **spec** (onde os testes irão ficar).

No arquivo **spec/rails_helper.rb** iremos adicionar uma configuração para que nossos testes olhem para o arquivo de configuração da engine.

{% gist https://gist.github.com/amandasposito/d389d0992891c4672691#file-spec_helper-rb %}

Para testarmos nosso fluxo, podemos gerar um simples model:

```
$ rails generate model article title:string text:text

$ bundle exec rake db:migrate RAILS_ENV=test
```

Dentro da nossa dummy\_app de testes, vamos configurar as rotas em **spec/dummy_app/config/** para apontar para a raíz de nossa aplicacação.

{% gist https://gist.github.com/amandasposito/1d5008ade337d530752d#file-routes-rb %}

Apenas para validarmos o fluxo, vamos inserir duas regras de validação no nosso modelo:

{% gist https://gist.github.com/amandasposito/707d8fb05dc4f331b634#file-article-rb %}

{% gist https://gist.github.com/amandasposito/df17cc3d10a3a6418147#file-article_spec-rb %}

E então rodar nossos testes com **bundle exec rspec**

![Nosso resultado final deverá ser assim](/assets/images/rspec-engine-tests.png)

Pronto, nossa engine está configurada com RSpec como suíte de testes, agora é só codificar.

O código final pode ser encontrado aqui [(https://github.com/amandasposito/rails-engines-rspec)](https://github.com/amandasposito/rails-engines-rspec).

Até mais!

[![Papo reto - rails engines](/assets/images/placeholder-engines.png)](https://www.youtube.com/watch?v=Jk1D759_epo)

### Referências

* http://rspec.info/
* http://guides.rubyonrails.org
* http://guides.rubyonrails.org/engines.html
