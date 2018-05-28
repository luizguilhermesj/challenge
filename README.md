# challenge
[diagram]: https://github.com/luizguilhermesj/challenge/blob/master/Solu%C3%A7%C3%A3o%201.png
![Diagrama da arquitetura][diagram]

Comecei normalizando o acesso às 3 bases, acredito que seja interessante criar 3 serviços distintos, um para cada uma das bases.  
Imagino que em diversas situações só é necessário acessar um ou dois dos serviços, isso nos provê uma arquitetura que permite escalar a infraestrutura de cada um deles de forma isolada.
Além disso, se a Base A é em arquivo de texto, a Base B é em SQL e a Base C é em NoSQL, fica transparente para quem consome os serviços, pois eles são responsáveis por cuidar da forma em que esses dados são acessados, compo por exemplo decriptar os dados da Base A, quem devem estar encriptados, uma vez que são os mais sensíveis.  

Todos esses serviços podem ficar em uma rede privada, para não permitir acesso que não passe pelo nosso serviço de autenticação.  

Coloquei um API Proxy interno para gerenciar essas requests e um serviço de dados na rede pública, que é o único ponto de contato, o único com acesso à rede privada e ele só sabe da existência do proxy, nada além disso.  

Também coloquei um serviço de identidade para que possamos autenticar os serviços que forem utilizar o serviço de dados. Ele seria o responsável por garantir quem tem acesso e à qual dado tem acesso.

### Rede privada
A ideia é que toda essa infraestrutura não esteja conectada à internet, ela deve ser separada e ter as medidas de seguraça adequadas para impedir acessos externos, com exceção do acesso do Serviço de Dados e somente ao serviço de proxy.

### Rede pública
Aqui já podemos receber requisições externas, de serviços de outros times, de clientes ou parceiros.

### Serviço de acesso Base A
Como dito, ele deve saber como lidar com a Base A, a tecnologia pode variar, por ser uma base de alta segurança e não ter necessidade de performance, provavelmente escolheria a linguagem mais fácil de integrar com a base.

### Serviço de acesso Base B
Este já tem uma necessidade de acesso mais rápido, portanto optaria por uma linguagem que consiga escalar consumindo menos recursos, como NodeJS, Python ou Golang

### ETL Base B
Dado que a Base B precisa ser utilizada por sistemas de ML, não acredito que utilizar o Serviço de acesso Base B seja a melhor solução. O ideal é extrair os dados e transformá-los no formato mais otimizado para os sistemas de ML

### Serviço de acesso Base C
Este precisa ser muito rápido ele pode ser similiar ao Serviço da base B, pensando em escalabilidade, porém como a informação não é crítica, penso que podemos replica-la em uma camada de cache, que pode ser apenas um banco de dados com performance otimizada ou algum serviço de indexação como Solr, ElasticSearch, etc.

### Serviço de Identidade
Responsável por identificar os serviços autorizados à utilizar o Serviço de Dados e quais são suas permissões, como por exemplo um serviço que tem acesso ao recurso `/debts`, mas não ao `/score`.

### Serviço de Dados
Este é o principal ponto de entrada do sistema. É nele que consequimos fazer requisições via uma API para obter as informações que precisamos. Portanto poderíamos ter estes endpoints, por exemplo:

```
/consumer/:cpf
/consumer/:cpf/debts
/consumer/:cpf/score
/consumer/:cpf/income
/consumer/:cpf/financial-movements
...
```

Provavelmente este serviço também manteria um log de todas as requests, ou acionaria outro serviço para isso, para podermos saber quais requests foram feitas e  também já imaginando tarifação das mesmas.

O que ele faz ao receber essas requests é basicamente chamar os serviços das bases, pois ele detém o conhecimento de onde buscar cada informação, ex.:

`/consumer/:cpf/debts`
 - faz uma request para http://proxy.local/a/debts/:cpf
 - responde um JSON com as dívidas do CPF informado

Ele também sabe juntar informações de mais de uma base para uma pergunta mais genérica, ex.:

`/consumer/:cpf/full-report`
 - faz uma request para http://proxy.local/a/report/:cpf
 - faz uma request para http://proxy.local/b/report/:cpf
 - faz uma request para http://proxy.local/c/report/:cpf
 - concatena as informações recebidas nas 3 requests
 - responde um JSON com todos os dados disponíveis para o CPF informado

 Este serviço também precisa ser escalável, então deve seguir o mesmo princípio dos serviços de acesso B e C, particularmente eu optaria por NodeJS, só porque eu gosto mais, porém acho que Golang resolveria esse problema de forma exemplar.  

 Outro ponto é que todas as requests dele precisam estar autenticadas pelo Serviço de Identidade, para que ele saiba se deve ou não responder as requests que recebe.  


# Infraestrutura / Escalabilidade / Disponibilidade
Acredito que uma arquitetura saudável para esses serviços seja um simples pool de Docker Containers para cada um deles. Gerenciados por um Kubernetes ou um DCOS, não devemos ter indisponibilidade e poderemos escalar tranquilamente. Se formos usar a AWS, dá para usar o ECS, mas não gosto muito da ideia, isso nos deixa muito dependentes da AWS.  
Para proxy eu já usei o Kong, funcionou muito bem, mas tem outras soluções que podemos usar, inclusive da própria AWS, o AWS API Gateway. Só usei ele para serviços do Lambda.  
Podemos também seguir na abordagem de serverless, usando o Lambda para cada um dos serviços. É interessante, tenho amigos que trabalham assim e dizem que funciona muito bem. Eu não tenho clareza dos Prós e Cons de um arquitetura serverless.

# Dados Armazenados
Eu não sei dizer exatamente, mas eu gosto de armazenar TUDO.  
Portanto, todas as requests feitas, quem está fazendo, que horas, etc.  
Com esses dados, nós podemos gerar outros, como por exemplo qual a demanda que temos em consultas simples de débitos e em qual período elas são mais comuns. Poderíamos ter preços diferenciados para incentivar a utilização do serviço em outros horários.  
Podemos identificar que clientes da região X do país utilizam muito o serviço Y que simplesmente é o serviço Z e o W juntos, poderíamos fazer uma pesquisa para entender o cenário diferenciado deles.  
Também podemos identificar que um cpf está sendo consultado por muitas empresas de crédito, podemos alertá-lo sobre isso.  
Podemos gerar uma notificação para o consumidor toda vez que seus dados são atualizados e quem foi o responsável.  
Tem muitas possibildiades. Dados tem muitas informações.  

Não consegui pensar em algum código útil, mas podemos bater um papo e eu faço algum código se quiserem =]
