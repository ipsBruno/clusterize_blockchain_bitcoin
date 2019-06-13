# Bitcoin Clusterize Addresses

## Instruções

#### 1 - Instale PHP7 e MySQL na sua maquina


#### 2 - Adicione as tabelas no seu banco de dados

###### Tabela: adresses_table
###### Colunas: address (varchar 64) e wallet (varchar 12)
###### Index: address, varchar



#### 3 - Configure a conexão ao banco de dados

application/config/database.php


#### 4 - Acesse com seu navegador

http://yourwebsite/index.php?/Welcome/index_block

Deixe rodando por alguns dias e acompanhe os logs em application/logs


    ERROR - 2019-06-13 11:54:15 --> Indexed Blocks [100] In [72] seconds
    ERROR - 2019-06-13 11:55:32 --> Indexed Blocks [100] In [75] seconds
    ERROR - 2019-06-13 11:56:48 --> Indexed Blocks [100] In [74] seconds
    ERROR - 2019-06-13 11:59:07 --> Indexed Blocks [100] In [77] seconds
    ERROR - 2019-06-13 12:00:27 --> Indexed Blocks [100] In [78] seconds
    ERROR - 2019-06-13 12:01:46 --> Indexed Blocks [100] In [78] seconds






#####

######

## Sobre o projeto

Esse projeto é uma versão aprimorada do meu antigo projeto pra clusterização de endereços

https://github.com/ipsBruno/tracking-wallet-bitcoin

**Porque fazer isso?**

A clusterizacao de endereços se mostra extremamente eficaz pra rastrear Bitcoins e marcar eles. Podendo desde consultar a solvência de uma exchange, até rastrear Bitcoins roubados, combater terrorismo, e crimes cibernéticos como pedofilia, etc

Existem duas formas de clusterizacao de endereço no Bitcoin. Uma é através da verificação de endereços com o mesmo input dentro das transações e a outra é através do change address. Nesse código estamos usando inputs (método mais confiável) pra validar as wallets
 
**Sobre**

Meu objetivo foi otimizar mais o código pra conseguir anexar todos blocos da blockchain dentro do banco de dados (mysql), uma vez que o meu código antigo (disponível aqui no github)  levava mais de 10 minutos por bloco o que significaria que nunca conseguiríamos nos manter atualizado perante a blockchain, afinal a cada 10 minutos é emitido um novo bloco na rede do Bitcoin


Tentei refatorar o código usando Neo4J (banco de dados de grafos indicado pra isso) pra montar essas clusterizações de forma mais correta possível, mas levou meses pra indexar toda blockchain no banco e ainda ficava demorada as querys de clusterizacao. Resumindo: Clusterizar os endereços de Bitcoin é um problema de grafos de complexabilidade algorítmica alta. Cada milhares de nodes (endereços) precisam ser ligados a outro node e/ou vários nodes (milhares de outros endereços)

Não estou utilizando change address nas clusterizacoes pela dificuldade/lentidão em verificação desses endereços. Apesar de tudo é um ponto que pretendo adicionar no futuro.


![Reactor, Blockchain Analysis from Chainalysis](https://www.chainalysis.com/img/reactor-2.png)

 Na imagem abaixo há um gráfico com um endereço maior em roxo, nesse 'case' os fundos de um usuário roubado haviam passado por alguns mixer até chegar neste endereço maior. A ideia da ferramenta é mostrar qual probabilidade de um endereço roubado ter chegado até outro endereço final

![](https://i.imgur.com/0lgTFyu.png)

Um adendo importante, com Neo4J dá pra cruzar dois endereços e verificar também se eles tem alguma ligação em comum (através da análise grafos [outputs em séries])

Por isso aconselho muito o uso dessa ferramenta em conjunto este aqui:
https://github.com/in3rsha/bitcoin-to-neo4j/blob/master/docs/cypher.md

![](https://github.com/in3rsha/bitcoin-to-neo4j/raw/master/docs/images/path_output.png)

![](https://github.com/in3rsha/bitcoin-to-neo4j/raw/master/docs/images/path_address.png)
```
MATCH (start :output {index:'$txid:vout'}), (end :output {index:'$txid:out'})
MATCH path=shortestPath( (start)-[:in|:out*]-(end) )
RETURN path
```

Note que assim dá pra fazer um estudo muito mais aprofundado na blockchain. 



**Como eu pude otimizar o código antigo?**
  
 São milhões de dados a serem cruzados, e como já expliquei lá em cima: cada milhares de inputs devem ser relacionado com outros milhares de inputs.

Bem, isso é BIG DATA e um problema de grafos chato de resolver, logo tive que tomar diversas ações pra otimizar a inserção dos dados e vou explica-las logo abaixo

0. - Tenha em mente que o bloco inicial é 580.000 ($finalBlock) e vai descendo de trás pra frente até chegar ao bloco 0. Um arquivo chamado ~~ultimo.log~~ é salvo com o número do bloco que paramos.

1. - Para aproveitar melhor a memória RAM do computador adicionei a funcionalidade de baixar dezenas/centenas de blocos por requisição e processar eles antes de enviar pro banco de dados
 
 2. -  Todos blocos tem seus milhões de endereços parseados direto na memória e encapsulados para serem adicionados em uma única query ou seja eu não faço 2 milhões de querys no banco em cada 100 blocos por input, mas sim apenas o suficiente com where_in/load file encapsuladas num único comando.

3. - Existem duas formas de clusterizacao de endereço no CÓDIGO. Uma é através da verificação de endereços com o mesmo input DENTRO do banco de dados e a outra é a verificação direto na MEMÓRIA, o que eu chamo de pré-clusterização antes de enviar as informações pro banco de dados

4. - Colocando aproximadamente 100 blocos por requisição o servidor ocupou até 6 GB de RAM. Anteriormente baixava apenas 1 bloco por vez. Agora são 100 blocos por vez (ou configurável ). Atualmente 100 blocos representa 60 MB por bloco processado

5. - Todas array declaradas ao longo do código estão sendo limpas logo após o uso pra economizar a RAM liberando espaço.

6. -  As requisições na blockchain.info são feitas em múltiplos threads. Se você não tiver uma API Key ficará limitado em 1 requisição por segundo, aconselho fazer o pedido de uma API pra eles.

7. -  Quando fiz a leitura dos inputs no meu full node notei a demora em processar eles. Por isso estou usando a API da Blockchain.info. Entendeu? Eles já retornam dos blocos com todas transações e seus respectivos inputs addresses. 


8. - Segundo meus testes com conexão de 1 GBPS uplink, HD normal (não-ssd) adicionava aproximadamente 1 bloco por segundo no banco de dados. Esse valor tende a aumentar conforme número de índices no banco de dados, mas acredito que em 7 dias com essas especificações dá pra processar toda blockchain no estágio que estamos hoje (580 mil blocos). O que é um tempo muito bom. O NEO4J por exemplo leva meses, e olha que é um banco dedicado pra processamento de grafos.

9. - Apesar da otimização não é aconselhável abrir várias guias do seu browser na tentativa de baixar e indexar os blocos mais rápidos. Não trabalhei dessa forma pensando em concorrência, os dados são encapsulados e enviados para o banco de uma forma que não pensei em concorrência. Por tanto tenha em mente isso.  Apenas uma guia no browser, e apenas um processo do servidor pra indexar os dados (sem multi-tasking)




### Como os dados ficam armazenados no banco?

Tudo ficará na tabela *addresses_table*.  A coluna wallet do banco de dados é referente ao código único da wallet do endereço. Podem existir mais de um endereço na mesma wallet, isso é a nossa clusterizacao funcionando. Ainda não defini qual wallet id e de qual exchange por exemplo. Mas para definir isso é só pegar um endereço de depósito já movimentado de uma exchange e pesquisar no banco de dados este endereço, o wallet desse endereço é a wallet da respectiva corretora. Agora buscando uma consulta pela wallet você terá todos endereços da corretora. 

**Veja o exemplo abaixo:**

    select * from addresses_table  where addr='addr_corretora'
    select * from addresses_table where wallet='wallet_id'

 
 ### Quão rastreáveis são os Bitcoins?

Bitcoins são rastreáveis. Sempre são. Independente de você usar mixer ou não sempre dará pra chegar nas carteiras de destino desses Bitcoins. A diferença é que com mixer podemos chegar a centenas, talvez milhares de endereços de saída e com uma boa análise tem como estimar a probabilidade e por fim descobrir qual endereço foi a destinação destes valores de forma precisa; principalmente se forem valores altos. 

Recentemente tive um caso bem sucedido de um cliente que teve sua carteira hackeada e mais de 300 BTC foram parar em uma corretora estrangeira. Com os relatórios em mãos e um boletim de ocorrência ou requisição judicial as chances de descobrir informações sobre o roubo são altas: mas por experiência própria as chances de ter os respectivos saldos de volta são mínimas

Hoje há aproximadamente 500 milhões de endereços a serem clusterizados até agora. Desses 500 milhões, 40% não podem ser atribuídos a uma única wallet. Ou seja 60% até podem ser rastreáveis

O  número aproximado de blocos hoje é de 580 mil. Cada novo bloco surge a 10 minutos. E a média de tamanho por casa requisição dele com os inputs incluso na API da blockchain.info e de 10 megabytes. Portanto tenha em mente que 580.000 * 10 megabytes serão transferidos na sua Internet.

*Apenas um disclaimer*: Por propósitos libertários, jamais pensei em fazer essa ferramenta para uso governamental pra controle da moeda ou cobrança de impostos. Assim como existe meios de rastrear existem meios de dar bypass na clusterizacao de inputs e posso explicar muito em breve. Na realidade basta entender como funciona o rastreio que você pode facilmente proteger sua privacidade dentro da blockchain.

### Porque MySQL e PHP?

O backend foi escrito inicialmente em NodeJS, mas por problemas com conexão ao banco e eventuais crashs resolvi migrar pra PHP que apesar de ser  síncrono atendeu bem o problema.

O banco utilizado foi o MySql, simplesmente porque foi o banco que atendeu minhas consultas e do qual eu julgo ter mais experiência. Tive muitos problemas com MongoDB, Elasticsearch e LevelDB que não atendeu minhas funções por ser only key-pair

MyISAM mostrou ser mais rápido em inserções que InnoDB. Também notei que a utilização de LOAD FILE e WHERE IN me ajudou a otimizar o número de querys executadas por segundo. Apesar disso tudo não fiz nenhuma configuração específica no **my.cnf**


Uma versão melhorada do mesmo algoritmo poderia ser feita em C++. O desempenho poderia ser absurdamente maior, contudo considero 7 dias (sem *ssd* ainda) um prazo bem bom para clusterizar toda blockchain. As ferramentas aí disponíveis hoje levam meses e até onde eu sei existe somente o site walletexplorer.com que faz essa funcionalidade e se mantem online e aberto ao público.

### Configurações Recomendas

    8 GB RAM
    512 GB HD
    Conexão com Internet de pelo menos 10 MB /seg



### Agradecimentos

 - Carlos Heitor Lain (Pagcripto)  Me auxiliou em servidores de testes e
   nas configurações deles 
   
  -  Rodrigo Souza (Bitcambio) Me ajudou com algumas dicas nos bancos de
   dados que deveriam ser usados como opção
   
  - Leandro Trindade (Access) Me auxiliou numa função alternativa pra
   clusterizar os endereços dentro da memória

### Contato

Meu contato é email@brunodasilva.com.

Tenho alguns problemas ao escrever textos, se achar alguma frase sem sentido ou tenha ficado confuso entre em contato comigo que posso explicar melhor.

Estou buscando patrocínios pra manter uma ferramenta similar a **chainalysis.com** on-line, gratuita e aberta ao público. Pra isso precisarei gastar um valor alto em servidores, portanto se quiser me ajudar nessa empreitada aceito doações em:




**1Kgijykax9RHuuaR84vi4W5eGKLc85DX4V**


### Links Legais
http://randomwalker.info/teaching/spring-2014-privacy-technologies/btctrackr.pdf
https://github.com/Iwontbecreative/Bitcoin-adress-clustering
https://bitfury.com/content/downloads/clustering_whitepaper.pdf
https://github.com/in3rsha/bitcoin-to-neo4j
https://medium.com/tokenanalyst/how-to-load-bitcoin-into-neo4j-in-one-day-b555219ed9d2
https://neo4j.com/blog/import-bitcoin-blockchain-neo4j/
https://www.walletexplorer.com


