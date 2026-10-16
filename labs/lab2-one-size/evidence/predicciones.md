# Predicciones del Laboratorio 2

Se escriben antes de correr cada paso.

## Paso 1: carga y espacio

Fabricio Mena · 15/10/2026, 18:34 · perfil completo (10 906 858 filas en Postgres, 1 000 000 en Valkey)

| Métrica | Postgres | Valkey | Gana |
| --- | ---: | ---: | --- |
| Velocidad de carga | ~150 000 filas/s | ~30 000 filas/s | Postgres, ~5× |
| Espacio del mes completo | ~1 850 MiB en disco | ~6 000 MiB en RAM | Postgres, ~3× |
| Bytes por fila | ~180 B | ~550 B | Postgres, ~3× |

- Postgres guarda cada columna con su tipo (un `smallint` ocupa 2 bytes, un `double` 8) y los nombres
  de las columnas una sola vez, en el esquema. Estimo unos 150 B por fila en la tabla y unos 20 B por
  fila en el índice de la llave primaria.
- Valkey guarda todo como texto y repite los 19 nombres de campo en cada viaje (`pickup_longitude`,
  `improvement_surcharge`, etc.). Solo los nombres ya suman unos 280 B por fila.
- `COPY` manda el CSV tal cual y el servidor lo procesa. En Valkey el cliente de Python tiene que armar
  cada `HSET` con los 19 pares campo-valor. El pipeline evita esperar la red en cada fila, pero ese
  trabajo en el cliente sigue ahí.
- Un año son unos 130 M de viajes. En Postgres serían unos 22 GiB de disco, que caben en una laptop. En
  Valkey serían unos 70 GiB de RAM, que no caben. Con el `--maxmemory 4gb` del curso, ni siquiera el mes
  completo cabría en Valkey.

## Paso 2: consulta de almacén

*Pendiente (Persona 2).*

## Paso 3: escrituras y feed

*Pendiente (Persona 3).*
