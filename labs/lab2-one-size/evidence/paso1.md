# Paso 1: carga y espacio

Corrida final del 17/10/2026 en la máquina de `README.md` (Ryzen 7 5700U, WSL 2), perfil completo.
Salida completa con las líneas `OK:` en `salida_cargar.txt`.

## Resultados

| Métrica | Postgres | Valkey |
| --- | ---: | ---: |
| Filas cargadas | 10 906 858 | 1 000 000 |
| Tiempo de carga | 66,0 s (`COPY`) + 10,0 s (índice PK) | 52,2 s |
| Filas por segundo | 165 323 | 19 141 |
| Espacio medido | 1 653,6 MiB tabla + 233,7 MiB índice = 1 887,7 MiB en disco | 523,9 MiB en RAM |
| Bytes por fila | 181,5 | 549,4 |
| Mes completo | 1 887,7 MiB en disco | 5 714,2 MiB en RAM (extrapolado) |

## Predicción contra lo medido

| Métrica | Predicción | Medido |
| --- | ---: | ---: |
| Postgres, filas/s | ~150 000 | 165 323 |
| Valkey, filas/s | ~30 000 | 19 141 |
| Postgres, bytes por fila | ~180 | 181,5 |
| Valkey, bytes por fila | ~550 | 549,4 |
| Valkey, RAM del mes | ~6 000 MiB | 5 714 MiB |

El espacio salió como se esperaba. La carga en Valkey fue más lenta de lo previsto, así que Postgres
ganó por 8,6× y no por 5×: armar cada `HSET` en Python cuesta más de lo que se estimó.

## Preguntas

**¿Qué tienen en común los dos métodos de carga? ¿Cómo se resuelve el overhead de cargar muchos datos?**

Los dos evitan mandar un comando por fila y esperar la respuesta de cada uno. `COPY` manda el CSV como
un solo flujo de bytes, en una sola transacción, y el servidor lo parsea; no hay un `INSERT` ni un
commit por fila. El pipeline junta 10 000 `HSET` en el cliente y los manda juntos con `execute()`. En
los dos casos el costo fijo de cada envío se paga una vez por bloque y no una vez por fila.

**¿Por qué el pipeline cambia el tiempo?**

Sin pipeline, cada `r.hset()` manda el comando y espera la respuesta antes de mandar el siguiente: serían
1 000 000 de viajes de ida y vuelta por la red. Con lotes de 10 000 son 100 viajes. Valkey hace el mismo
trabajo; lo que se ahorra es el tiempo esperando la red.

Aun con pipeline, Valkey cargó a 19 141 filas/s, 8,6 veces menos que `COPY` (165 323 filas/s). Con
`COPY`, Python solo pasa líneas de texto y el servidor hace el resto. Con Valkey, Python tiene que armar
cada `HSET` con 19 pares campo-valor.

**¿Qué pasa si el TODO 1 falla después del `COPY`?**

La conexión usa autocommit, así que cuando se llama al TODO 1 el `COPY`, la llave primaria y el `VACUUM`
ya están confirmados. La tabla `trips` queda cargada, pero el script se detiene: no se registran las
medidas de disco y la carga de Valkey no se ejecuta. Al corregir y volver a correr, `pg_trips.sql` hace
`DROP TABLE` y la carga se repite completa, unos 76 s en esta máquina.

**Compare `bytes_por_fila` y `ram_extrapolada_total` contra `disco_total`.**

Valkey usa 549,4 B por viaje y Postgres 181,5 B (159 de la tabla y 22 del índice): 3,0 veces menos.
Para el mes completo serían 5 714 MiB de RAM en Valkey contra 1 888 MiB de disco en Postgres.

Postgres guarda cada valor con su tipo (un `smallint` ocupa 2 bytes) y los nombres de las columnas una
sola vez, en el esquema. Valkey guarda todo como texto y repite los 19 nombres de campo en cada viaje.
Además, 5 714 MiB no caben en el Valkey del curso (`--maxmemory 4gb`), así que el mes completo no se
podría cargar.

**¿Cuál motor podría guardar un año entero en su laptop?**

Postgres. Un año son unos 131 M de viajes (12 meses como este). En Postgres serían unos 22 GiB de disco;
en Valkey, unos 67 GiB de RAM. Esta laptop tiene 16 GB de RAM y casi 950 GB de disco libre.

## Ensayo: «Ningún motor gana todas las cargas de trabajo»

En el Paso 1 gana Postgres. Cargó el mes completo a 165 323 filas/s, 8,6 veces más rápido que Valkey
(19 141 filas/s), y cada viaje ocupa 181 B en disco contra 549 B en RAM, 3,0 veces menos. En el Paso 2
también gana Postgres: el `GROUP BY` sobre el mismo millón de filas tarda 171 ms contra 13 883 ms en
Valkey, 81 veces menos, porque Valkey no tiene esa primitiva y el cliente tiene que traer cada fila para
agruparla (sección 4.2 del paper).

En el Paso 3 gana Valkey. El update por llave le toma 0,26 ms contra 2,28 ms de Postgres (8,8×); la
lectura queda casi empatada (0,32 contra 0,34 ms). En el feed, `valkey_pipeline` procesa 51 040
mensajes/s, 1,7 veces más que la mejor variante de Postgres (`postgres_lote`, 29 792) y 105 veces más
que un commit por mensaje (486). Es el mismo patrón del paper, donde StreamBase procesó 160 000 mensajes
por segundo y el RDBMS 900 (sección 4): el motor que procesa el mensaje sin guardarlo primero gana ese
tipo de carga (sección 4.1).

Los números también muestran los límites de Valkey. Todo tiene que caber en RAM, y el mes completo
(5 714 MiB) ya no cabe en los 4 GB que tiene asignados. No tiene tipos ni esquema, por eso ocupa 3 veces
más. No tiene `GROUP BY`. Y su ventaja en el Paso 3 depende de no escribir a disco: con
`appendfsync always` baja a 240 mensajes/s y pierde contra Postgres por 2,0×.

Cada motor gana donde su diseño calza con la carga, y ninguno gana todo. Es lo que el paper plantea en
las secciones 1 y 7: un solo motor no puede ser el mejor para cargas tan distintas.
