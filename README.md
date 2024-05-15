# Upgrade sin Downtime

 > En este repositorio se encuentran los pasos para reproducir el ejercicio demostrado durante el meetup de apertura del PostgreSQL User Group México el 23 de mayo de 2024.

Durante este ejercicio se realizará una demostración de actualización de la version principal de PostgreSQL de 14 a 16 "sin" downtime para la aplicación. Para ello se utilizarán las siguientes características incluidas en los paquetes base de PostgreSQL:

- Replicación lógica entre diferentes versions de PostgreSQL (14 -> 16).
- Características agregadas (`target_session_attrs`) a las ultimas versiones de librería cliente `libpq` ([comenzando en version 14](https://www.postgresql.org/docs/release/14.0/)).


## Pre-requisitos
Para este ejercicio se requiere acces a tres servidores o maquinas virtuales que tengan comunicación de red entre ellos con resolución de nombres (DNS, hosts, etc). Puede utilizarse cualquier variante de sistema operativo Linux pero para este caso particular se usará Ubuntu 22.04. Las  máquinas tendran los siguientes roles:

- **pgclient**. Esta máquina será utilizada para ejecutar la herramienta [`pgbench`](https://www.postgresql.org/docs/current/pgbench.html) y así simular carga aplicativa. Además se usará para una instancia de [Percona Monitoring and Management (PMM)](https://www.percona.com/software/database-tools/percona-monitoring-and-management) para tener una mejor referencia visual del ejercicio. 
- **pg14**. Esta máquina será nuestra instancia principal de PostgreSQL, como el nombre indica ejecutará la version 14.
- **pg16**. Esta máquina será la nueva instancia principal, ejecutará la versión 16 de PostgreSQL y al final del ejercicio está soportará la carga de la aplicación.


## Instalando el software

Para este ejercicio usaremos los paquetes de comunidad de PostgreSQL (pgdg), las instrucciones específicas dependiendo del sistema operativo y versión pueden consultarse en la documentación en linea:

https://www.postgresql.org/download/

### pgclient
Instalaremos los paquetes de cliente de PostgreSQL, incluyendo la librería `libpq` y la herramienta `pgbench`. 
```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt install -y postgresql-client-14 postgresql-contrib=14+238
```
> La herramienta `pgbench` se instala con el paquete `postgresql-contrib`, en Ubuntu hay un par de puntos a tener en cuenta:
> 1. Si no se indica una version del paquete `postgresql-contrib` el administrador de paquetes **apt** instalará la última versión disponible; como en este caso usaremos la version 14, por ello se especificó `postgresql-contrib=14+238`. Para consultar las versiones disponibles de un paquete se puede usar el siguiente comando:
> ```bash
> sudo apt-cache policy postgresql-contrib
> ```
> 2. En Ubuntu, el paquete `postgresql-contrib` depende algunos paquetes más, incluyendo el paquete que contiene el *postgres server* (`postgresql-14`), como resultado se instalará e inicializará un nuevo servidor de PostgreSQL, en este caso no lo necesitamos en la máquina **pgclient**, por lo que lo eliminaremos con el siguiente comando:
> ```bash
> sudo pg_dropcluster 14 main --stop
> ```

Para ilustrar la actividad en las bases de datos, usaremos PMM. Para revisar instrucciones específicas para diferentes plataformas se pueden referir a la [documentación en linea](https://docs.percona.com/percona-monitoring-and-management/quickstart/index.html).
```bash
curl -fsSL https://www.percona.com/get/pmm | sudo /bin/bash
```
> Para acceder al Docker daemon con un usuario diferente de **root** podemos seguir las instrucciones en la [documentación de Docker](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user):
> ```bash
> sudo groupadd docker
> sudo usermod -aG docker $USER
> newgrp docker
> ```
> Después podremos ejecutar comandos de Docker con usuarios que no sean root, por ejemplo:
> ```bash
> docker ps
> docker exec pmm-server 'pmm-admin status'
> ```

### pg14
Instalaremos los paquetes del servidor de PostgreSQL version 14, los siguientes comandos instalarán los paquetes e inicializarán un nuevo servicio de PostgreSQL 14. 
```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt -y install postgresql-14
```
### pg16
Instalaremos los paquetes del servidor de PostgreSQL version 16, los siguientes comandos instalarán los paquetes e inicializarán un nuevo servicio de PostgreSQL 16.     
```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt -y install postgresql-16
```

## Configuración inicial

Como se indico originalmente, el objetivo del ejercicio es hacer una actualización/migración de PG14 a PG16 sin afectar la disponibilidad del servicio a la aplicación cliente. Para ello necesitamos la siguiente configuración básica.

### pg14
Haremos los siguientes ajustes a los parámetros de postgres para permitir la replicación lógica, y la conectividad de red desde equipos remotos.

Agregamos las siguientes lineas al archivo `pg_hba.conf` de tal modo que se puedan establecer sesiones remotas desde las otras máquinas.
```bash
echo 'host all  all pgclient    scram-sha-256' | sudo -u postgres tee -a /etc/postgresql/14/main/pg_hba.conf
echo 'host all  all pg16    scram-sha-256' | sudo -u postgres tee -a /etc/postgresql/14/main/pg_hba.conf
```
Creamos un usuario para ser dueño de la nueva base de datos llamado `pgbench` y la propia base de datos con el mismo nombre. 
```bash
sudo -u postgres psql -c "CREATE USER pgbench WITH PASSWORD 'pgbench123'"
sudo -u postgres createdb pgbench -O pgbench
```
Ajustamos los siguientes parámetros de PostgreSQL y reiniciamos el servicio, esto permitirá la conectividad por las interfaces de red de la máquina servidor y la configuración de la replicación logíca más adelante.
```bash
sudo -u postgres psql << EOF 
    ALTER SYSTEM SET listen_addresses TO '*';
    ALTER SYSTEM SET wal_level TO 'logical';
EOF
sudo systemctl restart postgresql@14-main
```

### pg16
Para unificar configuraciones y permitir la conectividad remota aplicaremos los mismos ajustes a las instancia de PG 16 y reiniciamos el servicio.
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

Para este ejemplo, la aplicación que generará la carga será el propio [`pgbench`](https://www.gnu.org/software/screen/manual/screen.html) por lo que desde la máquina **pgclient** iniciaremos la carga con el siguiente comando, ejecutalo desde una terminal que se pueda dejar abierta durante el ejercicio o usa alguna herramienta como [`screen`](https://www.gnu.org/software/screen/manual/screen.html) para ejecutarla en segundo plano. 
```bash
pgbench --client=20 \   # Se ejecutaran 20 sesiones cliente simultaneas
        --connect \     # Se iniciará una nueva conexión por cada transacción
        --jobs=4 \      # Numero de hilos o procesos a nivel de sistema operativo, util si se tienen multiples CPUs
        --time=3600 \   # La prueba se ejecutará durante una hora o hasta que se cancele con `ctrl+c`
        --progress=5 \  # La herramienta reportará cada 5 segundos las estadísticas principales
        "host=pg14,pg16 target_session_attrs=read-write connect_timeout=20 user=pgbench dbname=pgbench"
```
> La ultima linea del comando `pgbench` es la cadena de conexión a la base de datos, aquí estamos usando algunas de las características especiales de la librería `libpq` para ajustar como y a donde se deben de conectar las sesiones cliente:
> - **host=**. Esta opción admite multiples servidores o IPs, los cuales serán intentados en el orden de aparición.
> - **target_session_attrs=**. Con este parámetro estamos indicando que las sesiones cliente deben establecerse solo en instancias que admiten operaciones de lectura y escritura, en el caso de instancias replica solo admiten operaciones de lectura. 
> - **connect_timeout=**. Esta opción define en cuantos segundos debe de esperar el cliente antes de marcar como fallida una conexión. En este caso el valor de 20 es muy alto, pero nos ayudará cuando hagamos el cambio a la nueva instancias de PostgreSQL 16.
> Referencia: https://www.postgresql.org/docs/current/libpq-connect.html
