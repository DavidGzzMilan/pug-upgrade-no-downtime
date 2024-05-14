# Upgrade sin Downtime

 > En este repositorio se encuentran los pasos para reproducir el ejercicio demostrado durante el meetup de apertura del PostgreSQL User Group México el 23 de mayo de 2024.

Durante este ejercicio se realizará una demostración de actualización de la version principal de PostgreSQL de 14 a 16 "sin" downtime para la aplicación. Para ello se utilizarán las siguientes características incluidas en los paquetes base de PostgreSQL:

- Replicación lógica entre diferentes versions de PostgreSQL (14 -> 16).
- Características agregadas (`target_session_attrs`) a las ultimas versiones de librería cliente `libpq` ([comenzando en version 14](https://www.postgresql.org/docs/release/14.0/)).


## Pre-requisitos
Para este ejercicio se requiere acces a tres servidores o maquinas virtuales que tengan comunicación de red entre ellos. Puede utilizarse cualquier variante de sistema operativo Linux pero para este caso particular se usará Ubuntu 22.04. Las  máquinas tendran los siguientes roles:

- **pgclient**. Esta máquina será utilizada para ejecutar la herramienta [`pgbench`](https://www.postgresql.org/docs/current/pgbench.html) y así simular carga aplicativa. Además se usará para una instancia de [Percona Monitoring and Management (PMM)](https://www.percona.com/software/database-tools/percona-monitoring-and-management) para tener una mejor referencia visual del ejercicio. 
- **pg14**. Esta máquina será nuestra instancia principal de PostgreSQL, como el nombre indica ejecutará la version 14.
- **pg16**. Esta máquina será la nueva instancia principal, ejecutará la versión 16 de PostgreSQL y al final del ejercicio está soportará la carga de la aplicación.


## Instalando el software

Para este ejercicio usaremos los paquetes de comunidad de PostgreSQL (pgdg), las instrucciones específicas dependiendo del sistema operativo y versión pueden consultarse en la documentación en linea:

https://www.postgresql.org/download/

### pgclient

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
```bash
psql --version
```

### pg14
```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt -y install postgresql-14
```
### pg16    
```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt -y install postgresql
```
