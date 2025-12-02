# exercicio-neo4j-recomendacao

A atividade foi resolvida utilizando como base dados disponíveis no Kaggle: [Dataset](https://www.kaggle.com/datasets/undefinenull/million-song-dataset-spotify-lastfm?resource=download).

## Importação dos dados de músicas

```
CALL {
  LOAD CSV WITH HEADERS FROM 'file:///Music%20Info.csv' AS row
  WITH row, split(row.tags, ",") AS tags

  MERGE (t:Track { trackId: row.track_id })
  SET t.name = row.name,
      t.year = toInteger(row.year),
      t.duration = toInteger(row.duration),
      t.danceability = toFloat(row.danceability),
      t.energy = toFloat(row.energy),
      t.key = toInteger(row.key),
      t.loudness = toFloat(row.loudness),
      t.mode = toInteger(row.mode),
      t.speechiness = toFloat(row.speechiness),
      t.acousticness = toFloat(row.acousticness),
      t.instrumentalness = toFloat(row.instrumentalness),
      t.liveness = toFloat(row.liveness),
      t.valence = toFloat(row.valence),
      t.tempo = toFloat(row.tempo),
      t.timeSignature = toInteger(row.time_signature)

  MERGE (a:Artist { name: row.artist })
  MERGE (a)-[:PERFORMED]->(t)

  FOREACH (_ IN CASE WHEN row.genre IS NOT NULL AND row.genre <> "" THEN [1] ELSE [] END |
    MERGE (g:Genre { name: row.genre })
    MERGE (t)-[:HAS_GENRE]->(g)
  )

  WITH t, tags
  UNWIND tags AS tag
  WITH t, tag WHERE tag IS NOT NULL AND trim(tag) <> ""
  MERGE (ta:Tag { name: trim(tag) })
  MERGE (t)-[:HAS_TAG]->(ta)
} IN TRANSACTIONS OF 1000 ROWS;
```
## Criando índice para tracks

Criando um índice no campo trackId
```
CREATE INDEX FOR (t:Track) on t.trackId
```

## Importando dados de usuários

Importando os dados de usuário e contagem de reprodução das faixas. Foi necessário realizar a limitação a 100.000 registros devido às limitações de memória do container docker utilizado.

```
CALL {
  LOAD CSV WITH HEADERS FROM 'file:///User%20Listening%20History.csv' AS row
  WITH row
  LIMIT 100000

    MATCH(t:Track{trackId:row.track_id})

    MERGE(u:User{userId:row.user_id})

    MERGE(u)-[r:PLAY]->(t)
    SET r.playCount = toInteger(row.playcount)

} IN TRANSACTIONS OF 1000 ROWS;
```

## Criando índice para usuários

Criando um índice para usuários no campo userId

```
CREATE INDEX FOR (u:User) on u.userId
```

## Schema dos dados importados
![](schema.png)

O esquema pode ser visualizado pelo comando
```
CALL db.schema.visualization()
```


## Recomendação com base nas tags em comum

Recomendando 10 músicas com tags semelhantes a Under the Bridge do Red Hot Chili Peppers
```
MATCH (t1:Track {trackId: "TRIODZU128E078F3E2"})-[:HAS_TAG]->(tag)<-[:HAS_TAG]-(t2:Track)
WHERE t1 <> t2
RETURN t2.name AS recommended,
       count(tag) AS commonTags
ORDER BY commonTags DESC
LIMIT 10;
```

## Recomendação com base nas músicas ouvidas por um usuário

Recomendação com base no usuário 4e11f45d732f4861772b2906f81a7d384552ad12 do tipo: Quem ouviu Track X também ouviu a Track Y.

```
MATCH (target:User {userId: "4e11f45d732f4861772b2906f81a7d384552ad12"})-[:PLAY]->(t)
MATCH (other:User)-[:PLAY]->(t)
WHERE other <> target

MATCH (other)-[:PLAY]->(rec)
WHERE NOT (target)-[:PLAY]->(rec)

RETURN rec.name AS recommended, count(*) AS score
ORDER BY score DESC
LIMIT 10;
```

## Recomendação com base nos atributos numéricos implementando Distância Euclidiana

Verificando as músicas similares a Under the bridge do Red Hot Chilli Peppers. A distância euclidiana foi calculada pele fórmula:
$\[d(p, q) = \sqrt{(p_1 - q_1)^2 + (p_2 - q_2)^2 + \cdots + (p_n - q_n)^2}\]$ utilizando os parâmetros danceability, energy, valence e tempo.

```
MATCH (t1:Track {trackId: "TRIODZU128E078F3E2"})
MATCH (t2:Track)
WHERE t1 <> t2
WITH t1, t2,
     sqrt(
       (t1.danceability - t2.danceability)^2 +
       (t1.energy - t2.energy)^2 +
       (t1.valence - t2.valence)^2 +
       (t1.tempo - t2.tempo)^2
     ) AS dist
RETURN t2.name AS recommended, dist
ORDER BY dist ASC
LIMIT 10;
```
