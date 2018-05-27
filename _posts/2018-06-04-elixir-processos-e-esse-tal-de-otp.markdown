---
layout: post
title:  "Elixir, processos e esse tal de OTP"
author: "Amanda Sposito"
date:   2018-04-11 11:21:00 -0300
categories:
  - elixir
  - otp
  - pt-BR
---

Uma das grandes características da linguagem Elixir é a maneira como ela lida com concorrência, como isso traz benefícios no dia-a-dia e como isso agrega valor ao software final.

Quando vamos estudar sobre concorrência, uma das coisas que surgem durante esse aprendizado é uma tal sigla que escutamos muito, chamada OTP.

### OTP

A sigla significa *Open Telecom Platform*, mas ninguém a usa exclusivamente para telecom hoje em dia. No livro *Designing for Scalability with Erlang/OTP* dos autores Francesco Cesarini e Steve Vinoski, eles definem OTP como três componentes principais que interagem entre si: o primeiro sendo o próprio Erlang, o segundo um [conjunto de bibliotecas](http://erlang.org/doc/applications.html) disponíveis com a virtual machine e o terceiro um conjunto de princípios de design do sistemas.

Por isso talvez você já tenha ouvido falar no termo *OTP compliant*, que quer dizer que a aplicação segue os princípios de design de sistemas estabelecidos pelo Erlang/OTP.

Um dos itens desse conjunto de princípios consiste na utilização de uma arquitetura de processos para resolver os problemas da sua aplicação.

### Processos

Quando falamos de processo em Elixir, estamos nos referindo aos processos da máquina virtual do Erlang, e não aos processos do sistema operacional. Os processos da BEAM (máquina virtual do Erlang) são muito mais leves e baratos do que aqueles  do sistema operacional, e rodam em todos os *cores* disponíveis na máquina por serem mais leves, em média 2k, portanto, podemos criar milhares deles em nossa aplicação.

Eles são isolados uns dos outros e se comunicam através de mensagens, dessa maneira nos ajudam a dividir a carga de trabalho e a rodar coisas em paralelo.

Por Elixir ser uma linguagem funcional, uma de suas características é a imutabilidade, o que nos ajuda a deixar o estado explícito. Dessa maneira, quando separamos nosso código em tarefas independentes que podem rodar concorrentemente, não precisamos nos preocupar em controlar seu estado utilizando mecanismos complexos para garantir isso, coisas como *[mutex](https://en.wikipedia.org/wiki/Mutual_exclusion)* e a utilização de *[threads](https://en.wikipedia.org/wiki/Thread_(computing))* não são mais necessárias.

### Como processos funcionam em Elixir?

É muito comum associarmos processos e a troca de mensagens existentes entre eles com o [modelo de atores para concorrência](https://en.wikipedia.org/wiki/Actor_model). Isso acontece porque cada processo em Elixir é independente e totalmente isolado um do outro. Um processo pode guardar estado, mas isso não é compartilhado e a única maneira de compartilhar algo entre processos é através do envio de mensagens.

Cada processo possui uma *mailbox* que, como o nome sugere, é responsável por receber mensagens de outros processos. Vale dizer também que quando enviamos uma mensagem, tudo acontece assincronamente, dessa maneira não  bloqueamos o processamento aguardando por uma resposta.

Uma maneira de pensarmos nisso é imaginarmos processos como celulares que utilizam SMSs para troca de informações. Cada celular tem um local para armazenar essas mensagens até que elas sejam tratadas e isso acontece de maneira independente e isolada.

![sending-receiveing-messages](/images/elixir-process/sending-receiveing-messages.png)

Vale mencionar também que podemos vincular um processo ao outro. Isso é importante  porque é com base nisso que podemos identificar falhas e tomar alguma ação para lidar com elas.

Quando temos processos que monitoram outros processos, damos o nome de Supervisor. Podemos ter vários deles em nossa aplicação e quando temos mais de um *Supervisor* monitorando processos é o que chamamos de Árvore de Supervisão.

![supervision-tree](/images/elixir-process/supervision-tree.png)

Isso é muito importante porque é através disso que obtemos a tolerância a falhas. Quando temos um problema com o nosso código, o que não queremos é que isso afete o usuário final. Através do monitoramento de processos podemos identificar quando algo inesperado acontece e assim tomar uma ação sobre isso, finalizando o processo com problema e iniciando-o novamente.  Assim, o processo volta ao seu estado inicial nos dando tempo para agir e resolver o problema que causou o erro.

### Como isso funciona para concorrência?

Quando a BEAM inicia, ela também inicia uma thread chamada *Scheduler* que é responsável por rodar cada processo concorrentemente na CPU.

Para poder obter toda vantagem do hardware, a BEAM inicia um *Scheduler* para cada core disponível, ou seja, um computador que possui quatro cores, vai ter quatro *schedulers*, cada um rodando vários processos concorrentemente.

![scheduler](/images/elixir-process/scheduler.png)

Processos são a base para o modelo de concorrência que usamos em Elixir. Muitas das funcionalidades que precisamos ao utilizá-los possuem alguma abstração para nos ajudar, portanto, dessa maneira não precisamos nos preocupar com os detalhes de implementação de funções mais primitivas, como *spawn, send e receive*.

Quando utilizamos essas funções primitivas, precisamos nos preocupar com muito mais detalhes que são passíveis de erros para conseguirmos atingir nosso objetivo e deixar nosso código OTP compliant. Normalmente, os desenvolvedores(as) não usam essas funções, ao invés disso, usam abstrações como *Task, GenServer e Agent*. Mas isso já é assunto para outro post.

