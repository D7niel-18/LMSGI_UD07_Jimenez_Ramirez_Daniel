# Manual de Explotación — WillmanTech S.L.

**Fecha:** 21-05-2026
**Norma usada:** ISO/IEC/IEEE 26514:2022

---
## 1. Introducción y Arquitectura

WillmanTech S.L. usa un ERP para gestionar sus ventas, facturas y clientes. El sistema corre en dos contenedores Docker que se levantan juntos con Docker Compose:

- **web**: el propio ERP (puerto 8069, accesible desde el navegador)
- **db**: la base de datos PostgreSQL (solo accesible internamente)

Los módulos activos en el sistema son: `account` (facturación), `sale` (ventas), `crm` (clientes), `purchase` (compras), `stock` (almacén) y `report` (informes PDF).

---

## 2. Instalación desde cero

### Requisitos previos
- Docker y Docker Compose instalados
- Mínimo 4 GB de RAM y 20 GB de disco libre

### Pasos

**1.** Crear el fichero docker-compose.yml

```
services:  # Define los servicios que se van a ejecutar

  odoo:  # Servicio principal: Odoo
    image: odoo:latest  # Usa la última version
    container_name: odoo  # Nombre asignado al contenedor
    restart: unless-stopped  # Reinicia automáticamente
    depends_on:
      - db  # Indica que este servicio depende del servicio de base de datos
    ports:
      - "8200:8069"  # Pone el puerto 8069 del contenedor al 8200 del host
    volumes:
      - odoo-data:/var/lib/odoo  # Volumen persistente para datos de Odoo
      - ./config:/etc/odoo  # Carpeta local para archivos de configuración
      - ./addons:/mnt/extra-addons  # Carpeta local para módulos personalizados
    environment:
      - HOST=db  # Nombre del servicio db
      - USER=odoo  # Usuario de la base de datos
      - PASSWORD=odoo  # Contraseña de la base de datos
    command: odoo -d odoo --db_user=odoo --db_password=odoo -i base  
    # Comando que inicia Odoo, crea/usa la DB "odoo" e instala el módulo base

  db:
    image: postgres:16.0  # Usa PostgreSQL versión 16
    container_name: db  # Nombre del contenedor de base de datos
    restart: unless-stopped  # Reinicio automático salvo parada manual
    environment:
      - POSTGRES_USER=odoo  # Usuario de la base de datos
      - POSTGRES_PASSWORD=odoo  # Contraseña del usuario
      - POSTGRES_DB=odoo  # Nombre de la base de datos inicial
      - PGDATA=/var/lib/postgresql/pgdata  # Ruta interna donde se almacenan los datos
    volumes:
      - db-data:/var/lib/postgresql/data  # Volumen persistente para PostgreSQL

volumes:  # Definición de volúmenes persistentes
  odoo-data:  # Volumen para almacenar datos de Odoo
  db-data:  # Volumen para almacenar datos de PostgreSQL
```

**2.** Levantar el contenedor
Con el comando docker compose -d up

---

## 3. Seguridad y Control de Acceso
El sistema tiene tres tipos de usuario:

**Administrador**: Configurar el sistema, crear usuarios, ver todos los datos
**Contable**: Crear y validar facturas, ver informes. No puede tocar la configuración
**Comercial**: Gestionar clientes y pedidos de venta. No ve la contabilidad

### Política de contraseñas
- Mínimo 10 caracteres con mayúsculas, minúsculas, números y símbolos
- Caducan cada 90 días
- La cuenta se bloquea tras 3 intentos fallidos

Para crear un usuario: **Ajustes → Usuarios → Nuevo**, rellenar los datos y asignar el rol correspondiente.

---

## 4. Backup y Restauración

### Hacer una copia de seguridad
Para realizar la copia de seguridad de la base de datos en POSTGRESQL tenemos que poner el siguiente comando
pg_dump -U postgres -d db12 > backup.sql

**-U**: Indicamos el usuario de la base de datos
**-d**: Indicamos la base de datos
**> backup.sql**: Le decimos que lo meta en un archivo .sql

### Restaurar una copia de seguridad

Primero debemos para el contenedor y luego

Para restaurar la base de datos necesitamos el siguiente comando:
psql -h localhost -U usuario_postgres -d nombre_base_datos -f ruta/al/archivo.sql

**-h:** La dirección del servidor
**-U:** El usuario
**-d:** El nombre de la base de datos destino que creaste en la fase previa.
**-f:** La ruta exacta donde se encuentra guardado tu archivo .sql

Y volvemos a arrancar

---

## 5. Flujo de Facturación y Generación de PDF

### Cómo crear una factura

1. Ir a **Facturación → Clientes → Facturas → Nuevo**
2. Seleccionar el cliente y rellenar las líneas (producto, cantidad, precio)
3. Pulsar **Confirmar** — la factura recibe un número definitivo (ej: `FACT/2026/00001`)
4. Pulsar **Imprimir** para obtener el PDF

### Cómo se genera el PDF (pipeline)

Cuando se pulsa "Imprimir", el sistema hace esto internamente:

```
Plantilla QWeb (XML)
        ↓
El motor QWeb rellena la plantilla con los datos reales de la factura
        ↓
Se genera un documento HTML con el diseño de la factura
        ↓
wkhtmltopdf convierte ese HTML a PDF (como si lo imprimiera un navegador)
        ↓
El PDF llega al usuario listo para descargar o enviar por correo
```
wkhtmltopdf es una herramienta de código abierto de línea de comandos que permite convertir páginas web o archivos HTML directamente a formato PDF o imágenes
```
