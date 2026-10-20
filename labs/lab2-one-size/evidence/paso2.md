# Paso 2: consulta de almacén

Consulta analítica `GROUP BY payment_type` ejecutada en PostgreSQL 16 y Valkey 8.

Corrida final del 17/10/2026 en la máquina de `README.md` (Ryzen 7 5700U, WSL 2), la misma de los pasos 1 y 3. Perfil de carga: PostgreSQL con 10 906 858 viajes y Valkey con 1 000 000 de viajes. Salida completa en `salida_analitica.txt`.

La ejecución de `analitica.py` finalizó correctamente:

`OK: TODO 3: el GROUP BY a mano da los mismos conteos y promedios que Postgres`

## Resultados

| Métrica | PostgreSQL | Valkey |
| --- | ---: | ---: |
| Consulta sobre tabla completa | 1064,854 ms | — |
| Consulta sobre el mismo subconjunto | 170,836 ms | 13882,670 ms |
| Pregunta no anticipada | 1329,447 ms | — |
| Tiempo de EXPLAIN ANALYZE | 1445,133 ms | — |

Sobre el mismo subconjunto de un millón de viajes, PostgreSQL fue aproximadamente **81,3 veces más rápido** que Valkey.

## Análisis del plan de PostgreSQL

**Parallel Seq Scan on trips**

El plan utilizó un escaneo secuencial paralelo con 2 workers lanzados y la participación del proceso principal. El nodo reportó `loops=3`, con aproximadamente 3 635 619 filas por ejecución, lo que representa alrededor de 10,9 millones de registros procesados en total.

El tiempo del escaneo fue de aproximadamente 459,170 ms, mientras que la consulta completa tardó 1445,133 ms según `EXPLAIN ANALYZE`.

**Buffers y espacio leído**

El plan registró `Buffers: shared read=211656`. Considerando bloques de 8 KiB:

211656 × 8 / 1024 = **1653,5625 MiB**

Esta cantidad es prácticamente igual a los 1653,562 MiB reportados como `disco_tabla` en el Paso 1. Por tanto, el volumen de páginas leídas equivale prácticamente al tamaño completo de la tabla.

Es importante diferenciar este valor de `disco_total`, que incluye los índices y alcanza 1887,703 MiB.

**Agregación en dos fases**

PostgreSQL utiliza `Partial HashAggregate` para calcular agregaciones parciales por `payment_type`. Posteriormente, los resultados pasan por `Gather Merge` y `Finalize GroupAggregate`, que produce las agrupaciones finales.

En Valkey, por el contrario, fue necesario realizar esta operación manualmente desde Python mediante `SCAN`, `HMGET` en pipeline y diccionarios para acumular las sumas y los conteos.

**Nodo dominante y relación con row-store**

El nodo que más tiempo consume es el `Partial HashAggregate` junto con el `Parallel Seq Scan` que lo alimenta: 1420 ms de los 1445 ms de la consulta (98 %). De ese tiempo, 459 ms son el escaneo, que lee las 211 656 páginas de la tabla, y el resto es agrupar y promediar 10,9 millones de filas.

La consulta solamente necesita `payment_type` y `fare_amount` (el escaneo entrega `width=7` bytes por fila), pero PostgreSQL almacena los datos por filas, así que lee páginas con las 20 columnas, unos 159 bytes por fila. Para usar 2 columnas leyó la tabla entera.

En la sección 5.1 del paper, Stonebraker y Çetintemel explican que los sistemas orientados a filas favorecen las escrituras de registros, mientras que los almacenes columnares pueden resultar más eficientes para consultas analíticas que utilizan pocas columnas.

## Pregunta no anticipada

Se ejecutó una consulta para calcular la propina promedio por hora del día en los viajes con `payment_type = 1`:

```sql
SELECT extract(hour FROM pickup_at) AS hora,
       AVG(tip_amount) AS propina
FROM trips
WHERE payment_type = 1
GROUP BY 1
ORDER BY 1;
```

PostgreSQL respondió esta consulta en aproximadamente **1329,447 ms**.

Esta prueba demuestra una ventaja del modelo relacional: es posible formular nuevas consultas mediante SQL sin modificar previamente la estructura de los datos. Esto resulta útil para los data warehouses, donde son frecuentes las consultas *ad hoc* sobre datos históricos, como explican Stonebraker y Çetintemel en la sección 2.

## Borrador del ensayo: «Que Valkey pierda el GROUP BY no refuta el argumento del data warehouse»

En el Paso 2, PostgreSQL fue aproximadamente 81,3 veces más rápido que Valkey al ejecutar la agregación sobre el mismo subconjunto de viajes. PostgreSQL obtuvo una mediana de 170,836 ms y Valkey necesitó 13882,670 ms. Sin embargo, esto no contradice el argumento de Stonebraker sobre los almacenes de datos.

La prueba no compara un sistema orientado a filas contra uno columnar. Valkey es un almacén clave-valor y no dispone de una primitiva equivalente a `GROUP BY`. Por esta razón, fue necesario recuperar los valores mediante `SCAN` y `HMGET` y realizar la agregación desde el cliente. PostgreSQL, en cambio, ejecutó la agregación dentro del motor mediante `Partial HashAggregate` y `Finalize GroupAggregate`.

Además, el plan muestra que PostgreSQL leyó aproximadamente 1653,6 MiB de páginas, una cantidad equivalente al tamaño completo de la tabla. Esto se relaciona con la sección 5.1 del paper, donde se argumenta que el almacenamiento columnar puede ser más eficiente para consultas que solo requieren algunos atributos de grandes cantidades de registros.

Por lo tanto, que PostgreSQL supere a Valkey en este experimento demuestra la importancia de contar con las primitivas adecuadas para cada carga de trabajo, pero no demuestra que un sistema orientado a filas sea necesariamente la mejor opción para un data warehouse.