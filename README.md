# Upgrade sin Downtime

 > En este repositorio se encuentran los pasos para reproducir el ejercicio demostrado durante el meetup de apertura del PostgreSQL User Group México el 23 de mayo de 2024.

Durante este ejercicio se realizará una demostración de actualización de la versión principal de PostgreSQL de 14 a 16 "sin" downtime para la aplicación. Para ello se utilizarán las siguientes características incluidas en los paquetes base de PostgreSQL:

- Replicación lógica entre diferentes versiones de PostgreSQL (14 -> 16).
- Características agregadas (`target_session_attrs`) a las últimas versiones de librería cliente `libpq` ([comenzando en version 14](https://www.postgresql.org/docs/release/14.0/)).


## Pre-requisitos
Para este ejercicio se requiere acceso a tres servidores o máquinas virtuales que tengan comunicación de red entre ellos con resolución de nombres (DNS, hosts, etc). Puede utilizarse cualquier variante de sistema operativo Linux pero para este caso particular se usará Ubuntu 22.04. Las  máquinas tendrán los siguientes roles:

- **pgclient**. Esta máquina será utilizada para ejecutar la herramienta [`pgbench`](https://www.postgresql.org/docs/current/pgbench.html) y así simular carga aplicativa. 
- **pg14**. Esta máquina será nuestra instancia principal de PostgreSQL, como el nombre indica, ejecutará la versión 14.
- **pg16**. Esta máquina será la nueva instancia principal, ejecutará la versión 16 de PostgreSQL y al final del ejercicio soportará la carga de la aplicación.


## Instalando el software

Para este ejercicio usaremos los paquetes de comunidad de PostgreSQL (pgdg), las instrucciones específicas dependiendo del sistema operativo y versión pueden consultarse en la documentación en línea:

https://www.postgresql.org/download/

### pgclient
Instalaremos los paquetes de cliente de PostgreSQL, incluyendo la librería `libpq` y la herramienta `pgbench`. 
```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt install -y postgresql-client-14 postgresql-contrib=14+238
```
> La herramienta `pgbench` se instala con el paquete `postgresql-contrib`, en Ubuntu hay un par de puntos a tener en cuenta:
> 1. Si no se indica una versión del paquete `postgresql-contrib` el administrador de paquetes **apt** instalará la última versión disponible; como en este caso usaremos la versión 14, por ello se especificó `postgresql-contrib=14+238`. Para consultar las versiones disponibles de un paquete se puede usar el siguiente comando:
> ```bash
> sudo apt-cache policy postgresql-contrib
> ```
> 2. En Ubuntu, el paquete `postgresql-contrib` depende algunos paquetes más, incluyendo el paquete que contiene el *postgres server* (`postgresql-14`), como resultado se instalará e inicializará un nuevo servidor de PostgreSQL, en este caso no lo necesitamos en la máquina **pgclient**, por lo que lo eliminaremos con el siguiente comando:
> ```bash
> sudo pg_dropcluster 14 main --stop
> ```

### pg14
Instalaremos los paquetes del servidor de PostgreSQL versión 14, los siguientes comandos instalarán los paquetes e inicializarán un nuevo servicio de PostgreSQL 14. 
```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt -y install postgresql-14
```
### pg16
Instalaremos los paquetes del servidor de PostgreSQL versión 16, los siguientes comandos instalarán los paquetes e inicializarán un nuevo servicio de PostgreSQL 16.     
```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt -y install postgresql-16
```

## Configuración inicial

Como se indicó originalmente, el objetivo del ejercicio es hacer una actualización/migración de PG14 a PG16 sin afectar la disponibilidad del servicio a la aplicación cliente. Para ello necesitamos la siguiente configuración básica.

### pg14
Haremos los siguientes ajustes a los parámetros de postgres para permitir la replicación lógica, y la conectividad de red desde equipos remotos.

Agregamos las siguientes líneas al archivo `pg_hba.conf` de tal modo que se puedan establecer sesiones remotas desde las otras máquinas.
```bash
echo 'host all  all pgclient    scram-sha-256' | sudo -u postgres tee -a /etc/postgresql/14/main/pg_hba.conf
echo 'host all  all pg16    scram-sha-256' | sudo -u postgres tee -a /etc/postgresql/14/main/pg_hba.conf
```
Creamos un usuario para ser dueño de la nueva base de datos llamado `pgbench` y la propia base de datos con el mismo nombre. 
```bash
sudo -u postgres psql -c "CREATE USER pgbench WITH PASSWORD 'pgbench123'"
sudo -u postgres createdb pgbench -O pgbench
```
Ajustamos los siguientes parámetros de PostgreSQL y reiniciamos el servicio, esto permitirá la conectividad por las interfaces de red de la máquina servidor y la configuración de la replicación lógica más adelante.
```bash
sudo -u postgres psql << EOF 
    ALTER SYSTEM SET listen_addresses TO '*';
    ALTER SYSTEM SET wal_level TO 'logical';
EOF
sudo systemctl restart postgresql@14-main
```

### pg16
Para unificar configuraciones y permitir la conectividad remota aplicaremos los mismos ajustes a la instancia de PG 16 y reiniciamos el servicio.
```bash
echo 'host all  all pgclient    scram-sha-256' | sudo -u postgres tee -a /etc/postgresql/16/main/pg_hba.conf
echo 'host all  all pg14    scram-sha-256' | sudo -u postgres tee -a /etc/postgresql/16/main/pg_hba.conf
```
```bash
sudo -u postgres psql -c "ALTER SYSTEM SET listen_addresses TO '*'"
sudo systemctl restart postgresql@16-main
```

### pgclient
Creamos un [archivo de contraseñas](https://www.postgresql.org/docs/current/libpq-pgpass.html) para evitar tener que introducir la contraseña del usuario **pgbench** cada vez.
```bash
echo '*:5432:pgbench:pgbench:pgbench123' >> ~/.pgpass
chmod 0600 ~/.pgpass
```
Ahora vamos a inicializar una base de datos de pruebas con `pgbench` en el servidor **pg14**.
```bash
pgbench --initialize --scale=30 --host=pg14 --username=pgbench pgbench
```


## Simulación de carga de aplicación

Para este ejemplo, la aplicación que generará la carga será el propio `pgbench` por lo que desde la máquina **pgclient** iniciaremos la carga con el siguiente comando, ejecutalo desde una terminal que se pueda dejar abierta durante el ejercicio o usa alguna herramienta como [`screen`](https://www.gnu.org/software/screen/manual/screen.html) para ejecutarla en segundo plano. 
```bash
pgbench --client=20 \
        --connect \
        --jobs=4 \
        --time=3600 \
        --progress=5 \
        "host=pg14,pg16 target_session_attrs=read-write connect_timeout=20 user=pgbench dbname=pgbench"
```
> `--client=20`    Se ejecutarán 20 sesiones cliente simultáneas
> `--connect`      Se iniciará una nueva conexión por cada transacción
> `--jobs=4`       Número de hilos o procesos a nivel de sistema operativo, útil si se tienen múltiples CPUs
> `--time=3600`    La prueba se ejecutará durante una hora o hasta que se cancele con `ctrl+c`
> `--progress=5`   La herramienta reportará cada 5 segundos las estadísticas principales
>
> La última línea del comando `pgbench` es la cadena de conexión a la base de datos, aquí estamos usando algunas de las características especiales de la librería `libpq` para ajustar cómo y a dónde se deben de conectar las sesiones cliente:
> - **host=**. Esta opción admite múltiples servidores o IPs, los cuales serán intentados en el orden de aparición.
> - **target_session_attrs=**. Con este parámetro estamos indicando que las sesiones cliente deben establecerse sólo en instancias que admiten operaciones de lectura y escritura, en el caso de instancias de réplica, estás solo admiten operaciones de lectura. 
> - **connect_timeout=**. Esta opción define cuántos segundos debe de esperar el cliente antes de marcar como fallida una conexión. En este caso el valor de 20 es muy alto, pero nos ayudará cuando hagamos el cambio a la nueva instancia de PostgreSQL 16.
> 
> Referencia: https://www.postgresql.org/docs/current/libpq-connect.html


## Replicación lógica
Los siguientes pasos crearán una replicación lógica básica entre las instancias PG14 y PG16, de tal manera que todos los cambios de datos que pasan en PG14 se repliquen en PG16, para que eventualmente se puedan "mover" la carga de la aplicación a la nueva instancia.

### pg14
Creamos un usuario para la replicación lógica, configuramos su contraseña y asignamos los permisos requeridos.
```bash
sudo -u postgres createuser --no-superuser --replication replicator
sudo -u postgres psql pgbench << EOF
    ALTER USER replicator PASSWORD 'replicator123';
    GRANT pg_read_all_data TO replicator ;
EOF
```
Para este ejemplo modificaremos una tabla de `pgbench` para agregar un mecanismo de identificación de datos, esto ya que esta tabla en particular no tiene llave primaria. En un ambiente real, esta modificación puede causar una sobrecarga excesiva en el sistema, por lo que debe de analizarse cuidadosamente.
```bash
sudo -u postgres psql pgbench -c 'ALTER TABLE public.pgbench_history REPLICA IDENTITY FULL'
```
Creamos una [publicación](https://www.postgresql.org/docs/current/logical-replication-publication.html) para todas las tablas en la base de datos `pgbench`.
```bash
sudo -u postgres psql pgbench -c 'CREATE PUBLICATION my_pub FOR ALL TABLES'
```

### pg16
Creamos un archivo de contraseñas para el usuario **replicator**.
```bash
echo '*:5432:*:replicator:replicator123' >> ~/.pgpass
chmod 0600 ~/.pgpass
```
Con la ayuda de las herramientas [pg_dumpall](https://www.postgresql.org/docs/current/app-pg-dumpall.html) y [pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html) copiaremos los usuarios y estructura (sin datos) de todas las tablas desde PG14 a PG16.
```bash
pg_dumpall --roles-only --host=pg14 --username=replicator | sudo -u postgres psql
pg_dump --create --schema-only --no-publications --host=pg14 --username=replicator --dbname=pgbench | sudo -u postgres psql
```
Crearemos la [subscripción](https://www.postgresql.org/docs/current/sql-createsubscription.html) con lo que se copiarán los datos desde PG14 y se establecerá la replicación lógica.
```bash
sudo -u postgres psql pgbench << EOF
    CREATE SUBSCRIPTION my_sub CONNECTION 'host=pg14 user=replicator password=replicator123 port=5432 dbname=pgbench' PUBLICATION my_pub
EOF
```
Ahora modificáremos el parámetro [`default_transaction_read_only`](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-DEFAULT-TRANSACTION-READ-ONLY) para indicar que por ahora la instancia de PG16 solo admitirá sesiones de lectura, para así evitar que por error el cliente establezca sesiones en esta instancia. 
```bash
sudo -u postgres psql pgbench << EOF
    ALTER SYSTEM SET default_transaction_read_only TO 'off';
    SELECT pg_reload_conf();
EOF
```

## Moviendo carga aplicativa a nuevo PG16
En este punto la replicación lógica se está encargando de replicar los datos entre ambas instancias, de PG14 a PG16, y la conexión cliente está configurada para establecer sesión a cualquiera de las dos instancias siempre y cuando acepten operaciones de lectura y escritura. Para "mover" la carga de la aplicación a la nueva instancia ejecutaremos los siguientes pasos:

### pg16
```bash
sudo -u postgres psql pgbench << EOF
    ALTER SYSTEM SET default_transaction_read_only TO 'off';
    SELECT pg_reload_conf();
EOF
```

### pg14
```bash
sudo -u postgres psql pgbench << EOF
    ALTER SYSTEM SET default_transaction_read_only TO 'on';
    SELECT pg_reload_conf();
EOF
```





