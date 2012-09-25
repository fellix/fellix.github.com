---
layout: post
title: Handling your Exceptions
keywords: exceptions, handler, handling, api
description: "gerenciar exception com ruby on rails é bem simples, mas como fazer para tratar respostas a requisições via API usando JSON?"
---

Gerenciar exceptions com o Ruby on Rails é bem simples. Atualmente exceptions _404_ (not found) e _500_ (internal server error) tem um arquivo html na pasta _public_, quando uma dessas exceptions acontece, de acordo com o http status o rails renderiza o html correspondente.

### Outras abordagens

Em alguns casos precisamos ir mais a fundo nessa abordagem do rails, somente editar o html não é o que necessitamos em alguns casos. Mas não vou entrar nesses detalhes por hora, minha questão é quando estamos trabalhando com uma API.

Imagine comigo que estamos recebendo e enviando JSON, o usuário envia uma requisição com um JSON e o servidor responde com outro JSON. Digamos que aconteça alguma falha no servidor, algum registo inconsistente na base que não foi migrado corretamente, algo que tenhamos um erro com status _500_ (quando um exception qualquer é lançada e não foi tratada), obviamente, o body da sua resposta, será exatamente o conteudo html da página 500, se por um acaso, o cliente tentar consumir esse *"JSON"*, ele vai ver algum erro informando que o conteúdo não é um JSON válido.

Dentro do *ActiveSupport* temos o módulo *Rescuable*, contem um pequeno método chamado *[rescue_from](http://api.rubyonrails.org/classes/ActiveSupport/Rescuable/ClassMethods.html#method-i-rescue_from)*, o uso dele é simples:

{% highlight ruby %}
rescue_from Exception, :with => :some_method
{% endhighlight %}

onde o primeiro argumento pode ser uma ou mais exceptions, e a opção _with_ é um método responsável por gerenciar essa requisição.

em outras palavras, quando a _Exception_ acontecer o rails vai chamar *some_method* para identificar o que fazer com essa exception.

### Tratando requisições a API

Quando se está trabalhando somente com JSON, enviando e recebendo, é necessário fazer alguns ajustes pra garantir que a resposta será um JSON, mesmo em um bug do software, que seria normalmente uma resposta 500 ao cliente.

Tratar isso no rails é bem simples

{% highlight ruby %}
class ApplicationController < ActionController::Base
  respond_to :json
  
  rescue_from Exception, :with => :render_internal_error
  
  private
  
  def render_internal_error(exception)
    render :json => { :errors => ["erro de processamento interno"] }, :status => :internal_server_error 
  end
end
{% endhighlight %}

dessa forma nosso usuário vai ter uma mensagem, que ainda não é legal de se receber, mas não vai receber um HTML pra fazer parser, e o status continua sendo 500 usando o symbol *:internal_server_error*.

para ver mais desses symbols nas respostas do rails da uma olhada [aqui](http://www.codyfauser.com/2008/7/4/rails-http-status-code-to-symbol-mapping)

Você também pode tratar exceptions como *ActiveRecord::RecordNotFound*, para mostrar uma mensagem mais amigavél. Da forma que foi mostrado aqui, estou capturando uma exception genérica, evite esse tipo de abordagem, use as exceptions expecíficas do seu problema, e deixe a generalização em ultimo caso.

até a próxima.