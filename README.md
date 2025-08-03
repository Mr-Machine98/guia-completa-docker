# Docker Guide

Este es una guia de docker, del curso de java Microservicios Gui'a Completa de Docker & Kubernetes.
> [!NOTA]
> Comandos y configuraciones basadas en un contenido de microservicios de la siguiente forma:  
```text
                   ┌──────────────┐           ┌──────────────┐
                   │  PostgreSQL  │◄────────►│  ms-cursos   │◄─────┐
                   └──────────────┘           └──────────────┘     │
                                                                 ◄─────► Comunicación entre MS
                   ┌──────────────┐           ┌──────────────┐     │
                   │   MySQL      │◄────────►│ ms-usuarios   │◄────┘
                   └──────────────┘           └──────────────┘
```
## Generando archivo Jar para dockerizar

Enpaquetamos la aplicacio'n o aplicaicones con: 
```sh
mvn clean package -DskipTests
```
Ese jar que se genera es el que podemos correr en produccio'n.

- Podemos levantar nuestra app empaquetada con el siguiente comando:
```bash
java -jar ./target/ms-usuarios-0.0.1-SNAPSHOT.jar
```

## Creando archivo dockerfile usando imagen OpenJDK

Para crear una imagen simpre se debe de guiar de otra imagen, creamos el archivo `Dockerfile` y establecemos lo siguiente:
```docker
# Nos basamos de una imagen jdk 17
FROM openjdk:17.0.2

# Esta es nuestra carpeta de trabajo dentro de nuestro OS
WORKDIR /app

# Copie desde nuestro proyecto el .jar empaquetado
# y mande eso a la carpeta de destino usando "."
# que por defecto la carpeta es /app
COPY ./target/ms-usuarios-0.0.1-SNAPSHOT.jar .

# Documenta que la aplicación escucha en el puerto 8001.
# Esto no abre el puerto; simplemente informa a otros usuarios del contenedor que ese puerto debería abrirse al usar la imagen. Para abrirlo en ejecución debes usar -p 8001:8001 en docker run.
EXPOSE 8001

# Establecemos el punto de entrada, significa que cuando
# levantamos un contenedor basado en esta imagen, siempre
# se va a un punto de entrada ejecutando el comando, osea el siguiente:
ENTRYPOINT ["java", "-jar", "ms-usuarios-0.0.1-SNAPSHOT.jar"]
```
## Creando nuestra imagen y contenedor
Construya una imagen basada al archivo ```DockerFile``` que se encuentra ubicado en la misma ubicacio'n donde estamos, misma ruta, con el ".":
```bash
docker build .
```
Para ver nuestras ima'genes creadas:
```bash
docker images
```
Para crear un contenedor basado a una imagen, podriamos usar el ```IMAGE ID``` para levantar un contenedor:
```bash
docker run a64b0db5cdc7
```

## Importante
> Nota: si nuestra app dockerizada no tiene una `BD`, podemos decirle externamente que se comunique con la `BD` de nuestro computador base con `host.docker.internal`, esto tambien aplica para comunicaciones tipo cliente (feign)
```yml
spring.datasource.url=jdbc:mysql://host.docker.internal:3306/ms_usuarios
```
## Visualizacion de contenedores
Para ver los contenedores que solo se estan ejecutando, puedes realizarlo con:
```bash
docker ps
```
y para verlos todos...
```bash
docker ps -a
```
Para detener un contenendor
```bash
docker ps -a
```
## Eliminacion de contenedores
Para eliminar un contendor escribimos
`docker rm ${id_container}`
```bash
docker rm bbc34a3daa81
```
Eliminar imagen `docker rmi ${id_image}`
```bash
docker rmi a64b0db5cdc7
```
## Asignacion de puertos a nuestro contenedor
Para que nuestro contenedor se logre comunicar definimos puertos, el __Exterior__ e __Interior__, el exterior permite la comunicacio'n desde nuestro compu base a nuestro contenedor; y el interior permite la comunicacio'n desde dentro del contenedor haci'a nuestra app Dockerizada expuesta en dicho `id_puerto`.
- docker run -p `${external_port}`:`${internal_port}` `${id_image}`
```bash
docker run -p 9012:8001 a459a2b96657
```
## Asignacion de tag a imagen al momento de crearla
Para crear una imagen con un tag en especifico usamos el comando:
```bash
docker build -t usuarios .
```
Donde `-t` es tag seguido de `${tag_name}`
## Optimizando dockerfile
Para que automaticamente el docker file genere el package para nuestra app, podemos agregar esta linea de comando en el `dockerfile`, y hacer automaticamente todo:
```docker
# Nos basamos de una imagen jdk 17
FROM openjdk:17.0.2

WORKDIR /app/ms-usuarios

COPY ./pom.xml /app
COPY ./ms-usuarios /app/ms-usuarios

# Comando
RUN ./mvn clean package -DskipTests

EXPOSE 8001

ENTRYPOINT ["java", "-jar", "./target/ms-usuarios-0.0.1-SNAPSHOT.jar"]
```
>[!NOTE]
>
> Realizar este dockerfile tiene sus desventajas ya que se vuelve pesado, siempre descargara las dependencias

Para ejecutar este docker file es basado en el siguiente contenido de estructura de carpetas

```bash
/d/Eclipse/workspace2/curso-kubernetes/
├── pom.xml
├── ms-usuarios/
│   ├── dockerfile
│   ├── pom.xml
│   ├── src/
│   └── target/ms-usuarios-0.0.1-SNAPSHOT.jar
```
Comando a ejecutar:
```bash
docker build -t usuarios1 . -f ms-usuarios/dockerfile
```
> docker build

Le dice a Docker que construya una imagen.
> -t usuarios1

Etiqueta la imagen como usuarios1, así puedes luego hacer docker run usuarios1.
>.

Es el contexto de construcción, es decir, el punto de partida desde donde Docker puede copiar archivos al contenedor. En este caso es el directorio actual: /d/Eclipse/workspace2/curso-kubernetes.
> -f ./ms-usuarios/dockerfile

Le dice a Docker que use el archivo dockerfile que está en la ruta ./ms-usuarios/ como el Dockerfile.

# SUPER OPTIMIZACION Explicación sencilla del Dockerfile para el microservicio `ms-usuarios` Multi-stage

```docker
FROM openjdk:17-jdk-alpine as builder

WORKDIR /app/ms-usuarios

COPY ./pom.xml /app
COPY ./ms-usuarios/.mvn ./.mvn
COPY ./ms-usuarios/mvnw .
COPY ./ms-usuarios/pom.xml .

RUN ./mvnw clean package -Dmaven.test.skip -Dmaven.main.skip -Dspring-boot.repackage.skip && rm -r ./target/

COPY ./ms-usuarios/src ./src

RUN ./mvnw clean package -DskipTests

FROM openjdk:17-jdk-alpine

WORKDIR /app

COPY --from=builder /app/ms-usuarios/target/ms-usuarios-0.0.1-SNAPSHOT.jar .

EXPOSE 8001

ENTRYPOINT ["java", "-jar", "ms-usuarios-0.0.1-SNAPSHOT.jar"]
```

Este `Dockerfile` se encarga de construir una imagen para ejecutar una aplicación Java (Spring Boot) llamada `ms-usuarios`. Se usa una técnica llamada **multietapa** para que la imagen final sea más ligera.

---

## 🧱 Primera etapa: `builder` (construcción del proyecto)

```dockerfile
FROM openjdk:17-jdk-alpine as builder
```
- Usa una imagen base de Java 17 sobre Alpine (ligera).
- Le ponemos un nombre a esta etapa: `builder`.

```dockerfile
WORKDIR /app/ms-usuarios
```
- Se crea una carpeta donde se trabajará dentro del contenedor.

```dockerfile
COPY ./pom.xml /app
COPY ./ms-usuarios/.mvn ./.mvn
COPY ./ms-usuarios/mvnw .
COPY ./ms-usuarios/pom.xml .
```
- Se copian archivos necesarios para preparar el proyecto Maven sin copiar todo el código fuente aún.
- Esto ayuda a que Docker use cache si el código no ha cambiado.

```dockerfile
RUN ./mvnw clean package -Dmaven.test.skip -Dmaven.main.skip -Dspring-boot.repackage.skip && rm -r ./target/
```
- Se ejecuta un primer `package` muy ligero para descargar las dependencias.
- Se omiten los tests, la compilación y el empaquetado del `.jar`.
- Luego se borra la carpeta `/target` para limpiar.

```dockerfile
COPY ./ms-usuarios/src ./src
```
- Ahora sí se copia el código fuente del proyecto.

```dockerfile
RUN ./mvnw clean package -DskipTests
```
- Se compila el proyecto y se genera el archivo `.jar`.
- No se ejecutan los tests.

---

## 🚀 Segunda etapa: imagen final para ejecutar

```dockerfile
FROM openjdk:17-jdk-alpine
```
- Otra vez Java 17, pero ahora solo para ejecutar, no para construir.

```dockerfile
WORKDIR /app
```
- Se define la carpeta de trabajo.

```dockerfile
COPY --from=builder /app/ms-usuarios/target/ms-usuarios-0.0.1-SNAPSHOT.jar .
```
- Se copia solo el `.jar` generado desde la etapa `builder`.

```dockerfile
EXPOSE 8001
```
- Indica que el contenedor usará el puerto 8001.

```dockerfile
ENTRYPOINT ["java", "-jar", "ms-usuarios-0.0.1-SNAPSHOT.jar"]
```
- Este es el comando que se ejecutará al iniciar el contenedor.

---

## ✅ ¿Por qué usar multietapa?

- **Más ligera**: Solo contiene lo necesario para correr la app.
- **Más rápida**: Aprovecha la cache de dependencias si el código no cambió.
- **Más limpia**: El código fuente y herramientas de construcción no quedan en la imagen final.

---

## Continuacion Comandos
```bash
docker start {container_id}
```
- Para volver a levantar un contenedor utilizamos.
---
```bash
docker run -d -p 8081:8001 usuarios3
```
- Para arrancar un contenedor que no adjunte los logs a la consola
---
```bash
docker attach {container_id}
docker logs -f {container_id}
```
- Para adjuntar los logs a una consola
---
```bash
docker stop {container_id}
```
- Detener un contenedor
---
```bash
docker logs {container_id}
```
- Mostrar logs del contenedor
---
```bash
docker start -a {container_id}
```
- Levantar contenedor con los logs adjuntos bloqueando la consola
---
```bash
docker container prune
```
- Elimina de un totazo todos los contenedores que esten en estatus `EXITED`
---
```bash
docker image prune --all
```
- Elimina de un totazo  las imagenes que no esten utilizando o referenciando un contenedor
---
```bash
docker run -p 8001:8001 -d --rm {image_id}
```
- Levanta un contenedor pero cuando lo detengamos se eliminara automaticamente.
---
## Ingresando en modo interactivo en contenedores
 Para poder ingresar en modo interactivo debemos modificar el `ENTRYPORT` por `CMD`, esto porque `ENTRYPORT` es mas estricto en terminos que si y solo si se va ejecutar el comando que le definamos, mientras que `CMD` permite que podamos ignorar el comando y colocar uno nuevo desde consola.
 ```dockerfile
 # Mas estricto solo ejecutara este comando asi le colocamos otro desde el docker run
 ENTRYPOINT ["java", "-jar", "ms-usuarios-0.0.1-SNAPSHOT.jar"]
 ```
 ```dockerfile
 # Mas flexible, admite otro comando si le colocamos otro comando desde el docker run
 CMD ["java", "-jar", "ms-usuarios-0.0.1-SNAPSHOT.jar"]
 ```
 por lo cual si usamos `CMD` podemos utilizar el modo interactivo y entrar al contenedor cuando se levante...
 ## 🐳 Comando Docker explicado: `docker run -p 8001:8001 --rm -it usuarios3 /bin/sh`

Este comando crea y ejecuta un contenedor a partir de la imagen `usuarios3`, mapea el puerto 8001, y abre una terminal interactiva dentro del contenedor.

---

### 🔍 Comando completo

```bash
docker run -p 8001:8001 --rm -it usuarios3 /bin/sh
```
---
## Copiando archivos hacia/desde el contenedor en ejecucion

Este comando copia un archivo `Login.java` desde tu máquina local hacia un contenedor Docker en ejecución.
`docker cp ./Login.java ${container_id}:/app/Login.java`

```bash
docker cp ./Login.java 20a7bc65b1a1:/app/Login.java
```
Si queremos hacer el proceso inverso de copiar archivos desde el contenedor a nuestra maquina local, es el siguiente comando:
```bash
docker cp 20a7bc65b1a1:/app/Login.java Login2.java
```
---
##  Más detalles sobre las imagenes y contenedores con el comando inspect
Para inspeccionar imagenes bata con el comando: `$ docker image inspect ${image_id}`
saldra todo el detalle de la imagen en cuestion: 
```bash
docker image inspect usuarios
```
---
El comando para los contenedores es similar
```bash
docker container inspect f455c0afaa94
```
---
##  Agregando TAG's versionamiento imagenes & contenedores
Para las imagenes queremos crear un versionamiento de una imagen como la siguiente forma, que se muestra en la columna `TAG`:
```bash
mrmac@machine98 MINGW64 /d/Eclipse/workspace2/curso-kubernetes (master)
$ docker images
REPOSITORY   TAG        IMAGE ID       CREATED          SIZE
usuarios     0-0-1-ms   c017eb38d0bd   25 minutes ago   390MB
usuarios     latest     c017eb38d0bd   25 minutes ago   390MB
```
podemos utilizar el comando:
```bash
docker build -t usuarios:0-0-1-ms . -f ./ms-usuarios/dockerfile
```
crea la imagen con su TAG.
Seguido podemos crear un contenedor con esta nueva imagen y su nuevo versionamiento pero ademas podemos agregarle un nombre al contenedor con las banderas `--name ${name}`:
```bash
docker run -p 9020:8001 --rm -d --name ms-usuarios  usuarios:0-0-1-ms
```
```bash
mrmac@machine98 MINGW64 /d/Eclipse/workspace2/curso-kubernetes (master)
$ docker ps
CONTAINER ID   IMAGE               COMMAND                  CREATED          STATUS          PORTS                                         NAMES
9e3e741640b4   usuarios:0-0-1-ms   "java -jar ms-usuari…"   6 seconds ago    Up 5 seconds    0.0.0.0:9020->8001/tcp, [::]:9020->8001/tcp   ms-usuarios
```
---

## Comunicacion entre contenedores

Redes en docker sirven par poder comunicar microservicios.

- Para crear una red usamos el comando: docker network create `${network_name}`
```bash
$ docker network create spring
```
---

Para listar las redes:
```bash
mrmac@machine98 MINGW64 /d/Eclipse/workspace2/curso-kubernetes (master)
$ docker network ls
NETWORK ID     NAME       DRIVER    SCOPE
481142e5c15e   bridge     bridge    local
216aaae2fcda   host       host      local
9ba3fe21921b   minikube   bridge    local
378b04c0833d   none       null      local
8f37a9b847c1   spring     bridge    local
```
## Asignacion Red a Contenedores
Para asignar una red a n docker basta cone el comando **docker run -p 8002:8002 -d --rm --name ms-cursos --network `${network_name}`  cursos:0.0.1**, ejemplo:
```bash
$ docker run -p 8001:8001 -d --rm --name ms-usuarios --network spring usuarios:0.0.1
2e4d1d4740dee352bca353e6471080c939609cae031317ee43ff3a09680042e4
```
```bash
$ docker run -p 8002:8002 -d --rm --name ms-cursos --network spring  cursos:0.0.1
c03295d8191556564e932e23741580e99c75d75dcb71436661a165968f0b2513
```
---
## Dockerizacion de Bases de Datos
Para descagarse una imagen de `DockerHub` usamos el comando (mysql version 8):
```shell
$ docker pull mysql:8
```
Ejemplo levantando un contenedor de BD
```bash
mrmac@machine98 MINGW64 /d/Eclipse/workspace2/curso-kubernetes (master)
$ docker images
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
usuarios     0.0.1     0b231448a6b3   2 days ago     390MB
cursos       0.0.1     fdfead054b85   2 days ago     388MB
mysql        8         a0f6c7786c75   3 months ago   777MB

$ docker run -p 3307:3306 -d --name mysql8 --network spring -e MYSQL_ROOT_PASSWORD=admin -e MYSQL_DATABASE=ms_usuarios mysql:8

```
Lanza un contenedor de MySQL en segundo plano, accesible desde tu máquina en el puerto 3307, con:
- Contraseña de root: admin
- Base de datos inicial: ms_usuarios
- Nombre del contenedor: mysql8
- Conectado a la red Docker llamada spring
---
Podemos descargar una imagen usando el commando `docker run`, ya que como localmente no encuentra la imagen automaticamente la descarga si hacer `docker pull`.
```bash
$ docker run -p 5433:5432 --name postgres14 --network spring -e POSTGRES_PASSWORD=admin -e POSTGRES_DB=ms_cursos -d postgres:14-alpine
```

## Comunicacion microservicio (Codigo) con microservicio con BD
Para comunicar un microservicio con codigo que utiliza una bd y dockerizamos esta bd para que se encuentre alojada en un micro ejemplo:

```text
                   ┌──────────────┐           ┌──────────────┐
                   │  PostgreSQL  │◄────────►│  ms-cursos    |
                   └──────────────┘           └──────────────┘
```
Necesitamos que estos 2 micros pertenescan al misma red, y para que se conozcan estos micros se debe hacer mediante el nombre que se les haya asignado, por ejemplo si el micro `ms-cursos` desea comunicarse con el microservicio bd `postgres`, en el codigo de `aplication.proporties` debe estar el nombre del micro que tiene alojada la `bd` en este caso postgres ejemplo:

```properties
spring.application.name=ms-usuarios
server.port=8001
# remplazar esta linea 
spring.datasource.url=jdbc:mysql://host.docker.internal:3306/ms_usuarios
spring.datasource.username=root
spring.datasource.password=admin
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.jpa.database-platform=org.hibernate.dialect.MySQL8Dialect
spring.jpa.generate-ddl=true
logging.level.org.hibernate.SQL=debug
logging.file.path=/app/logs
```
Se remplaza la linea `spring.datasource.url=`
```properties
spring.application.name=ms-usuarios
server.port=8001
# Por esta
spring.application.name=ms-usuarios
server.port=8001
spring.datasource.url=jdbc:mysql://mysql8:3306/ms_usuarios
spring.datasource.username=root
spring.datasource.password=admin
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.jpa.database-platform=org.hibernate.dialect.MySQL8Dialect
spring.jpa.generate-ddl=true
logging.level.org.hibernate.SQL=debug
logging.file.path=/app/logs
```
`mysql8` es le nombre del contenedor que tiene bd al estar este en la misma red que ms-usuarios se conoceran por su nombre.
# Persistencia de datos en los contenedores
`Volumenes`, son herramientas que nos ayuda a guardar los datos por ejemplo de algun contenedor de base de datos. Por ejemplo, creo un contenedor de bases de datos basado en `postgres`, y ya he hecho operaciones de datos. Si elimino el contenedor estos datos se eliminan, el hecho es que estos datos persistan, para eso existen los volumenes.

# 🐳 Explicación detallada del comando Docker BD + Volumen

```bash
docker run -p 3307:3306 -d --name mysql8 --network spring 
-e MYSQL_ROOT_PASSWORD=admin 
-e MYSQL_DATABASE=ms_usuarios 
-v data-mysql:/var/lib/mysql 
mysql:8
```

Este comando ejecuta un contenedor de MySQL 8 configurado para ser usado por una aplicación en Spring Boot. A continuación se detalla cada parte del comando:

---

## 📌 Parámetros del comando

| Opción | Descripción |
|--------|-------------|
| `-p 3307:3306` | Mapea el puerto **3306 del contenedor** (puerto de MySQL) al **puerto 3307 del host**. Puedes conectarte desde tu PC a `localhost:3307`. |
| `-d` | Ejecuta el contenedor en segundo plano (modo "detached"). |
| `--name mysql8` | Asigna el nombre `mysql8` al contenedor, útil para referenciarlo fácilmente. |
| `--network spring` | Conecta el contenedor a una red llamada `spring`, permitiendo comunicación con otros contenedores (por ejemplo, microservicios). |
| `-e MYSQL_ROOT_PASSWORD=admin` | Define la contraseña del usuario `root` como `admin`. |
| `-e MYSQL_DATABASE=ms_usuarios` | Crea automáticamente una base de datos llamada `ms_usuarios` al iniciar por primera vez. |
| `-v data-mysql:/var/lib/mysql` | Usa un volumen persistente llamado `data-mysql` para almacenar los datos de la base de datos. Esto permite que los datos no se pierdan al reiniciar o eliminar el contenedor. |
| `mysql:8` | Utiliza la imagen oficial de MySQL en su versión 8. |

---

Para listar los volumenes usar comando:
```bash
docker volume ls
```
resultado:
```powershell
PS C:\Users\mrmac> docker volume ls
DRIVER    VOLUME NAME
local     2f692bf2c3411980534940c069fcf7113201e5f262105eb0e66ba7d7f9daed33
local     3719c9ed10b981ebdb5dc897d430a8faa5ec7ab6a969349f23d2399a98ed0fa2
local     data-mysql
local     data-postgres
PS C:\Users\mrmac> 
```
# Crear un contenedor cliente que consulte a otro contenedor que aloja una BD mysql, configurandolo en la misma red.

# 🐳 Explicación del flujo de conexión a MySQL desde un contenedor Docker

Este documento explica paso a paso el proceso y significado de los comandos usados para conectarse a un contenedor MySQL desde otro contenedor temporal.

---

## 🧱 Comando ejecutado desde PowerShell (Windows)

```powershell
PS C:\Users\mrmac> docker run -it --rm --network spring mysql:8 bash
```

### 🔍 ¿Qué hace este comando?

- `docker run`: Ejecuta un nuevo contenedor.
- `-it`: Modo interactivo con terminal.
- `--rm`: El contenedor se elimina al salir.
- `--network spring`: Conecta este contenedor a la red Docker llamada `spring`.
- `mysql:8`: Imagen base de MySQL versión 8.
- `bash`: Inicia una terminal `bash` en lugar del servidor MySQL.

Esto lanza un contenedor **temporal** con consola interactiva.

---

## 🐚 Dentro del contenedor: Conexión a MySQL

```bash
bash-5.1# mysql -h mysql8 -u root -p
```

### 🔍 ¿Qué hace este comando?

- `mysql`: Cliente MySQL.
- `-h mysql8`: Host al que se quiere conectar. En este caso, es el nombre del contenedor MySQL (`mysql8`) dentro de la red Docker.
- `-u root`: Usuario root.
- `-p`: Solicita contraseña.

### 🔐 Respuesta del sistema

```text
Enter password:
```

Aquí debes ingresar la contraseña del usuario root (en este ejemplo, `admin` si usaste `-e MYSQL_ROOT_PASSWORD=admin` al crear el contenedor de MySQL).

> ⚠️ Nota: Al escribir la contraseña, **no verás nada en pantalla**, es una medida de seguridad normal.

---

## ✅ Resultado esperado

Si todo está correcto, verás algo como:

```sql
mysql>
```

Eso significa que estás **dentro del cliente MySQL** y conectado correctamente al contenedor `mysql8`.

---

## 🎯 ¿Qué se logró?

- Verificaste que **la red Docker `spring` funciona**.
- Confirmaste que un contenedor cliente puede conectarse a otro (el servidor MySQL).
- Probaste la conectividad interna de tus microservicios en red.

---

## 🧪 Tip: Probar comandos SQL

Dentro de `mysql>` puedes probar comandos como:

```sql
SHOW DATABASES;
USE ms_usuarios;
SHOW TABLES;
```

---
Un simil seria con un cliente postges de la siguiente manera:
```powershell
PS C:\Users\mrmac> docker run -it --rm --network spring postgres:14-alpine bash 
82754a87e4a3:/# psql -h postgres14 -U postgres
Password for user postgres: 
psql (14.18)
Type "help" for help.

postgres=# \l
```
---
# Argumentos y variables de entorno
`ARGUMENTOS:` varaible que solo se pueden configurar  en el tiempo de construccion de la imagen, osea en el `dockerfile`, no en el contenedor.

`ENVIROMENT:` se pueden configurar en cualquier momento, en el `dockerfile`  y disponible tambien en nuestro `codigo`, o en el `contenedor`, etc.

## Variables de entorno 
Ya sabemos que nostros tenemos nuestro codigo, y que con ayuda de docker sacamos una imagen de ello, un croquis de lo que sera el contenedor, al sacar ese croquis podemos predefinir opciones `(ejemplo en el codigo que contiene la imagen)`, que nos permitan en tiempo de ejecucion cambiar esos parametros, sea alterar el puerto del contenedor, la base de datos etc.
```properties
spring.application.name=ms-usuarios
server.port=${PORT:8001}
        .
        .
        .
```
Ejemplo, `server.port=${PORT:8001}`, construimos una imagen basado a este proyecto y al tenener la imagen y definir en el `dockerfile` una variable referenciando al nombre que le pusimos en el `application.properties` con la palabra defnina `ENV` ejemplo.

```dockerfile
COPY --from=builder /app/ms-usuarios/target/ms-usuarios-0.0.1-SNAPSHOT.jar .

# Este es el puerto interno del contenedor
ENV PORT=8000

EXPOSE ${PORT}

CMD ["java", "-jar", "ms-usuarios-0.0.1-SNAPSHOT.jar"]
```
y el comando para levantar el cotntenedor debe tener el mismo puerto interno.
```powershell
docker run -p 8001:8000 -d --rm --name ms-usuarios-con-env --network spring usuarios:latest
```
---
> # Nota
> Ahora veamos como se puede hacer de forma dinamica

Podemos desde los comandos `docker` cambiar el puerto de nuestro contenedor ya que habiamos definido desde nuestro `application.properties` la variable `${PORT}`
```properties
spring.application.name=ms-usuarios
server.port=${PORT:8001}
        .
        .
        .
```
```powershell
docker run -p 8001:8090 --env PORT=8090 -d --rm --name ms-usuarios-ENV-command --network spring usuarios:latest
```
la aplicacion levantara con 8090.

---

> ## NOTA
> Podemos tambien apoyarnos de un archivo de configuracion .env para poder setterar las variables dinamicamente desde el comando `docker run`

Miremos, creamos el file `.env` y ponemos la varaible:

```
PORT=8091
```
luego con el comando docker run levantamos un contenedor y utilizamos este archivo para setear la varaible dinamicamente

```docker
docker run -p 8001:8091 --env-file ./ms-usuarios/.env -d --rm --name ms-usuarios-ENV-command --network spring usuarios:latest
```
---
---
---
---
---
---
---
Ahora seguimos con los `argumentos`, varaibles que podemos utilizar solo en el dockerfile, nada que tenga que ver en ejecucion del contenedor.

Ejemplo crearmos un argumento en el `dockerfile`
```dockerfile
ARG MS_NAME=ms-usuarios
```
Un ejemplo claro de uso.

```dockerfile
FROM openjdk:17-jdk-alpine as builder
ARG MS_NAME=ms-usuarios
WORKDIR /app/${MS_NAME}

COPY ./pom.xml /app
COPY ./${MS_NAME}/.mvn ./.mvn
COPY ./${MS_NAME}/mvnw .
COPY ./${MS_NAME}/pom.xml .

RUN ./mvnw clean package -Dmaven.test.skip -Dmaven.main.skip -Dspring-boot.repackage.skip && rm -r ./target/

COPY ./${MS_NAME}/src ./src

RUN ./mvnw clean package -DskipTests

# -----------------------------------------------------

FROM openjdk:17-jdk-alpine

WORKDIR /app

RUN mkdir ./logs
ARG MS_NAME=ms-usuarios
ARG TARGET_FOLDER=/app/${MS_NAME}/target/
COPY --from=builder ${TARGET_FOLDER}ms-usuarios-0.0.1-SNAPSHOT.jar .
ARG PORT_APP=8001
# Este es el puerto interno del contenedor
ENV PORT ${PORT_APP}

EXPOSE ${PORT}

CMD ["java", "-jar", "ms-usuarios-0.0.1-SNAPSHOT.jar"]
```
Podemos sobre escribir arguentos desde el comando `docker build` utilizando la bandera `--build-arg {VARIALBE_NAME=value}`, ejemplo:
```docker
docker build -t usuarios . -f ./ms-usuarios/dockerfile --build-arg PORT_APP=8080
```
---
---
---
---
## Seguimos con mas ejemplos de variables de entorno
Podemos mejorar y convertir datos del `aplication.properties` en varaibles de entorno:
```properties
spring.application.name=ms-usuarios
server.port=${PORT:8001}
spring.datasource.url=jdbc:mysql://mysql8:3306/ms_usuarios
spring.datasource.username=root
spring.datasource.password=admin
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.jpa.database-platform=org.hibernate.dialect.MySQL8Dialect
spring.jpa.generate-ddl=true
logging.level.org.hibernate.SQL=debug

logging.file.path=/app/logs
```
Por lo siguiente
```properties
spring.application.name=ms-usuarios
server.port=${PORT:8001}
spring.datasource.url=jdbc:mysql://${DB_HOST:mysql8:3306}/${DB_DATABASE:ms_usuarios}
spring.datasource.username=${DB_USERNAME:root}
spring.datasource.password=${DB_PASSWORD:admin}
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.jpa.database-platform=org.hibernate.dialect.MySQL8Dialect
spring.jpa.generate-ddl=true
logging.level.org.hibernate.SQL=debug

logging.file.path=/app/logs
```
y el archivo de configuraciones quedaria de la siguiente manera:
```
PORT=8001
DB_HOST=mysql8:3306
DB_DATABASE=ms_usuarios
DB_USERNAME=root
DB_PASSWORD=admin
```
Levantamos un contenedor usando el archivo de `.env` de configuraciones el de arriba.
```ps
docker run -p 8001:8001 -d --rm --name ms-usuarios --network spring --env-file ./ms-usuarios/.env usuarios:latest
```
podemos ver las variables de entrono setteadas con el comando:
```ps
docker container inspect ms-usuarios
```
resultado
```bash
 "StdinOnce": false,
            "Env": [
                "PORT=8001",
                "DB_HOST=mysql8:3306",
                "DB_DATABASE=ms_usuarios",
                "DB_USERNAME=root",
                "DB_PASSWORD=admin",
                "PATH=/opt/openjdk-17/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "JAVA_HOME=/opt/openjdk-17",
                "JAVA_VERSION=17-ea+14"
            ],
            "Cmd": [
                "java",
               
```
---
---
---
---
---
## Docker Compose optimizando todos los comandos en un archivo para ejecutar todo de una !!!!
Docker compose trabaja con las imagenes, las utliza, solo cambia la forma en como se crea y ejecuta los contenedores, basicamente creo un archivo manifiesto que declara configuraciones para que me levante todo mi proyecto.

Un ejemplo de manifesto:

```yml
version: "3.9"
services:
   mysql8:
      image: mysql:8
      ports:
         - "3307:3306"
      environment:
         MYSQL_ROOT_PASSWORD: admin
         MYSQL_DATABASE: ms_usuarios
      volumes:
         - data-mysql:/var/lib/mysql
      restart: always
      networks:
         - spring
   postgres14:
      image: postgres:14-alpine
      ports:
         - "5433:5432"
      environment:
         POSTGRES_PASSWORD: admin
         POSTGRES_DB: ms_cursos
      volumes:
         - data-postgres:/var/lib/postgresql/data
      restart: always
      networks:
         - spring
   ms-usuarios:
      image: usuarios:latest
      ports:
         - "8001:8001"
      env_file: ./ms-usuarios/.env
      networks:
         - spring
      depends_on:
         - mysql8
      restart: always
   ms-cursos:
      image: cursos:latest
      ports:
         - "8002:8002"
      env_file:
         - ./ms-cursos/.env
      networks:
         - spring
      depends_on:
         - postgres14 
      restart: always
volumes:
   data-mysql:
   data-postgres:
networks:
   spring:
```

Observe que las declaraciones son iguales al levantar con los comandos que hemos venido trabajando

## Conmandos DockerCompose

El comando `docker-compose up -d` debe ejecutarse en la ruta donde se encuentre el archivo `docker-compose.yml`, este comando ejecuta el manifiesto y le agrega la bandera `-d`, para que los logs no se adjunten en la consola, ejemplo:
```ps
mrmac@machine98 MINGW64 /d/Eclipse/workspace2/curso-kubernetes (master)
$ docker-compose up -d
time="2025-07-31T22:41:12-05:00" level=warning msg="D:\\Eclipse\\workspace2\\curso-kubernetes\\docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion"
[+] Running 7/7
 ✔ Network curso-kubernetes_spring           Created                                                                                                                                                                           0.1s 
 ✔ Volume "curso-kubernetes_data-mysql"      Created                                                                                                                                                                           0.0s 
 ✔ Volume "curso-kubernetes_data-postgres"   Created                                                                                                                                                                           0.0s 
 ✔ Container curso-kubernetes-mysql8-1       Started                                                                                                                                                                           0.7s 
 ✔ Container curso-kubernetes-postgres14-1   Started                                                                                                                                                                           0.7s 
 ✔ Container curso-kubernetes-ms-usuarios-1  Started                                                                                                                                                                           1.0s 
 ✔ Container curso-kubernetes-ms-cursos-1    Started
```
## Detiene y Elimina los contenedores

Con `docker-compose down`, detiene y elimina los contenedores que estan arriba:

```ps
mrmac@machine98 MINGW64 /d/Eclipse/workspace2/curso-kubernetes (master)
$ docker-compose down

time="2025-08-02T09:25:59-05:00" level=warning msg="D:\\Eclipse\\workspace2\\curso-kubernetes\\docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion"
[+] Running 5/5
 ✔ Container curso-kubernetes-ms-cursos-1    Removed                                                                                                                                                                           0.5s 
 ✔ Container curso-kubernetes-ms-usuarios-1  Removed                                                                                                                                                                           0.7s 
 ✔ Container curso-kubernetes-postgres14-1   Removed                                                                                                                                                                           0.3s 
 ✔ Container curso-kubernetes-mysql8-1       Removed                                                                                                                                                                           1.5s 
 ✔ Network curso-kubernetes_spring           Removed                                                                                                                                                                           0.4s 

mrmac@machine98 MINGW64 /d/Eclipse/workspace2/curso-kubernetes (master)
```

Para eliminar en conjunto con los volumenes que crea automaticamente agregamos la bandera `-v`.

## docker-compose down -v
```ps
mrmac@machine98 MINGW64 /d/Eclipse/workspace2/curso-kubernetes (master)
$ docker-compose down -v
time="2025-08-02T09:30:48-05:00" level=warning msg="D:\\Eclipse\\workspace2\\curso-kubernetes\\docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion"
[+] Running 2/2
 ✔ Volume curso-kubernetes_data-postgres  Removed                                                                                                                                                                              0.0s 
 ✔ Volume curso-kubernetes_data-mysql     Removed                                                                                                                                                                              0.0s 

mrmac@machine98 MINGW64 /d/Eclipse/workspace2/curso-kubernetes (master)
$
```
> Recorderis
> --
> Para listar los `volumenes` usamos:
>> docker volume ls
>
> ~

## Mas ejemplos con el docker-compose.yml
```yml
version: "3.9"
services:
   mysql8:
      #Agregamos nombres
      container_name: mysql8
      image: mysql:8
      ports:
         - "3307:3306"
      environment:
         MYSQL_ROOT_PASSWORD: admin
         MYSQL_DATABASE: ms_usuarios
      volumes:
         - data-mysql:/var/lib/mysql
      restart: always
      networks:
         - spring
   postgres14:
      #Agregamos nombres
      container_name: postgres14
      image: postgres:14-alpine
      ports:
         - "5433:5432"
      environment:
         POSTGRES_PASSWORD: admin
         POSTGRES_DB: ms_cursos
      volumes:
         - data-postgres:/var/lib/postgresql/data
      restart: always
      networks:
         - spring
   ms-usuarios:
      #Agregamos nombres
      container_name: ms-usuarios
      image: usuarios:latest
      ports:
         - "8001:8001"
      env_file: ./ms-usuarios/.env
      networks:
         - spring
      depends_on:
         - mysql8
      restart: always
   ms-cursos:
      #Agregamos nombres
      container_name: ms-cursos
      image: cursos:latest
      ports:
         - "8002:8002"
      env_file:
         - ./ms-cursos/.env
      networks:
         - spring
      depends_on:
         - postgres14 
      restart: always
volumes:
   data-mysql:
      # Podemos reutilizar un volumen ya existente y lo usamos colocando su nombre
      name: data-mysql
   data-postgres:
      # Podemos reutilizar un volumen ya existente y lo usamos colocando su nombre
      name: data-postgres
networks:
   spring:
```
## Encender y pausar Contenedores con docker docker-compose

Comandos para prender y apagar contenedores usando `docker-compose`


Enciende los micros

```docker
docker-compose start
```


Detiene los micros

```docker
docker-compose stop
```

## Build Images en docker-compose

Podemos construir imagenes haciendo el simil del comando:
```ps
docker build -t usuarios:0-0-1-ms . -f ./ms-usuarios/dockerfile
```
en el archivo de manifiesto de `docker-compose.yml`
Ejemplo:
```yml
version: "3.9"

services:

   mysql8:
      #Agregamos nombres
      container_name: mysql8
      image: mysql:8
      ports:
         - "3307:3306"
      environment:
         MYSQL_ROOT_PASSWORD: admin
         MYSQL_DATABASE: ms_usuarios
      volumes:
         - data-mysql:/var/lib/mysql
      restart: always
      networks:
         - spring
   
   postgres14:
      #Agregamos nombres
      container_name: postgres14
      image: postgres:14-alpine
      ports:
         - "5433:5432"
      environment:
         POSTGRES_PASSWORD: admin
         POSTGRES_DB: ms_cursos
      volumes:
         - data-postgres:/var/lib/postgresql/data
      restart: always
      networks:
         - spring
   
   ms-usuarios:
      #Agregamos nombres
      container_name: ms-usuarios
      # Aqui sobreescribimos image por build para para construir la image desde aqui
      build:
         context: ./
         dockerfile: ./ms-usuarios/dockerfile
      ports:
         - "8001:8001"
      env_file: ./ms-usuarios/.env
      networks:
         - spring
      depends_on:
         - mysql8
      restart: always
      
   ms-cursos:
      #Agregamos nombres
      container_name: ms-cursos
      # Aqui sobreescribimos image por build para para construir la image desde aqui
      build:
         context: ./
         dockerfile: ./ms-cursos/dockerfile
      ports:
         - "8002:8002"
      env_file:
         - ./ms-cursos/.env
      networks:
         - spring
      depends_on:
         - postgres14
         - ms-usuarios
      restart: always
volumes:
   data-mysql:
      # Podemos reutilizar un volumen ya existente y lo usamos colocando su nombre
      name: data-mysql
   data-postgres:
      # Podemos reutilizar un volumen ya existente y lo usamos colocando su nombre
      name: data-postgres
networks:
   spring:
```

utilizando esta configuracion en el manifiesto podemos construir las images.

Si queremos por una razon un cambio en el codigo no se, podemos reconstruir las imagenes obligando al docker-compose a hacerlo con el siguiente comando:

```ps
mrmac@machine98 MINGW64 /d/Eclipse/workspace2/curso-kubernetes (master)
$ docker-compose up --build -d
```
Este comando levanta y reconstruye las imagenes.

Para SOLO construir las imagenes solo se utiliza el comando:
```ps
docker-compose build
```



