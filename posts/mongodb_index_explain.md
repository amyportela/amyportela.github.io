## MongoDB - Criar índice e avaliar com Explain("executionStats")


Tenho trabalhado com melhoria de performance no mongodb e para me organizar achei interessante documentar esses trabalho que tenho tido com índices, acompanhamento de queries e também parâmetros de configuração.

Neste primeiro documento quero exemplificar o processo de criação de um índice do tipo *single field* no mongodb e identificar se o índice é funcional para a query.

Resumidamente um índice é uma estrutura que, através de ponteiros, é associada a uma coleção de dados em campos específicos para acelerar o tempo de acesso a esses dados.
Atualmente [o MongoDB trabalha com mais de um tipo de índice](https://docs.mongodb.com/manual/indexes/).
Portanto é importante conhecer a estrutura do seu documento e da sua query para entender qual o melhor tipo de índice a ser aplicado.

Iniciando com um exemplo na minha collection ***mycollection*** tenho **1.500.000** de documentos, contém apenas o campo _id do documento (default) e apenas um campo **“x”** que existe em todos os documentos

```
> db.mycollection.count()
1500000

> db.mycollection.find()
{ "_id" : ObjectId("60822de25600d9024ad6eb40"), "x" : 1 }
{ "_id" : ObjectId("60822de25600d9024ad6eb41"), "x" : 2 }
{ "_id" : ObjectId("60822de25600d9024ad6eb42"), "x" : 3 }
{ "_id" : ObjectId("60822de25600d9024ad6eb43"), "x" : 4 }
{ "_id" : ObjectId("60822de25600d9024ad6eb44"), "x" : 5 }


> db.mycollection.count({"x":{$exists:true}})
1500000
``` 

Também por default as collections com documentos possuem o índice do _id do tipo *unique* para que não exista o mesmo numero de ObjectId na collection.

```


> db.mycollection.getIndexes()
[ { "v" : 2, "key" : { "_id" : 1 }, "name" : "_id_" } ]

```

Ao executar a pesquisa por um único registro, a média de duração da minha query é de 600 milissegundos e todos os documentos da collection (1.500.000) foram percorridos.

Antes de obsevar o retorno, aqui são os pontos que acho importante acompanhar para este exemplo:  

`queryPlanner.winningPlan` detalha o plano selecionado pelo otimizador de consulta exibindo o estágio. Se o plano utilizado pela query estiver usando um índice, irá aparecer aqui.
O que não é o caso do exemplo abaixo onde vemos `COLLSCAN`, ou seja, nenhum ídice está sendo utilizado.

`executeStats.executionTimeMillis` é o tempo geral de execução da query. Este tempo não é apenas o tempo em que a consulta é executada, também inclui o tempo que leva para gerar / selecionar o plano de execução.

`executeStats.totalDocsExamined` é a quantidade de documentos que foi percorrida durante a execução da query.

```
> db.mycollection.find({"x":32})
{ "_id" : ObjectId("60822de25600d9024ad6eb5f"), "x" : 32 }
>
> db.mycollection.find({"x":32}).explain("executionStats")
{
        "queryPlanner" : {
                "plannerVersion" : 1,
                "namespace" : "mycollection.mycollection",
                "indexFilterSet" : false,
                "parsedQuery" : {
                        "x" : {
                                "$eq" : 32
                        }
                },
                "winningPlan" : {
                        "stage" : "COLLSCAN",
                        "filter" : {
                                "x" : {
                                        "$eq" : 32
                                }
                        },
                        "direction" : "forward"
                },
                "rejectedPlans" : [ ]
        },
        "executionStats" : {
                "executionSuccess" : true,
                "nReturned" : 1,
                "executionTimeMillis" : 662,
                "totalKeysExamined" : 0,
                "totalDocsExamined" : 1500000,
                "executionStages" : {
                        "stage" : "COLLSCAN",
                        "filter" : {
                                "x" : {
                                        "$eq" : 32
                                }
                        },
                        "nReturned" : 1,
                        "executionTimeMillisEstimate" : 50,
                        "works" : 1500002,
                        "advanced" : 1,
                        "needTime" : 1500000,
                        "needYield" : 0,
                        "saveState" : 1500,
                        "restoreState" : 1500,
                        "isEOF" : 1,
                        "direction" : "forward",
                        "docsExamined" : 1500000
                }
        "ok" : 1
}
```
###### *Para ver mais detalhes da query no ambiente usei*  `explain("executionStats")`. *Existe mais opções de verbosity para o explain(), pode ser visto [aqui](https://docs.mongodb.com/manual/reference/command/explain/).*

Em outras situações de avaliação de performance, pode ser importante entender melhor outros pontos desse output, mais detalhes de cada campo pode ser visto [aqui](https://docs.mongodb.com/manual/reference/explain-results/)

Para melhorar a performance da minha query no ambiente vou criar o índice usando a key que uso para consulta, no caso o campo ***"x"***

```
> db.mycollection.createIndex({"x":1})
{
        "createdCollectionAutomatically" : false,
        "numIndexesBefore" : 1,
        "numIndexesAfter" : 2,
        "ok" : 1
}
> db.mycollection.getIndexes()
[
        {
                "v" : 2,
                "key" : {
                        "_id" : 1
                },
                "name" : "_id_"
        },
        {
                "v" : 2,
                "key" : {
                        "x" : 1
                },
                "name" : "x_1"
        }
]
```
Podemos observar no retorno abaixo que o `queryPlanner.winningPlan` agora possui mais detalhes, no caso  `queryPlanner.winningPlan.inputStage.indexName`, que é o nome do índice que criei no passo anterior.

O tempo de execução da query (`executeStats.executionTimeMillis`) que era na média de 600 milissegundos diminuiu para 0 e a quantidade de documentos que ele percorreu (`executeStats.totalDocsExamined`) que era 1.500.000 reduziu para 1.

```
> db.mycollection.find({"x":32})
{ "_id" : ObjectId("60822de25600d9024ad6eb5f"), "x" : 32 }

> db.mycollection.find({"x":32}).explain("executionStats")
{
        "queryPlanner" : {
                "plannerVersion" : 1,
                "namespace" : "mycollection.mycollection",
                "indexFilterSet" : false,
                "parsedQuery" : {
                        "x" : {
                                "$eq" : 32
                        }
                },
                "winningPlan" : {
                        "stage" : "FETCH",
                        "inputStage" : {
                                "stage" : "IXSCAN",
                                "keyPattern" : {
                                        "x" : 1
                                },
                                "indexName" : "x_1",
                                "isMultiKey" : false,
                                "multiKeyPaths" : {
                                        "x" : [ ]
                                },
                                "isUnique" : false,
                                "isSparse" : false,
                                "isPartial" : false,
                                "indexVersion" : 2,
                                "direction" : "forward",
                                "indexBounds" : {
                                        "x" : [
                                                "[32.0, 32.0]"
                                        ]
                                }
                        }
                },
                "rejectedPlans" : [ ]
        },
        "executionStats" : {
                "executionSuccess" : true,
                "nReturned" : 1,
                "executionTimeMillis" : 0,
                "totalKeysExamined" : 1,
                "totalDocsExamined" : 1,
                "executionStages" : {
                        "stage" : "FETCH",
                        "nReturned" : 1,
                        "executionTimeMillisEstimate" : 0,
                        "works" : 2,
                        "advanced" : 1,
                        "needTime" : 0,
                        "needYield" : 0,
                        "saveState" : 0,
                        "restoreState" : 0,
                        "isEOF" : 1,
                        "docsExamined" : 1,
                        "alreadyHasObj" : 0,
                        "inputStage" : {
                                "stage" : "IXSCAN",
                                "nReturned" : 1,
                                "executionTimeMillisEstimate" : 0,
                                "works" : 2,
                                "advanced" : 1,
                                "needTime" : 0,
                                "needYield" : 0,
                                "saveState" : 0,
                                "restoreState" : 0,
                                "isEOF" : 1,
                                "keyPattern" : {
                                        "x" : 1
                                },
                                "indexName" : "x_1",
                                "isMultiKey" : false,
                                "multiKeyPaths" : {
                                        "x" : [ ]
                                },
                                "isUnique" : false,
                                "isSparse" : false,
                                "isPartial" : false,
                                "indexVersion" : 2,
                                "direction" : "forward",
                                "indexBounds" : {
                                        "x" : [
                                                "[32.0, 32.0]"
                                        ]
                                },
                                "keysExamined" : 1,
                                "seeks" : 1,
                                "dupsTested" : 0,
                                "dupsDropped" : 0
                        }
                }
        },
        "ok" : 1
}

```

Acho importante frisar que este exemplo foi feito apenas localmente, portanto o tempo de execução para uma aplicação pode estar diferente.
