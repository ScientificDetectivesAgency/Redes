Análisis de redes con PgRouting

Para comenzar a trabajar esta vez no usaremos shapefiles independientes, utilizaremos el backup de una base de datos ya elaborada que hay que cargar desde pgadmin. Las tablas con las que vamos a trabajar son las siguientes: 

| Tabla  | Descripción|
| ------------- | ------------- |
| osm_cdmx  | Red CDMX (OSM)  |
| estaciones  | Estaciones simuladas de bomberos  |
| imt_chiapas  | Red Chiapas (IMT)  |
| cultivo  | Zonas de cultivo de café  |
| acopios  | Lugar para el almacenamiento de cafe  |
| beneficios  | Punto de procesado y tostado de café  |
| exportadoras  | Comercializadoras de café  |


[1] Generalmente este formato ya contiene las extensiones con las que se va a trabajar. Sin embargo como parte de la preparación de los datos crearemos la extensión pgrouting en pgadmin con el siguiente comando: 

```sql
create extension pgrouting;
```

Esto nos permite trabajar con todas las funciones disponibles de esta librería especializada en análisis de redes sobre datos geoespaciales. [Aquí](https://docs.pgrouting.org/2.4/en/index.html) puedes consultar la documentación de algunas de las funciones disponibles.

Dado que esta librería trabaja con datos geoespaciales generalmente vamos a trabajar con datos de vías de comunicación. Para comenzar a trabajar con redes necesitamos construir una topología. Esto significa que para cualquier arco (linea) de las vías de comunicación, los extremos de ese arco estarán unidos a un nodo único y a su vez a otros arcos. Una vez que todos los arcos están conectados a los nodos, tenemos un gráfico que se puede utilizar para hacer calculos con pgrouting.

Para esta sección vamos a trabajar con dos tipos de redes una que corresponde a la Ciudad de México descargada de [Open Street Maps](https://www.openstreetmap.org/export#map=8/15.295/-92.568) y otra generada por el Instituto Mexicano del Transporte (IMT) que pertenece al estado de Chiapas.

### El café y Chiapas 

Primero vamos a trabajar con un problema sencillo sobre la comercialización del café, como habrás observado en el backupt que restauramos tenemos la red de chiapas y tablas de puntos que corresponden a infraestructura involucrada en la comercialización y procesamiento de café (cultivos, beneficios, acopios y exportadoras). 

[2] Para crear la topología, necesitamos agregar dos campos para almacenar los nodos de orígen y destino de cada segmento:

```sql
alter table imt_chiapas add column source integer;
alter table imt_chiapas add column target integer;
```

[3] Ahora, vamos a llamar a la función ```select pgr_createTopology('lines', tolerancia, 'geom', 'id')```, para crear los nodos y asignar los identificadores correspondientes. Los argumentos de la función son los siguientes:

**lines:** Tabla con las geometrías
**tolerancia:** Distancia (en las unidades de la proyección) máxima para considerar dos lineas unidas.
**geom:** Columna con geometría.
**id:** columna con el identificador único.

En nuestro caso:
```sql
select pgr_createTopology('imt_chiapas', 0.05, 'geom', 'gid');
```
Como pueden ver, esta función crea la tabla ```imt_chiapas_vertices_pgr```, idealmente esta tabla contiene todos los nodos de la red, examínenla en Qgis.

[4]Tener la topología calculada ahora nos permite trabajar con puntos que por la forma en que fueron georeferenciados no siempre caen exactamente sobre la red, asignandoles el id del nodo  de la tabla ```imt_chiapas_vertices_pgr```: 


```sql
alter table #tabla_puntos# add column closest_node bigint; 
update #tabla_puntos# set closest_node = c.closest_node
from  
(select b.id as #id_puntos#, (
  SELECT a.id
  FROM #tabla_vertices# As a
  ORDER BY b.geom <-> a.the_geom LIMIT 1
)as closest_node
from  #tabla_puntos# b) as c
where c.#id_puntos# = #tabla_puntos#.id
```
**Nota:** Calcula el nodo más cercano de cultivos, beneficios, acopios y exportadoras.

 Para comenzar a usar PgRouting necesitamos determinar el costo que le vamos a asignar a cada arco, con costo nos referimos a una variable numerica que refleje la capacidad de transportarse por un segmento dado, puede ser tiempo, velocidad, longitud, pendiente, etc. 

Si exploramos la tabla con ``` select * from imt_chiapas limit 10``` veremos las siguientes columnas:
```geom, id,	tipo_vial,	cond_pav,	recubri	carriles,	condicion,	velocidad```

[5] Como verás como un costo posible solo tenemos la velocidad, entonces vamos a calcular longitud y tiempo:

```sql
alter table imt_chiapas add column costo double precision;
alter table imt_chiapas add column longitud float;
alter table imt_chiapas add column tiempo float;

update imt_chiapas set longitud = st_length(geom)/1000 
update imt_chiapas set tiempo = (longitud/maxspeed::float)*60
```
[6] Una vez calculados los costos, podemos comenzar a trabajar con los datos. Lo primero que vamos a hacer es explorar las relaciones entre la infraestructura relacionada con su producción y calcular rutas con diferentes costos con [Dijkstra](http://docs.pgrouting.org/2.0/en/src/dijkstra/doc/index.html#pgr-dijkstra):

```sql
select b.geom, a.*
from
(select node, edge as id, cost from pgr_dijkstra(
  ' SELECT  id,
           source::int4,
           target::int4,
           #COLUMNA DE COSTO#::float8 AS cost
    FROM  #TABLA RED#', #NodoDeOrigen#, #NodoDeDestino#, directed:=false)) as a
join #TABLA RED# b
on a.id = b.id
```
[7] Una parte importante de explorar las relaciones en la cadena de producción de café es identificar qué productores llevan su café a procesar, almacenar y vender en los diferentes puntos de infrestructura, para evaluar y es posible optimizar la forma en que se dan actualmente estas relaciones, para ello vamos a utilizar [pgr_dijkstraCost](https://docs.pgrouting.org/2.2/en/src/dijkstra/doc/pgr_dijkstraCost.html), esta función _calcula la suma de los costos de la ruta más corta para un subconjunto de pares de nodos de la red_.

```sql
create table cultivo_beneficios as 
SELECT DISTINCT ON (start_vid)
       start_vid, end_vid, agg_cost
FROM   (SELECT * FROM pgr_dijkstraCost(
    'select id, source, target, costo as cost from calles',
    array(select distinct(closest_node) from #NodoInicial#),
    array(select distinct(closest_node) from #NodoFinal# ),
	 directed:=false)
) as sub
ORDER  BY start_vid, agg_cost asc;
```
**NOTA:** Calcularlo para cada una de las etapas de la cadena productiva de cafe, _cultivo-acopio, acopio-beneficio, beneficio-exportadora_.

[8]

:shipit:


### Trabajando con Open Street Maps 

Como verás como un costo posible solo tenemos la velocidad (maxspeed), entonces vamos a calcular longitud y tiempo.

```sql
alter table osm_cdmx add column costo double precision;
alter table osm_cdmx add column longitud float;
alter table osm_cdmx add column tiempo float;

update osm_cdmx set longitud = st_length(geom)/1000 
update osm_cdmx set tiempo = (longitud/maxspeed::float)*60
```
Finalmente, para terminar esta parte del ejercicio, vamos a calcular una ruta usando dos algoritmos diferentes, primero vamos a usar [Dijkstra](http://docs.pgrouting.org/2.0/en/src/dijkstra/doc/index.html#pgr-dijkstra):
```sql
```
:shipit:
