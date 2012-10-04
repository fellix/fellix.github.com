---
layout: post
title: JSONP on Rails
keywords: JSONP,jsonp,rails,javascript,json-p,cors,cross-domain ajax
description: "Utilizando Ruby on Rails para processar requisição Cross-domain utilizando JSONP"
---

Nos últimos dias venho fazendo alguns testes em um projeto pessoal, para implantar requisições usando jsonp. Então resolvi escrever aqui sobre essa implementação.

### JSONP

JSONP, é um acrônimo para *JSON with padding*, que não é nada mais que um complemento (leia-se hack) do JSON tradicional. A intenção é usa-lo para fazer requisição em um domínio diferente do seu e processar os dados.

Devido a [same origin policy](http://en.wikipedia.org/wiki/Same_origin_policy), os browsers não permitem você fazer requisições usando *XMLHttpRequest* para domínios externos, ou seja, você pode fazer requisições para */post* mas não pode fazer para *http://some.external.com/post*.

Felizmente, ou infelizmente, existe uma tag que não fica atrelada a essa política, a tag *script*, então o uso de JSONP consiste em usar a tag script para fazer uma requisição a um servidor e executar um callback para processar o resultado. Abaixo um exemplo bem simples de uma requisição usando a tag script:

{% highlight html %}
<script type="text/javascript"
        src="http://some.external.com/post/1.json">
</script>
{% endhighlight %}

Quando o html for executado esse script será processado e a requisição será feita ao servidor recuperando os dados, mas reparem que essa requisição nada mais é do que uma request para recuperar um JSON, não uma requisição JSONP, ainda precisamos do nosso callback, esse script convertido em JSONP ficaria algo parecido com o exemplo abaixo:

{% highlight html %}
<script type="text/javascript"
        src="http://some.external.com/post/1.json?callback=showPost">
</script>
{% endhighlight %}

Dessa forma, quando a requisição for completada a *function* *showPost* será executada. Lembrando que o servidor deve ter a capacidade de processar esse callback.

### Rails reconhecendo JSONP

Se você não conhece a [ActionController::Responser](http://weblog.rubyonrails.org/2009/8/31/three-reasons-love-responder/), ou a [gem responders](https://github.com/plataformatec/responders) sugiro você dar uma olhada, não que você precise disso para usar o JSONP no seu projeto Rails, é interessante para você deixar separado como gerenciar cada um dos tipos de requisições que sua aplicação venha a ter, nesse exemplo estamos usando a gem responders para processar requisições JSONP.

primeiro passo é indicar para o Rails reconhecer jsonp.

*config/initializers/mime_types.rb*

{% highlight ruby %}
Mime::Type.register_alias "application/javascript", :jsonp
{% endhighlight %}

depois criaremos nosso responder

*lib/jsonp_responder.rb*

{% highlight ruby %}
module JSONPResponder
  def to_jsonp
  	render :json => resource.to_json, :callback => options[:callback]
  end
end
{% endhighlight %}

Simplesmente converte o modelo em json, e manda o callback informado para a opção callback, disponível apartir do Rails 3.

*lib/application_responder.rb*

{% highlight ruby %}
require 'jsonp_responder'

class ApplicationResponder < ActionController::Responder
  include Responders::FlashResponder
  include Responders::HttpCacheResponder
  include JSONPResponder

end
{% endhighlight %}

e disponibilizamos no nosso *ApplicationResponder* o módulo recém criado.
A única alteração que precisamos fazer no nosso controller, se ele segue o padrão dos responders, é dizer que ele responde a jsonp e também passar o callback como parametro.

*app/controllers/posts_controller.rb*

{% highlight ruby %}
class PostsController < ApplicationController
  respond_to :jsonp

  def index
    @posts = Post.all
    
    respond_with @posts, :callback => params[:callback]
  end   
end
{% endhighlight %}

com isso nossa aplicação já responde a jsonp.

### Testando

Para testar vamos criar uma pequena página HTML, que fara a requisição ao servidor, veja o código abaixo:

*sample.html*

{% highlight html %}
<html>
  <head>
    <script type="text/javascript">
      function callback(data) {
        console.log(data);
      }
    </script>
    <script type="text/javascript"
    		src="http://localhost:3000/posts.jsonp?callback=callback"></script>
  </head>
  <body>
  </body>
</html>
{% endhighlight %}

Agora só abrir essa página e você verá na console do seu browser o resultado da requisição. Não precisa executar essa página dentro do Rails, pode abrir direto no navegador.

Ainda pretendo aprofundar um pouco mais no assunto, mas não hoje, fica para um próximo post. Até lá.
