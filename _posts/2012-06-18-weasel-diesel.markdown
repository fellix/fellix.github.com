---
layout: post
title: Weasel-Diesel
keywords: Weasel-Diesel,rspec,rspec custom matchers,webservice,api
description: "Visão geral do uso do Weasel-Diesel com Rails, e como usar para testar seus serviços."
---

Alguns dias atrás [Matt Aimonet](http://twitter.com/merbist) escreveu em seu [blog um artigo sobre repensar o desenvolvimento de web services](http://matt.aimonetti.net/posts/2012/06/13/rethinking-web-service-development/), antes de continuar lendo esse post seria interessante ler o dele.

Não vou rescrever o que ele já escreveu, vou abordar mais a parte técnica da proposta dele, o [Weasel-Diesel](https://github.com/mattetti/Weasel-Diesel). A primeira vista quando olhei o código da DSL pensei:

> Que sintaxe horrível, como isso pode ser usável?

Mas essa normalmente é a primeira reação quando vemos algo que foge do que estamos acostumados, até das estranhesas que vemos no nosso dia-a-dia. Então deixei a idéia amadurecer, ou melhor, deixei estar, voltei para minhas atividades diárias. A noite, voltei para o artigo do Matt, li ele com mais calma e resolvi olhar o fonte, e a [aplicação sinatra de exemplo disponível](https://github.com/mattetti/sinatra-web-api-example), então pensei em como aplicar o conceito do Weasel-Diesel em uma aplicação Rails.

### Weasel-Diesel

Se você ainda não leu sobre o Weasel-Diesel no github ou no artigo do Matt, vou explicar um pouco o que ele é, se você já leu e sabe o que ele é, pode pular essa seção.

Weasel-Diesel é uma DSL para descrever serviços web, a intenção dele é ser simplista e desacoplado, ou seja, você pode usar seu framework favorito, o objetivo é prover somente uma forma de escrever o que cada serviço deve ser, qual seu input e output, e principalmente, a documentação desse serviço.

Por que a documentação é importante? Porque dificilmente você cria um serviço que ninguém poderá usar, e normalmente quem irá usar são outras pessoas, quando o serviço é pra você mesmo, ou para seus próprios projetos, não existe necessidade de documentação. Mas na maioria dos casos documentação importa e importa muito.

E a implementação? o Weasel-Diesel não tem a implementação do serviço, cabe a você escrevê-la do jeito que achar melhor e da forma que achar melhor, nas próximas seções vou mostrar alguns exemplos de como usar esse descritivo do Weasel-Diesel para validar os testes da sua implementação.

### Integrando com Rails

Como a documentação do WD só mostra exemplos usando Sinatra, resolvi fazer um teste usando o Rails. Adicionar ao projeto é super simples, basta abrir seu Gemfile é adicionar:

{% highlight ruby %}
gem 'weasel_diesel'
{% endhighlight %}

rodar o bundle, e pronto, já temos disponível em qualquer parte da nossa app o helper *describe_service*, sem arquivos de inicialização, ou qualquer configuração.

### Criando um serviço

Nosso serviço vai ser simples, recuperar os produtos cadastrados. A nossa url vai ser */products* e vamos simplesmente enviar um array json contendo o nome e o valor de cada produto cadastrado, a definição será algo assim:

{% highlight ruby %}
describe_service '/products' do |service|
  service.formats :json
  service.http_verb :get
  service.disable_auth

  service.response do |response|
    response.array :products do |node|
      node.string :name
      node.string :amount
    end
  end

  service.documentation do |doc|
    doc.overall <<-DOC
      Consulta a lista de produtos cadastrados
    DOC

    doc.example <<-DOC
      ## USAGE
      curl -i -H "Accept: application/json" http://localhost:3000/products
    DOC
  end
end
{% endhighlight %}

Vamos as explicações, primeiro passo é dizer qual serviço estamos descrevendo, nesse caso é */products*, dizemos que o formato dele é *json*, o verbo http é *get*, e desabilitamos a autenticação, lembrando que isso é só descritivo, não quer dizer que temos a autenticação habilitada realmente. Depois definimos qual vai ser a resposta do serviço, que vai ser um array, e cada elemento desse array vai ter o nome e o valor, ambos do tipo _string_.

Por ultimo, mas não menos importante, definimos a documentação, dizemos o que esse serviço é e como pode ser usado, nesse caso usando curl. Pronto, nosso serviço está definido, o que precisamos é implementá-lo.

### Implementando o serviço

Não vou entrar nos detalhes do nosso modelo por hora, o que nos interessa é a implementação do controller:

{% highlight ruby %}
class ProductsController < ApplicationController
  respond_to :json

  def index
    @products = Product.all
    respond_with({:products => @products})
  end
end
{% endhighlight %}

Esse código é bem simples, definimos um controller que responde a json, e simplesmente vai renderizar os produtos como json. Vamos escrever uma spec de request para testar a interação do usuário com o nosso serviço e ver se está tudo certo, vou usar o [RSpec](http://rspec.info/) e [Capybara](https://github.com/jnicklas/capybara) pra escrever esse teste.

{% highlight ruby %}
require 'spec_helper'

feature 'Product API' do
  include Rack::Test::Methods

  before do
    [
      {:name => "Macbook Pro",:amount => 2999},
      {:name => "Macbook Air", :amount => 3299},
      {:name => 'Ultrabook HP',  :amount => 1799}
    ].each do |attributes|
      Product.create attributes
    end
  end

  scenario 'list all the products' do
    header 'Accept', 'application/json'
    result = get '/products'

    service = WSList.all.find{|s| s.verb == :get && s.url == "/products"}
    valid, errors = service.validate_hash_response(JSON(result.body))

    assert valid, errors.join(" & ")
  end
end
{% endhighlight %}

Explicando o código: primeiro o include a _Rack::Test::Methods_ incluindo isso no nosso teste temos alguns helpers para fazer requests no nosso próprio domínio, inclusive podemos definir o header da request que estamos fazendo, [veja mais sobre isso aqui](http://www.anthonyeden.com/2010/11/testing-rest-apis-with-cucumber-and-rack-test/).

o bloco before simplesmente inserimos alguns registros na base, e depois escrevemos nosso scenario de teste, reparem que defino no header que essa request aceita json, dessa forma não preciso escrever a url como */products.json*. e fazemos uma requisição get a nossa url e recuperamos o resultado.

Precisamos encontrar a definição desse serviço para podermos testar nossa resposta, o WD disponibiliza uma classe _WSList_, aonde você consegue ter todos os serviços definidos, um simples find pra comparar o verbo e a url e temos nossa definição do serviço. Chamamos o método auxiliar validate_hash_response e passamos o resultado da nossa requisição, ele vai retornar um boolean, e um array de strings contento os erros caso tenha algum.

Também não podemos deixar de lado alguns ajustes a serem feitos no nosso *spec/spec_helper.rb*:

{% highlight ruby %}
ENV["RAILS_ENV"] ||= 'test'
require File.expand_path("../../config/environment", __FILE__)
require 'rspec/rails'
require 'rspec/autorun'

require 'json_response_verification'
require 'params_verification'
WeaselDiesel.send(:include, JSONResponseVerification)

Dir[Rails.root.join("spec/support/**/*.rb")].each {|f| require f}
Dir[Rails.root.join("app/api/**/*.rb")].each {|f| require f}

RSpec.configure do |config|
  config.use_transactional_fixtures = true
  config.infer_base_class_for_anonymous_controllers = false
end
{% endhighlight %}

O detalhe fica nos requires de *json_response_verification* e de *params_verification*, ambos disponibilizados pelo WD, e incluimos o módulo *JSONResponseVerification* no *WeaselDiesasel*, assim temos o metodo *validate_hash_response* disponível para os testes.

o assert no RSpec é disponibilizado pelo minitest, o que é algo que destoa um pouco do nosso teste, vamos criar um matcher pra resolver esse problema.

### Extendendo o RSpec

Para extender ao RSpec precisamos criar um novo matcher, já que nenhum dos padrões do RSpec vão fazer o que o WD faz, validar o json de resultado de acordo com o que foi descrito do serviço. Criar um matcher pro RSpec é relativamente simples, o que precisamos é definir uma classe que vai trabalhar com essa validação e um metodo para chamar no assert do RSpec:

{% highlight ruby %}
class BeApiResponse
  def initialize verb, url
    @service = WSList.all.find{|s| s.verb == verb.to_sym && s.url == "#{url}"}
    raise "API method [#{verb}] #{url} not found" unless @service
  end

  def matches? response
    @valid, @errors = @service.validate_hash_response(JSON(response.body))
    @valid
  end

  def description
    "be api response for [#{@service.verb}] #{@service.url}"
  end

  def failure_message
    @errors.join(' & ')
  end
end

def have_api_response verb, url
  BeApiResponse.new(verb, url)
end
{% endhighlight %}

a classe que foi criada recebe dois parametros o verbo http e a url que estamos buscando, a responsabilidade de encontrar o serviço ficou a cargo dessa classe, que vai simplemente mandar uma mensagem de erro caso não encontre o verbo para a url informada. O truque vem no metodo _matches?_ ele recebe um objeto e valida ele usando a validação do Weasel-Dieasel, e retorna se o elemento é válido ou não para a API, e a mensagem de falha são os erros concatenados.

por ultimo definimos o método que vamos usar pra fazer o matchers dentro do nosso teste, que vai receber o verbo e a url. E agora refatoramos nosso scenario para ficar assim:

{% highlight ruby %}
scenario 'list all the products' do
  header 'Accept', 'application/json'
  result = get '/products'

  result.should have_api_response(:get, '/products')
end
{% endhighlight %}

podemos reaproveitar o código em todos os testes.

### Conclusão

Apesar de em uma primeira olhada achar o Weasel-Diesel um tanto estranho, depois que você começa a fazer alguns testes e a trabalhar com ele você passa a ver ele com outros olhos, aproveitei esse espaço para escrever sobre o uso do WD para testar usa aplicação, mas você também pode usá-lo para gerar a documentação, [baixe o codigo de exemplo usado aqui](https://github.com/fellix/wd-sample) e execute o comando:

{% highlight bash %}
rake doc:services
{% endhighlight %}

e você verá uma simples documentação gerada, baseada nos serviços descritos. O código usado para gerar é o mesmo da aplicação de exemplo disponibilizada pelo Matt Aimonet, com algumas alterações pro nosso exemplo.

Se você escreve serviços usando Ruby, talvez valha a pena dar uma olhada no Weasel-Dielsel.