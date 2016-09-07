---
layout: post
title:  "Criando um chat utilizando Elixir e Phoenix"
date:   2015-12-08 22:33:37 -0300
categories:
  - elixir
  - phoenix
---

Há um tempo tenho estudado sobre Elixir e motivada pelo [Papo Reto da Bluesoft](http://youtube.com/bluesoftbr), resolvi escrever um post sobre como construir um chat simples utilizando Elixir e Phoenix.

Para começarmos, primeiro devemos instalar o Elixir, no [site](http://elixir-lang.org/install.html) deles existem os passos que devemos seguir.

Feito isso, é a vez de instalar o framework Phoenix. Novamente, basta seguir os passos do [site](http://www.phoenixframework.org/docs/installation) deles.

Para esse post, estou utilizando a versão 1.1.1 do Elixir e 1.0.4 do Phoenix.

Depois que tudo estiver instalado, no terminal, iremos digitar o comando abaixo:

{% highlight bash %}
$ mix phoenix.new demo_chat ~/projects
{% endhighlight %}

Este comando será responsável por criar uma aplicação com a estrutura de pastas padrão com o nome de *"demo_chat"* dentro da pasta *"projects"*.

![Estrutura padrão de arquivos](/assets/images/estrutura-basica-phoenix.png)

Para iniciarmos a aplicação basta digitar no terminal:

```
$ mix phoenix.server
```

Ela irá iniciar na porta 4000 e o resultado deverá ser assim:

![Página padrão de um projeto recém criado](/assets/images/default-phoenix-page.png)

Neste exemplo estou usando jquery e bootstrap, ambos adicionados na página **app.html.eex** que fica em **web > templates > layout**.

Feito isso vamos começar a configurar o arquivo **app.js** para lidar com nosso chat. Ele se encontra em **web > static > assets > js**.

A estrutura do arquivo é bem simples, a parte mais importante para nosso exemplo agora é importar o arquivo de Sockets. Ele será responsável pela nossa conexão.

```
import {Socket} from "deps/phoenix/web/static/js/phoenix"
```

Feito isso, o arquivo esta divido em algumas funções. Para nos juntarmos ao channel, que iremos configurar mais adiante, iremos adicionar um trecho de código que conecta ao nosso channel no carregamento da página.

```
let socket = new Socket("/socket");

socket.connect();

socket.onClose( e => console.log("Closed connection") );

var channel = socket.channel("rooms:lobby", {});

channel.join()
       .receive( "error", () => console.log("Failed to connect") )
       .receive( "ok", () => console.log("Connected") )
```

O arquivo final ficará parecido com este:

```
import "deps/phoenix_html/web/static/js/phoenix_html"
import {Socket} from "deps/phoenix/web/static/js/phoenix"

class App {
  static init() {
    var username = $("#username");
      var msgBody  = $("#message");

      //joins the channel
      let socket = new Socket("/socket");
      socket.connect();
      socket.onClose( e => console.log("Closed connection") );

      var channel = socket.channel("rooms:lobby", {});

      //handle the responses
      channel.join()
             .receive( "error", () => console.log("Failed to connect") )
             .receive( "ok",    () => console.log("Connected") )

      msgBody.off("keypress")
        .on("keypress", e => {
          if (e.keyCode == 13) {

            //pushes the message to the server
            channel.push("new:message", {
                    user: username.val(),
                    body: msgBody.val()
                  });

            msgBody.val("");
          }
        });

      //receiving broadcast messages
      channel.on( "new:message", msg => this.renderMessage(msg) )
  }

  static renderMessage(msg) {
    var messages = $("#messages")
    var user = this.sanitize(msg.user || "New User")
    var body = this.sanitize(msg.body)

    //append the message to the page
    messages.append(`<p><strong>[${user}]</strong>: ${body}</p>`)
  }

  static sanitize(str) {
    return $("<div/>").text(str).html()
  }
}

$( () => App.init() )

export default App
```

Certo, com o javascript pronto, vamos adicionar um channel. No arquivo que fica no caminho **web > channels > user_socket.ex** , iremos adicionar a seguinte linha de código:

```
channel “rooms:*”, DemoChat.RoomChannel
```

Essa linha é responsável por tratar qualquer mensagem que nós mandarmos que tenha o tópico **rooms:** , configurado no nosso javascript, através da variável **channel**.

Antes de falar do próximo passo, vamos falar um pouco sobre Channels.

Channels em Phoenix, são a parte responsável por fazer o meio de campo, de uma maneira simples, entre enviar e receber mensagens.

["Com Channels, nem os remetentes nem receptores têm de ser processos Elixir. Eles podem ser qualquer coisa que podemos ensinar para se comunicar através de um canal — um cliente de JavaScript, um aplicativo iOS, outro aplicativo Phoenix, o nosso relógio."](http://www.phoenixframework.org/docs/channels)

Agora iremos criar o arquivo **room_channel.ex** em **web > channels**.

Iremos adicionar uma função para nos conectarmos ao channel. Ela recebe o tópico, a mensagem e o socket; retornando o status de **:ok** para indicar sucesso na conexão.

```
def join("rooms:lobby", message, socket) do
  {:ok, socket}
end
```

E outra função que irá lidar com as mensagens que chegam ao servidor. Ele será responsável por realizar o broadcast da mensagem a todos os participantes.

```
def handle_in("new:message", msg, socket) do
  broadcast! socket, "new:message", %{user: msg["user"], body: msg["body"]}
  {:noreply, socket}
end
```

O javascript já está configurado para escutar a mensagem do broadcast:

```
channel.on( “new:message”, msg => this.renderMessage(msg) )
```

Após configurar a renderização da mensagem na tela, o chat está finalizado.

O resultado final você encontra [aqui](http://papo-reto-demo-chat.herokuapp.com/).

Até mais!

[![Elixir e Phoenix - Papo Reto](/assets/images/placeholder-video-chat-phoenix.png)](https://www.youtube.com/watch?v=xcKDGZntkdg)

### Referências

* Phoenix Guides
* How to build an Elixir chat app with Phoenix — Chris McCord
* Building a chat application using Elixir and Phoenix
* Deploying a Phoenix app to Heroku
