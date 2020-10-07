# Elastic-Shakespeare

<img src="calculon.webp" height="350">

En este ejercicio vamos a analizar un dataset de obras de Shakespeare y analizar un poco los datos.

Nota: Vamos a utilizar la versión 7.9.2 de Elasticsearch y de Kibana

La primera parte de este ejercicio es descargar e importar los datos de sus obras en modo _bulk_ (o con un 'bulk-request'):

Queda en cada uno la forma de consultar la DB, ya sea con curl, la consola de Kibana o algún driver de algún lenguaje de programación. Pero tengan en cuenta que con curl pueden llegar a haber problemas a la hora de hacer queries por los \" \".

1. Descargar los datos: https://raw.githubusercontent.com/glenacota/elastic-training-repo/master/datasets-to-go/shakespeare_dataset.json

2. Hacer un bulk-request:

```
curl -XPOST "elasticsearch:9200/shakespeare/_doc/_bulk?pretty" \
-H "Content-Type: application/json" --data-binary @path/to/shakespeare_dataset.json
```

Nota: No olvidarse del '@' en el comando

Ahora que ya tenemos los datos, podemos empezar a jugar.

### Testing

Primero vamos a testear que haya cargado todos los datos, para eso vamos a contar cuantos registros hay y vamos a preguntar por 3 registros, el primero, alguno en el medio y el último. Si tenemos todos estos, vamos a asumir que tenemos todos los datos

Nota: Los índices arrancan en 0.

1. Hacer un count de todos los índices insertados

`curl -X GET localhost:9200/shakespeare/_count`

y deberíamos obtener:

```
{"count":111396,"_shards":{"total":1,"successful":1,"skipped":0,"failed":0}}
```

2. Obtener el primer elemento

`curl -x GET localhost:9200/shakespeare/_doc/0?pretty`

Deberíamos obtener lo siguiente:

```
{
  "_index" : "shakespeare",
  "_type" : "_doc",
  "_id" : "0",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "type" : "act",
    "line_id" : 1,
    "play_name" : "Henry IV",
    "speech_number" : "",
    "line_number" : "",
    "speaker" : "",
    "text_entry" : "ACT I"
  }
}
```

Ahora vamos a pedir el elemento 73330 y el último elemento, 111395. En orden, deberíamos tener lo siguiente:

`curl -X GET localhost:9200/shakespeare/_doc/73330?pretty`

```
{
  "_index" : "shakespeare",
  "_type" : "_doc",
  "_id" : "73330",
  "_version" : 1,
  "_seq_no" : 73330,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "type" : "line",
    "line_id" : 73331,
    "play_name" : "Othello",
    "speech_number" : 63,
    "line_number" : "2.3.176",
    "speaker" : "OTHELLO",
    "text_entry" : "Honest Iago, that lookst dead with grieving,"
  }
}
```

`curl -X GET localhost:9200/shakespeare/_doc/111395?pretty`

```
{
  "_index" : "shakespeare",
  "_type" : "_doc",
  "_id" : "111395",
  "_version" : 1,
  "_seq_no" : 111395,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "type" : "line",
    "line_id" : 111396,
    "play_name" : "A Winters Tale",
    "speech_number" : 38,
    "line_number" : "",
    "speaker" : "LEONTES",
    "text_entry" : "Exeunt"
  }
}
```

Listo. Si estas 3 consultas funcionaron bien, asumimos que tenemos todo el dataset cargado.

## Oh Romeo...

Nunca leí Romeo y Julieta pero podríamos debatir que es una de las obras más conocidas por Shakespeare (Aunque él no escribió la historia sino que hizo la adaptación a teatro, de lo que se dice que fue una historia verdadera).

1. Ver si está la obra

Primero que nada, revisemos si está la obra en nuestra base.

`curl -XGET "http://localhost:9200 /shakespeare/_search?q=play_name="Romeo and Juliet""`

Deberíamos recibir algunas entradas, por lo que confirmamos que la obra está presente en la DB.

2. "Oh Romeo.."

Me pregunto cuantas veces aparece está esta frase en la obra, sin importar quien lo diga.

Recordemos que el atributo "play_name" nos indica la obra y el atributo "text_entry" nos indica el díalogo de ese personaje.

Como Elasticsearch nos ofrece también búsquedas simulares, podemos filtrar algunas respuestas indeseadas incrementando el score, para esto podríamos utilizar un score mínimo de 20.

```
curl -XGET "http://elasticsearch:9200/_search" -H 'Content-Type: application/json' -d'{  "min_score": 20,    "query": {        "bool": {            "must": [{                "match": { "play_name": "Romeo and Juliet"}            }, {                "match": { "text_entry": "O Romeo"}            }]        }    }}'
```

Obtenemos como respuesta

```
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 6,
    "successful" : 6,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 23.532429,
    "hits" : [
      {
        "_index" : "shakespeare",
        "_type" : "_doc",
        "_id" : "86168",
        "_score" : 23.532429,
        "_source" : {
          "type" : "line",
          "line_id" : 86169,
          "play_name" : "Romeo and Juliet",
          "speech_number" : 4,
          "line_number" : "2.2.35",
          "speaker" : "JULIET",
          "text_entry" : "O Romeo, Romeo! wherefore art thou Romeo?"
        }
      },
      {
        "_index" : "shakespeare",
        "_type" : "_doc",
        "_id" : "86918",
        "_score" : 22.828098,
        "_source" : {
          "type" : "line",
          "line_id" : 86919,
          "play_name" : "Romeo and Juliet",
          "speech_number" : 40,
          "line_number" : "3.1.117",
          "speaker" : "BENVOLIO",
          "text_entry" : "O Romeo, Romeo, brave Mercutios dead!"
        }
      },
      {
        "_index" : "shakespeare",
        "_type" : "_doc",
        "_id" : "87057",
        "_score" : 22.828098,
        "_source" : {
          "type" : "line",
          "line_id" : 87058,
          "play_name" : "Romeo and Juliet",
          "speech_number" : 6,
          "line_number" : "3.2.43",
          "speaker" : "Nurse",
          "text_entry" : "Though heaven cannot: O Romeo, Romeo!"
        }
      }
    ]
  }
}
```

## Agregaciones

Nota: es importante aclarar que para las funciones de agregación sobre strings, se tiene que utilizar el 'keyword' en vez del valor del atributo. Por ejemplo, en vez de usar "play_name", tenemos que usar "play_name.keyword". Además, podemos usar el atributo "size" en "0" en la query para hacer un análogo a "DISTINCT" en SQL.

1. Primero vamos a ver cuantas obras tenemos:

Para esto podemos utilizar el atributo "cardinality"

```
curl -XGET "http://elasticsearch:9200/shakespeare/_search" -H 'Content-Type: application/json' -d'{  "size": 0,  "aggs" : {    "plays": { "cardinality": {"field": "play_name.keyword"} }  }}'
```

2. Obtener la misma lista pero en orden

Investigar sobre el campo "order" para la función de agregación. Si utilizan "terms", recuerden que por default hay un máximo de elementos que traen los queries, podemos extender el límite utilizando "size: 1000" dentro del elemento "terms".

```
curl -XGET "http://elasticsearch:9200/_search" -H 'Content-Type: application/json' -d'{  "size": 0,  "aggs": {    "plays": {      "terms": {        "field": "play_name.keyword",        "order": { "_key": "asc" },        "size": 1000      }    }  }}'
```

3. Obtener las obras con al menos 3900 líneas

Investigar el término "min_doc_counts" para la función de agregación. Recuerden el comentario sobre "size" del inciso anterior.

```
curl -XGET "http://elasticsearch:9200/_search" -H 'Content-Type: application/json' -d'{  "size": 0,  "aggs": {    "plays": {      "terms": {        "field": "play_name.keyword",        "order": { "_key": "asc" },        "size": 1000,        "min_doc_count": 3900      }    }  }}'
```

4. Cuantos personajes tiene la obra "The Merchant Of Venice" ?

```
curl -XGET "http://elasticsearch:9200/shakespeare/_search?q=play_name="Merchant Of Venice"" -H 'Content-Type: application/json' -d'{  "size": 0,  "aggs": {        "actores": {          "terms": {"field": "speaker.keyword"}        }    }}'
```
