# Oracle Database Free con Docker

En estas instrucciones se utilizará Docker Compose a través de la consola para
levantar un contenedor con OracleDB local funcional.

Se utiliza un volumen para que los datos persistan cuando se baje el contenedor.

## Requisitos

- Docker
- Docker compose

Revisar que Docker esté funcionando correctamente:

```bash
$ docker compose version
Docker Compose version v2.39.3  # o la version que esté instalada
```

> [!WARNING]
>
> Cuando termines de usar la BD, se recomienda bajar la instancia para evitar el
> consumo innecesario de recursos del computador. Si no lo hacemos, lo más
> probable es que se inicie el contenedor automáticamente cada vez que se inicie
> Docker:
>
> ```bash
> docker compose down  # desde la carpeta con el compose.yml
> ```

> [!TIP]
>
> Para eliminar completamente la BD, incluyendo la configuración y los datos, se
> puede eliminar el volúmen:
>
> ```bash
> docker compose down -v
> ```

## 1. Levantar el contenedor

Ejecutar el siguiente comando desde la misma ruta donde se encuentre el archivo
`compose.yml`:

```bash
docker compose up -d
```

## 2. Conexión administrativa

Para crear el usuario nos tenemos que conectar a la conexión administrativa
Container DataBase (CDB). Para ello apuntamos al **Service Name** `FREE`.

> [!NOTE]
>
> La conexión administrativa es una conexión especial con el usuario `SYSTEM`
> que viene por defecto en las BD Oracle (El password de esta conexión lo
> definimos nosotros dentro del `compose.yml` o de otras formas manuales).
>
> La conexión, se usa para configurar la BD, lo que incluye la creación de
> esquemas, la administración de usuarios, sus permisos y otros funcionamientos.

Como ya viene creada por defecto, en SQL Developer (o el cliente SQL que
utilicemos) simplemente se agrega la conexión:

| Campo            | Valor              | Nota                                             |
| ---------------- | ------------------ | ------------------------------------------------ |
| **Name**         | `OracleAdmin_CDB`  | O cualquier nombre                               |
| **Username**     | `SYSTEM`           |                                                  |
| **Password**     | `MiPasswordSeguro` | Definido en el `compose.yaml`                    |
| **Hostname**     | `localhost`        |                                                  |
| **Port**         | `1521`             |                                                  |
| **Service Name** | **`FREE`**         | **¡Importante!** No usar SID. Usar Service Name. |

## 3. Creación de usuarios

Una vez establecida la conexión en `OracleAdmin_CDB`, necesitamos:

- Crear un usuario
- Crear un esquema para ese usuario
- Otorgar permisos al usuario para que pueda escribir en el esquema creado.

Para todo esto, tenemos 2 opciones. Crear usuarios de _uno en uno_ o
automatizarlo con un script:

### Crear múltiples usuarios

Para los nombres de usuario vamos a utilizar esta forma `BDY1102_X` donde `X` es
el número de la práctica y `014V` la contraseña común:

```sql
ALTER SESSION SET CONTAINER = FREEPDB1;

BEGIN
   FOR i IN 1..17 LOOP
      EXECUTE IMMEDIATE 'CREATE USER BDY1102_' || i ||
                        ' IDENTIFIED BY "014V"' ||
                        ' DEFAULT TABLESPACE USERS' ||
                        ' TEMPORARY TABLESPACE TEMP';

      EXECUTE IMMEDIATE 'ALTER USER BDY1102_' || i || ' QUOTA UNLIMITED ON USERS';
      EXECUTE IMMEDIATE 'GRANT CREATE SESSION, RESOURCE TO BDY1102_' || i;
      EXECUTE IMMEDIATE 'ALTER USER BDY1102_' || i || ' DEFAULT ROLE RESOURCE';
   END LOOP;
END;
/
```

> [!NOTE]
>
> La primera línea determina el contenedor (no confundir con contenedores
> Docker) en el que se crean los usuarios. Entonces, para modificar permisos a
> futuro, verificar que se esté trabajando en el contenedor `FREEPDB1`:
>
> ```sql
> ALTER SESSION SET CONTAINER = FREEPDB1;
> ```

### Alternativa: Crear solo 1 usuario

```sql
CREATE USER C##BDY1102 IDENTIFIED BY "BDY1101practica_1"
DEFAULT TABLESPACE USERS
TEMPORARY TABLESPACE TEMP;
CREATE ROLE C##RESOURCE CONTAINER=ALL;
GRANT CREATE SESSION, CREATE TABLE, CREATE SEQUENCE, CREATE VIEW TO C##RESOURCE CONTAINER=ALL;
GRANT C##RESOURCE TO C##BDY1102 CONTAINER=ALL;
ALTER USER C##BDY1102 DEFAULT ROLE C##RESOURCE;
ALTER USER C##BDY1102 QUOTA UNLIMITED ON USERS;
```

Y nos conectamos a la **Pluggable Database (PDB)** con este usuario recién
creado. Estas son las credenciales (que definimos en las queries anteriores):

| Campo            | Valor               | Nota                                  |
| ---------------- | ------------------- | ------------------------------------- |
| **Name**         | `BDY1102_PDB`       | Conexión de trabajo                   |
| **Username**     | `C##BDY1102`        | El usuario creado en el paso anterior |
| **Password**     | `BDY1101practica_1` |                                       |
| **Hostname**     | `localhost`         |                                       |
| **Port**         | `1521`              |                                       |
| **Service Name** | **`FREEPDB1`**      | Apuntamos a la PDB específica         |

**Listo, todo debería estar funcionando.**

---

## 4. Conexión a los esquemas

Como ya tenemos creados los usuarios y esquemas, solo nos queda conectarnos a la
DB para utilizarla. En nuestro cliente SQL debemos:

- Crear la conexión con las credenciales que definimos
- Conectarnos

Estos son los datos de conexión para los esquemas con los usuarios
correspondientes. En el ejemplo se utiliza el usuario `1`:

| Campo            | Valor          | Nota                                             |
| ---------------- | -------------- | ------------------------------------------------ |
| **Username**     | `BDY1102_1`    | Cambiar el número según la práctica.             |
| **Password**     | `014V`         | Contraseña única para todos los esquemas.        |
| **Hostname**     | `localhost`    |                                                  |
| **Port**         | `1521`         |                                                  |
| **Service Name** | **`FREEPDB1`** | **Obligatorio:** Conexión a la PDB (no cambiar). |

---

## Fuente

Toda la info viene de aquí:

https://container-registry.oracle.com/ords/ocr/ba/database/free

Se adaptaron las instrucciones de Podman a Docker y finalmente al compose.
