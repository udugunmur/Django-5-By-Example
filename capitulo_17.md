# Parte 4: Creación de una plataforma de E-Learning

## Capítulo 17: Puesta en producción (Going Live)

### Introducción

En el capítulo anterior, construiste un servidor de chat en tiempo real para estudiantes utilizando Django Channels. Ahora que has creado una plataforma de e-learning completamente funcional, necesitas configurar un entorno de producción para que pueda ser accesible a través de Internet. Hasta ahora, has estado trabajando en un entorno de desarrollo, utilizando el servidor de desarrollo de Django para ejecutar tu sitio. En este capítulo, aprenderás a configurar un entorno de producción que sea capaz de servir tu proyecto Django de una manera segura y eficiente.

Este capítulo cubrirá los siguientes temas:

- Configuración de los ajustes de Django para múltiples entornos
- Uso de Docker Compose para ejecutar múltiples servicios
- Configuración de un servidor web con uWSGI y Django
- Servir PostgreSQL y Redis con Docker Compose
- Uso del framework de comprobación del sistema de Django
- Servir NGINX con Docker
- Servir archivos estáticos a través de NGINX
- Asegurar conexiones a través de Transport Layer Security (TLS) / Secure Sockets Layer (SSL)
- Uso del servidor Asynchronous Server Gateway Interface (ASGI) Daphne para Django Channels
- Creación de un middleware personalizado de Django
- Implementación de comandos de gestión (*management commands*) personalizados de Django

En los capítulos anteriores, los diagramas al inicio representaban vistas, plantillas y funcionalidades de extremo a extremo. Este capítulo, sin embargo, cambia el enfoque hacia la configuración de un entorno de producción. En su lugar, encontrarás diagramas específicos para ilustrar la configuración del entorno a lo largo del capítulo.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter17](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter17).

Todos los módulos de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente que acompaña a este capítulo. Puedes seguir las instrucciones para instalar cada módulo de Python a continuación o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Creación de un entorno de producción

Es hora de desplegar tu proyecto Django en un entorno de producción. Comenzarás configurando los ajustes de Django para múltiples entornos y luego configurarás un entorno de producción.

#### Gestión de configuraciones para múltiples entornos

En proyectos del mundo real, tendrás que lidiar con múltiples entornos. Por lo general, tendrás al menos un entorno local para el desarrollo y un entorno de producción para servir tu aplicación. También podrías tener otros entornos, como entornos de prueba (*testing*) o de preproducción (*staging*).

Algunos ajustes del proyecto serán comunes a todos los entornos, pero otros serán específicos de cada entorno. Por lo general, utilizarás un archivo base que define los ajustes comunes y un archivo de configuración por entorno que anula los ajustes necesarios y define los adicionales.

Gestionaremos los siguientes entornos:

- `local`: El entorno local para ejecutar el proyecto en tu máquina.
- `prod`: El entorno para desplegar tu proyecto en un servidor de producción.

Crea un directorio `settings/` junto al archivo `settings.py` del proyecto `educa`. Cambia el nombre del archivo `settings.py` a `base.py` y muévelo al nuevo directorio `settings/`.

Crea los siguientes archivos adicionales dentro de la carpeta `settings/` para que el nuevo directorio se vea de la siguiente manera:

```text
settings/
    __init__.py
    base.py
    local.py
    prod.py
```

Estos archivos son los siguientes:

- `base.py`: El archivo de configuración base, que contiene las configuraciones comunes (anteriormente `settings.py`).
- `local.py`: Configuraciones personalizadas para tu entorno local.
- `prod.py`: Configuraciones personalizadas para el entorno de producción.

Has movido los archivos de configuración a un directorio un nivel por debajo, por lo que debes actualizar la configuración `BASE_DIR` en el archivo `settings/base.py` para que apunte al directorio principal del proyecto.

Al manejar múltiples entornos, crea un archivo de configuración base y un archivo de configuración para cada entorno. Los archivos de configuración de entorno deben heredar las configuraciones comunes y sobrescribir las configuraciones específicas del entorno.

Edita el archivo `settings/base.py` y reemplaza la siguiente línea:

```python
BASE_DIR = Path(__file__).resolve().parent.parent
```

Reemplaza la línea anterior con la siguiente:

```python
BASE_DIR = Path(__file__).resolve().parent.parent.parent
```

Apuntas a un directorio superior agregando `.parent` a la ruta `BASE_DIR`. Configuremos los ajustes para el entorno local.

#### Configuraciones del entorno local

En lugar de utilizar una configuración predeterminada para los ajustes `DEBUG` y `DATABASES`, los definirás para cada entorno de forma explícita. Estos ajustes serán específicos del entorno. Edita el archivo `educa/settings/local.py` y añade las siguientes líneas:

```python
from .base import *

DEBUG = True

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
```

Este es el archivo de configuración para tu entorno local. En este archivo, importas todas las configuraciones definidas en el archivo `base.py` y defines las configuraciones `DEBUG` y `DATABASES` para este entorno. Las configuraciones `DEBUG` y `DATABASES` siguen siendo las mismas que has estado utilizando para el desarrollo.

Ahora, elimina las configuraciones `DATABASES` y `DEBUG` del archivo de configuración `base.py`.

Los comandos de administración de Django no detectarán automáticamente el archivo de configuración a utilizar porque el archivo de configuración del proyecto no es el archivo `settings.py` predeterminado. Al ejecutar comandos de administración, debes indicar el módulo de configuración que deseas utilizar agregando una opción `--settings`, de la siguiente manera:

```bash
python manage.py runserver --settings=educa.settings.local
```

A continuación, validaremos el proyecto y la configuración del entorno local.

#### Ejecución del entorno local

Ejecutemos el entorno local utilizando la nueva estructura de configuración. Asegúrate de que Redis se esté ejecutando o inicia el contenedor Docker de Redis en una consola con el siguiente comando:

```bash
docker run -it --rm --name redis -p 6379:6379 redis:7.2.4
```

Ejecuta el siguiente comando de administración en otra consola, desde el directorio del proyecto:

```bash
python manage.py runserver --settings=educa.settings.local
```

Abre `http://127.0.0.1:8000/` en tu navegador y comprueba que el sitio se cargue correctamente. Ahora estás sirviendo tu sitio utilizando la configuración para el entorno local.

Si no deseas pasar la opción `--settings` cada vez que ejecutas un comando de administración, puedes definir la variable de entorno `DJANGO_SETTINGS_MODULE`. Django la utilizará para identificar el módulo de configuración a utilizar. Si estás utilizando Linux o macOS, puedes definir la variable de entorno ejecutando el siguiente comando en la consola:

```bash
export DJANGO_SETTINGS_MODULE=educa.settings.local
```

Si estás utilizando Windows, puedes ejecutar el siguiente comando en la consola:

```cmd
set DJANGO_SETTINGS_MODULE=educa.settings.local
```

Cualquier comando de administración que ejecutes después de esto utilizará la configuración definida en la variable de entorno `DJANGO_SETTINGS_MODULE`.

Detén el servidor de desarrollo de Django desde la consola presionando las teclas Ctrl + C y detén el contenedor Docker de Redis desde la consola presionando también las teclas Ctrl + C.

El entorno local funciona bien. Preparemos los ajustes para el entorno de producción.

#### Configuraciones del entorno de producción

Comencemos añadiendo las configuraciones iniciales para el entorno de producción. Edita el archivo `educa/settings/prod.py` y haz que se vea de la siguiente manera:

```python
from .base import *

DEBUG = False

ADMINS = [
    ('Antonio M', 'email@mydomain.com'),
]

ALLOWED_HOSTS = ['*']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
```

Estas son las configuraciones para el entorno de producción:

- `DEBUG`: Establecer `DEBUG` en `False` es necesario para cualquier entorno de producción. De lo contrario, la información de seguimiento (*traceback*) y los datos de configuración confidenciales quedarán expuestos a todos.
- `ADMINS`: Cuando `DEBUG` es `False` y una vista genera una excepción, toda la información se enviará por correo electrónico a las personas enumeradas en la configuración `ADMINS`. Asegúrate de reemplazar la tupla nombre/correo electrónico con tu propia información.
- `ALLOWED_HOSTS`: Por razones de seguridad, Django solo permitirá que los hosts incluidos en esta lista sirvan el proyecto. Por ahora, permites todos los hosts utilizando el símbolo de asterisco, `*`. Limitarás los hosts que se pueden utilizar para servir el proyecto más adelante.
- `DATABASES`: Mantienes la configuración de base de datos predeterminada que apunta a la base de datos SQLite de tu entorno local. Configurarás la base de datos de producción más adelante.

A lo largo de las siguientes secciones de este capítulo, completarás el archivo de configuración para tu entorno de producción.

Has organizado con éxito las configuraciones para manejar múltiples entornos. Ahora, construirás un entorno de producción completo configurando diferentes servicios con Docker.

---

### Uso de Docker Compose

Inicialmente utilizaste Docker en el Capítulo 3, Ampliar tu aplicación de blog, y has estado usando Docker a lo largo de este libro para ejecutar contenedores para diferentes servicios, como PostgreSQL, Redis y RabbitMQ.

Cada contenedor Docker combina el código fuente de la aplicación con las bibliotecas del sistema operativo y las dependencias necesarias para ejecutar la aplicación. Al utilizar contenedores de aplicaciones, puedes mejorar la portabilidad de tu aplicación. Para el entorno de producción, utilizaremos Docker Compose para construir y ejecutar múltiples contenedores Docker.

Docker Compose es una herramienta para definir y ejecutar aplicaciones multi-contenedor. Puedes crear un archivo de configuración para definir los diferentes servicios y usar un solo comando para iniciar todos los servicios desde tu configuración. Puedes encontrar información sobre Docker Compose en [https://docs.docker.com/compose/](https://docs.docker.com/compose/).

Para el entorno de producción, crearás una aplicación distribuida que se ejecuta en múltiples contenedores Docker. Cada contenedor Docker ejecutará un servicio diferente. Inicialmente definirás los siguientes tres servicios y agregarás servicios adicionales en las siguientes secciones:

- **Servicio web**: Un servidor web para servir el proyecto Django
- **Servicio de base de datos**: Un servicio de base de datos para ejecutar PostgreSQL
- **Servicio de caché**: Un servicio para ejecutar Redis

Comencemos instalando Docker Compose.

#### Instalación de Docker Compose a través de Docker Desktop

Puedes ejecutar Docker Compose en macOS, Linux de 64 bits y Windows. La forma más rápida de instalar Docker Compose es instalando Docker Desktop. La instalación incluye Docker Engine, la interfaz de línea de comandos y Docker Compose.

Instala Docker Desktop siguiendo las instrucciones en [https://docs.docker.com/compose/install/compose-desktop/](https://docs.docker.com/compose/install/compose-desktop/).

Abre la aplicación Docker Desktop y haz clic en Containers:

> *Figura 17.1: La interfaz de Docker Desktop*

Después de instalar Docker Compose, debes crear una imagen de Docker para tu proyecto Django.

#### Creación de un Dockerfile

Necesitas crear una imagen de Docker para ejecutar el proyecto Django. Un `Dockerfile` es un archivo de texto que contiene los comandos para que Docker ensamble una imagen de Docker. Prepararás un Dockerfile con los comandos para construir la imagen de Docker para el proyecto Django.

Junto al directorio del proyecto `educa`, crea un nuevo archivo y nómbralo `Dockerfile`. Añade el siguiente código al nuevo archivo:

```dockerfile
# Pull official base Python Docker image
FROM python:3.12.3

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Set work directory
WORKDIR /code

# Install dependencies
RUN pip install --upgrade pip
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copy the Django project
COPY . .
```

Este código realiza las siguientes tareas:

1. Se utiliza la imagen Docker principal oficial de Python 3.12.3. Puedes encontrar la imagen oficial de Python en [https://hub.docker.com/_/python](https://hub.docker.com/_/python).
2. Se establecen las siguientes variables de entorno:
   - `PYTHONDONTWRITEBYTECODE`: Esto evita que Python escriba archivos `.pyc`.
   - `PYTHONUNBUFFERED`: Esto garantiza que las transmisiones stdout y stderr de Python se envíen directamente a la terminal sin ser almacenadas primero en búfer.
3. El comando `WORKDIR` se utiliza para definir el directorio de trabajo de la imagen.
4. Se actualiza el paquete `pip` de la imagen.
5. El archivo `requirements.txt` se copia en el directorio de trabajo (`.`) de la imagen principal de Python.
6. Los paquetes de Python en `requirements.txt` se instalan en la imagen usando `pip`.
7. El código fuente del proyecto Django se copia del directorio local al directorio de trabajo (`.`) de la imagen.

Con este Dockerfile, has definido cómo se ensamblará la imagen de Docker que servirá a Django. Puedes encontrar la referencia de Dockerfile en [https://docs.docker.com/reference/dockerfile/](https://docs.docker.com/reference/dockerfile/).

#### Adición de los requisitos de Python

Se utiliza un archivo `requirements.txt` en el Dockerfile que creaste para instalar todos los paquetes de Python necesarios para el proyecto.

Junto al directorio del proyecto `educa`, crea un nuevo archivo y nómbralo `requirements.txt`. Si aún no lo has hecho, añade las siguientes líneas al archivo `requirements.txt`:

```text
asgiref>=3.8.1
Django~=5.2.0
Pillow==10.3.0
sqlparse==0.5.0
django-braces==1.16.0
django-embed-video==1.4.10
pymemcache==4.0.0
django-debug-toolbar==5.0.1
redis==5.2.1
django-redisboard==8.4.0
djangorestframework==3.15.2
requests==2.32.3
channels[daphne]==4.2.0
channels-redis==4.2.1
psycopg==3.2.6
uwsgi==2.0.28
python-decouple==3.8
setuptools==80.9.0
```

Además de los paquetes de Python que instalaste en los capítulos anteriores, el archivo `requirements.txt` incluye los siguientes paquetes:

- `psycopg`: Este es el adaptador de PostgreSQL. Utilizarás PostgreSQL para el entorno de producción.
- `uwsgi`: Un servidor web WSGI. Configurarás este servidor web más adelante para servir a Django en el entorno de producción.
- `python-decouple`: Paquete para cargar variables de entorno fácilmente.

Comencemos configurando la aplicación Docker en Docker Compose. Crearemos un archivo Docker Compose con la definición para el servidor web, la base de datos y los servicios de Redis.

#### Creación de un archivo Docker Compose

Para definir los servicios que se ejecutarán en diferentes contenedores Docker, utilizaremos un archivo Docker Compose. El archivo Compose es un archivo de texto en formato YAML que define servicios, redes y volúmenes de datos para una aplicación Docker. YAML es un lenguaje de serialización de datos legible por humanos. Puedes ver un ejemplo de un archivo YAML en [https://yaml.org/](https://yaml.org/).

Junto al directorio del proyecto `educa`, crea un nuevo archivo y nómbralo `docker-compose.yml`. Añade el siguiente código:

```yaml
services:
  web:
    build: .
    command: python /code/educa/manage.py runserver 0.0.0.0:8000
    restart: always
    volumes:
      - .:/code
    ports:
      - "8000:8000"
    environment:
      - DJANGO_SETTINGS_MODULE=educa.settings.prod
```

En este archivo, defines un servicio `web`. Las secciones para definir este servicio son las siguientes:

- `build`: Esto define los requisitos de construcción para una imagen de contenedor de servicio. Proporcionas una ruta relativa con un solo punto (`.`) para apuntar al mismo directorio donde se encuentra el archivo Compose. Docker Compose buscará un Dockerfile en esta ubicación.
- `command`: Esto sobrescribe el comando predeterminado del contenedor. Ejecutas el servidor de desarrollo de Django usando el comando `runserver`. El proyecto se sirve en el host `0.0.0.0`, que es la IP predeterminada de Docker, en el puerto 8000.
- `restart`: Esto define la política de reinicio para el contenedor. Con `always`, el contenedor siempre se reinicia si se detiene. Esto es útil para un entorno de producción en el que deseas minimizar el tiempo de inactividad.
- `volumes`: Los datos en los contenedores Docker no son permanentes. Los volúmenes son el método preferido para persistir los datos generados y utilizados por los contenedores Docker. En esta sección, montas el directorio local `.` en el directorio `/code` de la imagen.
- `ports`: Esto expone los puertos del contenedor. El puerto de host 8000 está asignado al puerto de contenedor 8000, en el que se ejecuta el servidor de desarrollo de Django.
- `environment`: Esto define variables de entorno. Estableces la variable de entorno `DJANGO_SETTINGS_MODULE` para usar el archivo de configuración de producción de Django `educa.settings.prod`.

Ten en cuenta que en la definición del archivo Docker Compose, estás utilizando el servidor de desarrollo de Django para servir la aplicación. El servidor de desarrollo de Django no es adecuado para uso en producción, por lo que lo reemplazarás más adelante con un servidor web Python WSGI.

En este punto, asumiendo que tu directorio principal se llama `Chapter17`, la estructura de archivos debería verse de la siguiente manera:

```text
Chapter17/
    Dockerfile
    docker-compose.yml
    educa/
        manage.py
        ...
    requirements.txt
```

Abre una consola en el directorio principal, donde se encuentra el archivo `docker-compose.yml`, y ejecuta el siguiente comando:

```bash
docker compose up
```

Esto iniciará la aplicación Docker definida en el archivo Docker Compose. Verás una salida que incluye las siguientes líneas:

```text
chapter17-web-1  | Performing system checks...
chapter17-web-1  |
chapter17-web-1  | System check identified no issues (0 silenced).
chapter17-web-1  | March 10, 2024 - 12:03:28
chapter17-web-1  | Django version 5.2.0, using settings 'educa.settings.prod'
chapter17-web-1  | Starting ASGI/Daphne version 4.1.0 development server at http://0.0.0.0:8000/
chapter17-web-1  | Quit the server with CONTROL-C.
```

¡El contenedor Docker para tu proyecto Django se está ejecutando!

Abre `http://0.0.0.0:8000/admin/` con tu navegador. Deberías ver el formulario de inicio de sesión del sitio de administración de Django:

> *Figura 17.2: El formulario de inicio de sesión del sitio de administración de Django sin estilos CSS aplicados*

Los estilos CSS no se cargan. Estás utilizando `DEBUG=False`, por lo que los patrones de URL para servir archivos estáticos no se incluyen en el archivo `urls.py` principal del proyecto. Recuerda que el servidor de desarrollo de Django no es adecuado para servir archivos estáticos. Configurarás un servidor para servir archivos estáticos más adelante en este capítulo.

Si accedes a cualquier otra URL de tu sitio, es posible que obtengas un error HTTP 500 porque aún no has configurado una base de datos para el entorno de producción.

Echa un vistazo a la aplicación Docker Desktop:

> *Figura 17.3: La aplicación chapter17 y el contenedor web-1 en Docker Desktop*

La aplicación Docker `chapter17` se está ejecutando y tiene un solo contenedor llamado `web-1`, que se ejecuta en el puerto 8000.

Bajo Images, verás la imagen construida para el servicio web:

> *Figura 17.4: La aplicación chapter17 y el contenedor web-1 en Docker Desktop*

A continuación, agregarás un servicio PostgreSQL y un servicio Redis a tu aplicación Docker.

#### Configuración del servicio PostgreSQL

Edita el archivo `docker-compose.yml` y añade las siguientes líneas:

```yaml
services:
  db:
    image: postgres:16.2
    restart: always
    volumes:
      - ./data/db:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
  web:
    build: .
    command: python /code/educa/manage.py runserver 0.0.0.0:8000
    restart: always
    volumes:
      - .:/code
    ports:
      - "8000:8000"
    environment:
      - DJANGO_SETTINGS_MODULE=educa.settings.prod
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    depends_on:
      - db
```

Con estos cambios, defines un servicio llamado `db` con las siguientes subsecciones:

- `image`: El servicio utiliza la imagen base de Docker `postgres`.
- `restart`: La política de reinicio se establece en `always`.
- `volumes`: Montas el directorio `./data/db` en el directorio de la imagen `/var/lib/postgresql/data` para persistir la base de datos de modo que los datos almacenados en la base de datos se mantengan después de que se detenga la aplicación Docker. Esto creará la ruta local `data/db/`.
- `environment`: Utilizas las variables `POSTGRES_DB` (nombre de la base de datos), `POSTGRES_USER` y `POSTGRES_PASSWORD` con valores predeterminados.

La definición para el servicio `web` ahora incluye las variables de entorno de PostgreSQL para Django. Creas una dependencia de servicio usando `depends_on` para que el servicio `web` se inicie después del servicio `db`. Esto garantizará el orden de inicialización del contenedor, pero no garantizará que PostgreSQL esté completamente iniciado antes de que se inicie el servidor web Django. Para resolver esto, debes usar un script que espere la disponibilidad del host de la base de datos y su puerto TCP. Docker recomienda que utilices la herramienta `wait-for-it` para controlar la inicialización del contenedor.

Descarga el script bash `wait-for-it.sh` de [https://github.com/vishnubob/wait-for-it/blob/master/wait-for-it.sh](https://github.com/vishnubob/wait-for-it/blob/master/wait-for-it.sh) y guarda el archivo junto al archivo `docker-compose.yml`. Luego, edita el archivo `docker-compose.yml` y modifica la definición del servicio web de la siguiente manera:

```yaml
  web:
    build: .
    command: ["./wait-for-it.sh", "db:5432", "--", "python", "/code/educa/manage.py", "runserver", "0.0.0.0:8000"]
    restart: always
    volumes:
      - .:/code
    environment:
      - DJANGO_SETTINGS_MODULE=educa.settings.prod
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    depends_on:
      - db
```

En esta definición de servicio, utilizas el script de shell `wait-for-it.sh` para esperar a que el host `db` esté listo y acepte conexiones en el puerto 5432, el puerto predeterminado para PostgreSQL, antes de iniciar el servidor de desarrollo de Django.

Editemos los ajustes de Django. Edita el archivo `educa/settings/prod.py` y añade el siguiente código:

```python
from decouple import config
from .base import *

DEBUG = False

ADMINS = [
    ('Antonio M', 'email@mydomain.com'),
]

ALLOWED_HOSTS = ['*']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('POSTGRES_DB'),
        'USER': config('POSTGRES_USER'),
        'PASSWORD': config('POSTGRES_PASSWORD'),
        'HOST': 'db',
        'PORT': 5432,
    }
}
```

En el archivo de configuración de producción, utilizas los siguientes ajustes:

- `ENGINE`: Utilizas el backend de base de datos de Django para PostgreSQL.
- `NAME`, `USER` y `PASSWORD`: Utilizas la función `config()` de `python-decouple` para recuperar las variables de entorno `POSTGRES_DB` (nombre de la base de datos), `POSTGRES_USER` y `POSTGRES_PASSWORD`. Has establecido estas variables de entorno en el archivo Docker Compose.
- `HOST`: Utilizas `db`, que es el nombre de host del contenedor para el servicio de base de datos definido en el archivo Docker Compose. El nombre de host de un contenedor tiene como valor predeterminado el ID del contenedor en Docker. Por eso usas el nombre de host `db`.
- `PORT`: Utilizas el valor 5432, que es el puerto predeterminado para PostgreSQL.

Detén la aplicación Docker desde la consola presionando las teclas Ctrl + C o usando el botón de parada en la aplicación Docker Desktop. Luego, inicia Compose nuevamente con el siguiente comando:

```bash
docker compose up
```

La primera ejecución después de agregar el servicio `db` al archivo Docker Compose llevará más tiempo porque PostgreSQL necesita inicializar la base de datos. La salida contendrá las siguientes dos líneas:

```text
db-1   | ... database system is ready to accept connections ...
web-1  | Starting ASGI/Daphne version 4.1.0 development server at http://0.0.0.0:8000/
```

Tanto la base de datos PostgreSQL como la aplicación Django están listas. La base de datos de producción está vacía, por lo que debes aplicar las migraciones de la base de datos.

#### Aplicación de migraciones de bases de datos y creación de un superusuario

Abre una consola diferente en el directorio principal, donde se encuentra el archivo `docker-compose.yml`, y ejecuta el siguiente comando:

```bash
docker compose exec web python /code/educa/manage.py migrate
```

El comando `docker compose exec` te permite ejecutar comandos en el contenedor. Utilizas este comando para ejecutar el comando de administración `migrate` en el contenedor Docker `web`.

Finalmente, crea un superusuario con el siguiente comando:

```bash
docker compose exec web python /code/educa/manage.py createsuperuser
```

Se han aplicado las migraciones a la base de datos y has creado un superusuario. Puedes acceder a `http://localhost:8000/admin/` con las credenciales de superusuario. Los estilos CSS todavía no se cargarán porque aún no has configurado el servicio de archivos estáticos.

Has definido servicios para servir Django y PostgreSQL usando Docker Compose. A continuación, agregarás un servicio para servir a Redis en el entorno de producción.

#### Configuración del servicio Redis

Agreguemos un servicio de Redis al archivo Docker Compose. Para este propósito, utilizarás la imagen oficial de Redis en Docker.

Edita el archivo `docker-compose.yml` y añade las siguientes líneas:

```yaml
services:
  db:
    # ...
  cache:
    image: redis:7.2.4
    restart: always
    volumes:
      - ./data/cache:/data
  web:
    # ...
    depends_on:
      - db
      - cache
```

En el código anterior, defines el servicio `cache` con las siguientes subsecciones:

- `image`: El servicio utiliza la imagen base de Docker `redis`.
- `restart`: La política de reinicio se establece en `always`.
- `volumes`: Montas el directorio `./data/cache` en el directorio `/data` de la imagen donde se persistirán las escrituras de Redis. Esto creará la ruta local `data/cache/`.

En la definición del servicio `web`, agregas el servicio `cache` como una dependencia, de modo que el servicio `web` se inicie después del servicio `cache`. El servidor Redis se inicializa rápido, por lo que no necesitas usar la herramienta `wait-for-it` en este caso.

Edita el archivo `educa/settings/prod.py` y añade las siguientes líneas:

```python
REDIS_URL = 'redis://cache:6379'
CACHES['default']['LOCATION'] = REDIS_URL
CHANNEL_LAYERS['default']['CONFIG']['hosts'] = [REDIS_URL]
```

En estas configuraciones, utilizas el nombre de host `cache` que Docker Compose genera automáticamente utilizando el nombre del servicio de caché y el puerto 6379 utilizado por Redis. Modificas la configuración `CACHES` de Django y la configuración `CHANNEL_LAYERS` utilizada por Channels para utilizar la URL de producción de Redis.

Detén la aplicación Docker desde la consola presionando las teclas Ctrl + C o usando el botón de parada en la aplicación Docker Desktop. Luego, inicia Compose nuevamente con el siguiente comando:

```bash
docker compose up
```

Abre la aplicación Docker Desktop. Ahora deberías ver la aplicación Docker `chapter17` ejecutando un contenedor para cada servicio definido en el archivo Docker Compose: `db`, `cache` y `web`:

> *Figura 17.5: La aplicación chapter17 con los contenedores db-1, web-1 y cache-1 en Docker Desktop*

Todavía estás sirviendo a Django con el servidor de desarrollo de Django, que, como sabes, está diseñado solo para desarrollo y no está optimizado para uso en producción. Reemplacémoslo con el servidor web WSGI Python.

---

### Servir Django a través de WSGI y NGINX

La principal plataforma de despliegue de Django es WSGI. WSGI significa *Web Server Gateway Interface* y es el estándar para servir aplicaciones de Python en la web.

Cuando generas un nuevo proyecto usando el comando `startproject`, Django crea un archivo `wsgi.py` dentro del directorio de tu proyecto. Este archivo contiene un objeto ejecutable (*callable*) de aplicación WSGI, que es un punto de acceso a tu aplicación.

WSGI se utiliza tanto para ejecutar tu proyecto con el servidor de desarrollo de Django como para implementar tu aplicación con el servidor de tu elección en un entorno de producción. Puedes obtener más información sobre WSGI en [https://wsgi.readthedocs.io/en/latest/](https://wsgi.readthedocs.io/en/latest/).

En las siguientes secciones utilizaremos uWSGI, un servidor web de código abierto que implementa la especificación WSGI.

#### Uso de uWSGI

A lo largo de este libro, has estado utilizando el servidor de desarrollo de Django para ejecutar proyectos en tu entorno local. Sin embargo, el servidor de desarrollo no está diseñado para uso en producción, y desplegar tu aplicación en un entorno de producción requerirá un servidor web estándar.

uWSGI es un servidor de aplicaciones de Python extremadamente rápido. Se comunica con tu aplicación Python utilizando la especificación WSGI. uWSGI traduce las solicitudes web a un formato que tu proyecto Django puede procesar.

Configuremos uWSGI para servir el proyecto Django. Ya agregaste `uwsgi==2.0.28` al archivo `requirements.txt` del proyecto, por lo que uWSGI ya se está instalando en la imagen de Docker del servicio web.

Edita el archivo `docker-compose.yml` y modifica la definición del servicio web de la siguiente manera:

```yaml
  web:
    build: .
    command: ["./wait-for-it.sh", "db:5432", "--", "uwsgi", "--ini", "/code/config/uwsgi/uwsgi.ini"]
    restart: always
    volumes:
      - .:/code
    environment:
      - DJANGO_SETTINGS_MODULE=educa.settings.prod
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    depends_on:
      - db
      - cache
```

Asegúrate de eliminar la sección `ports`. uWSGI será accesible con un socket, por lo que no necesitas exponer un puerto en el contenedor.

El nuevo comando para la imagen ejecuta `uwsgi` y le pasa el archivo de configuración `/code/config/uwsgi/uwsgi.ini`. Creemos el archivo de configuración para uWSGI.

#### Configuración de uWSGI

uWSGI te permite definir una configuración personalizada en un archivo `.ini`. Junto al archivo `docker-compose.yml`, crea la ruta de archivo `config/uwsgi/uwsgi.ini`. Asumiendo que tu directorio principal se llama `Chapter17`, la estructura de archivos debería verse de la siguiente manera:

```text
Chapter17/
    config/
        uwsgi/
            uwsgi.ini
    Dockerfile
    docker-compose.yml
    educa/
        manage.py
        ...
    requirements.txt
```

Edita el archivo `config/uwsgi/uwsgi.ini` y añade el siguiente código:

```ini
[uwsgi]
socket=/code/educa/uwsgi_app.sock
chdir = /code/educa/
module=educa.wsgi:application
master=true
chmod-socket=666
uid=www-data
gid=www-data
vacuum=true
```

En el archivo `uwsgi.ini`, defines las siguientes opciones:

- `socket`: Este es el socket Unix/TCP para enlazar el servidor.
- `chdir`: Esta es la ruta al directorio de tu proyecto, para que uWSGI cambie a ese directorio antes de cargar la aplicación Python.
- `module`: Este es el módulo WSGI a utilizar. Lo estableces en la función invocable de la aplicación contenida en el módulo `wsgi` de tu proyecto.
- `master`: Esto habilita el proceso maestro.
- `chmod-socket`: Estos son los permisos de archivo que se aplicarán al archivo de socket. En este caso, utilizas 666 para que NGINX pueda leer/escribir en el socket.
- `uid`: Este es el ID de usuario del proceso una vez que se inicia.
- `gid`: Este es el ID de grupo del proceso una vez que se inicia.
- `vacuum`: Usar `true` le indica a uWSGI que limpie los archivos temporales o los sockets UNIX que cree.

La opción `socket` está diseñada para la comunicación con algún tipo de enrutador de terceros, como NGINX. Vas a ejecutar uWSGI usando un socket y vas a configurar NGINX como tu servidor web, que se comunicará con uWSGI a través del socket.

Puedes encontrar la lista de opciones disponibles de uWSGI en [https://uwsgi-docs.readthedocs.io/en/latest/Options.html](https://uwsgi-docs.readthedocs.io/en/latest/Options.html).

No podrás acceder a tu instancia de uWSGI desde tu navegador ahora, ya que se ejecuta a través de un socket. Para completar el entorno, utilizaremos NGINX delante de uWSGI, para administrar las solicitudes HTTP y pasar las solicitudes de la aplicación a uWSGI a través del socket. Completemos el entorno de producción.

#### Uso de NGINX

Cuando estás sirviendo un sitio web, tienes que servir contenido dinámico, pero también necesitas servir archivos estáticos, como hojas de estilo CSS, archivos JavaScript e imágenes. Si bien uWSGI es capaz de servir archivos estáticos, agrega una sobrecarga innecesaria a las solicitudes HTTP y, por lo tanto, se recomienda configurar un servidor web, como NGINX, delante de él.

NGINX es un servidor web enfocado en alta concurrencia, rendimiento y bajo uso de memoria. NGINX también actúa como un proxy inverso, recibiendo solicitudes HTTP y WebSocket y enrutándolas a diferentes backends.

Por lo general, utilizarás un servidor web, como NGINX, delante de uWSGI para servir archivos estáticos de manera eficiente, y reenviarás solicitudes dinámicas a los trabajadores de uWSGI. Al usar NGINX, también puedes aplicar diferentes reglas y beneficiarte de sus capacidades de proxy inverso.

Agregaremos el servicio NGINX al archivo Docker Compose utilizando la imagen oficial de Docker de NGINX. Puedes encontrar información sobre la imagen oficial de Docker de NGINX en [https://hub.docker.com/_/nginx](https://hub.docker.com/_/nginx).

Edita el archivo `docker-compose.yml` y añade las siguientes líneas:

```yaml
services:
  db:
    # ...
  cache:
    # ...
  web:
    # ...
  nginx:
    image: nginx:1.25.5
    restart: always
    volumes:
      - ./config/nginx:/etc/nginx/templates
      - .:/code
    ports:
      - "80:80"
```

Has añadido la definición para el servicio `nginx` con las siguientes subsecciones:

- `image`: El servicio utiliza la imagen base de Docker `nginx`.
- `restart`: La política de reinicio se establece en `always`.
- `volumes`: Montas el volumen `./config/nginx` en el directorio `/etc/nginx/templates` de la imagen de Docker. Aquí es donde NGINX buscará una plantilla de configuración predeterminada. También montas el directorio local `.` en el directorio `/code` de la imagen, para que NGINX pueda tener acceso a archivos estáticos.
- `ports`: Expones el puerto 80, que está asignado al puerto de contenedor 80. Este es el puerto predeterminado para HTTP.

Configuremos el servidor web NGINX.

#### Configuración de NGINX

Crea la siguiente ruta de archivo bajo el directorio `config/`:

```text
config/
    uwsgi/
        uwsgi.ini
    nginx/
        default.conf.template
```

Edita el archivo `nginx/default.conf.template` y añade el siguiente código:

```nginx
# upstream for uWSGI
upstream uwsgi_app {
    server unix:/code/educa/uwsgi_app.sock;
}

server {
    listen 80;
    server_name www.educaproject.com educaproject.com;
    error_log stderr warn;
    access_log /dev/stdout main;

    location / {
        include /etc/nginx/uwsgi_params;
        uwsgi_pass uwsgi_app;
    }
}
```

Esta es la configuración básica para NGINX. En esta configuración, configuras un componente upstream llamado `uwsgi_app`, que apunta al socket creado por uWSGI. Utilizas el bloque `server` con la siguiente configuración:

- Le dices a NGINX que escuche en el puerto 80.
- Estableces el nombre del servidor en `www.educaproject.com` y `educaproject.com`. NGINX atenderá las solicitudes entrantes para ambos dominios.
- Utilizas `stderr` para la directiva `error_log` para que los registros de errores se escriban en el archivo de error estándar. El segundo parámetro determina el nivel de registro. Utilizas `warn` para recibir advertencias y errores de mayor gravedad.
- Apuntas `access_log` a la salida estándar con `/dev/stdout`.
- Especificas que cualquier solicitud bajo la ruta `/` debe enrutarse con el socket `uwsgi_app` a uWSGI.
- Incluyes los parámetros de configuración predeterminados de uWSGI que vienen con NGINX. Estos se encuentran en `/etc/nginx/uwsgi_params`.

NGINX ya está configurado. Puedes encontrar la documentación de NGINX en [https://nginx.org/en/docs/](https://nginx.org/en/docs/).

Detén la aplicación Docker desde la consola presionando las teclas Ctrl + C o usando el botón de parada en la aplicación Docker Desktop. Luego, inicia Compose nuevamente con el siguiente comando:

```bash
docker compose up
```

Abre la URL `http://localhost/` en tu navegador. No es necesario añadir un puerto a la URL porque estás accediendo al host a través del puerto HTTP estándar 80. Deberías ver la página de lista de cursos sin estilos CSS:

> *Figura 17.6: La página de lista de cursos servida con NGINX y uWSGI*

El siguiente diagrama muestra el ciclo de solicitud/respuesta del entorno de producción que has configurado:

> *Figura 17.7: El ciclo de solicitud/respuesta del entorno de producción*

Lo siguiente sucede cuando el navegador del cliente envía una solicitud HTTP:

1. NGINX recibe la solicitud HTTP.
2. NGINX delega la solicitud a uWSGI a través de un socket.
3. uWSGI pasa la solicitud a Django para su procesamiento.
4. Django devuelve una respuesta HTTP que se devuelve a NGINX, que a su vez la devuelve al navegador del cliente.

Si revisas la aplicación Docker Desktop, deberías ver que hay cuatro contenedores ejecutándose:

- El servicio `db` ejecuta PostgreSQL
- El servicio `cache` ejecuta Redis
- El servicio `web` ejecuta uWSGI y Django
- El servicio `nginx` ejecuta NGINX

Continuemos con la configuración del entorno de producción. En lugar de acceder a nuestro proyecto mediante `localhost`, configuraremos el proyecto para utilizar el nombre de host `educaproject.com`.

#### Uso de un nombre de host

Utilizarás el nombre de host `educaproject.com` para tu sitio. Como estás utilizando un nombre de dominio de muestra, debes redirigirlo a tu host local.

Si estás utilizando Linux o macOS, edita el archivo `/etc/hosts` y añade la siguiente línea:

```text
127.0.0.1 educaproject.com www.educaproject.com
```

Si estás utilizando Windows, edita el archivo `C:\Windows\System32\drivers\etc\hosts` y añade la misma línea.

Al hacerlo, estás enrutando los nombres de host `educaproject.com` y `www.educaproject.com` a tu servidor local. En un servidor de producción, no necesitarás hacer esto, ya que tendrás una dirección IP fija y apuntarás tu nombre de host a tu servidor en la configuración DNS de tu dominio.

Abre `http://educaproject.com/` en tu navegador. Deberías poder ver tu sitio, todavía sin ningún recurso estático cargado. Tu entorno de producción está casi listo.

Ahora, puedes restringir los hosts que pueden servir a tu proyecto Django. Edita el archivo de configuración de producción `educa/settings/prod.py` de tu proyecto y cambia la configuración `ALLOWED_HOSTS`, de la siguiente manera:

```python
ALLOWED_HOSTS = ['educaproject.com', 'www.educaproject.com']
```

Django solo servirá a tu aplicación si se ejecuta bajo alguno de estos nombres de host. Puedes leer más sobre la configuración `ALLOWED_HOSTS` en [https://docs.djangoproject.com/en/5.2/ref/settings/#allowed-hosts](https://docs.djangoproject.com/en/5.2/ref/settings/#allowed-hosts).

El entorno de producción está casi listo. Continuemos configurando NGINX para servir archivos estáticos.

#### Servir archivos estáticos y multimedia

uWSGI es capaz de servir archivos estáticos sin problemas, pero no es tan rápido y eficaz como NGINX. Para obtener el mejor rendimiento, utilizarás NGINX para servir archivos estáticos en tu entorno de producción. Configurarás NGINX para servir tanto los archivos estáticos de tu aplicación (hojas de estilo CSS, archivos JavaScript e imágenes) como los archivos multimedia cargados por los instructores para los contenidos del curso.

Edita el archivo `settings/base.py` y añade la siguiente línea justo debajo de la configuración `STATIC_URL`:

```python
STATIC_ROOT = BASE_DIR / 'static'
```

Este es el directorio raíz para todos los archivos estáticos del proyecto. A continuación, recopilarás los archivos estáticos de las diferentes aplicaciones de Django en el directorio común.

#### Recopilación de archivos estáticos

Cada aplicación en tu proyecto Django puede contener archivos estáticos en un directorio `static/`. Django proporciona un comando para recopilar archivos estáticos de todas las aplicaciones en una sola ubicación. Esto simplifica la configuración para servir archivos estáticos en producción. El comando `collectstatic` recopila los archivos estáticos de todas las aplicaciones del proyecto en la ruta definida con la configuración `STATIC_ROOT`.

Detén la aplicación Docker desde la consola presionando las teclas Ctrl + C o usando el botón de parada en la aplicación Docker Desktop. Luego, inicia Compose nuevamente con el siguiente comando:

```bash
docker compose up
```

Abre otra consola en el directorio principal, donde se encuentra el archivo `docker-compose.yml`, y ejecuta el siguiente comando:

```bash
docker compose exec web python /code/educa/manage.py collectstatic
```

Ten en cuenta que alternativamente puedes ejecutar el siguiente comando en la consola, desde el directorio del proyecto `educa/`:

```bash
python manage.py collectstatic --settings=educa.settings.local
```

Ambos comandos tendrán el mismo efecto ya que el directorio local base está montado en la imagen de Docker. Django te preguntará si deseas sobrescribir los archivos existentes en el directorio raíz. Escribe `yes` y presiona Enter. Verás la siguiente salida:

```text
171 static files copied to '/code/educa/static'.
```

Los archivos ubicados en el directorio `static/` de cada aplicación presente en la configuración `INSTALLED_APPS` se han copiado al directorio de proyecto global `/educa/static/`.

#### Servir archivos estáticos con NGINX

Edita el archivo `config/nginx/default.conf.template` y añade las siguientes líneas al bloque `server`:

```nginx
server {
    # ...
    location / {
        include /etc/nginx/uwsgi_params;
        uwsgi_pass uwsgi_app;
    }

    location /static/ {
        alias /code/educa/static/;
    }

    location /media/ {
        alias /code/educa/media/;
    }
}
```

Estas directivas le indican a NGINX que sirva directamente los archivos estáticos ubicados en las rutas `/static/` y `/media/`. Estas rutas son las siguientes:

- `/static/`: Corresponde a la ruta del ajuste `STATIC_URL`. La ruta de destino corresponde al valor de la configuración `STATIC_ROOT`. La utilizas para servir los archivos estáticos de tu aplicación desde el directorio montado en la imagen Docker de NGINX.
- `/media/`: Corresponde a la ruta del ajuste `MEDIA_URL`, y su ruta de destino corresponde al valor de la configuración `MEDIA_ROOT`. La utilizas para servir los archivos multimedia subidos a los contenidos del curso desde el directorio montado en la imagen Docker de NGINX.

La Figura 17.8 muestra la configuración actual del entorno de producción:

> *Figura 17.8: El ciclo de solicitud/respuesta del entorno de producción, incluidos los archivos estáticos*

Los archivos bajo las rutas `/static/` y `/media/` ahora son servidos por NGINX directamente, en lugar de ser reenviados a uWSGI. Las solicitudes a cualquier otra ruta todavía son pasadas por NGINX a uWSGI a través del socket UNIX.

Detén la aplicación Docker desde la consola presionando las teclas Ctrl + C o usando el botón de parada en la aplicación Docker Desktop. Luego, inicia Compose nuevamente con el siguiente comando:

```bash
docker compose up
```

Abre `http://educaproject.com/` en tu navegador. Deberías ver la siguiente pantalla:

> *Figura 17.9: La página de lista de cursos servida con NGINX y uWSGI*

Los recursos estáticos, como las hojas de estilo CSS y las imágenes, ahora se cargan correctamente. NGINX ahora atiende las solicitudes HTTP de archivos estáticos directamente, en lugar de reenviarlas a uWSGI.

Has configurado con éxito NGINX para servir archivos estáticos. A continuación, vas a realizar algunas comprobaciones en tu proyecto Django para validarlo para un entorno de producción y vas a servir tu sitio bajo HTTPS.

---

### Asegurar tu sitio con SSL/TLS

El protocolo TLS es el estándar para servir sitios web a través de una conexión segura. El predecesor de TLS es SSL. Aunque SSL ahora está en desuso, en múltiples bibliotecas y documentación en línea encontrarás referencias a ambos términos: TLS y SSL. Se recomienda encarecidamente que sirvas tus sitios web a través de HTTPS.

En esta sección, vas a comprobar tu proyecto Django para detectar cualquier problema y validarlo para un despliegue en producción. También prepararás el proyecto para que se sirva a través de HTTPS. Luego, vas a configurar un certificado SSL/TLS en NGINX para servir tu sitio de forma segura.

#### Comprobación de tu proyecto para producción

Django incluye un framework de comprobación del sistema para validar tu proyecto en cualquier momento. El framework de comprobación inspecciona las aplicaciones instaladas en tu proyecto Django y detecta problemas comunes. Las comprobaciones se activan implícitamente cuando ejecutas comandos de administración como `runserver` y `migrate`. Sin embargo, puedes activar comprobaciones explícitamente con el comando de administración `check`.

Puedes leer más sobre el framework de comprobación del sistema de Django en [https://docs.djangoproject.com/en/5.2/topics/checks/](https://docs.djangoproject.com/en/5.2/topics/checks/).

Confirmemos que el framework de comprobación no genera ningún problema para tu proyecto. Abre la consola en el directorio del proyecto `educa` y ejecuta el siguiente comando para verificar tu proyecto:

```bash
python manage.py check --settings=educa.settings.prod
```

Verás la siguiente salida:

```text
System check identified no issues (0 silenced).
```

El framework de verificación del sistema no identificó ningún problema. Si utilizas la opción `--deploy`, el framework de comprobación del sistema realizará comprobaciones adicionales que son relevantes para una implementación en producción.

Ejecuta el siguiente comando desde el directorio del proyecto `educa`:

```bash
python manage.py check --deploy --settings=educa.settings.prod
```

Verás una salida como la siguiente:

```text
System check identified some issues:

WARNINGS:
?: (security.W004) You have not set a value for the SECURE_HSTS_SECONDS setting. ...
?: (security.W008) Your SECURE_SSL_REDIRECT setting is not set to True...
?: (security.W009) Your SECRET_KEY has less than 50 characters, less than 5 unique characters, or it's prefixed with 'django-insecure-'...
?: (security.W012) SESSION_COOKIE_SECURE is not set to True. ...
?: (security.W016) You have 'django.middleware.csrf.CsrfViewMiddleware' in your MIDDLEWARE, but you have not set CSRF_COOKIE_SECURE ...

System check identified 5 issues (0 silenced).
```

El framework de comprobación ha identificado cinco problemas (cero errores y cinco advertencias). Todas las advertencias están relacionadas con configuraciones de seguridad.

Abordemos el problema `security.W009`. Edita el archivo `educa/settings/base.py` y modifica la configuración `SECRET_KEY` eliminando el prefijo `django-insecure-` y agregando caracteres aleatorios adicionales para generar una cadena con al menos 50 caracteres.

Ejecuta el comando `check` nuevamente y verifica que el problema `security.W009` ya no se genere. El resto de las advertencias están relacionadas con la configuración de SSL/TLS. Las abordaremos a continuación.

#### Configuración de tu proyecto Django para SSL/TLS

Django viene con configuraciones específicas para soporte SSL/TLS. Vas a editar las configuraciones de producción para servir tu sitio a través de HTTPS.

Edita el archivo de configuración `educa/settings/prod.py` y añade las siguientes configuraciones:

```python
# Security
CSRF_COOKIE_SECURE = True
SESSION_COOKIE_SECURE = True
SECURE_SSL_REDIRECT = True
```

Estas configuraciones son las siguientes:

- `CSRF_COOKIE_SECURE`: Utiliza una cookie segura para la protección contra la falsificación de solicitudes entre sitios (CSRF). Con `True`, los navegadores solo transferirán la cookie a través de HTTPS.
- `SESSION_COOKIE_SECURE`: Utiliza una cookie de sesión segura. Con `True`, los navegadores solo transferirán la cookie a través de HTTPS.
- `SECURE_SSL_REDIRECT`: Esto indica si las solicitudes HTTP deben redirigirse a HTTPS.

Django ahora redirigirá las solicitudes HTTP a HTTPS; las cookies de sesión y CSRF se enviarán solo a través de HTTPS.

Ejecuta el siguiente comando desde el directorio principal de tu proyecto:

```bash
python manage.py check --deploy --settings=educa.settings.prod
```

Solo queda una advertencia, `security.W004`:

```text
?: (security.W004) You have not set a value for the SECURE_HSTS_SECONDS setting. ...
```

Esta advertencia está relacionada con la política de seguridad de transporte estricto HTTP (HSTS). La política HSTS evita que los usuarios eludan las advertencias y se conecten a un sitio con un certificado SSL caducado, autofirmado o no válido. En la siguiente sección, utilizaremos un certificado autofirmado para nuestro sitio, por lo que ignoraremos esta advertencia.

Cuando eres dueño de un dominio real, puedes solicitar a una Autoridad de Certificación (CA) de confianza que emita un certificado SSL/TLS para él, de modo que los navegadores puedan verificar su identidad. En ese caso, puedes darle un valor a `SECURE_HSTS_SECONDS` superior a 0, que es el valor predeterminado. Puedes obtener más información sobre la política HSTS en [https://docs.djangoproject.com/en/5.2/ref/middleware/#http-strict-transport-security](https://docs.djangoproject.com/en/5.2/ref/middleware/#http-strict-transport-security).

Has solucionado con éxito el resto de los problemas planteados por el framework de comprobación. Puedes leer más sobre la lista de verificación de implementación de Django en [https://docs.djangoproject.com/en/5.2/howto/deployment/checklist/](https://docs.djangoproject.com/en/5.2/howto/deployment/checklist/).

#### Creación de un certificado SSL/TLS

Crea un nuevo directorio dentro del directorio del proyecto `educa` y nómbralo `ssl`. Luego, genera un certificado SSL/TLS desde la línea de comandos con el siguiente comando:

```bash
openssl req -x509 -newkey rsa:2048 -sha256 -days 3650 -nodes \
  -keyout ssl/educa.key -out ssl/educa.crt \
  -subj '/CN=*.educaproject.com' \
  -addext 'subjectAltName=DNS:*.educaproject.com'
```

Esto generará una clave privada y un certificado SSL/TLS de 2048 bits que es válido por 10 años. Este certificado se emite para el nombre de host `*.educaproject.com`. Este es un certificado comodín (*wildcard*); al utilizar el carácter comodín `*` en el nombre de dominio, el certificado se puede utilizar para cualquier subdominio de `educaproject.com`, como `www.educaproject.com` o `django.educaproject.com`. Después de generar el certificado, el directorio `educa/ssl/` contendrá dos archivos: `educa.key` (la clave privada) y `educa.crt` (el certificado).

Necesitarás al menos OpenSSL 1.1.1 o LibreSSL 3.1.0 para usar la opción `-addext`. Puedes verificar la ubicación de OpenSSL en tu máquina con el comando `which openssl` y puedes verificar la versión con el comando `openssl version`.

Alternativamente, puedes usar el certificado SSL/TLS provisto en el código fuente de este capítulo. Encontrarás el certificado en [https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter17/educa/ssl/](https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter17/educa/ssl/). Ten en cuenta que debes generar una clave privada y no utilizar este certificado en producción.

#### Configuración de NGINX para usar SSL/TLS

Edita el archivo `docker-compose.yml` y añade la siguiente línea:

```yaml
services:
  # ...
  nginx:
    # ...
    ports:
      - "80:80"
      - "443:443"
```

El host del contenedor NGINX será accesible a través del puerto 80 (HTTP) y del puerto 443 (HTTPS). El puerto de host 443 se asigna al puerto de contenedor 443.

Edita el archivo `config/nginx/default.conf.template` del proyecto `educa` y modifica el bloque `server` para incluir SSL/TLS, de la siguiente manera:

```nginx
server {
    listen 80;
    listen 443 ssl;
    ssl_certificate /code/educa/ssl/educa.crt;
    ssl_certificate_key /code/educa/ssl/educa.key;
    server_name www.educaproject.com educaproject.com;
    # ...
}
```

Con el código anterior, NGINX ahora escucha tanto a HTTP sobre el puerto 80 como a HTTPS sobre el puerto 443. Indicas la ruta al certificado SSL/TLS con `ssl_certificate` y la clave del certificado con `ssl_certificate_key`.

Detén la aplicación Docker desde la consola presionando las teclas Ctrl + C o usando el botón de parada en la aplicación Docker Desktop. Luego, inicia Compose nuevamente con el siguiente comando:

```bash
docker compose up
```

Abre `https://educaproject.com/` con tu navegador. Deberías ver un mensaje de advertencia similar al siguiente:

> *Figura 17.10: Una advertencia de certificado no válido*

Esta pantalla puede variar según tu navegador. Te advierte que tu sitio no está utilizando un certificado válido o de confianza; el navegador no puede verificar la identidad de tu sitio. Esto se debe a que firmaste tu propio certificado en lugar de obtener uno de una CA de confianza. Cuando eres dueño de un dominio real, puedes solicitar a una CA de confianza que emita un certificado SSL/TLS para él, de modo que los navegadores puedan verificar su identidad. Si deseas obtener un certificado de confianza para un dominio real, puedes consultar el proyecto Let’s Encrypt creado por la Fundación Linux. Es una CA sin fines de lucro que simplifica la obtención y renovación gratuita de certificados SSL/TLS de confianza. Puedes encontrar más información en [https://letsencrypt.org](https://letsencrypt.org/).

Haz clic en el enlace o botón que proporciona información adicional y elige visitar el sitio web, ignorando las advertencias. El navegador puede pedirte que agregues una excepción para este certificado o que verifiques que confías en él. Si estás utilizando Chrome, es posible que no veas ninguna opción para continuar al sitio web. Si este es el caso, escribe `thisisunsafe` y presiona Enter directamente en Chrome en la página de advertencia. Chrome luego cargará el sitio web. Ten en cuenta que haces esto con tu propio certificado emitido; no confíes en ningún certificado desconocido ni eludas las comprobaciones del certificado SSL/TLS del navegador para otros dominios.

Cuando accedas al sitio, el navegador mostrará un icono de candado junto a la URL:

> *Figura 17.11: La barra de direcciones del navegador, incluido un icono de candado de conexión segura*

Otros navegadores pueden mostrar una advertencia que indica que el certificado no es de confianza:

> *Figura 17.12: La barra de direcciones del navegador, incluido un mensaje de advertencia*

Es posible que tu navegador marque el certificado como no seguro, pero lo estás utilizando únicamente con fines de prueba. Ahora estás sirviendo tu sitio de forma segura a través de HTTPS.

#### Redirección del tráfico HTTP a HTTPS

Estás redirigiendo las solicitudes HTTP a HTTPS con Django usando la configuración `SECURE_SSL_REDIRECT`. Cualquier solicitud que use `http://` se redirige a la misma URL usando `https://`. Sin embargo, esto se puede manejar de una manera más eficiente usando NGINX.

Edita el archivo `config/nginx/default.conf.template` y añade las siguientes líneas:

```nginx
# upstream for uWSGI
upstream uwsgi_app {
    server unix:/code/educa/uwsgi_app.sock;
}

server {
    listen 80;
    server_name www.educaproject.com educaproject.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    ssl_certificate /code/educa/ssl/educa.crt;
    ssl_certificate_key /code/educa/ssl/educa.key;
    server_name www.educaproject.com educaproject.com;
    # ...
}
```

En este código, eliminas la directiva `listen 80;` del bloque `server` original, de modo que la plataforma solo esté disponible a través de HTTPS (puerto 443). Encima del bloque `server` original, agregas un bloque `server` adicional que solo escucha en el puerto 80 y redirige todas las solicitudes HTTP a HTTPS. Para lograr esto, devuelves un código de respuesta HTTP 301 (redirección permanente) que redirige a la versión `https://` de la URL solicitada utilizando las variables `$host` y `$request_uri`.

Abre una consola en el directorio principal, donde se encuentra el archivo `docker-compose.yml`, y ejecuta el siguiente comando para recargar NGINX:

```bash
docker compose exec nginx nginx -s reload
```

Esto ejecuta el comando `nginx -s reload` en el contenedor `nginx`. Ahora estás redirigiendo todo el tráfico HTTP a HTTPS usando NGINX.

Tu entorno ahora está protegido con TLS/SSL. Para completar la configuración del entorno de producción, el único paso restante es integrar Daphne para manejar solicitudes asíncronas y hacer que las salas de chat de nuestros cursos funcionen en producción.

---

### Configuración de Daphne para Django Channels

En el Capítulo 16, Creación de un servidor de chat, utilizaste Django Channels para construir un servidor de chat usando WebSockets y utilizaste Daphne para atender solicitudes asíncronas reemplazando el comando estándar de Django `runserver`. Agregaremos Daphne a nuestro entorno de producción.

Creemos un nuevo servicio en el archivo Docker Compose para ejecutar el servidor web Daphne.

Edita el archivo `docker-compose.yml` y añade las siguientes líneas dentro del bloque `services`:

```yaml
  daphne:
    build: .
    working_dir: /code/educa/
    command: ["../wait-for-it.sh", "db:5432", "--", "daphne", "-b", "0.0.0.0", "-p", "9001", "educa.asgi:application"]
    restart: always
    volumes:
      - .:/code
    environment:
      - DJANGO_SETTINGS_MODULE=educa.settings.prod
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    depends_on:
      - db
      - cache
```

La definición del servicio `daphne` es muy similar a la del servicio `web`. La imagen para el servicio `daphne` también se crea con el Dockerfile que creaste previamente para el servicio `web`. Las principales diferencias son las siguientes:

- `working_dir` cambia el directorio de trabajo de la imagen a `/code/educa/`.
- `command` ejecuta la aplicación `educa.asgi:application` definida en el archivo `educa/asgi.py` con `daphne` en el nombre de host `0.0.0.0` y el puerto 9001. También utiliza el script de shell `wait-for-it` para esperar a que la base de datos PostgreSQL esté lista antes de inicializar el servidor web.

Dado que estás ejecutando Django en producción, Django verifica los `ALLOWED_HOSTS` al recibir solicitudes HTTP. Implementaremos la misma validación para las conexiones WebSocket.

Edita el archivo `educa/asgi.py` de tu proyecto y añade las siguientes líneas:

```python
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.security.websocket import AllowedHostsOriginValidator
from channels.auth import AuthMiddlewareStack

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'educa.settings')

django_asgi_app = get_asgi_application()

from chat.routing import websocket_urlpatterns

application = ProtocolTypeRouter({
    'http': django_asgi_app,
    'websocket': AllowedHostsOriginValidator(
        AuthMiddlewareStack(
            URLRouter(websocket_urlpatterns)
        )
    ),
})
```

La configuración de Channels ahora está lista para producción.

#### Uso de conexiones seguras para WebSockets

Has configurado NGINX para usar conexiones seguras con SSL/TLS. Necesitas cambiar las conexiones `ws` (WebSocket) para usar el protocolo `wss` (WebSocket Secure) ahora, de la misma manera que las conexiones HTTP ahora se sirven a través de HTTPS.

Edita la plantilla `chat/room.html` de la aplicación `chat` y busca la siguiente línea en el bloque `domready`:

```javascript
const url = 'ws://' + window.location.host +
```

Reemplaza esa línea con la siguiente:

```javascript
const url = 'wss://' + window.location.host +
```

Al utilizar `wss://` en lugar de `ws://`, te estás conectando explícitamente a un WebSocket seguro.

#### Inclusión de Daphne en la configuración de NGINX

En tu configuración de producción, ejecutarás Daphne en un socket UNIX y utilizarás NGINX delante de él. NGINX pasará las solicitudes a Daphne según la ruta solicitada. Expondrás Daphne a NGINX a través de una interfaz de socket UNIX, al igual que la configuración de uWSGI.

Edita el archivo `config/nginx/default.conf.template` y haz que se vea de la siguiente manera:

```nginx
# upstream for uWSGI
upstream uwsgi_app {
    server unix:/code/educa/uwsgi_app.sock;
}

# upstream for Daphne
upstream daphne {
    server daphne:9001;
}

server {
    listen 80;
    server_name www.educaproject.com educaproject.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    ssl_certificate /code/educa/ssl/educa.crt;
    ssl_certificate_key /code/educa/ssl/educa.key;
    server_name www.educaproject.com educaproject.com;
    error_log stderr warn;
    access_log /dev/stdout main;

    location / {
        include /etc/nginx/uwsgi_params;
        uwsgi_pass uwsgi_app;
    }

    location /ws/ {
        proxy_pass http://daphne;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_redirect off;
    }

    location /static/ {
        alias /code/educa/static/;
    }

    location /media/ {
        alias /code/educa/media/;
    }
}
```

En esta configuración, estableces un nuevo upstream llamado `daphne`, que apunta al host `daphne` y al puerto 9001. En el bloque `server`, configuras la ubicación `/ws/` para reenviar solicitudes a Daphne. Utilizas la directiva `proxy_pass` para pasar solicitudes a Daphne e incluyes algunas directivas de proxy adicionales.

Con esta configuración, NGINX pasará cualquier solicitud de URL que comience con el prefijo `/ws/` a Daphne y el resto a uWSGI, excepto los archivos bajo las rutas `/static/` o `/media/`, que serán servidos directamente por NGINX.

La Figura 17.13 muestra la configuración de producción final, incluido el servidor Daphne:

> *Figura 17.13: El ciclo de solicitud/respuesta del entorno de producción, incluido Daphne*

NGINX se ejecuta delante de uWSGI y Daphne como un servidor proxy inverso. NGINX está de cara a la web y pasa las solicitudes al servidor de aplicaciones (uWSGI o Daphne) según su prefijo de ruta. Además de esto, NGINX también sirve archivos estáticos y redirige las solicitudes no seguras a seguras. Esta configuración reduce el tiempo de inactividad, consume menos recursos del servidor y proporciona un mayor rendimiento y seguridad.

Detén la aplicación Docker desde la consola presionando las teclas Ctrl + C o usando el botón de parada en la aplicación Docker Desktop. Luego, inicia Compose nuevamente con el siguiente comando:

```bash
docker compose up
```

Usa tu navegador para crear un curso de muestra con un usuario instructor, inicia sesión con un usuario que esté inscrito en el curso y abre `https://educaproject.com/chat/room/1/` con tu navegador. Deberías poder enviar y recibir mensajes:

> *Figura 17.14: Mensajes de la sala de chat del curso servidos con NGINX y Daphne*

Daphne está funcionando correctamente y NGINX le está pasando solicitudes WebSocket. Todas las conexiones están aseguradas con SSL/TLS.

¡Felicitaciones! Has creado una pila personalizada lista para producción utilizando NGINX, uWSGI y Daphne. Podrías realizar una mayor optimización para obtener un rendimiento adicional y una seguridad mejorada a través de los ajustes de configuración en NGINX, uWSGI y Daphne. ¡Sin embargo, esta configuración de producción es un gran comienzo!

Has utilizado Docker Compose para definir y ejecutar servicios en múltiples contenedores. Ten en cuenta que puedes usar Docker Compose tanto para entornos de desarrollo local como para entornos de producción. Puedes encontrar información adicional sobre el uso de Docker Compose en producción en [https://docs.docker.com/compose/production/](https://docs.docker.com/compose/production/).

Para entornos de producción más avanzados, necesitarás distribuir dinámicamente contenedores a través de un número variable de máquinas. Para ello, en lugar de Docker Compose, necesitarás un orquestador como el modo Docker Swarm o Kubernetes. Puedes encontrar información sobre el modo Docker Swarm en [https://docs.docker.com/engine/swarm/](https://docs.docker.com/engine/swarm/) y sobre Kubernetes en [https://kubernetes.io/docs/home/](https://kubernetes.io/docs/home/).

Ten en cuenta que la administración de sistemas e infraestructura en la nube exige experiencia en configuración, optimización y seguridad. Para garantizar un entorno de producción seguro y eficiente, considera incorporar a un experto en sistemas/DevOps o mejorar tu propia experiencia en estas áreas.

Ahora que tenemos un entorno completo que procesa solicitudes HTTP de manera eficiente, es un buen momento para profundizar en el middleware para el procesamiento de solicitudes/respuestas en toda nuestra aplicación.

---

### Creación de un middleware personalizado

Ya conoces la configuración `MIDDLEWARE`, que contiene los middlewares para tu proyecto. Puedes pensar en ello como un sistema de complementos de bajo nivel, que te permite implementar ganchos (*hooks*) que se ejecutan en el proceso de solicitud/respuesta. Cada middleware es responsable de alguna acción específica que se ejecutará para todas las solicitudes o respuestas HTTP.

Debes evitar agregar procesamiento costoso al middleware, ya que se ejecuta en cada solicitud individual.

La Figura 17.15 muestra la ejecución del middleware en Django:

> *Figura 17.15: Ejecución de middleware en Django*

Cuando se recibe una solicitud HTTP, el middleware se ejecuta en el orden de aparición en la configuración `MIDDLEWARE`. Cuando Django ha generado una respuesta HTTP, la respuesta pasa a través de todo el middleware de vuelta en orden inverso.

La Figura 17.16 muestra el orden de ejecución de los componentes de middleware incluidos en la configuración `MIDDLEWARE` al crear un proyecto con el comando de administración `startproject`:

> *Figura 17.16: Orden de ejecución para los componentes de middleware predeterminados*

El middleware se puede escribir como una función, de la siguiente manera:

```python
def my_middleware(get_response):
    def middleware(request):
        # Code executed for each request before
        # the view (and later middleware) are called.
        response = get_response(request)
        # Code executed for each request/response after
        # the view is called.
        return response
    return middleware
```

Una fábrica de middleware es un objeto ejecutable (*callable*) que toma un invocable `get_response` y devuelve un middleware. El invocable middleware toma una solicitud y devuelve una respuesta, al igual que una vista. El invocable `get_response` podría ser el siguiente middleware en la cadena o la vista real en el caso del último middleware enumerado.

Si algún middleware devuelve una respuesta sin llamar a su función invocable `get_response`, se cortocircuita el proceso; no se ejecuta ningún middleware adicional (ni la vista), y la respuesta regresa a través de las mismas capas por las que pasó la solicitud.

El orden de los componentes del middleware en la configuración `MIDDLEWARE` es muy importante porque cada componente puede depender de los datos establecidos en la solicitud por otros componentes del middleware ejecutados previamente.

Al agregar un nuevo middleware a la configuración `MIDDLEWARE`, asegúrate de colocarlo en la posición correcta.

Puedes encontrar más información sobre el middleware en [https://docs.djangoproject.com/en/5.2/topics/http/middleware/](https://docs.djangoproject.com/en/5.2/topics/http/middleware/).

#### Creación de un middleware de subdominio

Vas a crear un middleware personalizado para permitir que se pueda acceder a los cursos a través de un subdominio personalizado. Cada URL de detalle del curso, que se parece a `https://educaproject.com/course/django/`, también será accesible a través del subdominio que utiliza el slug del curso, como `https://django.educaproject.com/`. Los usuarios podrán utilizar el subdominio como un atajo para acceder a los detalles del curso. Cualquier solicitud a subdominios se redirigirá a cada URL de detalle del curso correspondiente.

El middleware puede residir en cualquier lugar dentro de tu proyecto. Sin embargo, se recomienda crear un archivo `middleware.py` en el directorio de tu aplicación.

Crea un nuevo archivo dentro del directorio de la aplicación `courses` y nómbralo `middleware.py`. Añade el siguiente código:

```python
from django.urls import reverse
from django.shortcuts import get_object_or_404, redirect
from .models import Course


def subdomain_course_middleware(get_response):
    """
    Subdomains for courses
    """
    def middleware(request):
        host_parts = request.get_host().split('.')
        if len(host_parts) > 2 and host_parts[0] != 'www':
            # get course for the given subdomain
            course = get_object_or_404(Course, slug=host_parts[0])
            course_url = reverse('course_detail', args=[course.slug])
            # redirect current request to the course_detail view
            url = '{}://{}{}'.format(
                request.scheme,
                '.'.join(host_parts[1:]),
                course_url
            )
            return redirect(url)
        response = get_response(request)
        return response
    return middleware
```

Cuando se recibe una solicitud HTTP, realizas las siguientes tareas:

1. Obtienes el nombre de host que se está utilizando en la solicitud y lo divides en partes. Por ejemplo, si el usuario está accediendo a `mycourse.educaproject.com`, generas la lista `['mycourse', 'educaproject', 'com']`.
2. Verificas si el nombre de host incluye un subdominio comprobando si la división generó más de dos elementos. Si el nombre de host incluye un subdominio y este no es `www`, intentas obtener el curso con el slug proporcionado en el subdominio.
3. Si no se encuentra un curso, lanzas una excepción HTTP 404. De lo contrario, rediriges el navegador a la URL de detalle del curso.

Edita el archivo `settings/base.py` del proyecto y añade `'courses.middleware.subdomain_course_middleware'` al final de la lista `MIDDLEWARE`, de la siguiente manera:

```python
MIDDLEWARE = [
    # ...
    'courses.middleware.subdomain_course_middleware',
]
```

El middleware ahora se ejecutará en cada solicitud.

Recuerda que los nombres de host autorizados para servir tu proyecto Django se especifican en la configuración `ALLOWED_HOSTS`. Cambiemos esta configuración para que cualquier subdominio posible de `educaproject.com` pueda servir a tu aplicación.

Edita el archivo `educa/settings/prod.py` y modifica la configuración `ALLOWED_HOSTS`, de la siguiente manera:

```python
ALLOWED_HOSTS = ['.educaproject.com']
```

Un valor que comienza con un punto se utiliza como comodín de subdominio; `'.educaproject.com'` coincidirá con `educaproject.com` y cualquier subdominio para este dominio, por ejemplo, `course.educaproject.com` y `django.educaproject.com`.

#### Servir múltiples subdominios con NGINX

Necesitas que NGINX pueda servir tu sitio con cualquier subdominio posible. Edita el archivo `config/nginx/default.conf.template` en estas dos ocurrencias:

```nginx
server_name www.educaproject.com educaproject.com;
```

Reemplaza las apariciones de la línea anterior con la siguiente:

```nginx
server_name *.educaproject.com educaproject.com;
```

Al utilizar el asterisco, esta regla se aplica a todos los subdominios de `educaproject.com`. Para probar tu middleware localmente, debes agregar los subdominios que deseas probar a `/etc/hosts`. Para probar el middleware con un objeto `Course` con el slug `django`, añade la siguiente línea a tu archivo `/etc/hosts`:

```text
127.0.0.1 django.educaproject.com
```

Detén la aplicación Docker desde la consola presionando las teclas Ctrl + C o usando el botón de parada en la aplicación Docker Desktop. Luego, inicia Compose nuevamente con el siguiente comando:

```bash
docker compose up
```

Luego, abre `https://django.educaproject.com/` en tu navegador. El middleware encontrará el curso por el subdominio y redirigirá tu navegador a `https://educaproject.com/course/django/`.

¡Tu middleware de subdominio personalizado está funcionando!

Ahora, profundizaremos en un tema final que es extremadamente útil para los proyectos: automatizar tareas y hacerlas disponibles como comandos.

---

### Implementación de comandos de gestión personalizados

Django permite que tus aplicaciones registren comandos de administración personalizados para la utilidad `manage.py`. Por ejemplo, utilizaste los comandos de administración `makemessages` y `compilemessages` en el Capítulo 11, Añadir internacionalización a tu tienda, para crear y compilar archivos de traducción.

Un comando de administración consiste en un módulo de Python que contiene una clase `Command` que hereda de `django.core.management.base.BaseCommand` o una de sus subclases. Puedes crear comandos simples o hacer que acepten argumentos posicionales y opcionales como entrada.

Django busca comandos de administración en el directorio `management/commands/` para cada aplicación activa en la configuración `INSTALLED_APPS`. Cada módulo encontrado se registra como un comando de administración que lleva su nombre.

Puedes obtener más información sobre los comandos de administración personalizados en [https://docs.djangoproject.com/en/5.2/howto/custom-management-commands/](https://docs.djangoproject.com/en/5.2/howto/custom-management-commands/).

Vas a crear un comando de administración personalizado para recordar a los estudiantes que se inscriban en al menos un curso. El comando enviará un recordatorio por correo electrónico a los usuarios que hayan estado registrados durante más tiempo que un período especificado y que aún no estén inscritos en ningún curso.

Crea la siguiente estructura de archivos dentro del directorio de la aplicación `students`:

```text
management/
    __init__.py
    commands/
        __init__.py
        enroll_reminder.py
```

Edita el archivo `enroll_reminder.py` y añade el siguiente código:

```python
import datetime
from django.conf import settings
from django.contrib.auth.models import User
from django.core.mail import send_mass_mail
from django.core.management.base import BaseCommand
from django.db.models import Count
from django.utils import timezone


class Command(BaseCommand):
    help = 'Sends an e-mail reminder to users registered more' \
           'than N days that are not enrolled into any courses yet'

    def add_arguments(self, parser):
        parser.add_argument('--days', dest='days', type=int)

    def handle(self, *args, **options):
        emails = []
        subject = 'Enroll in a course'
        date_joined = timezone.now().today() - datetime.timedelta(
            days=options['days'] or 0
        )
        users = User.objects.annotate(
            course_count=Count('courses_joined')
        ).filter(course_count=0, date_joined__date__lte=date_joined)
        for user in users:
            message = f"""Dear {user.first_name},
We noticed that you didn't enroll in any courses yet. What are you waiting for?"""
            emails.append(
                (
                    subject,
                    message,
                    settings.DEFAULT_FROM_EMAIL,
                    [user.email]
                )
            )
        send_mass_mail(emails)
        self.stdout.write(f'Sent {len(emails)} reminders')
```

Este es tu comando `enroll_reminder`. El código anterior funciona de la siguiente manera:

1. La clase `Command` hereda de `BaseCommand`.
2. Incluyes un atributo `help`. Este atributo proporciona una breve descripción del comando que se imprime si ejecutas el comando `python manage.py help enroll_reminder`.
3. Utilizas el método `add_arguments()` para añadir el argumento con nombre `--days`. Este argumento se utiliza para especificar el número mínimo de días que un usuario debe estar registrado, sin haberse inscrito en ningún curso, para recibir el recordatorio.
4. El método `handle()` contiene el comando real. Obtienes el atributo `days` analizado desde la línea de comandos. Si no se establece, utilizas `0`, de modo que se envíe un recordatorio a todos los usuarios que no se hayan inscrito en un curso, independientemente de cuándo se hayan registrado. Utilizas la utilidad `timezone` proporcionada por Django para recuperar la fecha actual consciente de la zona horaria con `timezone.now().date()`. (Puedes configurar la zona horaria para tu proyecto con la configuración `TIME_ZONE`). Recuperas los usuarios que han estado registrados durante más de los días especificados y que aún no están inscritos en ningún curso. Logras esto anotando el QuerySet con el número total de cursos en los que está inscrito cada usuario. Generas el correo electrónico de recordatorio para cada usuario y lo agregas a la lista `emails`. Finalmente, envías los correos electrónicos utilizando la función `send_mass_mail()`, que está optimizada para abrir una sola conexión SMTP para enviar todos los correos electrónicos, en lugar de abrir una conexión por correo electrónico enviado.

Has creado tu primer comando de administración. Abre la consola y ejecuta tu comando:

```bash
docker compose exec web python /code/educa/manage.py \
    enroll_reminder --days=20 --settings=educa.settings.prod
```

Si no tienes un servidor SMTP local en ejecución, puedes consultar el Capítulo 2, Mejorar tu blog con funciones avanzadas, donde configuraste los ajustes de SMTP para tu primer proyecto Django. Alternativamente, puedes agregar la siguiente configuración al archivo `base.py` para que Django envíe correos electrónicos a la salida estándar durante el desarrollo:

```python
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

Django también incluye una utilidad para llamar a comandos de administración mediante Python. Puedes ejecutar comandos de administración desde tu código de la siguiente manera:

```python
from django.core import management

management.call_command('enroll_reminder', days=20)
```

¡Felicitaciones! Ahora puedes crear comandos de administración personalizados para tus aplicaciones.

Los comandos de administración de Django se pueden programar para que se ejecuten automáticamente mediante herramientas como `cron` o `Celery Beat`. Cron es un programador de trabajos basado en tiempo en sistemas operativos tipo Unix que permite a los usuarios programar scripts o comandos para que se ejecuten en momentos e intervalos específicos. Puedes leer más sobre cron en [https://en.wikipedia.org/wiki/Cron](https://en.wikipedia.org/wiki/Cron). Por otro lado, Celery Beat es un programador que trabaja con Celery para ejecutar funciones a intervalos designados. Puedes obtener más información sobre Celery Beat en [https://docs.celeryq.dev/en/stable/userguide/periodic-tasks.html](https://docs.celeryq.dev/en/stable/userguide/periodic-tasks.html). Al usar cron o Celery Beat, puedes asegurarte de que tus tareas se ejecuten con regularidad sin intervención manual.

Para acceder al Apéndice del libro, que contiene más información sobre Django 5.1 y 5.2, sigue este enlace: [https://packt.link/1g7Af](https://packt.link/1g7Af).

---

### Resumen

En este capítulo, creaste un entorno de producción utilizando Docker Compose. Configuraste NGINX, uWSGI y Daphne para servir tu aplicación en producción. Aseguraste tu entorno utilizando SSL/TLS. También implementaste un middleware personalizado y aprendiste a crear comandos de administración personalizados.

Has llegado al final de este libro. ¡Felicitaciones! Has aprendido las habilidades necesarias para crear aplicaciones web exitosas con Django. Este libro te ha guiado a través del proceso de desarrollo de proyectos de la vida real y la integración de Django con otras tecnologías. Ahora, estás listo para crear tu propio proyecto Django, ya sea un prototipo simple o una aplicación web a gran escala.

¡Buena suerte con tu próxima aventura con Django!

---

### Ampliación de tu proyecto mediante IA

En esta sección, se te presenta una tarea para ampliar tu proyecto, acompañada de un prompt de muestra para ChatGPT que te ayudará. Para interactuar con ChatGPT, visita [https://chat.openai.com/](https://chatgpt.com/). Si esta es tu primera interacción con ChatGPT, puedes volver a visitar la sección *Ampliación de tu proyecto mediante IA* en el Capítulo 3, Ampliar tu aplicación de blog.

Hemos desarrollado una completa plataforma de e-learning. Sin embargo, cuando los estudiantes están inscritos en múltiples cursos, cada uno con varios módulos, puede resultar difícil para ellos recordar dónde lo dejaron por última vez. Para abordar esto, usemos ChatGPT junto con Redis para almacenar y recuperar el progreso de cada estudiante dentro de un curso. Para obtener orientación, consulta el prompt proporcionado en [https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter17/prompts/task.md](https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter17/prompts/task.md).

Cuando estés refinando tu código de Python, ChatGPT puede ayudarte a explorar diferentes estrategias de refactorización. Comenta tu enfoque actual y ChatGPT puede brindarte consejos para hacer que tu código sea más pitónico, utilizando principios como *Don't Repeat Yourself* (DRY) y diseño modular para un código más limpio y fácil de mantener.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter17](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter17)
- **Descripción general de Docker Compose:** [https://docs.docker.com/compose/](https://docs.docker.com/compose/)
- **Instalación de Docker Compose:** [https://docs.docker.com/compose/install/compose-desktop/](https://docs.docker.com/compose/install/compose-desktop/)
- **Imagen oficial de Python en Docker:** [https://hub.docker.com/_/python](https://hub.docker.com/_/python)
- **Referencia de Dockerfile:** [https://docs.docker.com/reference/dockerfile/](https://docs.docker.com/reference/dockerfile/)
- **Archivo requirements.txt para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter17/requirements.txt](https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter17/requirements.txt)
- **Ejemplo de archivo YAML:** [https://yaml.org/](https://yaml.org/)
- **Sección build de Dockerfile:** [https://docs.docker.com/compose/compose-file/build/](https://docs.docker.com/compose/compose-file/build/)
- **Política de reinicio de Docker:** [https://docs.docker.com/config/containers/start-containers-automatically/](https://docs.docker.com/config/containers/start-containers-automatically/)
- **Volúmenes de Docker:** [https://docs.docker.com/storage/volumes/](https://docs.docker.com/storage/volumes/)
- **Especificación de Docker Compose:** [https://docs.docker.com/compose/compose-file/](https://docs.docker.com/compose/compose-file/)
- **Imagen oficial de PostgreSQL en Docker:** [https://hub.docker.com/_/postgres](https://hub.docker.com/_/postgres)
- **Script bash wait-for-it.sh para Docker:** [https://github.com/vishnubob/wait-for-it/blob/master/wait-for-it.sh](https://github.com/vishnubob/wait-for-it/blob/master/wait-for-it.sh)
- **Orden de inicio de servicios en Compose:** [https://docs.docker.com/compose/startup-order/](https://docs.docker.com/compose/startup-order/)
- **Imagen oficial de Redis en Docker:** [https://hub.docker.com/_/redis](https://hub.docker.com/_/redis)
- **Documentación de WSGI:** [https://wsgi.readthedocs.io/en/latest/](https://wsgi.readthedocs.io/en/latest/)
- **Lista de opciones de uWSGI:** [https://uwsgi-docs.readthedocs.io/en/latest/Options.html](https://uwsgi-docs.readthedocs.io/en/latest/Options.html)
- **Imagen oficial de NGINX en Docker:** [https://hub.docker.com/_/nginx](https://hub.docker.com/_/nginx)
- **Documentación de NGINX:** [https://nginx.org/en/docs/](https://nginx.org/en/docs/)
- **Configuración ALLOWED_HOSTS:** [https://docs.djangoproject.com/en/5.2/ref/settings/#allowed-hosts](https://docs.djangoproject.com/en/5.2/ref/settings/#allowed-hosts)
- **Framework de comprobación del sistema de Django:** [https://docs.djangoproject.com/en/5.2/topics/checks/](https://docs.djangoproject.com/en/5.2/topics/checks/)
- **Política HTTP Strict Transport Security con Django:** [https://docs.djangoproject.com/en/5.2/ref/middleware/#http-strict-transport-security](https://docs.djangoproject.com/en/5.2/ref/middleware/#http-strict-transport-security)
- **Lista de verificación de despliegue de Django:** [https://docs.djangoproject.com/en/5.2/howto/deployment/checklist/](https://docs.djangoproject.com/en/5.2/howto/deployment/checklist/)
- **Directorio de certificados SSL/TLS autogenerados:** [https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter17/educa/ssl/](https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter17/educa/ssl/)
- **Autoridad de certificación Let's Encrypt:** [https://letsencrypt.org/](https://letsencrypt.org/)
- **Uso de Docker Compose en producción:** [https://docs.docker.com/compose/production/](https://docs.docker.com/compose/production/)
- **Modo Docker Swarm:** [https://docs.docker.com/engine/swarm/](https://docs.docker.com/engine/swarm/)
- **Kubernetes:** [https://kubernetes.io/docs/home/](https://kubernetes.io/docs/home/)
- **Middleware de Django:** [https://docs.djangoproject.com/en/5.2/topics/http/middleware/](https://docs.djangoproject.com/en/5.2/topics/http/middleware/)
- **Creación de comandos de administración personalizados:** [https://docs.djangoproject.com/en/5.2/howto/custom-management-commands/](https://docs.djangoproject.com/en/5.2/howto/custom-management-commands/)
- **cron:** [https://en.wikipedia.org/wiki/Cron](https://en.wikipedia.org/wiki/Cron)
- **Celery Beat:** [https://docs.celeryq.dev/en/stable/userguide/periodic-tasks.html](https://docs.celeryq.dev/en/stable/userguide/periodic-tasks.html)

