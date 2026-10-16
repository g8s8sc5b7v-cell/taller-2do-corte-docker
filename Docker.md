# Docker: fundamentos y su rol en la Arquitectura de Software

**Curso:** Arquitectura de Software
**Taller:** Parte 1 — Investigativa
**Tema:** Docker 
**Estudiante:** Cristobal Casas Meza - T00084236
**Fecha:** 16 de octubre de 2026

---

## Introducción

Hoy es muy raro encontrar un proyecto de software profesional que no use contenedores. Docker se convirtió en la herramienta estándar para empaquetar aplicaciones y llevarlas de una máquina a otra sin sorpresas. En esta materia, además, Docker es la pieza concreta sobre la que corren muchos de los estilos arquitectónicos que hemos estudiado: un microservicio, un broker de eventos o una base de datos se ejecutan, en la práctica, como contenedores.

Este informe investiga qué es Docker, de qué se compone, qué funcionalidades ofrece, cuáles son sus comandos principales, qué problema resolvió y por qué es tan relevante al diseñar y desplegar arquitecturas de software.

---

## Desarrollo

### 1. ¿Qué es Docker?

**Definición.** Docker es una plataforma que permite empaquetar una aplicación junto con todo lo que necesita para funcionar (código, runtime, librerías y configuración) en una unidad estandarizada llamada **contenedor**. Ese paquete se puede ejecutar de la misma forma en cualquier computador que tenga Docker instalado.

**Cómo funciona a alto nivel.** Un contenedor no es una máquina completa, sino un *proceso normal del sistema operativo* al que se le aplica aislamiento. El kernel de Linux ofrece dos mecanismos para lograrlo: los *namespaces*, que hacen que el proceso solo "vea" sus propios procesos, red y sistema de archivos, y los *cgroups*, que limitan cuánta CPU y memoria puede consumir. Como el contenedor es solo un proceso aislado, **comparte el kernel del sistema operativo del host** en lugar de traer el suyo propio.

**Contenedor vs. máquina virtual.**

Para entender la diferencia conviene pensar en las capas de cada enfoque, de abajo hacia arriba. En una **máquina virtual** primero está el hardware, encima el sistema operativo del host y luego un *hipervisor* que reparte los recursos; sobre el hipervisor, cada VM trae su propio sistema operativo invitado completo, con sus librerías, y solo al final la aplicación. En un **contenedor** también empieza con el hardware y el sistema operativo del host (con su kernel), pero encima va únicamente el Docker Engine, y sobre este cada contenedor trae solo la aplicación y sus librerías: no hay sistema operativo invitado ni hipervisor, porque todos los contenedores usan el kernel del host.

Una VM necesita un sistema operativo invitado completo por cada instancia, lo que implica gigabytes de espacio y minutos de arranque. Un contenedor reutiliza el kernel del host y solo incluye la aplicación y sus dependencias, por lo que pesa megabytes y arranca en segundos. Por eso es más ligero y permite correr muchos más a la vez. *(Nota: en Windows y macOS, Docker Desktop ejecuta internamente una VM Linux pequeña que hace de host para los contenedores.)*

### 2. ¿Cuáles son sus componentes?

| Componente | Descripción |
|---|---|
| **CLI** (`docker`) | La herramienta de línea de comandos con la que el usuario da órdenes. No ejecuta nada por sí misma: envía las peticiones al daemon. |
| **Daemon** (`dockerd`) | El motor de Docker. Corre en segundo plano y hace el trabajo real: construye imágenes, crea, arranca y detiene contenedores, gestiona redes y volúmenes. |
| **Imagen** | Plantilla de solo lectura, formada por capas, que contiene la aplicación y sus dependencias. Es el "molde" para crear contenedores. |
| **Contenedor** | Instancia en ejecución de una imagen. Se puede crear, iniciar, detener y borrar; de una misma imagen salen muchos contenedores. |
| **Dockerfile** | Archivo de texto con las instrucciones paso a paso para construir una imagen (`FROM`, `COPY`, `RUN`, `CMD`...). |
| **Registro** (ej. Docker Hub) | Repositorio donde se almacenan y comparten imágenes. Se descargan con `pull` y se publican con `push`. |

**Cómo se relacionan.** El usuario escribe un comando en la CLI, esta lo envía al daemon mediante una API, y el daemon actúa sobre imágenes, contenedores y el registro.

En cuanto al flujo, la CLI y el daemon se comunican mediante una API REST: la CLI hace la petición y el daemon la ejecuta. Es el daemon quien toma el Dockerfile para construir la imagen, quien crea el contenedor a partir de esa imagen y quien se conecta con el registro para descargar (`pull`) o publicar (`push`) imágenes. De esta forma la imagen queda en el centro de todo: el Dockerfile la produce, el contenedor se crea a partir de ella y el registro la almacena y la distribuye.

**La cadena clave: Dockerfile → Imagen → Contenedor.** El *Dockerfile* es la receta; con `docker build` el daemon la ejecuta y produce una *imagen*; con `docker run` esa imagen se convierte en un *contenedor* en ejecución. El **registro** entra en dos momentos: antes de ejecutar, para *descargar* una imagen que alguien ya construyó (`docker pull`, o implícitamente en `docker run`), y después de construir, para *publicar* la nuestra y compartirla (`docker push`). Una analogía útil para programadores: la imagen es como una *clase* y el contenedor como un *objeto* instanciado de esa clase.

### 3. ¿Cuáles son sus principales funcionalidades?

1. **Portabilidad.** La misma imagen corre igual en el portátil del desarrollador, en el servidor de pruebas y en la nube, sin importar el sistema operativo instalado. *Ejemplo:* una API en Node.js empaquetada en Windows se ejecuta sin cambios en un servidor Linux de AWS.

2. **Aislamiento.** Cada contenedor tiene su propio sistema de archivos, red y procesos, así que las aplicaciones no se pisan entre sí. *Ejemplo:* un proyecto antiguo que necesita Python 2.7 y otro nuevo que usa Python 3.12 corren en la misma máquina sin conflicto.

3. **Versionado de imágenes.** Las imágenes se identifican con etiquetas (*tags*), lo que permite elegir y fijar versiones exactas y volver a una anterior si algo falla. *Ejemplo:* `mysql:8` y `mysql:5.7` conviven; si la versión `mi-api:2.0` falla, se regresa a `mi-api:1.9`.

4. **Reproducibilidad del entorno.** Como el entorno está descrito en un Dockerfile, cualquier integrante del equipo obtiene exactamente lo mismo. *Ejemplo:* un estudiante nuevo clona el repositorio, ejecuta `docker compose up` y tiene API y base de datos funcionando en minutos.

5. **Escalabilidad.** Arrancar más instancias es rápido y barato porque solo hay que crear más contenedores de la misma imagen. *Ejemplo:* ante un pico de tráfico, se levantan 5 contenedores de la API detrás de un balanceador de carga.

6. **Base para CI/CD.** Las imágenes se pueden construir y probar automáticamente en cada cambio y luego desplegar. *Ejemplo:* en cada `push` a GitHub, un pipeline construye la imagen, corre las pruebas dentro de un contenedor y la publica en un registro.

### 4. ¿Cuáles son sus principales comandos?

**a) Tabla de comandos**

| Comando | ¿Qué hace? | Ejemplo |
|---|---|---|
| `docker pull` | Descarga una imagen desde un registro. | `docker pull mysql:8` |
| `docker run` | Crea y arranca un contenedor a partir de una imagen. | `docker run -d --name web -p 8080:80 nginx` |
| `docker ps` | Lista los contenedores que están en ejecución. | `docker ps` |
| `docker ps -a` | Lista todos los contenedores, incluidos los detenidos. | `docker ps -a` |
| `docker logs` | Muestra la salida (logs) de un contenedor. | `docker logs -f web` |
| `docker exec` | Ejecuta un comando dentro de un contenedor que ya está corriendo. | `docker exec -it web bash` |
| `docker stop` / `start` | Detiene / vuelve a iniciar un contenedor existente. | `docker stop web` · `docker start web` |
| `docker rm` | Elimina un contenedor (debe estar detenido, o usar `-f`). | `docker rm web` |
| `docker images` | Lista las imágenes guardadas localmente. | `docker images` |
| `docker rmi` | Elimina una imagen local. | `docker rmi nginx` |
| `docker build` | Construye una imagen a partir de un Dockerfile. | `docker build -t mi-api:1.0 .` |
| `docker compose up` | Levanta todos los servicios definidos en un `compose.yaml`. | `docker compose up -d` |
| `docker push` | Publica una imagen en un registro. | `docker push usuario/mi-api:1.0` |

**b) Tres comandos en detalle**

**`docker run`** — es el comando más usado: crea y arranca el contenedor. Parámetros comunes:

- `-d` (*detached*): lo ejecuta en segundo plano para no ocupar la terminal.
- `--name`: le da un nombre propio, más fácil de recordar que el aleatorio.
- `-p host:contenedor`: publica un puerto; `-p 8080:80` hace que `localhost:8080` llegue al puerto 80 del contenedor.
- `-e CLAVE=valor`: define una variable de entorno (configuración, contraseñas de desarrollo).
- `-v origen:destino`: monta un volumen para que los datos sobrevivan aunque el contenedor se elimine.

```bash
docker run -d --name mysql-dev -p 3307:3306 \
  -e MYSQL_ROOT_PASSWORD=secreto \
  -v mysql-data:/var/lib/mysql \
  mysql:8
```

*Cuándo lo usaría:* para tener una base de datos MySQL lista para desarrollo sin instalarla en mi computador. Uso el puerto 3307 en el host por si ya tengo un MySQL local en el 3306, y el volumen para no perder los datos al borrar el contenedor.

**`docker build`** — construye una imagen leyendo un Dockerfile. Parámetros comunes:

- `-t nombre:etiqueta`: asigna nombre y versión a la imagen.
- `-f ruta`: indica un Dockerfile con otro nombre o ubicación.
- `--no-cache`: fuerza reconstruir todas las capas sin reutilizar caché.
- `.` (al final): el *contexto de construcción*, es decir, la carpeta cuyos archivos puede usar el Dockerfile.

```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

```bash
docker build -t mi-api:1.0 .
```

*Cuándo lo usaría:* cuando termino una nueva versión de mi API y quiero empaquetarla para probarla, compartirla con el equipo o subirla a un registro.

**`docker exec`** — ejecuta un comando dentro de un contenedor que ya está corriendo (a diferencia de `run`, no crea uno nuevo). Parámetros comunes:

- `-i` (modo interactivo) y `-t` (terminal), casi siempre juntos como `-it`.
- `-u usuario`: ejecuta el comando con otro usuario.
- `-e CLAVE=valor`: pasa una variable de entorno solo a ese comando.

```bash
docker exec -it mysql-dev mysql -u root -p
```

*Cuándo lo usaría:* para depurar. Si la aplicación se comporta raro, entro al contenedor a revisar archivos de configuración o consultar directamente la base de datos.

### 5. ¿Qué problema resuelve?

**El escenario "en mi máquina funciona".** Una aplicación no depende solo de su código, sino también de todo el entorno que la rodea. Cuando ese entorno cambia de una máquina a otra, la app puede fallar aunque el código sea idéntico. Las causas típicas son:

- **Versiones de dependencias distintas:** una librería o un runtime (Node, Python, Java) en una versión diferente.
- **Sistema operativo:** diferencias entre Windows, macOS y Linux (rutas, permisos, librerías del sistema).
- **Configuración:** variables de entorno, puertos o archivos de configuración que alguien olvidó.
- **Instalación manual:** pasos de instalación en un documento que nadie sigue igual, o que quedó desactualizado.

**Ejemplo antes vs. después.** Ana desarrolla una API con Node 18 y MySQL 8. Luis, que se une al proyecto, tiene Node 14 y MySQL 5.7. *Antes de Docker:* Luis sigue un README de 15 pasos, la instalación falla por versiones incompatibles y pierde medio día; cuando por fin funciona, en producción vuelve a fallar porque el servidor tiene otra configuración. *Con Docker:* el proyecto incluye un `Dockerfile` y un `compose.yaml` que definen Node 18 y MySQL 8. Luis ejecuta `docker compose up` y en pocos minutos tiene exactamente el mismo entorno que Ana, y el mismo que se despliega en producción.

### 6. ¿Por qué es importante dentro de una Arquitectura de Software?

**Un contenedor por servicio.** Conviene que la API, la base de datos y el broker de eventos corran cada uno en su propio contenedor porque así:

- cada servicio tiene sus propias dependencias y versiones, sin conflictos (incluso con lenguajes distintos);
- un fallo en un servicio queda contenido y no tumba a los demás;
- cada uno se puede actualizar, reiniciar o escalar de forma independiente;
- la arquitectura se refleja directamente en la infraestructura: un componente lógico = una imagen.

Un ejemplo mínimo de esto en `compose.yaml`:

```yaml
services:
  api:
    build: ./api
    ports: ["3000:3000"]
    depends_on: [db, broker]
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: secreto
  broker:
    image: rabbitmq:3-management
```

**Despliegue consistente.** La misma imagen que se construye una vez pasa sin modificarse por desarrollo, pruebas y producción. Lo único que cambia entre ambientes es la configuración (variables de entorno), no el software ni sus dependencias. Esto elimina la categoría de errores del tipo "en pruebas funcionaba".

**Escalabilidad y portabilidad a la nube.** Escalar un servicio es crear más contenedores de su imagen (por ejemplo, `docker compose up -d --scale api=3` con un balanceador delante). Además, prácticamente todos los proveedores y orquestadores (AWS ECS, Google Cloud Run, Azure Container Apps, Kubernetes) ejecutan contenedores, así que migrar de un proveedor a otro no exige reescribir la aplicación.

**Relación con los estilos arquitectónicos vistos:**

- **Microservicios:** es el encaje más natural. Cada microservicio se empaqueta como su propia imagen, se despliega y escala por separado y puede usar su propia tecnología.
- **SOA:** los servicios siguen siendo unidades independientes que se comunican por una interfaz; los contenedores simplifican desplegar y aislar cada servicio frente a esquemas más pesados basados en servidores compartidos.
- **Orientada a eventos:** el broker (RabbitMQ, Kafka) corre como contenedor, y los productores y consumidores de eventos son contenedores aparte. Si hay muchos eventos por procesar, se agregan más contenedores consumidores sin tocar el resto.

---

## Conclusión

Después de investigar, entiendo que Docker no es solo una herramienta para "instalar cosas más fácil", sino una pieza que hace posible, en la práctica, varias de las ideas centrales de la arquitectura de software: separar responsabilidades en componentes independientes, desplegarlos de forma repetible y escalarlos según la demanda. Lo más valioso para mí es la reproducibilidad: poder describir el entorno completo en archivos versionados junto al código y olvidarme del "en mi máquina funciona".

Usaría Docker en cualquier proyecto que tenga más de un componente, por ejemplo una aplicación con una API, una base de datos y un servicio de mensajería para notificaciones: con un solo `docker compose up` todo el equipo podría trabajar sobre el mismo entorno, y más adelante ese mismo empaquetado serviría para desplegarlo en la nube. Como siguiente paso me interesa aprender sobre orquestación (Kubernetes) para entender cómo se gestionan muchos contenedores en producción.

---

## Referencias

1. Docker Inc. *Docker overview*. Docker Docs. https://docs.docker.com/get-started/docker-overview/
2. Docker Inc. *Docker CLI reference*. Docker Docs. https://docs.docker.com/reference/cli/docker/
3. Docker Inc. *Dockerfile reference*. Docker Docs. https://docs.docker.com/reference/dockerfile/
4. Docker Inc. *Docker Hub*. https://hub.docker.com
5. Merkel, D. (2014). *Docker: lightweight Linux containers for consistent development and deployment*. Linux Journal, 2014(239).
6. Lewis, J. y Fowler, M. (2014). *Microservices*. https://martinfowler.com/articles/microservices.html
7. Material de clase: diapositiva `diapo-docker.html`, curso de Arquitectura de Software.