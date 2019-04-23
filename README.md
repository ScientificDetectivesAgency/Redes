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


Generalmente este formato ya contiene las extensiones con las que se va a trabajar. Sin embargo como parte de la preparación de los datos crearemos la extensión pgrouting con el siguiente comando: 

```sql
create extension pgrouting;
```

Esto nos permite trabajar con todas las funciones disponibles de esta librería especializada en análisis de redes sobre datos geoespaciales. [Aquí](https://docs.pgrouting.org/2.4/en/index.html) puedes consultar la documentación de algunas de las funciones disponibles.

Dado que esta librería trabaja con datos geoespaciales generalmente vamos a trabajar con datos de vías de comunicación. Para comenzar a trabajar con redes necesitamos construir una topología. Esto significa que para cualquier arco (linea) de las vías de comunicación, los extremos de ese arco estarán unidos a un nodo único y a su vez a otros arcos. Una vez que todos los arcos están conectados a los nodos, tenemos un gráfico que se puede utilizar para hacer calculos con pgrouting.

Para esta sección vamos a trabajar con dos tipos de redes una que corresponde a la Ciudad de México descargada de [Open Street Maps](https://www.openstreetmap.org/export#map=8/15.295/-92.568) y otra generada por el Instituto Mexicano del Transporte (IMT) que pertenece al estado de Chiapas.

Primero trabajaremos con la red de OSM (osm_cdmx), para comenzar a crear la topología necesitamos agregar dos campos para almacenar los nodos de orígen y destino de cada segmento:

```sql
alter table calles add column source integer;
alter table calles add column target integer;
```

Ahora, vamos a llamar a la función ```select pgr_createTopology('lines', tolerancia, 'geom', 'id')```, para crear los nodos y asignar los identificadores correspondientes. Los argumentos de la función son los siguientes:

**lines:** Tabla con las geometrías
**tolerancia:** Distancia (en las unidades de la proyección) máxima para considerar dos lineas unidas.
**geom:** Columna con geometría.
**id:** columna con el identificador único.

En nuestro caso:
```sql
select pgr_createTopology('osm_cdmx', 0.05, 'geom', 'gid');
```
Como pueden ver, esta función crea la tabla ```osm_cdmx_vertices_pgr```, idealmente esta tabla contiene todos los nodos de la red, examínenla en Qgis.

Ahora necesitamos determinar el costo que le vamos a asignar a cada arco, con costo nos referimos a una variable numerica que refleje la capacidad de transportarse por un segmento dado, puede ser tiempo, velocidad, longitud, pendiente, etc. 

Si exploramos la tabla con ``` select * from osm_cdmx limit 10``` veremos las siguientes columnas:

| columna  | 
| ------------- | 
| gid, geom, class_id |
| x1,	y1,	x2,	y2 |
|one_way, maxspeed	
|osm_id, source_osm, target_osm	|	

Como verás como un costo posible solo tenemos la velocidad (maxspeed), entonces vamos a calcular longitud y tiempo.

```sql
alter table osm_cdmx add column costo double precision;
alter table osm_cdmx add column longitud float;
alter table osm_cdmx add column tiempo float;

update osm_cdmx set longitud = st_length(geom)/1000 
update osm_cdmx set tiempo = (longitud/maxspeed::float)*60
```

:shipit:
