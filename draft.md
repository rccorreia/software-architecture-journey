# Draft

Boas práticas de arquitetura:

- Definir Fitness Functions
  Os requisitos não funcionais, quando feito o levantamento na definição desses requisitos, serão falados sobre disponibilidade, estabilidade, segurança, escalabilidade, etc. Esses requisitos precisam ter um indicador para saber qual é a meta, qual o objetivo, e estes devem ser muito claramente definidos. Esses são os fitness functions e são eles que vão nos indicar que conseguimos ou não atingir o objetivo. Deve possuir também uma estratégia de coleta e medição (durante o desenvolvimento e para o ambiente de produção) para verificar se estamos atingindo o objetivo. Um exemplo de ferramente para fazer essa coleta pode ser o application insights

**_Acima ficou um ótimo resumo de cuidados iniciais com arquitetura_**

- Estratégia de métricas para extrair resultados de forma continuada para sempre comparar com as Fitness Functions definidas anteriormente em produção

- Saber que partes diferentes do sistema podem ter _Disponibilidade_ ou _Resiliência_ (Fitness Functions) diferentes

- Ter claro quais são os componentes e responsabilidade desses componentes e quais deles tem que estar disponíveis a maior parte do tempo

- Disponibilidade:
  Combinação de dois números: Tempo de resposta (comportamento ideal) e quanto tempo sua aplicação precisa estar respondendo. Exemplo, quero que em 99% do tempo a minha aplicação responda, em determinados lugares críticos, em menos de 300 milisegundos

      99% do tempo disponível, em 1 dia, significa 14 minutos. O que é um tempo razoável. Podemos falar sobre em que momento do dia essa instabilidade é tolerável. Talvez essa instabilidade durante o pico de pedidos, no caso de um e-commerce, não seja tolerável.

Redução de efeitos da não disponibilidade:
Load Balancer. Replicar o serviço na horizontal e adicionar um loadbalancer na frente para minimizar o risco de ter o serviço indisponível.

Caso tenha necessidades diferentes ao longo do dia, é possível ligar essa réplica em horário comercial, por exemplo, e desligá-la a noite

Porém, não adianta essa estratégia caso as duas instâncias do serviço consumirem uma única instância de banco de dados. Esse será o gargalo e o ponto de instabilidade.

A solução é semelhante, trabalhando com uma réplica do banco de dados. Um sendo master e o outro slave. (Só uma aceita gravação, a outra, ou demais, aceitam apenas leitura. Nesse caso se a master tiver problemas, pode ter indisponibilidade na gravação.) (Pode procurar outra estratégia de replicação de banco de dados em que seja possível trabalhar com master e master)

Pode trabalhar também com uma estrutura de caching acima do banco de dados, ou seja, entre ele e o serviço. Isso reduz consideravelmente as demandas por consulta ao banco de dados.
Ou então planejar e projetar uma estrutura de cache externo, como CDN (akamai) para responder por alguns recursos e/ou trabalhar com client side caching.

Aumentar a disponibilidade implica no aumento da complexidade de desenvolvimento do software e gestão no ambiente de operação.
E complexidade é custo.
Por isso é importante saber claramente qual a métrica de disponibilidade que precisamos para que não seja feito acima da necessidade, adicionando custo, nem abaixo da necessidade, deixando o sistema fora em momentos críticos.

Caso existam dois serviços que possuam 99% de disponibilidade, e um dos serviços dependa do outro, e essa dependência seja síncrona, esse conjunto de serviços possui 98% de disponibilidade, pois pode acontecer de estarem indisponíveis na sequência, sendo que o serviço que depende do outro pode ficar fora tanto quando ele estiver indisponível quanto quando o outro também estiver.
Existem outros riscos dessa dependência também, como:

- A rede (comunicação) entre eles pode também falhar.
- Essa comunicação pode não ser segura também e é necessário adicionar criptografia ou autenticação nessa comunicação, aumentando o tempo de resposta
- Podem apresentar problemas de latência
- Largura da Banda, podendo impactar na velocidade de transferência.
- Transporte das mensagens e dados. É necessário um processo de serialização e deserialização em strings, por exemplo. Prestar atenção e respeitar o garbage collector.

Quanto maior o número de serviços dependentes e síncronos, maior a perda de disponibilidade.
Para microsserviços isso deve ser muito bem pensado, pois pode causar um impacto muito relevante. A arquitetura pensada acima, para apenas um serviço, se mostra ineficaz caso tenha muitos serviços em cadeia.

Três vilões de performance:

- Memória
- IO de disco
- Garbage Collector

Microsserviços deve ser usado apenas em casos muito específicos. Deve-se ter excelentes argumentos para trabalhar com microsserviços pois isso traz uma categoria nova de desafios para resolver problemas complexos.

Utilizar mensageria, como RabbitMQ, para transformar a comunicação entre seus serviços assíncrona. Dessa forma, o fato de nunca chegarmos a 100% de disponibilidade é amenizado pois temos uma virtual taxa de resposta de 100%, pois caso ele não esteja disponível, ele vai responder logo que voltar ao ar. Isso ameniza o problema de disponibilidade.

Resumindo disponibilidade:
Não existe 100%. Deve-se verificar a disponibilidade necessária e então aplicar estratégias como: adicionar Load Balancer, fazer distribuição de carga horizontal, mecanismo de replicação de banco de dados, seja Master e Slave ou Master e Master, trabalhar com boa infraestrutura de caching, seja ele externo e/ou client side, tudo isso para atingir esse objetivo, lembrando que quanto maior a complexidade e a necessidade de disponibilidade, maior o custo.

Escalabilidade:
Para atingir uma escalabilidade alta, partindo do ponto inicial que seria um serviço monolítico, deve-se atentar para três pontos centrais:

- Eixo x (horizontal) - criar novas instâncias do sistema, como clones exatamente iguais, amparadas por um load balancer que divide a carga de requisições.
  Para isso deve ser possível criar mais de uma instância do projeto, o que em muitos casos necessita de alteração no sistema para possibilitar isso.

- Eixo y (vertical) - separar o serviço monolítico por função ou em serviços menores
  Isso ajuda caso você queira escalar apenas uma parte de seu sistema, por exemplo a área de pagamentos de um e-commerce ou outra área que esteja com uma demanda alta pontualmente.
  Para isso devem ser feitas também alterações no projeto, separando responsabilidades e modificando para que elas se conversem.

- Eixo z - separar o serviço para ter uma infraestrutura dedicada a atender demandas específicas, como grandes polos econômicos, distribuindo processamento para clusters especializados em cada área.

Tempo de carga do serviço tem que ser baixa. Pois caso ele caia, tem que subir rapidamente, e caso seja necessário replicá-la, deve ser algo razoavelmente rápido para atender a demanda.

A escalabilidade fica extremamente mais fácil com conteinerização e kubernetes ou qualquer outro bom serviço de orquestração.

Introduzir nas aplicações prática de health checking. Introduzir um end-point na aplicação com propósito verificar a sanidade da aplicação, verificando se o banco está respondendo bem, se a memória está ok, se o disco está respondendo em um tempo bom, etc.
Com base nessa métrica, caso não esteja bem, seria interessante derrubar e subir novamente. O kubernets consegue verificar esse endpoint e tomar a decisão.

Deve ser levado em conta o conceito de graceful shutdown, que seria encerrar a aplicação de forma suave, para que o kubernets possa encerrar e subir outra instância sem causar problemas ou perda de dados. Uma ação pode ser utilizar o CancellationToken e o ApplicationLifetime, onde a aplicação informa se está encerrando ou não. Isso não é necessário em toda aplicação, mas é interessante nos pontos de gargálo ou pontos críticos do sistema.

Resiliência:
É pensar em uma aplicação que tem condições de se recuperar caso um problema aconteça. É muito caro e virtualmente impossível fazer um sistema que não apresente algum problema, até porque podem ter coisas que não dependem desse sistema em si.

Outro conteúdo para verificar: https://www.elemarjr.com/en/2018/02/thinking-about-microservices-the-fiefdom-and-the-emissaries/

Fonte: Palestra do Elemar Júnior no youtube: Práticas e Padrões de Arquitetura para Disponibilidade e Resiliência
https://youtu.be/1VvXG42mufA
