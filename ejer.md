# Elastic-Shakespeare

<img src="calculon.webp" height="350">

En este ejercicio vamos a analizar un dataset de obras de Shakespeare y analizar un poco los datos.

Nota: Vamos a utilizar la versión 7.9.2 de Elasticsearch y de Kibana

La primera parte de este ejercicio es descargar e importar los datos de sus obras en modo _bulk_ (o con un 'bulk-request'):

Queda en cada uno la forma de consultar la DB, ya sea con curl, la consola de Kibana o algún driver de algún lenguaje de programación. Pero tengan en cuenta que con curl pueden llegar a haber problemas a la hora de hacer queries por los \" \".

1. Descargar los datos: https://raw.githubusercontent.com/lpinilla/Elastic-Shakespeare/master/shakespeare_dataset.json

2. Hacer un bulk-request:

```
curl -XPOST "localhost:9200/shakespeare/_doc/_bulk?pretty" \
-H "Content-Type: application/json" --data-binary @path/to/shakespeare_dataset.json
```

Nota: No olvidarse del '@' en el comando

Ahora que ya tenemos los datos, podemos empezar a jugar.

### Testing

Primero vamos a testear que haya cargado todos los datos, para eso vamos a contar cuantos registros hay y vamos a preguntar por 3 registros, el primero, alguno en el medio y el último. Si tenemos todos estos, vamos a asumir que tenemos todos los datos

Nota: Los índices arrancan en 0.

#### Hacer un count de todos los índices insertados

`curl -X GET localhost:9200/shakespeare/_count`

y deberíamos obtener:

```
{"count":111396,"_shards":{"total":1,"successful":1,"skipped":0,"failed":0}}
```

#### Consultar datos aleatorios

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

### Ver si está la obra

Primero que nada, revisemos si está la obra en nuestra base.

`curl -XGET "http://localhost:9200 /shakespeare/_search?q=play_name="Romeo and Juliet""`

Deberíamos recibir algunas entradas, por lo que confirmamos que la obra está presente en la DB.

### "O Romeo.."

Me pregunto cuantas veces aparece está esta frase en la obra, sin importar quien lo diga.

Recordemos que el atributo "play_name" nos indica la obra y el atributo "text_entry" nos indica el texto de algún díalogo.

Como Elasticsearch por default nos ofrece también búsquedas simulares, podemos filtrar algunas respuestas indeseadas incrementando el score, para esto podríamos utilizar un score mínimo de 20.

```
curl -XGET "http://localhost:9200/_search" -H 'Content-Type: application/json' -d'{  "min_score": 20,    "query": {        "bool": {            "must": [{                "match": { "play_name": "Romeo and Juliet"}            }, {                "match": { "text_entry": "O Romeo"}            }]        }    }}'
```

## Agregaciones

Nota: es importante aclarar que para las funciones de agregación sobre strings, tal vez se tenga que utilizar el 'keyword' en vez del atributo. Por ejemplo, en vez de tener que usar "play_name", tengamos que usar "play_name.keyword". Además, podemos usar el atributo "size" en "0" en la query para hacer parecido a un "DISTINCT" en SQL.

### Primero vamos a ver cuantas obras tenemos en total en la db:

Para esto podemos utilizar el comando "cardinality"

```
curl -XGET "http://localhost:9200/shakespeare/_search" -H 'Content-Type: application/json' -d'{  "size": 0,  "aggs" : {    "plays": { "cardinality": {"field": "play_name.keyword"} }  }}'
```

### Obtener la misma lista pero en orden

Investigar sobre el comando "order" para la función de agregación. Si utilizan el comando "terms", recuerden que por default hay un máximo de elementos que traen los queries, podemos extender el límite utilizando "size: 1000" dentro del elemento "terms".

```
curl -XGET "http://localhost:9200/_search" -H 'Content-Type: application/json' -d'{  "size": 0,  "aggs": {    "plays": {      "terms": {        "field": "play_name.keyword",        "order": { "_key": "asc" },        "size": 1000      }    }  }}'
```

### Obtener las obras con al menos 3900 líneas

Investigar el comando "min_doc_counts" para la función de agregación. Recuerden el comentario sobre "size" del inciso anterior.

```
curl -XGET "http://localhost:9200/_search" -H 'Content-Type: application/json' -d'{  "size": 0,  "aggs": {    "plays": {      "terms": {        "field": "play_name.keyword",        "order": { "_key": "asc" },        "size": 1000,        "min_doc_count": 3900      }    }  }}'
```

### Cuantos personajes tiene la obra "The Merchant Of Venice" ?

```
curl -XGET "http://localhost:9200/shakespeare/_search?q=play_name="Merchant Of Venice"" -H 'Content-Type: application/json' -d'{  "size": 0,  "aggs": {        "actores": {          "terms": {"field": "speaker.keyword"}        }    }}'
```

Finalmente, desde Kibana, los request quedarían:

```
GET /shakespeare/_doc/0

GET /shakespeare/_doc/73330

GET /shakespeare/_doc/111395

GET /shakespeare/_search?q=play_name="Romeo and Juliet"

GET /_search
{
  "min_score": 20,
    "query": {
        "bool": {
            "must": [{
                "match": { "play_name": "Romeo and Juliet"}
            }, {
                "match": { "text_entry": "O Romeo"}
            }]
        }
    }
}

GET shakespeare/_search
{
  "size": 0,
  "aggs" : {
    "plays": { "cardinality": {"field": "play_name.keyword"} }
  }
}

GET /_search
{
  "size": 0,
  "aggs": {
    "plays": {
      "terms": {
        "field": "play_name.keyword",
        "order": { "_key": "asc" },
        "size": 1000
      }
    }
  }
}

GET /_search
{
  "size": 0,
  "aggs": {
    "plays": {
      "terms": {
        "field": "play_name.keyword",
        "order": { "_key": "asc" },
        "size": 1000,
        "min_doc_count": 3900
      }
    }
  }
}


GET shakespeare/_search?q=play_name="Merchant Of Venice"
{
  "size": 0,
  "aggs": {
        "actores": {
          "terms": {"field": "speaker.keyword"}
        }
    }
}
```

