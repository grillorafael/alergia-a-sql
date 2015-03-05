# Alergia à SQL
## Introdução

Quem nunca ouviu as seguintes frases: “Escrever SQL nos dias de hoje não é uma boa prática” ou “Tente não escrever SQL, usa os recursos do ORM”, ainda não trabalhou o suficiente.

O mercado de startups parece cada vez mais alérgico ao SQL, um dos recursos mais poderosos e antigos(1974!) no mundo dos bancos de dados.

O que as pessoas esquecem ou escolhem ignorar é a [capacidade que um banco de dados tem](http://rob.conery.io/2015/02/24/embracing-sql-in-postgres/) e pode te oferecer para resolver problemas banais.

Mas a questão é, como e porque devemos perder o medo do SQL?

## Escrever SQL é sim uma boa prática

Quando estamos envolvidos demais com os ORMs que utilizamos para fazer aplicações, tentamos cada vez mais resolver os problemas de acesso de dados utilizando os recursos do ORM que muitas vezes são muito bons e completos.

Porém, quando precisamos resolver consultas mais complexas, nós enfrentamos elas geralmente de duas formas diferentes:
1. Realizando diversas buscas e processando a nível de aplicação
1. Escrever uma String de SQL e executá-la com o ORM

Por mais vantajoso em termos de desempenho seja escrever a String da consulta dentro do ORM, muitas vezes preferimos processar essas consultas em tempo de execução utilizando os lindos recursos de nossas maravilhosas linguagens dinâmicas. Afinal, é deselegante escrever uma String com uma query SQL no meio do nosso Business Logic, né?

Esse pensamento é bastante comum, pouco eficiente e bastante errado. Quando assumimos fazer o item 1, perdemos os recursos de [otimização de query](http://en.wikipedia.org/wiki/Query_optimization) que o banco nos fornece e damos lugar um código que é muito provavelmente ineficiente e tem pouca garantia de consistência.
Quando assumimos o item 2, ganhamos o fator otimização porém perdemos no fator manutenibilidade.

Você pode alegar que não se pode ganhar todas né? Nesse caso, você pode sim ganhar todas.
## Functions e Procedures

Funções internas do banco de dados são normalmente chamadas de Procedure ou Routine. Essas procedures se comportam como uma função em qualquer linguagem de programação, ou seja, podemos chama-las através de queries, podemos utilizá-las em outras procedures e podemos também chama-las através do nosso ORM de preferência.

### Funções nativas do PostgreSQL
O PostgreSQL possui uma série de [funções implementadas por padrão](http://rob.conery.io/2015/02/24/embracing-sql-in-postgres/) que podem ajudar bastante no desenvolvimento de aplicações. Vamos exemplificar algumas que acreditamos serem as mais desconhecidas e interessantes.

#### Manipulação de Datas
O PostgreSQL possui um tipo de dados chamado de interval que é um tipo que lida com datas e horas. Através do tipo interval é possível manipular datas de diversas formas diferentes como exemplificado abaixo.

```
select '1 week' + now() as a_week_from_now;
        a_week_from_now
-------------------------------
 2015-03-03 10:08:12.156656+01
(1 row)
```

[TODO]

#### Exemplos

Não precisamos de muita análise para perceber o problema que existe nesse trecho de código. Podemos facilmente reescrever esse trecho e encapsula-lo em uma Procedure como exemplificado abaixo.

Antes
```javascript
var sql = '';
sql += ' SELECT p1.id, p1.nome, p1.descricao,  ST_AsGeoJSON(p1."poiGeometrico") geojson, a1.nome "acervo_nome", c1.nome "categoria_nome", p1."urlIcone", ';
sql += ' p2.id id2, p2.nome nome2 , p2.descricao descricao2,  ST_AsGeoJSON(p2."poiGeometrico") "geojson2", a2.nome "acervo_Nome2", c2.nome "categoria_nome2", p2."urlIcone" "urlIcone2", ';
sql += ' ST_Distance(ST_Transform(p1."poiGeometrico",3067), ST_Transform(p2."poiGeometrico",3067)) distancia ';
sql += ' FROM "Poi" p1, "Poi" p2, "Acervo" a1, "Acervo" a2, "Categoria" c1, "Categoria" c2 ';
sql += ' WHERE c1.id = :categoriaId1 AND c2.id = :categoriaId2 AND ';
sql += ' p1."AcervoId" = a1.id AND p2."AcervoId" = a2.id AND ';
sql += ' p1."CategoriumId" = c1.id AND p2."CategoriumId" = c2.id ';
sql += ' AND ST_Distance(ST_Transform(p1."poiGeometrico",3067), ST_Transform(p2."poiGeometrico",3067)) < :distancia ';

var parametros = {
    categoriaId1: parseInt(req.body.categoriaId1, 10),
    categoriaId2: parseInt(req.body.categoriaId2, 10),
    distancia: parseInt(req.body.distancia, 10)
};

db.sequelize.query(sql, null, {
    raw: true
}, parametros)
    .then(function(data) {
        res.send(data);
    });
```

Depois
```SQL
CREATE OR REPLACE FUNCTION deteccao_de_impacto(
    p_categoria_id1 integer,
    p_categoria_id2 integer,
    p_distancia integer
)
RETURNS TABLE (
    id integer,
    nome varchar,
    descricao varchar,
    geojson json,
    acervo_nome varchar,
    categoria_nome varchar,
    urlIcone varchar,
    id2 integer,
    nome2 varchar,
    descricao2 varchar,
    geojson2 json,
    acervo_Nome2 varchar,
    categoria_nome2 varchar,
    urlIcone2 varchar,
    distancia integer
) AS
$BODY$
#variable_conflict use_variable
BEGIN
    RETURN QUERY
    SELECT
        p1.id,
        p1.nome,
        p1.descricao,
        ST_AsGeoJSON(p1.poiGeometrico) geojson,
        a1.nome acervo_nome,
        c1.nome categoria_nome,
        p1.urlIcone,
        p2.id id2,
        p2.nome nome2,
        p2.descricao descricao2,
        ST_AsGeoJSON(p2.poiGeometrico) geojson2,
        a2.nome acervo_Nome2,
        c2.nome categoria_nome2,
        p2.urlIcone urlIcone2,
        ST_Distance(
            ST_Transform(p1.poiGeometrico, 3067), 
            ST_Transform(p2.poiGeometrico, 3067)
        ) distancia
    FROM 
        Poi p1,
        Poi p2,
        Acervo a1,
        Acervo a2,
        Categoria c1,
        Categoria c2
    WHERE 
        c1.id = p_categoria_id1 AND
        c2.id = p_categoria_id2 AND
        p1.AcervoId = a1.id AND
        p2.AcervoId = a2.id AND
        p1.CategoriumId = c1.id AND
        p2.CategoriumId = c2.id AND
        ST_Distance(
            ST_Transform(p1.poiGeometrico, 3067), 
            ST_Transform(p2.poiGeometrico, 3067)
        ) < p_distancia
    ;
END;
$BODY$
  LANGUAGE plpgsql VOLATILE
  COST 100;
ALTER FUNCTION deteccao_de_impacto(integer, integer, integer)
  OWNER TO postgres;
```

```javascript
var sql = "select deteccao_de_impacto(:categoriaId1, :categoriaId2, :distancia)";
var parametros = {
    categoriaId1: parseInt(req.body.categoriaId1, 10),
    categoriaId2: parseInt(req.body.categoriaId2, 10),
    distancia: parseInt(req.body.distancia, 10)
};

db.sequelize.query(sql, null, {
    raw: true
}, parametros)
    .then(function(data) {
        res.send(data);
    });
```

### Então vamos usar SQL para tudo!

“Agora que eu sei tudo de SQL! Vou usar SQL em tudo!”

### Não existe bala de prata

### Conclusões
