# Bitácora Laboratorio 1
¿Por qué se define primero una región primaria?
Aqui va la respuesta
¿Qué comando prueba la configuración resultante?
Aqui va la respuesta
¿Agregar una región crea latencia WAN real?
Aqui va la otra respuesta


Paso 4
ti4601=> SHOW CREATE TABLE catalogo_producto;
    table_name     |                             create_statement                             
-------------------+--------------------------------------------------------------------------
 catalogo_producto | CREATE TABLE public.catalogo_producto (                                 +
                   |         producto_id UUID NOT NULL DEFAULT gen_random_uuid(),            +
                   |         sku STRING NOT NULL,                                            +
                   |         nombre STRING NOT NULL,                                         +
                   |         precio DECIMAL(12,2) NOT NULL,                                  +
                   |         CONSTRAINT catalogo_producto_pkey PRIMARY KEY (producto_id ASC),+
                   |         UNIQUE INDEX catalogo_producto_sku_key (sku ASC),               +
                   |         CONSTRAINT check_precio CHECK (precio >= 0:::DECIMAL)           +
                   | ) LOCALITY GLOBAL
(1 row)

ti4601=> SHOW CREATE TABLE pedido;
 table_name |                                                    create_statement                                                    
------------+------------------------------------------------------------------------------------------------------------------------
 pedido     | CREATE TABLE public.pedido (                                                                                          +
            |         region public.crdb_internal_region NOT NULL,                                                                  +
            |         pedido_id UUID NOT NULL DEFAULT gen_random_uuid(),                                                            +
            |         producto_id UUID NOT NULL,                                                                                    +
            |         tienda STRING NOT NULL,                                                                                       +
            |         estado STRING NOT NULL DEFAULT 'CREADO':::STRING,                                                             +
            |         monto DECIMAL(12,2) NOT NULL,                                                                                 +
            |         version INT8 NOT NULL DEFAULT 0:::INT8,                                                                       +
            |         creado_en TIMESTAMPTZ NOT NULL DEFAULT now():::TIMESTAMPTZ,                                                   +
            |         CONSTRAINT pedido_pkey PRIMARY KEY (region ASC, pedido_id ASC),                                               +
            |         CONSTRAINT pedido_producto_id_fkey FOREIGN KEY (producto_id) REFERENCES public.catalogo_producto(producto_id),+
            |         CONSTRAINT check_monto CHECK (monto >= 0:::DECIMAL)                                                           +
            | ) LOCALITY REGIONAL BY ROW AS region
(1 row)


Paso 9:
que las regiones son lógicas;
que la máquina, los recursos disponibles y el número de ejecuciones se mantuvieron constantes;
que inferir el costo WAN requeriría latencia controlada, ausente hoy.

