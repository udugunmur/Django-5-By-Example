# Parte 1: Aplicación de Blog

## Capítulo 1: Creación de una aplicación de blog

### Introducción

En este libro, aprenderás a construir proyectos web de nivel profesional utilizando Django. Este capítulo inicial te guiará a través de los componentes fundamentales de una aplicación Django, desde la instalación hasta el despliegue. Si aún no has configurado Django en tu máquina, la sección **Instalación de Django** te guiará paso a paso durante el proceso de instalación.

Antes de comenzar nuestro primer proyecto en Django, repasemos lo que estás a punto de aprender. Este capítulo te ofrecerá una visión general del framework. Te guiará a través de los diferentes componentes principales para crear una aplicación web completamente funcional: modelos, plantillas, vistas y URLs. Obtendrás una comprensión clara de cómo funciona Django y cómo interactúan los diferentes componentes del framework entre sí.

También aprenderás la diferencia entre proyectos y aplicaciones en Django, y conocerás las configuraciones más importantes de Django. Construirás una aplicación de blog sencilla que permitirá a los usuarios navegar por todas las publicaciones publicadas y leer publicaciones individuales. También crearás una interfaz de administración sencilla para gestionar y publicar artículos. En los siguientes dos capítulos, ampliarás la aplicación de blog con funcionalidades más avanzadas.

Considera este capítulo como tu hoja de ruta para construir una aplicación Django completa. No te preocupes si algunos componentes o conceptos te parecen poco claros al principio. Los diferentes componentes del framework se explorarán en detalle a lo largo de este libro.

Este capítulo cubrirá los siguientes temas:

- Instalación de Python
- Creación de un entorno virtual de Python
- Instalación de Django
- Creación y configuración de un proyecto Django
- Construcción de una aplicación Django
- Diseño de modelos de datos
- Creación y aplicación de migraciones de modelos
- Configuración de un sitio de administración para tus modelos
- Trabajo con QuerySets y managers de modelos
- Creación de vistas, plantillas y URLs
- Comprensión del ciclo de petición/respuesta de Django

Comenzarás instalando Python en tu máquina.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter01](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Todos los paquetes de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente del capítulo. Puedes seguir las instrucciones para instalar cada paquete de Python en las siguientes secciones, o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

> [!NOTE]
> **Tu compra incluye una copia gratuita en PDF + extras exclusivos**
> Tu compra incluye una copia en PDF sin DRM de este libro, una prueba de 7 días de la biblioteca Packt+ (no se requiere tarjeta de crédito) y extras exclusivos adicionales. Consulta la sección de beneficios gratuitos con tu libro en el Prefacio para desbloquearlos al instante y maximizar tu aprendizaje.

---

### Instalación de Python

Django 5.2 es compatible con Python 3.10, 3.11, 3.12 y 3.13. En los ejemplos de este libro, utilizaremos Python 3.12.

Si utilizas Linux o macOS, es probable que ya tengas Python instalado, pero es posible que necesites actualizar a una de las versiones compatibles recién mencionadas. Puedes descargar el instalador visitando [python.org/downloads](https://subscription.packtpub.com/book/web-development/9781805125457/1). Alternativamente, utiliza tu método preferido para instalar una de las versiones más recientes. Si utilizas Windows, también puedes descargar un instalador de Python desde el sitio web de Python.

Abre el intérprete de línea de comandos de tu máquina. Si utilizas macOS, presiona Comando + barra espaciadora para abrir Spotlight y escribe `Terminal` para abrir Terminal.app. Si utilizas Windows, abre el menú Inicio y escribe `powers` en el cuadro de búsqueda. Luego, haz clic en la aplicación Windows PowerShell para abrirla. Alternativamente, puedes utilizar el símbolo del sistema básico escribiendo `cmd` en el cuadro de búsqueda y haciendo clic en la aplicación Símbolo del sistema para abrirla.

Verifica que Python 3 esté instalado en tu máquina escribiendo el siguiente comando en la consola:

```bash
python3 --version
```

Si ves lo siguiente, entonces Python 3 está instalado en tu computadora:

```text
Python 3.12.9
```

Si obtienes un error, prueba el comando `python` en lugar de `python3`. Si usas Windows, se recomienda que reemplaces `python` con el comando `py`.

Si tu versión de Python instalada es inferior a la 3.12, o si Python no está instalado en tu computadora, descarga Python 3.12 desde [https://www.python.org/downloads/](https://subscription.packtpub.com/book/web-development/9781805125457/1) y sigue las instrucciones para instalarlo. En el sitio de descargas, puedes encontrar instaladores de Python para Windows, macOS y Linux.

A lo largo de este libro, cuando se haga referencia a Python en la consola, utilizaremos el comando `python`, aunque algunos sistemas pueden requerir el uso de `python3`. Si estás utilizando Linux o macOS y el Python de tu sistema es Python 2, deberás usar `python3` para utilizar la versión de Python 3 que instalaste. Ten en cuenta que Python 2 llegó al final de su vida útil en enero de 2020 y ya no debe utilizarse.

En Windows, `python` es el ejecutable de Python de tu instalación predeterminada de Python, mientras que `py` es el lanzador de Python. El lanzador de Python para Windows se introdujo en Python 3.3. Detecta qué versiones de Python están instaladas en tu máquina y delega automáticamente a la versión más reciente.

Si usas Windows, deberías usar el comando `py`. Puedes leer más sobre el lanzador de Python de Windows en [https://docs.python.org/3/using/windows.html#launcher](https://subscription.packtpub.com/book/web-development/9781805125457/1).

A continuación, vas a crear un entorno de Python para tu proyecto e instalar las bibliotecas de Python necesarias.

---

### Creación de un entorno virtual de Python

Cuando escribes aplicaciones en Python, normalmente utilizarás paquetes y módulos que no están incluidos en la biblioteca estándar de Python. Es posible que tengas aplicaciones de Python que requieran una versión diferente del mismo módulo. Sin embargo, solo se puede instalar una versión específica de un módulo a nivel de todo el sistema. Si actualizas la versión de un módulo para una aplicación, podrías terminar rompiendo otras aplicaciones que requieran una versión anterior de ese módulo.

Para solucionar este problema, puedes utilizar entornos virtuales de Python. Con los entornos virtuales, puedes instalar módulos de Python en una ubicación aislada en lugar de instalarlos a nivel de todo el sistema. Cada entorno virtual tiene su propio binario de Python y puede tener su propio conjunto independiente de paquetes de Python instalados en su directorio `site-packages`.

Desde la versión 3.3, Python incluye la biblioteca `venv`, que proporciona soporte para crear entornos virtuales ligeros. Al utilizar el módulo `venv` de Python para crear entornos aislados de Python, puedes utilizar diferentes versiones de paquetes para diferentes proyectos. Otra ventaja de usar `venv` es que no necesitarás privilegios administrativos para instalar paquetes de Python.

Si estás utilizando Linux o macOS, crea un entorno aislado con el siguiente comando:

```bash
python -m venv my_env
```

Recuerda usar `python3` en lugar de `python` si tu sistema viene con Python 2 e instalaste Python 3.

Si estás utilizando Windows, usa el siguiente comando en su lugar:

```bash
py -m venv my_env
```

Esto utilizará el lanzador de Python en Windows.

El comando anterior creará un entorno de Python en un nuevo directorio llamado `my_env`. Cualquier biblioteca de Python que instales mientras tu entorno virtual esté activo irá al directorio `my_env/lib/python3.12/site-packages`.

Si estás utilizando Linux o macOS, ejecuta el siguiente comando para activar tu entorno virtual:

```bash
source my_env/bin/activate
```

Si estás utilizando Windows, usa el siguiente comando en su lugar:

```bash
.\my_env\Scripts\activate
```

El indicador de la consola incluirá el nombre del entorno virtual activo entre paréntesis, de esta forma:

```bash
(my_env) zenx@pc:~ zenx$
```

Puedes desactivar tu entorno en cualquier momento con el comando `deactivate`. Puedes encontrar más información sobre `venv` en [https://docs.python.org/3/library/venv.html](https://subscription.packtpub.com/book/web-development/9781805125457/1).

---

### Instalación de Django

Si ya has instalado Django 5.2, puedes omitir esta sección e ir directamente a la sección **Creación de tu primer proyecto**.

Django se distribuye como un módulo de Python y, por lo tanto, se puede instalar en cualquier entorno de Python. Si aún no has instalado Django, la siguiente es una guía rápida para instalarlo en tu máquina.

#### Instalación de Django con pip

El sistema de gestión de paquetes `pip` es el método preferido para instalar Django. Python 3.12 viene con `pip` preinstalado, pero puedes encontrar instrucciones de instalación de `pip` en [https://pip.pypa.io/en/stable/installation/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Ejecuta el siguiente comando en la consola para instalar Django con `pip`:

```bash
python -m pip install Django~=5.2
```

Esto instalará la versión 5.2 más reciente de Django en el directorio `site-packages` de Python de tu entorno virtual.

Ahora comprobaremos si Django se ha instalado correctamente. Ejecuta el siguiente comando en la consola:

```bash
python -m django --version
```

Si obtienes una salida que comienza con `5.2`, Django se ha instalado correctamente en tu máquina. Si recibes el mensaje `No module named django`, Django no está instalado en tu máquina. Si tienes problemas al instalar Django, puedes revisar las diferentes opciones de instalación descritas en [https://docs.djangoproject.com/en/5.2/intro/install/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Todos los paquetes de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente del capítulo mencionado anteriormente. Puedes seguir las instrucciones para instalar cada paquete de Python en las siguientes secciones, o puedes instalar todos los requisitos a la vez con el comando `pip install -r requirements.txt`.

---

### Visión general de Django

Django es un framework que consta de un conjunto de componentes que resuelven problemas comunes de desarrollo web. Los componentes de Django están débilmente acoplados (loosely coupled), lo que significa que se pueden gestionar de forma independiente. Esto ayuda a separar las responsabilidades de las diferentes capas del framework: la capa de base de datos no sabe nada sobre cómo se muestran los datos, el sistema de plantillas no sabe nada sobre las peticiones web, y así sucesivamente.

Django ofrece la máxima reutilización de código siguiendo el principio **DRY** (*don't repeat yourself*). Django también fomenta el desarrollo rápido y te permite usar menos código aprovechando las capacidades dinámicas de Python, como la introspección.

Puedes leer más sobre las filosofías de diseño de Django en [https://docs.djangoproject.com/en/5.2/misc/design-philosophies/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

#### Componentes principales del framework

Django sigue el patrón **MTV** (*Model-Template-View*). Es un patrón ligeramente similar al conocido patrón **MVC** (*Model-View-Controller*), donde la plantilla actúa como la vista y el propio framework actúa como el controlador.

Las responsabilidades en el patrón MTV de Django se dividen de la siguiente manera:

- **Model (Modelo):** Define la estructura lógica de los datos y es el manejador de datos entre la base de datos y la vista.
- **Template (Plantilla):** Es la capa de presentación. Django utiliza un sistema de plantillas de texto plano que conserva todo lo que el navegador web renderiza.
- **View (Vista):** Se comunica con la base de datos a través del modelo y transfiere los datos a la plantilla para su visualización.

El propio framework actúa como el controlador. Envía una petición a la vista correspondiente según la configuración de URLs de Django.

Al desarrollar cualquier proyecto de Django, siempre trabajarás con modelos, vistas, plantillas y URLs. En este capítulo, aprenderás cómo encajan entre sí.

#### La arquitectura de Django

La Figura 1.4 muestra cómo Django procesa las peticiones y cómo se gestiona el ciclo de petición/respuesta con los diferentes componentes principales de Django: URLs, vistas, modelos y plantillas:

> *Figura 1.4: La arquitectura de Django*

Así es como Django gestiona las peticiones HTTP y genera respuestas:

1. Un navegador web solicita una página mediante su URL y el servidor web pasa la petición HTTP a Django.
2. Django recorre sus patrones de URL configurados y se detiene en el primero que coincida con la URL solicitada.
3. Django ejecuta la vista correspondiente al patrón de URL coincidente.
4. La vista potencialmente utiliza modelos de datos para recuperar información de la base de datos.
5. Los modelos de datos proporcionan definiciones de datos y comportamientos. Se utilizan para consultar la base de datos.
6. La vista renderiza una plantilla (habitualmente HTML) para mostrar los datos y la devuelve con una respuesta HTTP.

Volveremos al ciclo de petición/respuesta de Django al final de este capítulo en la sección **El ciclo de petición/respuesta**.

Django también incluye puntos de enganche (*hooks*) en el proceso de petición/respuesta, llamados **middleware**. El middleware se ha omitido intencionadamente de este diagrama en aras de la simplicidad. Utilizarás middleware en diferentes ejemplos de este libro y aprenderás a crear middleware personalizado en el Capítulo 17, *Puesta en producción*.

Hemos cubierto los elementos fundamentales de Django y cómo procesa las peticiones. Exploremos las nuevas características introducidas en Django 5.

#### Nuevas características en Django 5

Django 5 introdujo varias características clave que utilizarás en los ejemplos de este libro. Esta versión también dejó obsoletas ciertas funciones y eliminó funcionalidades previamente desaprobadas. Estas son algunas de las principales novedades introducidas:

- **Filtros por facetas en el sitio de administración:** Ahora se pueden añadir filtros por facetas al sitio de administración. Cuando se activan, se muestran los recuentos de facetas para los filtros aplicados en la lista de objetos de administración. Esta característica se presenta en la sección *Adición de conteos de facetas a los filtros* de este capítulo.
- **Plantillas simplificadas para el renderizado de campos de formulario:** El renderizado de campos de formulario se ha simplificado con la capacidad de definir grupos de campos con plantillas asociadas. Esto tiene como objetivo hacer que el proceso de renderizado de elementos relacionados de un campo de formulario de Django, como etiquetas, widgets, textos de ayuda y errores, sea más ágil. Se puede encontrar un ejemplo de uso de grupos de campos en la sección *Creación de plantillas para el formulario de comentarios* del Capítulo 2, *Mejora de tu blog y adición de funciones sociales*.
- **Valores predeterminados calculados por la base de datos:** Django añade valores predeterminados calculados por la base de datos. Se presenta un ejemplo de esta característica en la sección *Adición de campos de fecha y hora* de este capítulo.
- **Campos de modelo generados por la base de datos:** Este es un nuevo tipo de campo que te permite crear columnas generadas por la base de datos. Se utiliza una expresión para establecer automáticamente el valor del campo cada vez que se modifica el modelo. El valor del campo se establece utilizando la sintaxis SQL `GENERATED ALWAYS`.
- **Más opciones para declarar opciones (*choices*) de campos de modelo:** Los campos que admiten opciones ya no requieren acceder al atributo `.choices` para acceder a los tipos de enumeración. Se puede utilizar directamente un mapeo o un elemento invocable (*callable*) en lugar de un iterable para expandir los tipos de enumeración. Las opciones con tipos de enumeración en este libro se han actualizado para reflejar estos cambios. Se puede encontrar un ejemplo de esto en la sección *Adición de un campo de estado* de este capítulo.

Django 5 también incluyó algunas mejoras en el soporte asíncrono. El soporte para la interfaz de puerta de enlace de servidor asíncrono (*ASGI*) se introdujo por primera vez en Django 3 y mejoró en Django 4.1 con controladores asíncronos para vistas basadas en clases y una interfaz ORM asíncrona. Django 5 añade funciones asíncronas al framework de autenticación, proporciona soporte para el envío asíncrono de señales y añade soporte asíncrono a múltiples decoradores integrados.

Desde entonces, se han lanzado Django 5.1 y 5.2. Django 5.1 eliminó la compatibilidad con Python 3.8 y 3.9, e introdujo las siguientes características principales:

- **La etiqueta de plantilla `{% querystring %}`:** Esta etiqueta de plantilla simplifica la modificación de parámetros de consulta en URLs, facilitando la generación de enlaces que mantienen los parámetros de consulta existentes mientras se añaden o modifican los específicos.
- **Soporte de pool de conexiones para PostgreSQL:** Establecer una nueva conexión puede llevar un tiempo relativamente largo, pero mantener las conexiones abiertas puede reducir la latencia.
- **LoginRequiredMiddleware:** Si está instalado, el nuevo middleware redirige todas las peticiones no autenticadas a una página de inicio de sesión. Las vistas pueden permitir peticiones no autenticadas mediante el nuevo decorador `login_not_required()`.

Por su parte, Django 5.2 introdujo las siguientes novedades destacadas:

- **Importaciones automáticas de modelos en la shell:** La shell de Django ahora importa automáticamente los modelos de tus aplicaciones instaladas, lo que hace que la ejecución de consultas sea mucho más rápida al eliminar la necesidad de importar los modelos que necesitas.
- **Claves primarias compuestas (*Composite Primary Keys*):** Django ahora está mejor alineado con la funcionalidad de SQL, con la capacidad de definir una clave primaria compuesta para tus modelos. En lugar de los valores enteros autoincrementales habituales, defines campos en el modelo cuyos valores se convierten en la clave primaria.
- **Sobrescritura de BoundField:** Ahora es más fácil sobrescribir `BoundField` en formularios, lo que simplifica enormemente la adición de clases para estilos.

Para obtener más información sobre los cambios introducidos en 5.1 y 5.2, consulta el Apéndice. Al tratarse de una versión basada en tiempo, no hay cambios drásticos en Django 5, lo que hace que actualizar aplicaciones de Django 4 a la versión 5.2 sea un proceso sencillo.

Puedes acceder al Apéndice a través del siguiente enlace: [https://packt.link/1g7Af](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Si deseas actualizar rápidamente un proyecto Django existente a la versión 5.2, puedes usar la herramienta `django-upgrade`. Este paquete reescribe los archivos de tu proyecto aplicando correctores hasta una versión de destino. Puedes encontrar instrucciones para usar `django-upgrade` en [https://github.com/adamchainz/django-upgrade](https://subscription.packtpub.com/book/web-development/9781805125457/1).

La herramienta `django-upgrade` está inspirada en el paquete `pyupgrade`. Puedes usar `pyupgrade` para actualizar automáticamente la sintaxis para versiones más recientes de Python. Puedes encontrar más información sobre `pyupgrade` en [https://github.com/asottile/pyupgrade](https://subscription.packtpub.com/book/web-development/9781805125457/1).

---

### Creación de tu primer proyecto

Tu primer proyecto de Django consistirá en una aplicación de blog. Esto te ofrecerá una sólida introducción a las capacidades y funcionalidades de Django.

Un blog es el punto de partida perfecto para construir un proyecto completo en Django, dada su amplia gama de características requeridas, desde la gestión básica de contenidos hasta funcionalidades avanzadas como comentarios, compartir publicaciones, búsqueda y recomendaciones de publicaciones. El proyecto del blog se cubrirá en los tres primeros capítulos de este libro.

En este capítulo, comenzaremos creando el proyecto Django y una aplicación Django para el blog. Luego crearemos nuestros modelos de datos y los sincronizaremos con la base de datos. Finalmente, crearemos un sitio de administración para el blog y construiremos las vistas, plantillas y URLs.

La Figura 1.5 muestra una representación de las páginas de la aplicación de blog que vas a crear:

> *Figura 1.5: Diagrama de funcionalidades construidas en el Capítulo 1*

La aplicación de blog consistirá en una lista de publicaciones que incluirá el título de la publicación, la fecha de publicación, el autor, un extracto de la publicación y un enlace para leer el artículo. La página de lista de publicaciones se implementará con la vista `post_list`. Aprenderás a crear vistas en este capítulo.

Cuando los lectores hagan clic en el enlace de una publicación en la página de lista de publicaciones, serán redirigidos a una vista individual (detalle) de la publicación. La vista de detalle mostrará el título, la fecha de publicación, el autor y el cuerpo completo de la publicación.

Comencemos creando el proyecto Django para nuestro blog. Django proporciona un comando que te permite crear una estructura inicial de archivos de proyecto.

Ejecuta el siguiente comando en tu consola:

```bash
django-admin startproject mysite
```

Esto creará un proyecto Django con el nombre `mysite`.

> [!WARNING]
> Evita nombrar proyectos con nombres de módulos integrados de Python o Django para prevenir conflictos.

Echemos un vistazo a la estructura generada del proyecto:

```text
mysite/
    manage.py
    mysite/
        __init__.py
        asgi.py
        settings.py
        urls.py
        wsgi.py
```

El directorio exterior `mysite/` es el contenedor de nuestro proyecto. Contiene los siguientes archivos:

- `manage.py`: Esta es una utilidad de línea de comandos utilizada para interactuar con tu proyecto. Por lo general, no necesitarás editar este archivo.
- `mysite/`: Este es el paquete de Python para tu proyecto, que consta de los siguientes archivos:
  - `__init__.py`: Un archivo vacío que le indica a Python que trate el directorio `mysite` como un módulo de Python.
  - `asgi.py`: Esta es la configuración para ejecutar tu proyecto como una aplicación ASGI con servidores web compatibles con ASGI. ASGI es el estándar emergente de Python para servidores y aplicaciones web asíncronas.
  - `settings.py`: Este archivo indica las configuraciones del proyecto y contiene los ajustes predeterminados iniciales.
  - `urls.py`: Este es el lugar donde residen tus patrones de URL. Cada URL definida aquí se asigna a una vista.
  - `wsgi.py`: Esta es la configuración para ejecutar tu proyecto como una aplicación WSGI (*Web Server Gateway Interface*) con servidores web compatibles con WSGI.

#### Aplicación de migraciones iniciales de la base de datos

Las aplicaciones Django requieren una base de datos para almacenar datos. El archivo `settings.py` contiene la configuración de la base de datos para tu proyecto en el ajuste `DATABASES`. La configuración predeterminada es una base de datos SQLite3. SQLite viene empaquetado con Python 3 y se puede utilizar en cualquiera de tus aplicaciones Python. SQLite es una base de datos ligera que puedes usar con Django para desarrollo. Si planeas desplegar tu aplicación en un entorno de producción, debes usar una base de datos con todas las funciones, como PostgreSQL, MySQL u Oracle. Puedes encontrar más información sobre cómo poner en marcha tu base de datos con Django en [https://docs.djangoproject.com/en/5.2/topics/install/#database-installation](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Tu archivo `settings.py` también incluye una lista llamada `INSTALLED_APPS` que contiene aplicaciones comunes de Django que se agregan a tu proyecto de forma predeterminada. Repasaremos estas aplicaciones más adelante en la sección **Configuración del proyecto**.

Las aplicaciones de Django contienen modelos de datos que se asignan a tablas de base de datos. Crearás tus propios modelos en la sección **Creación de los modelos de datos del blog**. Para completar la configuración del proyecto, debes crear las tablas asociadas con los modelos de las aplicaciones predeterminadas de Django incluidas en la configuración `INSTALLED_APPS`. Django viene con un sistema que te ayuda a gestionar las migraciones de bases de datos.

Abre la consola y ejecuta los siguientes comandos:

```bash
cd mysite
python manage.py migrate
```

Verás una salida que finaliza con las siguientes líneas:

```text
Applying contenttypes.0001_initial... OK
Applying auth.0001_initial... OK
Applying admin.0001_initial... OK
Applying admin.0002_logentry_remove_auto_add... OK
Applying admin.0003_logentry_add_action_flag_choices... OK
Applying contenttypes.0002_remove_content_type_name... OK
Applying auth.0002_alter_permission_name_max_length... OK
Applying auth.0003_alter_user_email_max_length... OK
Applying auth.0004_alter_user_username_opts... OK
Applying auth.0005_alter_user_last_login_null... OK
Applying auth.0006_require_contenttypes_0002... OK
Applying auth.0007_alter_validators_add_error_messages... OK
Applying auth.0008_alter_user_username_max_length... OK
Applying auth.0009_alter_user_last_name_max_length... OK
Applying auth.0010_alter_group_name_max_length... OK
Applying auth.0011_update_proxy_permissions... OK
Applying auth.0012_alter_user_first_name_max_length... OK
Applying sessions.0001_initial... OK
```

Las líneas anteriores son las migraciones de base de datos aplicadas por Django. Al aplicar las migraciones iniciales, se crean en la base de datos las tablas para las aplicaciones enumeradas en la configuración `INSTALLED_APPS`.

Aprenderás más sobre el comando de gestión `migrate` en la sección **Creación y aplicación de migraciones** de este capítulo.

#### Ejecución del servidor de desarrollo

Django viene con un servidor web ligero para ejecutar tu código rápidamente, sin necesidad de dedicar tiempo a configurar un servidor de producción. Cuando ejecutas el servidor de desarrollo de Django, este continúa buscando cambios en tu código. Se recarga automáticamente, liberándote de tener que recargarlo manualmente tras los cambios en el código. Sin embargo, es posible que no note algunas acciones, como agregar nuevos archivos a tu proyecto, por lo que tendrás que reiniciar el servidor manualmente en estos casos.

Inicia el servidor de desarrollo escribiendo el siguiente comando en la consola:

```bash
python manage.py runserver
```

Deberías ver algo como esto:

```text
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
January 01, 2024 - 10:00:00
Django version 5.2, using settings 'mysite.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

Ahora, abre `http://127.0.0.1:8000/` en tu navegador. Deberías ver una página que indica que el proyecto se está ejecutando correctamente, como se muestra en la Figura 1.6:

> *Figura 1.6: La página predeterminada del servidor de desarrollo de Django*

La captura de pantalla anterior indica que Django se está ejecutando. Si observas tu consola, verás la petición GET realizada por tu navegador:

```text
[03/Jun/2025 12:54:08] "GET / HTTP/1.1" 200 12068
```

Cada petición HTTP queda registrada en la consola por el servidor de desarrollo. Cualquier error que ocurra durante la ejecución del servidor de desarrollo también aparecerá en la consola.

Puedes ejecutar el servidor de desarrollo de Django en un host y puerto personalizados o indicarle a Django que cargue un archivo de configuración específico, de la siguiente manera:

```bash
python manage.py runserver 127.0.0.1:8001 --settings=mysite.settings
```

Cuando tengas que lidiar con múltiples entornos que requieran diferentes configuraciones, puedes crear un archivo de configuración diferente para cada entorno.

Este servidor solo está destinado al desarrollo y no es adecuado para su uso en producción. Para desplegar Django en un entorno de producción, debes ejecutarlo como una aplicación WSGI utilizando un servidor web como Apache, Gunicorn o uWSGI, o como una aplicación ASGI utilizando un servidor como Daphne o Uvicorn. Puedes encontrar más información sobre cómo desplegar Django con diferentes servidores web en [https://docs.djangoproject.com/en/5.2/howto/deployment/wsgi/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

El Capítulo 17, *Puesta en producción*, explica cómo configurar un entorno de producción para tus proyectos Django.

#### Configuración del proyecto

Abramos el archivo `settings.py` y echemos un vistazo a la configuración del proyecto. Hay varios ajustes que Django incluye en este archivo, pero estos son solo una parte de todos los ajustes disponibles de Django. Puedes ver todos los ajustes y sus valores predeterminados en [https://docs.djangoproject.com/en/5.2/ref/settings/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Revisemos algunos de los ajustes del proyecto:

- `DEBUG`: Es un valor booleano que activa y desactiva el modo de depuración del proyecto. Si se establece en `True`, Django mostrará páginas de error detalladas cuando su aplicación lance una excepción no detectada. Cuando pases a un entorno de producción, recuerda que debes configurarlo en `False`. Nunca despliegues un sitio en producción con `DEBUG` activado porque expondrás datos confidenciales relacionados con el proyecto.
- `ALLOWED_HOSTS`: No se aplica mientras el modo de depuración está activado o cuando se ejecutan las pruebas. Una vez que muevas tu sitio a producción y configures `DEBUG` en `False`, tendrás que agregar tu dominio/host a este ajuste para permitirle servir tu sitio Django.
- `INSTALLED_APPS`: Es un ajuste que tendrás que editar para todos los proyectos. Este ajuste le indica a Django qué aplicaciones están activas para este sitio. De forma predeterminada, Django incluye las siguientes aplicaciones:
  - `django.contrib.admin`: Un sitio de administración.
  - `django.contrib.auth`: Un framework de autenticación.
  - `django.contrib.contenttypes`: Un framework para manejar tipos de contenido.
  - `django.contrib.sessions`: Un framework de sesiones.
  - `django.contrib.messages`: Un framework de mensajería.
  - `django.contrib.staticfiles`: Un framework para gestionar archivos estáticos, como CSS, archivos JavaScript e imágenes.
- `MIDDLEWARE`: Es una lista que contiene el middleware a ejecutar.
- `ROOT_URLCONF`: Indica el módulo de Python donde se definen los patrones de URL raíz de tu aplicación.
- `DATABASES`: Es un diccionario que contiene la configuración de todas las bases de datos que se utilizarán en el proyecto. Siempre debe haber una base de datos predeterminada (`default`). La configuración predeterminada utiliza una base de datos SQLite3.
- `LANGUAGE_CODE`: Define el código de idioma predeterminado para este sitio Django.
- `USE_TZ`: Le indica a Django que active/desactive la compatibilidad con zonas horarias. Django viene con soporte para fechas y horas compatibles con zonas horarias (*timezone-aware*). Este ajuste se establece en `True` cuando creas un nuevo proyecto mediante el comando de gestión `startproject`.

No te preocupes si no entiendes mucho de lo que estás viendo aquí. Aprenderás más sobre las diferentes configuraciones de Django en los siguientes capítulos.

#### Proyectos y aplicaciones

A lo largo de este libro, te encontrarás con los términos **proyecto** (*project*) y **aplicación** (*application*) una y otra vez. En Django, un proyecto se considera una instalación de Django con ciertas configuraciones. Una aplicación es un grupo de modelos, vistas, plantillas y URLs. Las aplicaciones interactúan con el framework para proporcionar funcionalidades específicas y se pueden reutilizar en varios proyectos. Puedes pensar en un proyecto como tu sitio web, que contiene varias aplicaciones, como un blog, una wiki o un foro, que también pueden ser utilizadas por otros proyectos de Django.

La Figura 1.7 muestra la estructura de un proyecto Django:

> *Figura 1.7: La estructura de proyecto/aplicación de Django*

#### Creación de una aplicación

Creemos nuestra primera aplicación Django. Construiremos una aplicación de blog desde cero.

Ejecuta el siguiente comando en la consola desde el directorio raíz del proyecto:

```bash
python manage.py startapp blog
```

Esto creará la estructura básica de la aplicación, que se verá así:

```text
blog/
    __init__.py
    admin.py
    apps.py
    migrations/
        __init__.py
    models.py
    tests.py
    views.py
```

Estos archivos son los siguientes:

- `__init__.py`: Un archivo vacío que le indica a Python que trate el directorio `blog` como un módulo de Python.
- `admin.py`: Aquí es donde registras los modelos para incluirlos en el sitio de administración de Django (el uso de este sitio es opcional).
- `apps.py`: Incluye la configuración principal de la aplicación `blog`.
- `migrations/`: Este directorio contendrá las migraciones de base de datos de la aplicación. Las migraciones permiten a Django rastrear los cambios en tus modelos y sincronizar la base de datos en consecuencia. Este directorio contiene un archivo `__init__.py` vacío.
- `models.py`: Incluye los modelos de datos de tu aplicación; todas las aplicaciones de Django deben tener un archivo `models.py`, pero se puede dejar vacío.
- `tests.py`: Aquí es donde puedes agregar pruebas para tu aplicación.
- `views.py`: La lógica de tu aplicación va aquí; cada vista recibe una petición HTTP, la procesa y devuelve una respuesta.

Con la estructura de la aplicación lista, podemos comenzar a construir los modelos de datos para el blog.

---

### Creación de los modelos de datos del blog

Recuerda que un objeto de Python es una colección de datos y métodos. Las clases son el modelo para agrupar datos y funcionalidad juntos. Crear una nueva clase crea un nuevo tipo de objeto, lo que te permite crear instancias de ese tipo.

Un modelo de Django es una fuente de información sobre los comportamientos de tus datos. Consiste en una clase de Python que hereda de `django.db.models.Model`. Cada modelo se asigna a una única tabla de base de datos, donde cada atributo de la clase representa un campo de base de datos.

Cuando creas un modelo, Django te proporcionará una API práctica para consultar objetos en la base de datos de manera sencilla.

Definiremos los modelos de base de datos para nuestra aplicación de blog. Luego, generaremos las migraciones de base de datos para los modelos con el fin de crear las tablas correspondientes en la base de datos. Al aplicar las migraciones, Django creará una tabla para cada modelo definido en el archivo `models.py` de la aplicación.

#### Creación del modelo Post

Primero, definiremos un modelo `Post` que nos permitirá almacenar publicaciones de blog en la base de datos.

Agrega las siguientes líneas al archivo `models.py` de la aplicación `blog`:

```python
from django.db import models


class Post(models.Model):
    title = models.CharField(max_length=250)
    slug = models.SlugField(max_length=250)
    body = models.TextField()

    def __str__(self):
        return self.title
```

Este es el modelo de datos para las publicaciones del blog. Las publicaciones tendrán un título, una etiqueta corta llamada `slug` y un cuerpo (`body`). Echemos un vistazo a los campos de este modelo:

- `title`: Este es el campo para el título de la publicación. Es un campo `CharField` que se traduce en una columna `VARCHAR` en la base de datos SQL.
- `slug`: Este es un campo `SlugField` que se traduce en una columna `VARCHAR` en la base de datos SQL. Un slug es una etiqueta corta que contiene solo letras, números, guiones bajos o guiones. Una publicación con el título *Django Reinhardt: A legend of Jazz* podría tener un slug como `django-reinhardt-legend-jazz`. Usaremos el campo `slug` para crear URLs hermosas y optimizadas para SEO para publicaciones de blog en el Capítulo 2, *Mejora de tu blog y adición de funciones sociales*.
- `body`: Este es el campo para almacenar el cuerpo de la publicación. Es un campo `TextField` que se traduce en una columna `TEXT` en la base de datos SQL.

También hemos agregado un método `__str__()` a la clase del modelo. Este es el método predeterminado de Python para devolver una cadena con la representación legible por humanos del objeto. Django utilizará este método para mostrar el nombre del objeto en muchos lugares, como el sitio de administración de Django.

Echemos un vistazo a cómo se traducirán el modelo y sus campos en una tabla y columnas de base de datos. El siguiente diagrama muestra el modelo `Post` y la tabla de base de datos correspondiente que creará Django cuando sincronicemos el modelo con la base de datos:

> *Figura 1.8: Correspondencia inicial entre el modelo Post y la tabla de base de datos*

Django creará una columna de base de datos para cada uno de los campos del modelo: `title`, `slug` y `body`. Puedes ver cómo cada tipo de campo corresponde a un tipo de datos de base de datos.

De forma predeterminada, Django agrega un campo de clave primaria autoincremental a cada modelo. El tipo de campo para este campo se especifica en la configuración de cada aplicación o globalmente en el ajuste `DEFAULT_AUTO_FIELD`. Al crear una aplicación con el comando `startapp`, el valor predeterminado para `DEFAULT_AUTO_FIELD` es `BigAutoField`. Este es un entero de 64 bits que se incrementa automáticamente según los IDs disponibles. Si no especificas una clave primaria para tu modelo, Django agrega este campo automáticamente. También puedes definir uno de los campos del modelo para que sea la clave primaria configurando `primary_key=True` en él.

Ampliaremos el modelo `Post` con campos y comportamientos adicionales. Una vez completo, lo sincronizaremos con la base de datos creando una migración de base de datos y aplicándola.

#### Adición de campos de fecha y hora

Continuaremos agregando diferentes campos de fecha y hora al modelo `Post`. Cada publicación se publicará en una fecha y hora específicas. Por lo tanto, necesitamos un campo para almacenar la fecha y la hora de publicación. También queremos almacenar la fecha y la hora en que se creó el objeto `Post` y cuándo se modificó por última vez.

Edita el archivo `models.py` de la aplicación `blog` para que quede de la siguiente manera:

```python
from django.db import models
from django.utils import timezone


class Post(models.Model):
    title = models.CharField(max_length=250)
    slug = models.SlugField(max_length=250)
    body = models.TextField()
    publish = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return self.title
```

Hemos añadido un campo `publish` al modelo `Post`. Este es un campo `DateTimeField` que se traduce en una columna `DATETIME` en la base de datos SQL. Lo utilizaremos para almacenar la fecha y la hora en que se publicó la entrada. Utilizamos el método `timezone.now` de Django como valor predeterminado para el campo. Observa que importamos el módulo `timezone` para utilizar este método. `timezone.now` devuelve la fecha y hora actuales en un formato compatible con zonas horarias (*timezone-aware*). Puedes considerarlo como una versión compatible con zonas horarias del método estándar de Python `datetime.now`.

Otro método para definir valores predeterminados para campos de modelo es mediante valores predeterminados calculados por la base de datos. Introducida en Django 5, esta característica te permite utilizar funciones subyacentes de la base de datos para generar valores predeterminados. Por ejemplo, el siguiente código utiliza la fecha y hora actuales del servidor de base de datos como valor predeterminado para el campo `publish`:

```python
from django.db import models
from django.db.models.functions import Now


class Post(models.Model):
    # ...
    publish = models.DateTimeField(db_default=Now())
```

Para utilizar valores predeterminados generados por la base de datos, usamos el atributo `db_default` en lugar de `default`. En este ejemplo, utilizamos la función de base de datos `Now`. Cumple un propósito similar a `default=timezone.now`, pero en lugar de una fecha y hora generadas por Python, utiliza la función de base de datos `NOW()` para producir el valor inicial. Puedes leer más sobre el atributo `db_default` en [https://docs.djangoproject.com/en/5.2/ref/models/fields/#django.db.models.Field.db_default](https://subscription.packtpub.com/book/web-development/9781805125457/1). Puedes encontrar todas las funciones de base de datos disponibles en [https://docs.djangoproject.com/en/5.2/ref/models/database-functions/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Continuemos con la versión anterior del campo:

```python
class Post(models.Model):
    # ...
    publish = models.DateTimeField(default=timezone.now)
```

Edita el archivo `models.py` de la aplicación `blog` y agrega las siguientes líneas:

```python
from django.db import models
from django.utils import timezone


class Post(models.Model):
    title = models.CharField(max_length=250)
    slug = models.SlugField(max_length=250)
    body = models.TextField()
    publish = models.DateTimeField(default=timezone.now)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.title
```

Hemos añadido los siguientes campos al modelo `Post`:

- `created`: Este es un campo `DateTimeField`. Lo utilizaremos para almacenar la fecha y la hora en que se creó la publicación. Al usar `auto_now_add`, la fecha se guardará automáticamente al crear un objeto.
- `updated`: Este es un campo `DateTimeField`. Lo utilizaremos para almacenar la última fecha y hora en que se actualizó la publicación. Al usar `auto_now`, la fecha se actualizará automáticamente al guardar un objeto.

El uso de los campos de fecha y hora `auto_now_add` y `auto_now` en tus modelos de Django es muy beneficioso para realizar el seguimiento de los momentos de creación y última modificación de los objetos.

#### Definición de un orden de clasificación predeterminado

Las publicaciones de blogs se presentan habitualmente en orden cronológico inverso, mostrando primero las más recientes. Para nuestro modelo, definiremos una ordenación predeterminada. Esta ordenación surte efecto al recuperar objetos de la base de datos a menos que se indique un orden específico en la consulta.

Edita el archivo `models.py` de la aplicación `blog` como se muestra a continuación:

```python
from django.db import models
from django.utils import timezone


class Post(models.Model):
    title = models.CharField(max_length=250)
    slug = models.SlugField(max_length=250)
    body = models.TextField()
    publish = models.DateTimeField(default=timezone.now)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['-publish']

    def __str__(self):
        return self.title
```

Hemos añadido una clase `Meta` dentro del modelo. Esta clase define metadatos para el modelo. Utilizamos el atributo `ordering` para indicarle a Django que debe ordenar los resultados por el campo `publish`. Este ordenamiento se aplicará de forma predeterminada para las consultas a la base de datos cuando no se proporcione ningún orden específico en la consulta. Indicamos orden descendente usando un guion antes del nombre del campo, `-publish`. Las publicaciones se devolverán en orden cronológico inverso de forma predeterminada.

#### Adición de un índice de base de datos

Definamos un índice de base de datos para el campo `publish`. Esto mejorará el rendimiento para el filtrado de consultas o la ordenación de resultados por este campo. Esperamos que muchas consultas aprovechen este índice ya que estamos utilizando el campo `publish` para ordenar los resultados de forma predeterminada.

Edita el archivo `models.py` de la aplicación `blog` para que quede de la siguiente manera:

```python
from django.db import models
from django.utils import timezone


class Post(models.Model):
    title = models.CharField(max_length=250)
    slug = models.SlugField(max_length=250)
    body = models.TextField()
    publish = models.DateTimeField(default=timezone.now)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['-publish']
        indexes = [
            models.Index(fields=['-publish']),
        ]

    def __str__(self):
        return self.title
```

Hemos añadido la opción `indexes` a la clase `Meta` del modelo. Esta opción te permite definir índices de base de datos para tu modelo, que podrían comprender uno o varios campos, en orden ascendente o descendente, o expresiones funcionales y funciones de base de datos. Hemos añadido un índice para el campo `publish`. Utilizamos un guion antes del nombre del campo para definir el índice específicamente en orden descendente. La creación de este índice se incluirá en las migraciones de base de datos que generaremos más adelante para nuestros modelos de blog.

> [!NOTE]
> La ordenación de índices no es compatible con MySQL. Si utilizas MySQL para la base de datos, se creará un índice descendente como un índice normal.

Puedes encontrar más información sobre cómo definir índices para modelos en [https://docs.djangoproject.com/en/5.2/ref/models/indexes/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

#### Activación de la aplicación

Necesitamos activar la aplicación `blog` en el proyecto para que Django realice un seguimiento de la aplicación y pueda crear tablas de base de datos para sus modelos.

Edita el archivo `settings.py` y añade `blog.apps.BlogConfig` a la configuración `INSTALLED_APPS`. Debería verse de esta manera:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'blog.apps.BlogConfig',
]
```

La clase `BlogConfig` es la configuración de la aplicación. Ahora Django sabe que la aplicación está activa para este proyecto y podrá cargar los modelos de la aplicación.

#### Adición de un campo de estado

Una funcionalidad común para los blogs es guardar las publicaciones como borradores hasta que estén listas para su publicación. Añadiremos un campo de estado (`status`) a nuestro modelo que nos permitirá gestionar el estado de las publicaciones del blog. Utilizaremos los estados `Draft` (Borrador) y `Published` (Publicado) para las publicaciones.

Edita el archivo `models.py` de la aplicación `blog` para que quede de la siguiente manera:

```python
from django.db import models
from django.utils import timezone


class Post(models.Model):
    class Status(models.TextChoices):
        DRAFT = 'DF', 'Draft'
        PUBLISHED = 'PB', 'Published'

    title = models.CharField(max_length=250)
    slug = models.SlugField(max_length=250)
    body = models.TextField()
    publish = models.DateTimeField(default=timezone.now)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)
    status = models.CharField(
        max_length=2,
        choices=Status,
        default=Status.DRAFT
    )

    class Meta:
        ordering = ['-publish']
        indexes = [
            models.Index(fields=['-publish']),
        ]

    def __str__(self):
        return self.title
```

Hemos definido la clase de enumeración `Status` heredando de `models.TextChoices`. Las opciones disponibles para el estado de la publicación son `DRAFT` y `PUBLISHED`. Sus respectivos valores son `DF` y `PB`, y sus etiquetas o nombres legibles son `Draft` y `Published`.

Django proporciona tipos de enumeración que puedes heredar para definir opciones de forma sencilla. Estos se basan en el objeto `enum` de la biblioteca estándar de Python. Puedes leer más sobre `enum` en [https://docs.python.org/3/library/enum.html](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Los tipos de enumeración de Django presentan algunas modificaciones respecto a `enum`. Puedes conocer esas diferencias en [https://docs.djangoproject.com/en/5.2/ref/models/fields/#enumeration-types](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Podemos acceder a `Post.Status.choices` para obtener las opciones disponibles, `Post.Status.names` para obtener los nombres de las opciones, `Post.Status.labels` para obtener los nombres legibles por humanos y `Post.Status.values` para obtener los valores reales de las opciones.

También hemos añadido un nuevo campo `status` al modelo que es una instancia de `CharField`. Incluye un parámetro `choices` para limitar el valor del campo a las opciones en `Status`. También hemos establecido un valor predeterminado para el campo mediante el parámetro `default`. Usamos `DRAFT` como la opción predeterminada para este campo.

Es una buena práctica definir opciones dentro de la clase del modelo y utilizar los tipos de enumeración. Esto te permitirá hacer referencia fácilmente a las etiquetas, valores o nombres de las opciones desde cualquier lugar de tu código. Puedes importar el modelo `Post` y usar `Post.Status.DRAFT` como referencia para el estado *Draft* en cualquier parte de tu código.

Echemos un vistazo a cómo interactuar con las opciones de estado.

Ejecuta el siguiente comando en la consola para abrir la shell de Python:

```bash
python manage.py shell
```

> [!NOTE]
> Como novedad en Django 5.2, los modelos de tu `INSTALLED_APPS` se importarán automáticamente a la shell, por lo que la importación del modelo `Post` en el siguiente ejemplo sería innecesaria. Al pasar la opción de verbosidad con el comando shell usando un nivel 2 se mostrarían las importaciones, por ejemplo, `python manage.py shell --verbosity 2`. Para obtener más detalles, consulta el Apéndice.

A continuación, escribe las siguientes líneas:

```python
>>> from blog.models import Post
>>> Post.Status.choices
```

Obtendrás las opciones de la enumeración con pares de valor-etiqueta, de esta forma:

```python
[('DF', 'Draft'), ('PB', 'Published')]
```

Escribe la siguiente línea:

```python
>>> Post.Status.labels
```

Obtendrás los nombres legibles por humanos de los miembros de la enumeración, de la siguiente manera:

```python
['Draft', 'Published']
```

Escribe la siguiente línea:

```python
>>> Post.Status.values
```

Obtendrás los valores de los miembros de la enumeración, como se muestra a continuación. Estos son los valores que se pueden almacenar en la base de datos para el campo `status`:

```python
['DF', 'PB']
```

Escribe la siguiente línea:

```python
>>> Post.Status.names
```

Obtendrás los nombres de las opciones, de esta forma:

```python
['DRAFT', 'PUBLISHED']
```

Puedes acceder a un miembro de enumeración de búsqueda específico con `Post.Status.PUBLISHED` y también puedes acceder a sus propiedades `.name` y `.value`.

#### Adición de una relación de muchos a uno

Las publicaciones siempre las escribe un autor. Crearemos una relación entre usuarios y publicaciones que indicará qué usuario escribió qué publicaciones. Django viene con un framework de autenticación que gestiona cuentas de usuario. El framework de autenticación de Django viene en el paquete `django.contrib.auth` y contiene un modelo `User`. Para definir la relación entre usuarios y publicaciones, utilizaremos la configuración `AUTH_USER_MODEL`, que apunta a `auth.User` de forma predeterminada. Este ajuste te permite especificar un modelo de usuario diferente para tu proyecto.

Edita el archivo `models.py` de la aplicación `blog` para que quede de la siguiente manera:

```python
from django.conf import settings
from django.db import models
from django.utils import timezone


class Post(models.Model):
    class Status(models.TextChoices):
        DRAFT = 'DF', 'Draft'
        PUBLISHED = 'PB', 'Published'

    title = models.CharField(max_length=250)
    slug = models.SlugField(max_length=250)
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='blog_posts'
    )
    body = models.TextField()
    publish = models.DateTimeField(default=timezone.now)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)
    status = models.CharField(
        max_length=2,
        choices=Status,
        default=Status.DRAFT
    )

    class Meta:
        ordering = ['-publish']
        indexes = [
            models.Index(fields=['-publish']),
        ]

    def __str__(self):
        return self.title
```

Hemos importado la configuración del proyecto y hemos añadido un campo `author` al modelo `Post`. Este campo define una relación de muchos a uno (*many-to-one*) con el modelo de usuario predeterminado, lo que significa que cada publicación la escribe un usuario, y un usuario puede escribir cualquier número de publicaciones. Para este campo, Django creará una clave foránea (*foreign key*) en la base de datos utilizando la clave primaria del modelo relacionado.

El parámetro `on_delete` especifica el comportamiento a adoptar cuando se elimina el objeto referenciado. Esto no es específico de Django; es un estándar de SQL. Al usar `CASCADE`, especificas que cuando se elimine el usuario referenciado, la base de datos también eliminará todas las publicaciones de blog relacionadas. Puedes consultar todas las opciones posibles en [https://docs.djangoproject.com/en/5.2/ref/models/fields/#django.db.models.ForeignKey.on_delete](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Usamos `related_name` para especificar el nombre de la relación inversa, de `User` a `Post`. Esto nos permitirá acceder a los objetos relacionados fácilmente desde un objeto de usuario utilizando la notación `user.blog_posts`. Aprenderemos más sobre esto más adelante.

Django incluye diferentes tipos de campos que puedes usar para definir tus modelos. Puedes encontrar todos los tipos de campo en [https://docs.djangoproject.com/en/5.2/ref/models/fields/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

El modelo `Post` ya está completo y ahora podemos sincronizarlo con la base de datos.

#### Uso del campo CompositePrimaryKey

A menudo, una clave primaria única y autoincremental proporciona el rendimiento y la mantenibilidad necesarios para satisfacer las necesidades de un proyecto. Pero las bases de datos más grandes o aquellas con una estructura más compleja pueden beneficiarse de las claves primarias compuestas. Un caso de uso de ejemplo sería en una tabla intermedia (*joining table*) para una relación de muchos a muchos.

En Django 5.2, el campo `CompositePrimaryKey` está disponible, lo que permite a un desarrollador definir la clave primaria de una tabla como una combinación de valores de otros campos.

Los modelos con un campo `CompositePrimaryKey` no se pueden registrar actualmente en el administrador de Django. La compatibilidad con estos campos en el admin llegará en una versión futura y tiene el número de ticket de seguimiento de Django 35953 ([https://code.djangoproject.com/ticket/35953](https://subscription.packtpub.com/book/web-development/9781805125457/1)).

Se podría utilizar una clave compuesta para un nuevo modelo con el fin de rastrear las publicaciones favoritas de un usuario, donde los datos importantes provienen de dos claves foráneas (primero al usuario y luego a la publicación), y estas columnas ayudan a garantizar la unicidad en la clave compuesta:

```python
class FavouritePost(models.Model):
    pk = models.CompositePrimaryKey(
        "user", "post"
    )
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE
    )
    post = models.ForeignKey(
        'blog.Post',
        on_delete=models.CASCADE
    )
    created = models.DateTimeField(auto_now_add=True)
```

#### Creación y aplicación de migraciones

Ahora que tenemos un modelo de datos para las publicaciones del blog, necesitamos crear la tabla de base de datos correspondiente. Django viene con un sistema de migraciones que rastrea los cambios realizados en los modelos y les permite propagarse a la base de datos.

El comando `migrate` aplica migraciones para todas las aplicaciones enumeradas en `INSTALLED_APPS`. Sincroniza la base de datos con los modelos actuales y las migraciones existentes.

Primero, necesitaremos crear una migración inicial para nuestro modelo `Post`.

Ejecuta el siguiente comando en la consola desde el directorio raíz de tu proyecto:

```bash
python manage.py makemigrations blog
```

Deberías obtener una salida similar a la siguiente:

```text
Migrations for 'blog':
  blog/migrations/0001_initial.py
    - Create model Post
    - Create index blog_post_publish_bb7600_idx on field(s) -publish of model post
```

Django acaba de crear el archivo `0001_initial.py` dentro del directorio `migrations` de la aplicación `blog`. Esta migración contiene las sentencias SQL para crear la tabla de base de datos para el modelo `Post` y la definición del índice de base de datos para el campo `publish`.

Puedes echar un vistazo al contenido del archivo para ver cómo se define la migración. Una migración especifica dependencias sobre otras migraciones y operaciones a realizar en la base de datos para sincronizarla con los cambios del modelo.

Echemos un vistazo al código SQL que Django ejecutará en la base de datos para crear la tabla para tu modelo. El comando `sqlmigrate` toma los nombres de las migraciones y devuelve su SQL sin ejecutarlo.

Ejecuta el siguiente comando desde la consola para inspeccionar la salida SQL de tu primera migración:

```bash
python manage.py sqlmigrate blog 0001
```

La salida debería verse de la siguiente manera:

```sql
BEGIN;
--
-- Create model Post
--
CREATE TABLE "blog_post" (
    "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT,
    "title" varchar(250) NOT NULL,
    "slug" varchar(250) NOT NULL,
    "body" text NOT NULL,
    "publish" datetime NOT NULL,
    "created" datetime NOT NULL,
    "updated" datetime NOT NULL,
    "status" varchar(2) NOT NULL,
    "author_id" integer NOT NULL REFERENCES "auth_user" ("id") DEFERRABLE INITIALLY DEFERRED
);
--
-- Create blog_post_publish_bb7600_idx on field(s) -publish of model post
--
CREATE INDEX "blog_post_publish_bb7600_idx" ON "blog_post" ("publish" DESC);
CREATE INDEX "blog_post_slug_b95473f2" ON "blog_post" ("slug");
CREATE INDEX "blog_post_author_id_dd7a8485" ON "blog_post" ("author_id");
COMMIT;
```

La salida exacta depende de la base de datos que estés utilizando. La salida anterior se genera para SQLite. Como puedes ver en la salida, Django genera los nombres de tabla combinando el nombre de la aplicación y el nombre del modelo en minúsculas (`blog_post`), pero también puedes especificar un nombre de base de datos personalizado para tu modelo en la clase `Meta` del modelo usando el atributo `db_table`.

Django crea una columna `id` autoincremental que se utiliza como clave primaria para cada modelo, pero también puedes sobrescribir esto especificando `primary_key=True` en uno de los campos de tu modelo. La columna `id` predeterminada consiste en un entero que se incrementa automáticamente. Esta columna corresponde al campo `id` que se agrega automáticamente a tu modelo.

Se crean los siguientes tres índices de base de datos:

- Un índice en orden descendente sobre la columna `publish`. Este es el índice que definimos explícitamente con la opción `indexes` de la clase `Meta` del modelo.
- Un índice sobre la columna `slug` porque los campos `SlugField` implican un índice de forma predeterminada.
- Un índice sobre la columna `author_id` porque los campos `ForeignKey` implican un índice de forma predeterminada.

Comparemos el modelo `Post` con su correspondiente tabla de base de datos `blog_post`:

> *Figura 1.9: Correspondencia entre el modelo Post completo y la tabla de base de datos*

La Figura 1.9 muestra cómo los campos del modelo se corresponden con las columnas de la tabla de la base de datos.

Sincronicemos la base de datos con el nuevo modelo.

Ejecuta el siguiente comando en la consola para aplicar las migraciones existentes:

```bash
python manage.py migrate
```

Obtendrás una salida que termina con la siguiente línea:

```text
Applying blog.0001_initial... OK
```

Acabamos de aplicar migraciones para las aplicaciones enumeradas en `INSTALLED_APPS`, incluida la aplicación `blog`. Tras aplicar las migraciones, la base de datos refleja el estado actual de los modelos.

Si editas el archivo `models.py` para agregar, eliminar o cambiar los campos de los modelos existentes, o si agregas nuevos modelos, tendrás que crear una nueva migración usando el comando `makemigrations`. Cada migración le permite a Django realizar un seguimiento de los cambios en los modelos. Luego, tendrás que aplicar la migración usando el comando `migrate` para mantener la base de datos sincronizada con tus modelos.

---

### Creación de un sitio de administración para los modelos

Ahora que el modelo `Post` está sincronizado con la base de datos, podemos crear un sitio de administración sencillo para gestionar las publicaciones del blog.

Django viene con una interfaz de administración integrada que es muy útil para editar contenido. El sitio de Django se construye dinámicamente leyendo los metadatos del modelo y proporcionando una interfaz lista para producción para editar contenido. Puedes usarlo de inmediato, configurando cómo deseas que se muestren tus modelos en él.

La aplicación `django.contrib.admin` ya está incluida en la configuración `INSTALLED_APPS`, por lo que no es necesario que la agregues.

#### Creación de un superusuario

Primero, deberás crear un usuario para administrar el sitio de administración. Ejecuta el siguiente comando:

```bash
python manage.py createsuperuser
```

Verás la siguiente salida. Introduce tu nombre de usuario, correo electrónico y contraseña deseados, de la siguiente manera:

```text
Username (leave blank to use 'admin'): admin
Email address: admin@admin.com
Password: ********
Password (again): ********
```

Luego, verás el siguiente mensaje de éxito:

```text
Superuser created successfully.
```

Acabamos de crear un usuario administrador con los permisos más altos.

#### El sitio de administración de Django

Inicia el servidor de desarrollo con el siguiente comando:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/admin/` en tu navegador. Deberías ver la página de inicio de sesión de la administración, como se muestra en la Figura 1.10:

> *Figura 1.10: Pantalla de inicio de sesión del sitio de administración de Django*

Inicia sesión con las credenciales del usuario que creaste en el paso anterior. Verás la página de índice del sitio de administración, como se muestra en la Figura 1.11:

> *Figura 1.11: Página de índice del sitio de administración de Django*

Los modelos `Group` y `User` que puedes ver en la captura de pantalla anterior son parte del framework de autenticación de Django ubicado en `django.contrib.auth`. Si haces clic en **Users**, verás el usuario que creaste anteriormente.

#### Adición de modelos al sitio de administración

Añadamos los modelos de tu blog al sitio de administración. Edita el archivo `admin.py` de la aplicación `blog` y haz que se vea de la siguiente manera:

```python
from django.contrib import admin
from .models import Post

admin.site.register(Post)
```

Ahora, recarga el sitio de administración en tu navegador. Deberías ver tu modelo `Post` en el sitio, de la siguiente manera:

> *Figura 1.12: El modelo Post de la aplicación blog incluido en la página de índice del sitio de administración de Django*

Fue fácil, ¿verdad? Cuando registras un modelo en el sitio de administración de Django, obtienes una interfaz fácil de usar generada mediante la introspección de tus modelos que te permite listar, editar, crear y eliminar objetos de forma sencilla.

Haz clic en el enlace **Add** junto a **Posts** para agregar una nueva publicación. Observarás el formulario que Django ha generado dinámicamente para tu modelo, como se muestra en la Figura 1.13:

> *Figura 1.13: Formulario de edición del sitio de administración de Django para el modelo Post*

Django utiliza diferentes widgets de formulario para cada tipo de campo. Incluso los campos complejos, como `DateTimeField`, se muestran con una interfaz sencilla, como un selector de fecha de JavaScript.

Rellena el formulario y haz clic en el botón **SAVE**. Deberías ser redirigido a la página de lista de publicaciones con un mensaje de éxito y la publicación que acabas de crear, como se muestra en la Figura 1.14:

> *Figura 1.14: Vista de lista del sitio de administración de Django para el modelo Post con un mensaje de añadido correctamente*

#### Personalización de cómo se muestran los modelos

Ahora, veremos cómo personalizar el sitio de administración.

Edita el archivo `admin.py` de tu aplicación `blog` y cámbialo de la siguiente manera:

```python
from django.contrib import admin
from .models import Post


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ['title', 'slug', 'author', 'publish', 'status']
```

Le estamos indicando al sitio de administración de Django que el modelo se registra en el sitio utilizando una clase personalizada que hereda de `ModelAdmin`. En esta clase, podemos incluir información sobre cómo mostrar el modelo en el sitio de administración y cómo interactuar con él.

El atributo `list_display` te permite configurar los campos de tu modelo que deseas mostrar en la página de lista de objetos de administración. El decorador `@admin.register()` realiza la misma función que la función `admin.site.register()` que reemplazaste, registrando la clase `ModelAdmin` que decora.

Personalicemos el modelo de administración con algunas opciones más.

Edita el archivo `admin.py` de tu aplicación `blog` y cámbialo de la siguiente manera:

```python
from django.contrib import admin
from .models import Post


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ['title', 'slug', 'author', 'publish', 'status']
    list_filter = ['status', 'created', 'publish', 'author']
    search_fields = ['title', 'body']
    prepopulated_fields = {'slug': ('title',)}
    raw_id_fields = ['author']
    date_hierarchy = 'publish'
    ordering = ['status', 'publish']
```

Regresa a tu navegador y recarga la página de lista de publicaciones. Ahora, se verá así:

> *Figura 1.15: Vista de lista personalizada del sitio de administración de Django para el modelo Post*

Puedes ver que los campos mostrados en la página de lista de publicaciones son los que especificamos en el atributo `list_display`. La página de lista ahora incluye una barra lateral derecha que te permite filtrar los resultados por los campos incluidos en el atributo `list_filter`. Los filtros para campos `ForeignKey` como `author` solo se muestran en la barra lateral si existe más de un objeto en la base de datos.

Ha aparecido una barra de búsqueda en la página. Esto se debe a que hemos definido una lista de campos de búsqueda mediante el atributo `search_fields`. Justo debajo de la barra de búsqueda, hay enlaces de navegación para navegar a través de una jerarquía de fechas; esto se ha definido mediante el atributo `date_hierarchy`. También puedes ver que las publicaciones están ordenadas por las columnas `STATUS` y `PUBLISH` de forma predeterminada. Hemos especificado los criterios de ordenación predeterminados mediante el atributo `ordering`.

A continuación, haz clic en el enlace **ADD POST**. También notarás algunos cambios aquí. A medida que escribes el título de una nueva publicación, el campo `slug` se completa automáticamente. Le has indicado a Django que autocomplete el campo `slug` con la entrada del campo `title` mediante el atributo `prepopulated_fields`:

> *Figura 1.16: El slug del modelo ahora se autocompleta automáticamente a medida que escribes el título*

Además, el campo `author` ahora se muestra con un widget de búsqueda, que puede ser mucho mejor que un menú desplegable de selección cuando tienes miles de usuarios. Esto se logra con el atributo `raw_id_fields` y se ve así:

> *Figura 1.17: El widget para seleccionar objetos relacionados para el campo Author del modelo Post*

#### Adición de conteos de facetas a los filtros

Django 5.0 introdujo filtros por facetas en el sitio de administración, mostrando recuentos de facetas. Estos recuentos indican la cantidad de objetos correspondientes a cada filtro específico, lo que facilita la identificación de objetos coincidentes en la vista changelist del admin. A continuación, nos aseguraremos de que los filtros de facetas se muestren siempre para el modelo de administración `PostAdmin`.

Edita el archivo `admin.py` de tu aplicación `blog` y añade la siguiente línea:

```python
from django.contrib import admin
from .models import Post


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ['title', 'slug', 'author', 'publish', 'status']
    list_filter = ['status', 'created', 'publish', 'author']
    search_fields = ['title', 'body']
    prepopulated_fields = {'slug': ('title',)}
    raw_id_fields = ['author']
    date_hierarchy = 'publish'
    ordering = ['status', 'publish']
    show_facets = admin.ShowFacets.ALWAYS
```

Crea algunas publicaciones utilizando el sitio de administración y accede a `http://127.0.0.1:8000/admin/blog/post/`. Los filtros ahora deberían incluir los recuentos totales de facetas, como se muestra en la Figura 1.18:

> *Figura 1.18: Filtros del campo de estado que incluyen recuentos de facetas*

Con unas pocas líneas de código, hemos personalizado la forma en que se muestra el modelo en el sitio de administración. Hay muchas formas de personalizar y ampliar el sitio de administración de Django; aprenderás más sobre ellas más adelante en este libro.

Puedes encontrar más información sobre el sitio de administración de Django en [https://docs.djangoproject.com/en/5.2/ref/contrib/admin/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

---

### Trabajo con QuerySets y managers

Ahora que tenemos un sitio de administración completamente funcional para gestionar las publicaciones del blog, es un buen momento para aprender cómo leer y escribir contenido en la base de datos mediante programación.

El mapeador objeto-relacional (ORM) de Django es una potente API de abstracción de bases de datos que te permite crear, recuperar, actualizar y eliminar objetos fácilmente. Un ORM te permite generar consultas SQL utilizando el paradigma orientado a objetos de Python. Puedes pensar en él como una forma de interactuar con tu base de datos de una manera pythónica en lugar de escribir consultas SQL sin procesar.

El ORM asigna tus modelos a tablas de bases de datos y te proporciona una interfaz pythónica simple para interactuar con tu base de datos. El ORM genera consultas SQL y asigna los resultados a objetos de modelo. El ORM de Django es compatible con MySQL, PostgreSQL, SQLite, Oracle y MariaDB.

Recuerda que puedes definir la base de datos de tu proyecto en la configuración `DATABASES` del archivo `settings.py` de tu proyecto. Django puede trabajar con múltiples bases de datos a la vez y puedes programar enrutadores de bases de datos para crear esquemas de enrutamiento de datos personalizados.

Una vez que hayas creado tus modelos de datos, Django te proporciona una API gratuita para interactuar con ellos. Puedes encontrar la referencia de la API de modelos de la documentación oficial en [https://docs.djangoproject.com/en/5.2/ref/models/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

El ORM de Django se basa en QuerySets. Un `QuerySet` es una colección de consultas a la base de datos para recuperar objetos de su base de datos. Puedes aplicar filtros a los QuerySets para restringir los resultados de la consulta según los parámetros dados. El QuerySet equivale a una sentencia SQL `SELECT` y los filtros son cláusulas limitantes de SQL como `WHERE` o `LIMIT`.

A continuación, vas a aprender a crear y ejecutar QuerySets.

#### Creación de objetos

Ejecuta el siguiente comando en la consola para abrir la shell de Python:

```bash
python manage.py shell
```

A continuación, escribe las siguientes líneas:

```python
>>> from django.contrib.auth.models import User
>>> from blog.models import Post
>>> user = User.objects.get(username='admin')
>>> post = Post(title='Another post',
...             slug='another-post',
...             body='Post body.',
...             author=user)
>>> post.save()
```

Analicemos lo que hace este código.

Primero, estamos recuperando el objeto de usuario con el nombre de usuario `admin`:

```python
>>> user = User.objects.get(username='admin')
```

El método `get()` nos permite recuperar un único objeto de la base de datos. Este método ejecuta una sentencia SQL `SELECT` entre bastidores. Ten en cuenta que este método espera un resultado que coincida con la consulta. Si la base de datos no devuelve ningún resultado, este método generará una excepción `DoesNotExist`, y si la base de datos devuelve más de un resultado, generará una excepción `MultipleObjectsReturned`. Ambas excepciones son atributos de la clase del modelo en la que se realiza la consulta.

Luego, creamos una instancia de `Post` con un título, slug y cuerpo personalizados, y establecemos el usuario que recuperamos previamente como el autor de la publicación:

```python
>>> post = Post(title='Another post', slug='another-post', body='Post body.', author=user)
```

Este objeto está en la memoria y no se conserva en la base de datos; creamos un objeto de Python que se puede usar durante el tiempo de ejecución pero que no se guarda en la base de datos.

Finalmente, guardamos el objeto `Post` en la base de datos usando el método `save()`:

```python
>>> post.save()
```

Esta acción realiza una sentencia SQL `INSERT` entre bastidores.

Primero creamos un objeto en la memoria y luego lo guardamos en la base de datos. Sin embargo, puedes crear el objeto y guardarlo en la base de datos en una sola operación utilizando el método `create()`, de la siguiente manera:

```python
>>> Post.objects.create(title='One more post', slug='one-more-post', body='Post body.', author=user)
```

En ciertas situaciones, es posible que necesites obtener un objeto de la base de datos o crearlo si no existe. El método `get_or_create()` facilita esto al recuperar un objeto o crearlo si no lo encuentra. Este método devuelve una tupla con el objeto recuperado y un booleano que indica si se creó un nuevo objeto. El siguiente código intenta recuperar un objeto `User` con el nombre de usuario `user2`, y si no existe, lo creará:

```python
>>> user, created = User.objects.get_or_create(username='user2')
```

#### Actualización de objetos

Ahora, cambia el título del objeto `Post` anterior por uno diferente y guarda el objeto nuevamente:

```python
>>> post.title = 'New title'
>>> post.save()
```

Esta vez, el método `save()` realiza una sentencia SQL `UPDATE`.

> [!NOTE]
> Los cambios que realizas en un objeto de modelo no se guardan en la base de datos hasta que llamas al método `save()`.

#### Recuperación de objetos

Ya sabes cómo recuperar un único objeto de la base de datos utilizando el método `get()`. Accedimos a este método usando `Post.objects.get()`. Cada modelo de Django tiene al menos un manager, y el manager predeterminado se llama `objects`. Obtienes un objeto `QuerySet` utilizando el manager de tu modelo.

Para recuperar todos los objetos de una tabla, utilizamos el método `all()` en el manager `objects` predeterminado, de esta forma:

```python
>>> all_posts = Post.objects.all()
```

Así es como creamos un `QuerySet` que devuelve todos los objetos de la base de datos. Ten en cuenta que este `QuerySet` aún no se ha ejecutado. Los QuerySets de Django son perezosos (*lazy*), lo que significa que solo se evalúan cuando se les obliga a hacerlo. Este comportamiento hace que los QuerySets sean muy eficientes. Si no asignas el QuerySet a una variable, sino que lo escribes directamente en la shell de Python, la sentencia SQL del QuerySet se ejecuta porque lo estás forzando a generar una salida:

```python
>>> Post.objects.all()
<QuerySet [<Post: Who was Django Reinhardt?>, <Post: New title>]>
```

#### Filtrado de objetos

Para filtrar un `QuerySet`, puedes usar el método `filter()` del manager. Este método te permite especificar el contenido de una cláusula SQL `WHERE` mediante el uso de búsquedas de campo (*field lookups*).

Por ejemplo, puedes usar lo siguiente para filtrar objetos `Post` por su título:

```python
>>> Post.objects.filter(title='Who was Django Reinhardt?')
```

Este QuerySet devolverá todas las publicaciones con el título exacto *Who was Django Reinhardt?*. Revisemos la sentencia SQL generada con este QuerySet. Ejecuta el siguiente código en la shell:

```python
>>> posts = Post.objects.filter(title='Who was Django Reinhardt?')
>>> print(posts.query)
```

Al imprimir el atributo `query` del QuerySet, podemos obtener el SQL producido por él:

```sql
SELECT "blog_post"."id", "blog_post"."title", "blog_post"."slug", "blog_post"."author_id", "blog_post"."body", "blog_post"."publish", "blog_post"."created", "blog_post"."updated", "blog_post"."status" FROM "blog_post" WHERE "blog_post"."title" = Who was Django Reinhardt? ORDER BY "blog_post"."publish" DESC
```

La cláusula `WHERE` generada realiza una coincidencia exacta en la columna `title`. La cláusula `ORDER BY` especifica el orden predeterminado definido en el atributo `ordering` de las opciones `Meta` del modelo `Post`, ya que no hemos proporcionado ningún orden específico en el QuerySet. Aprenderás sobre la ordenación en breve. Ten en cuenta que el atributo `query` no forma parte de la API pública de QuerySet.

#### Uso de búsquedas de campo

El ejemplo anterior de QuerySet consiste en una búsqueda de filtro con una coincidencia exacta. La interfaz de QuerySet te proporciona múltiples tipos de búsqueda. Se utilizan dos guiones bajos para definir el tipo de búsqueda, con el formato `field__lookup`. Por ejemplo, la siguiente búsqueda produce una coincidencia exacta:

```python
>>> Post.objects.filter(id__exact=1)
```

Cuando no se proporciona ningún tipo de búsqueda específico, se asume que el tipo de búsqueda es exacto (`exact`). La siguiente búsqueda es equivalente a la anterior:

```python
>>> Post.objects.filter(id=1)
```

Echemos un vistazo a otros tipos comunes de búsqueda. Puedes generar una búsqueda insensible a mayúsculas y minúsculas con `iexact`:

```python
>>> Post.objects.filter(title__iexact='who was django reinhardt?')
```

También puedes filtrar objetos mediante una prueba de contención. La búsqueda `contains` se traduce en una búsqueda SQL utilizando el operador `LIKE`:

```python
>>> Post.objects.filter(title__contains='Django')
```

La cláusula SQL equivalente es `WHERE title LIKE '%Django%'`. También está disponible una versión insensible a mayúsculas y minúsculas, llamada `icontains`:

```python
>>> Post.objects.filter(title__icontains='django')
```

Puedes comprobar un iterable determinado (a menudo una lista, tupla u otro objeto QuerySet) con la búsqueda `in`. El siguiente ejemplo recupera publicaciones con un `id` que sea 1 o 3:

```python
>>> Post.objects.filter(id__in=[1, 3])
```

El siguiente ejemplo muestra la búsqueda de mayor que (`gt`, *greater than*):

```python
>>> Post.objects.filter(id__gt=3)
```

La cláusula SQL equivalente es `WHERE ID > 3`.

Este ejemplo muestra la búsqueda de mayor o igual que (`gte`, *greater than or equal to*):

```python
>>> Post.objects.filter(id__gte=3)
```

Este muestra la búsqueda de menor que (`lt`, *less than*):

```python
>>> Post.objects.filter(id__lt=3)
```

Este muestra la búsqueda de menor o igual que (`lte`, *less than or equal to*):

```python
>>> Post.objects.filter(id__lte=3)
```

Se puede realizar una búsqueda sensible/insensible a mayúsculas y minúsculas que comience por un texto con los tipos de búsqueda `startswith` e `istartswith`, respectivamente:

```python
>>> Post.objects.filter(title__istartswith='who')
```

Se puede realizar una búsqueda sensible/insensible a mayúsculas y minúsculas que termine con un texto con los tipos de búsqueda `endswith` e `iendswith`, respectivamente:

```python
>>> Post.objects.filter(title__iendswith='reinhardt?')
```

También existen diferentes tipos de búsqueda para fechas. Se puede realizar una búsqueda exacta de fecha de la siguiente manera:

```python
>>> from datetime import date
>>> Post.objects.filter(publish__date=date(2024, 1, 31))
```

Esto muestra cómo filtrar un campo `DateField` o `DateTimeField` por año:

```python
>>> Post.objects.filter(publish__year=2024)
```

También puedes filtrar por mes:

```python
>>> Post.objects.filter(publish__month=1)
```

Y puedes filtrar por día:

```python
>>> Post.objects.filter(publish__day=1)
```

Puedes encadenar búsquedas adicionales para fecha, año, mes y día. Por ejemplo, aquí hay una búsqueda para un valor mayor que una fecha determinada:

```python
>>> Post.objects.filter(publish__date__gt=date(2024, 1, 1))
```

Para buscar en campos de objetos relacionados, también se utiliza la notación de dos guiones bajos. Por ejemplo, para recuperar las publicaciones escritas por el usuario con el nombre de usuario `admin`, utiliza lo siguiente:

```python
>>> Post.objects.filter(author__username='admin')
```

También puedes encadenar búsquedas adicionales para los campos relacionados. Por ejemplo, para recuperar publicaciones escritas por cualquier usuario con un nombre de usuario que comience con `ad`, utiliza lo siguiente:

```python
>>> Post.objects.filter(author__username__startswith='ad')
```

También puedes filtrar por múltiples campos. Por ejemplo, el siguiente QuerySet recupera todas las publicaciones publicadas en 2024 por el autor con el nombre de usuario `admin`:

```python
>>> Post.objects.filter(publish__year=2024, author__username='admin')
```

#### Encadenamiento de filtros

El resultado de un QuerySet filtrado es otro objeto QuerySet. Esto te permite encadenar QuerySets entre sí. Puedes construir un QuerySet equivalente al anterior encadenando múltiples filtros:

```python
>>> Post.objects.filter(publish__year=2024) \
...             .filter(author__username='admin')
```

#### Exclusión de objetos

Puedes excluir ciertos resultados de tu QuerySet utilizando el método `exclude()` del manager. Por ejemplo, puedes recuperar todas las publicaciones publicadas en 2024 cuyos títulos no comiencen con `Why`:

```python
>>> Post.objects.filter(publish__year=2024) \
...             .exclude(title__startswith='Why')
```

#### Ordenación de objetos

El orden predeterminado se define en la opción `ordering` del `Meta` del modelo. Puedes sobrescribir el orden predeterminado utilizando el método `order_by()` del manager. Por ejemplo, puedes recuperar todos los objetos ordenados por su título de la siguiente manera:

```python
>>> Post.objects.order_by('title')
```

El orden ascendente está implícito. Puedes indicar orden descendente con un prefijo de signo negativo, de esta forma:

```python
>>> Post.objects.order_by('-title')
```

Puedes ordenar por múltiples campos. El siguiente ejemplo ordena los objetos primero por autor y luego por título:

```python
>>> Post.objects.order_by('author', 'title')
```

Para ordenar de forma aleatoria, utiliza la cadena `'?'`, de la siguiente manera:

```python
>>> Post.objects.order_by('?')
```

#### Limitación de QuerySets

Puedes limitar un QuerySet a un cierto número de resultados utilizando un subconjunto de la sintaxis de corte de listas (*array-slicing*) de Python. Por ejemplo, el siguiente QuerySet limita los resultados a 5 objetos:

```python
>>> Post.objects.all()[:5]
```

Esto se traduce en una cláusula SQL `LIMIT 5`. Ten en cuenta que no se admite la indexación negativa.

```python
>>> Post.objects.all()[3:6]
```

Lo anterior se traduce en una cláusula SQL `OFFSET 3 LIMIT 6`, para devolver del cuarto al sexto objeto.

Para recuperar un único objeto, puedes usar un índice en lugar de un corte (*slice*). Por ejemplo, usa lo siguiente para recuperar el primer objeto de las publicaciones en orden aleatorio:

```python
>>> Post.objects.order_by('?')[0]
```

#### Conteo de objetos

El método `count()` cuenta el número total de objetos que coinciden con el QuerySet y devuelve un entero. Este método se traduce en una sentencia SQL `SELECT COUNT(*)`. El siguiente ejemplo devuelve el número total de publicaciones con un `id` menor que 3:

```python
>>> Post.objects.filter(id__lt=3).count()
2
```

#### Comprobación de si un objeto existe

El método `exists()` te permite verificar si un QuerySet contiene algún resultado. Este método devuelve `True` si el QuerySet contiene algún elemento y `False` en caso contrario. Por ejemplo, puedes comprobar si hay publicaciones con un título que comience con `Why` utilizando el siguiente QuerySet:

```python
>>> Post.objects.filter(title__startswith='Why').exists()
False
```

#### Eliminación de objetos

Si deseas eliminar un objeto, puedes hacerlo desde una instancia de objeto utilizando el método `delete()`, de la siguiente manera:

```python
>>> post = Post.objects.get(id=1)
>>> post.delete()
```

Ten en cuenta que eliminar objetos también eliminará las relaciones dependientes para los objetos `ForeignKey` definidos con `on_delete` configurado en `CASCADE`.

#### Búsquedas complejas con objetos Q

Las búsquedas de campo mediante `filter()` se unen con un operador SQL `AND`. Por ejemplo, `filter(field1='foo', field2='bar')` recuperará objetos donde `field1` sea `foo` y `field2` sea `bar`. Si necesitas crear consultas más complejas, como consultas con declaraciones `OR`, puedes usar objetos `Q`.

Un objeto `Q` te permite encapsular una colección de búsquedas de campo. Puedes componer declaraciones combinando objetos `Q` con los operadores `&` (and), `|` (or) y `^` (xor).

Por ejemplo, el siguiente código recupera publicaciones con un título que comienza con la cadena `who` o `why` (sin distinción entre mayúsculas y minúsculas):

```python
>>> from django.db.models import Q
>>> starts_who = Q(title__istartswith='who')
>>> starts_why = Q(title__istartswith='why')
>>> Post.objects.filter(starts_who | starts_why)
```

En este caso, usamos el operador `|` para construir una declaración `OR`.

Puedes leer más sobre los objetos `Q` en [https://docs.djangoproject.com/en/5.2/topics/db/queries/#complex-lookups-with-q-objects](https://subscription.packtpub.com/book/web-development/9781805125457/1).

#### Cuándo se evalúan los QuerySets

La creación de un QuerySet no implica ninguna actividad en la base de datos hasta que se evalúa. Por lo general, los QuerySets devolverán otro QuerySet no evaluado. Puedes concatenar tantos filtros como desees a un QuerySet y no consultarás la base de datos hasta que se evalúe el QuerySet. Cuando se evalúa un QuerySet, se traduce en una consulta SQL a la base de datos.

Los QuerySets solo se evalúan en los siguientes casos:

- La primera vez que iteras sobre ellos
- Cuando los serializas (*pickle*) o almacenas en caché
- Cuando llamas a `repr()` o `len()` sobre ellos
- Cuando llamas explícitamente a `list()` sobre ellos
- Cuando los pruebas en una declaración, como `bool()`, `or`, `and` o `if`

#### Más sobre QuerySets

Utilizarás QuerySets en todos los proyectos de ejemplo que aparecen en este libro. Aprenderás a generar agregaciones sobre QuerySets en la sección *Recuperación de publicaciones por similitud* del Capítulo 3, *Extensión de tu aplicación de blog*.

Aprenderás a optimizar QuerySets en la sección *Optimización de QuerySets que involucran objetos relacionados* en el Capítulo 7, *Seguimiento de acciones de usuario*.

La referencia de la API de QuerySet se encuentra en [https://docs.djangoproject.com/en/5.2/ref/models/querysets/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Puedes leer más sobre cómo hacer consultas con el ORM de Django en [https://docs.djangoproject.com/en/5.2/topics/db/queries/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

#### Creación de managers de modelos

El manager predeterminado para cada modelo es el manager `objects`. Este manager recupera todos los objetos de la base de datos. Sin embargo, podemos definir managers personalizados para los modelos.

Creemos un manager personalizado para recuperar todas las publicaciones que tengan el estado `PUBLISHED`.

Hay dos formas de agregar o personalizar managers para tus modelos: puedes agregar métodos de manager adicionales a un manager existente o crear un nuevo manager modificando el QuerySet inicial que devuelve el manager. El primer método te proporciona una notación de QuerySet como `Post.objects.my_manager()`, y el segundo te proporciona una notación de QuerySet como `Post.my_manager.all()`.

Elegiremos el segundo método para implementar un manager que nos permita recuperar publicaciones usando la notación `Post.published.all()`.

Edita el archivo `models.py` de tu aplicación `blog` para añadir el manager personalizado de la siguiente manera:

```python
class PublishedManager(models.Manager):
    def get_queryset(self):
        return (
            super().get_queryset().filter(status=Post.Status.PUBLISHED)
        )


class Post(models.Model):
    # model fields
    # ...
    objects = models.Manager()  # The default manager.
    published = PublishedManager()  # Our custom manager.

    class Meta:
        ordering = ['-publish']
        indexes = [
            models.Index(fields=['-publish']),
        ]

    def __str__(self):
        return self.title
```

El primer manager declarado en un modelo se convierte en el manager predeterminado. Puedes usar el atributo `default_manager_name` de `Meta` para especificar un manager predeterminado diferente. Si no se define ningún manager en el modelo, Django crea automáticamente el manager predeterminado `objects` para él. Si declaras algún manager para tu modelo pero deseas conservar también el manager `objects`, debes agregarlo explícitamente a tu modelo. En el código anterior, hemos agregado el manager `objects` predeterminado y el manager personalizado `published` al modelo `Post`.

El método `get_queryset()` de un manager devuelve el QuerySet que se ejecutará. Hemos sobrescrito este método para crear un QuerySet personalizado que filtre las publicaciones por su estado y devuelva un QuerySet sucesivo que solo incluya las publicaciones con el estado `PUBLISHED`.

Ahora hemos definido un manager personalizado para el modelo `Post`. ¡Vamos a probarlo!

Inicia el servidor de desarrollo nuevamente con el siguiente comando en la consola:

```bash
python manage.py shell
```

Ahora puedes importar el modelo `Post` y recuperar todas las publicaciones publicadas cuyo título comience con `Who`, ejecutando el siguiente QuerySet:

```python
>>> from blog.models import Post
>>> Post.published.filter(title__startswith='Who')
```

Para obtener resultados para este QuerySet, asegúrate de configurar el campo `status` en `PUBLISHED` en el objeto `Post` cuyo título comience con la cadena *Who*.

---

### Creación de vistas de lista y de detalle

Ahora que comprendes cómo usar el ORM, estás listo para crear las vistas de la aplicación de blog. Una vista de Django es simplemente una función de Python que recibe una petición web y devuelve una respuesta web. Toda la lógica para devolver la respuesta deseada va dentro de la vista.

Primero, crearás las vistas de tu aplicación, luego definirás un patrón de URL para cada vista y, finalmente, crearás plantillas HTML para renderizar los datos generados por las vistas. Cada vista renderizará una plantilla, pasándole variables, y devolverá una respuesta HTTP con la salida renderizada.

#### Creación de vistas de lista y de detalle

Comencemos creando una vista para mostrar la lista de publicaciones.

Edita el archivo `views.py` de la aplicación `blog` y haz que se vea de la siguiente manera:

```python
from django.shortcuts import render
from .models import Post


def post_list(request):
    posts = Post.published.all()
    return render(
        request,
        'blog/post/list.html',
        {'posts': posts}
    )
```

Esta es nuestra primera vista de Django. La vista `post_list` toma el objeto `request` como único parámetro. Este parámetro es requerido por todas las vistas.

En esta vista, recuperamos todas las publicaciones con el estado `PUBLISHED` usando el manager `published` que creamos previamente.

Finalmente, usamos el atajo `render()` proporcionado por Django para renderizar la lista de publicaciones con la plantilla dada. Esta función toma el objeto `request`, la ruta de la plantilla y las variables de contexto para renderizar la plantilla dada. Devuelve un objeto `HttpResponse` con el texto renderizado (normalmente código HTML).

El atajo `render()` tiene en cuenta el contexto de la petición, por lo que cualquier variable establecida por los procesadores de contexto de plantilla es accesible por la plantilla dada. Los procesadores de contexto de plantilla son simplemente elementos invocables (*callables*) que establecen variables en el contexto. Aprenderás a usar procesadores de contexto en el Capítulo 4, *Creación de un sitio web social*.

Creemos una segunda vista para mostrar una única publicación. Agrega la siguiente función al archivo `views.py`:

```python
from django.http import Http404


def post_detail(request, id):
    try:
        post = Post.published.get(id=id)
    except Post.DoesNotExist:
        raise Http404("No Post found.")
    return render(
        request,
        'blog/post/detail.html',
        {'post': post}
    )
```

Esta es la vista `post_detail`. Esta vista toma el argumento `id` de una publicación. En la vista, intentamos recuperar el objeto `Post` con el `id` dado llamando al método `get()` en el manager `published`. Lanzamos una excepción `Http404` para devolver un error HTTP 404 si se genera la excepción `DoesNotExist` del modelo porque no se encuentra ningún resultado.

Finalmente, usamos el atajo `render()` para renderizar la publicación recuperada usando una plantilla.

#### Uso del atajo get_object_or_404

Django proporciona un atajo para llamar a `get()` en un manager de modelo determinado y genera una excepción `Http404` en lugar de una excepción `DoesNotExist` cuando no se encuentra ningún objeto.

Edita el archivo `views.py` para importar el atajo `get_object_or_404` y cambia la vista `post_detail` de la siguiente manera:

```python
from django.shortcuts import get_object_or_404, render

# ...


def post_detail(request, id):
    post = get_object_or_404(
        Post,
        id=id,
        status=Post.Status.PUBLISHED
    )
    return render(
        request,
        'blog/post/detail.html',
        {'post': post}
    )
```

En la vista de detalle, ahora usamos el atajo `get_object_or_404()` para recuperar la publicación deseada. Esta función recupera el objeto que coincide con los parámetros dados o una excepción HTTP 404 (*not found*) si no se encuentra ningún objeto.

#### Adición de patrones de URL para tus vistas

Los patrones de URL te permiten asignar URLs a vistas. Un patrón de URL se compone de un patrón de cadena, una vista y, opcionalmente, un nombre que te permite nombrar la URL en todo el proyecto. Django recorre cada patrón de URL y se detiene en el primero que coincide con la URL solicitada. Luego, Django importa la vista del patrón de URL coincidente y la ejecuta, pasando una instancia de la clase `HttpRequest` y los argumentos posicionales o de palabras clave.

Crea un archivo `urls.py` en el directorio de la aplicación `blog` y agrégale las siguientes líneas:

```python
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    # post views
    path('', views.post_list, name='post_list'),
    path('<int:id>/', views.post_detail, name='post_detail'),
]
```

En el código anterior, defines un espacio de nombres de aplicación con la variable `app_name`. Esto te permite organizar las URLs por aplicación y usar el nombre al hacer referencia a ellas. Defines dos patrones diferentes usando la función `path()`. El primer patrón de URL no toma ningún argumento y se asigna a la vista `post_list`. El segundo patrón se asigna a la vista `post_detail` y toma solo un argumento `id`, que coincide con un número entero, establecido por el convertidor de ruta `int`.

Utilizas corchetes angulares para capturar los valores de la URL. Cualquier valor especificado en el patrón de URL como `<parameter>` se captura como una cadena. Utilizas convertidores de ruta, como `<int:year>`, para hacer coincidir y devolver específicamente un número entero. Por ejemplo, `<slug:post>` coincidiría específicamente con un slug (una cadena que solo puede contener letras, números, guiones bajos o guiones). Puedes ver todos los convertidores de ruta proporcionados por Django en [https://docs.djangoproject.com/en/5.2/topics/http/urls/#path-converters](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Si el uso de `path()` y los convertidores no es suficiente para ti, puedes usar `re_path()` en su lugar para definir patrones de URL complejos con expresiones regulares de Python. Puedes obtener más información sobre la definición de patrones de URL con expresiones regulares en [https://docs.djangoproject.com/en/5.2/ref/urls/#django.urls.re_path](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Crear un archivo `urls.py` para cada aplicación es la mejor manera de hacer que tus aplicaciones sean reutilizables por otros proyectos.

A continuación, debes incluir los patrones de URL de la aplicación `blog` en los patrones de URL principales del proyecto.

Edita el archivo `urls.py` ubicado en el directorio `mysite` de tu proyecto y haz que se vea de la siguiente manera:

```python
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('admin/', admin.site.urls),
    path('blog/', include('blog.urls', namespace='blog')),
]
```

El nuevo patrón de URL definido con `include` hace referencia a los patrones de URL definidos en la aplicación `blog` para que se incluyan bajo la ruta `blog/`. Incluyes estos patrones bajo el espacio de nombres `blog`. Los espacios de nombres deben ser únicos en todo tu proyecto. Más adelante, harás referencia a las URLs de tu blog fácilmente usando el espacio de nombres seguido de dos puntos y el nombre de la URL, por ejemplo, `blog:post_list` y `blog:post_detail`. Puedes aprender más sobre los espacios de nombres de URL en [https://docs.djangoproject.com/en/5.2/topics/http/urls/#url-namespaces](https://subscription.packtpub.com/book/web-development/9781805125457/1).

---

### Creación de plantillas para tus vistas

Has creado vistas y patrones de URL para la aplicación de blog. Los patrones de URL asignan URLs a vistas y las vistas deciden qué datos se devuelven al usuario. Las plantillas definen cómo se muestran los datos; generalmente están escritas en HTML en combinación con el lenguaje de plantillas de Django. Puedes encontrar más información sobre el lenguaje de plantillas de Django en [https://docs.djangoproject.com/en/5.2/ref/templates/language/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Agreguemos plantillas a tu aplicación para mostrar publicaciones de una manera fácil de usar.

Crea los siguientes directorios y archivos dentro del directorio de tu aplicación `blog`:

```text
templates/
    blog/
        base.html
        post/
            list.html
            detail.html
```

La estructura anterior será la estructura de archivos para tus plantillas. El archivo `base.html` incluirá la estructura HTML principal del sitio web y dividirá el contenido en el área de contenido principal y una barra lateral. Los archivos `list.html` y `detail.html` heredarán del archivo `base.html` para renderizar las vistas de lista y detalle de publicaciones del blog, respectivamente.

Django tiene un potente lenguaje de plantillas que te permite especificar cómo se muestran los datos. Se basa en etiquetas de plantilla, variables de plantilla y filtros de plantilla:

- **Etiquetas de plantilla (*Template tags*):** Controlan el renderizado de la plantilla y se ven así: `{% tag %}`.
- **Variables de plantilla (*Template variables*):** Se reemplazan con valores cuando se renderiza la plantilla y se ven así: `{{ variable }}`.
- **Filtros de plantilla (*Template filters*):** Te permiten modificar variables para su visualización y se ven así: `{{ variable|filter }}`.

Puedes ver todas las etiquetas y filtros de plantilla integrados en [https://docs.djangoproject.com/en/5.2/ref/templates/builtins/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

#### Creación de una plantilla base

Edita el archivo `base.html` y agrega el siguiente código:

```html
{% load static %}
<!DOCTYPE html>
<html>
<head>
    <title>{% block title %}{% endblock %}</title>
    <link href="{% static "css/blog.css" %}" rel="stylesheet">
</head>
<body>
    <div id="content">
        {% block content %}
        {% endblock %}
    </div>
    <div id="sidebar">
        <h2>My blog</h2>
        <p>This is my blog.</p>
    </div>
</body>
</html>
```

`{% load static %}` le indica a Django que cargue las etiquetas de plantilla estáticas proporcionadas por la aplicación `django.contrib.staticfiles`, que se encuentra en la configuración `INSTALLED_APPS`. Después de cargarlas, puedes usar la etiqueta de plantilla `{% static %}` en toda esta plantilla. Con esta etiqueta de plantilla, puedes incluir los archivos estáticos, como el archivo `blog.css`, que encontrarás en el código de este ejemplo bajo el directorio `static/` de la aplicación `blog`. Copia el directorio `static/` del código que acompaña a este capítulo en la misma ubicación de tu proyecto para aplicar los estilos CSS a las plantillas. Puedes encontrar el contenido del directorio en [https://github.com/PacktPublishing/Django-5-by-example/tree/master/Chapter01/mysite/blog/static](https://subscription.packtpub.com/book/web-development/9781805125457/1).

Puedes ver que hay dos etiquetas `{% block %}`. Estas le indican a Django que deseas definir un bloque en esa área. Las plantillas que heredan de esta plantilla pueden rellenar los bloques con contenido. Has definido un bloque llamado `title` y un bloque llamado `content`.

#### Creación de la plantilla de lista de publicaciones

Editemos el archivo `post/list.html` y hagamos que se vea de la siguiente manera:

```html
{% extends "blog/base.html" %}

{% block title %}My Blog{% endblock %}

{% block content %}
    <h1>My Blog</h1>
    {% for post in posts %}
        <h2>
            <a href="{% url 'blog:post_detail' post.id %}">
                {{ post.title }}
            </a>
        </h2>
        <p class="date">
            Published {{ post.publish }} by {{ post.author }}
        </p>
        {{ post.body|truncatewords:30|linebreaks }}
    {% endfor %}
{% endblock %}
```

Con la etiqueta de plantilla `{% extends %}`, le indicas a Django que herede de la plantilla `blog/base.html`. Luego, completas los bloques `title` y `content` de la plantilla base con contenido. Iteras a través de las publicaciones y muestras su título, fecha, autor y cuerpo, incluido un enlace en el título a la URL de detalle de la publicación. Construimos la URL usando la etiqueta de plantilla `{% url %}` proporcionada por Django.

Esta etiqueta de plantilla te permite construir URLs dinámicamente por su nombre. Usamos `blog:post_detail` para hacer referencia a la URL `post_detail` en el espacio de nombres `blog`. Pasamos el parámetro requerido `post.id` para construir la URL de cada publicación.

> [!TIP]
> Utiliza siempre la etiqueta de plantilla `{% url %}` para crear URLs en tus plantillas en lugar de escribir URLs codificadas de forma fija (*hardcoded*). Esto hará que tus URLs sean mucho más fáciles de mantener.

En el cuerpo de la publicación, aplicamos dos filtros de plantilla: `truncatewords` trunca el valor al número de palabras especificado y `linebreaks` convierte la salida en saltos de línea HTML. Puedes concatenar tantos filtros de plantilla como desees; cada uno se aplicará a la salida generada por el anterior.

#### Acceso a nuestra aplicación

Cambia el estado de la publicación inicial a `Published`, como se muestra en la Figura 1.19, y crea algunas publicaciones nuevas, también con el estado `Published`.

> *Figura 1.19: El campo de estado para una publicación publicada*

Abre la consola y ejecuta el siguiente comando para iniciar el servidor de desarrollo:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/blog/` en tu navegador; verás todo en funcionamiento. Deberías ver algo como esto:

> *Figura 1.20: La página para la vista de lista de publicaciones*

#### Creación de la plantilla de detalle de publicación

A continuación, edita el archivo `post/detail.html`:

```html
{% extends "blog/base.html" %}

{% block title %}{{ post.title }}{% endblock %}

{% block content %}
    <h1>{{ post.title }}</h1>
    <p class="date">
        Published {{ post.publish }} by {{ post.author }}
    </p>
    {{ post.body|linebreaks }}
{% endblock %}
```

A continuación, puedes regresar a tu navegador y hacer clic en uno de los títulos de las publicaciones para echar un vistazo a la vista de detalle de la publicación. Deberías ver algo como esto:

> *Figura 1.21: La página para la vista de detalle de la publicación*

Echa un vistazo a la URL: debe incluir el ID de publicación generado automáticamente, como `/blog/1/`.

---

### El ciclo de petición/respuesta

Repasemos el ciclo de petición/respuesta de Django con la aplicación que construimos. El siguiente esquema muestra un ejemplo simplificado de cómo Django procesa las peticiones HTTP y genera respuestas HTTP:

> *Figura 1.22: El ciclo de petición/respuesta de Django*

Repasemos el proceso de petición/respuesta de Django:

1. Un navegador web solicita una página mediante su URL, por ejemplo, `https://domain.com/blog/33/`. El servidor web recibe la petición HTTP y se la transfiere a Django.
2. Django recorre cada patrón de URL definido en la configuración de patrones de URL. El framework comprueba cada patrón contra la ruta de URL dada, en orden de aparición, y se detiene en el primero que coincide con la URL solicitada. En este caso, el patrón `/blog/<id>/` coincide con la ruta `/blog/33/`.
3. Django importa la vista del patrón de URL coincidente y la ejecuta, pasando una instancia de la clase `HttpRequest` y los argumentos de palabras clave o posicionales. La vista utiliza los modelos para recuperar información de la base de datos. Mediante el ORM de Django, los QuerySets se traducen a SQL y se ejecutan en la base de datos.
4. La vista utiliza la función `render()` para renderizar una plantilla HTML pasando el objeto `Post` como una variable de contexto.
5. El contenido renderizado es devuelto como un objeto `HttpResponse` por la vista con el tipo de contenido `text/html` de forma predeterminada.

Siempre puedes utilizar este esquema como referencia básica de cómo procesa Django las peticiones. Este esquema no incluye el middleware de Django por simplicidad. Utilizarás middleware en diferentes ejemplos de este libro y aprenderás a crear middleware personalizado en el Capítulo 17, *Puesta en producción*.

---

### Comandos de gestión utilizados en este capítulo

En este capítulo, hemos introducido una variedad de comandos de gestión de Django. Necesitas familiarizarte con ellos, ya que se utilizarán a menudo a lo largo del libro. Revisemos los comandos que hemos cubierto en este capítulo:

- Para crear la estructura de archivos para un nuevo proyecto de Django llamado `mysite`:
  ```bash
  django-admin startproject mysite
  ```

- Para crear la estructura de archivos para una nueva aplicación de Django llamada `blog`:
  ```bash
  python manage.py startapp blog
  ```

- Para aplicar todas las migraciones de bases de datos:
  ```bash
  python manage.py migrate
  ```

- Para crear migraciones para los modelos de la aplicación `blog`:
  ```bash
  python manage.py makemigrations blog
  ```

- Para ver las sentencias SQL que se ejecutarán con la primera migración de la aplicación `blog`:
  ```bash
  python manage.py sqlmigrate blog 0001
  ```

- Para ejecutar el servidor de desarrollo de Django:
  ```bash
  python manage.py runserver
  ```

- Para ejecutar el servidor de desarrollo especificando host/puerto y archivo de configuración:
  ```bash
  python manage.py runserver 127.0.0.1:8001 --settings=mysite.settings
  ```

- Para ejecutar la shell de Django:
  ```bash
  python manage.py shell
  ```

- Para crear un superusuario utilizando el framework de autenticación de Django:
  ```bash
  python manage.py createsuperuser
  ```

Para obtener la lista completa de comandos de gestión disponibles, consulta [https://docs.djangoproject.com/en/5.2/ref/django-admin/](https://subscription.packtpub.com/book/web-development/9781805125457/1).

---

### Resumen

En este capítulo, aprendiste los conceptos básicos del framework web Django creando una sencilla aplicación de blog. Diseñaste los modelos de datos y aplicaste migraciones a la base de datos. También creaste las vistas, plantillas y URLs para tu blog.

En el próximo capítulo, mejorarás tu blog creando URLs canónicas para tus publicaciones y construyendo URLs optimizadas para SEO. También aprenderás cómo implementar la paginación de objetos y cómo crear vistas basadas en clases. Además, crearás formularios para permitir que tus usuarios recomienden publicaciones por correo electrónico y comenten en las publicaciones.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter01](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Descargar Python:** [https://www.python.org/downloads/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Lanzador de Python para Windows:** [https://docs.python.org/3/using/windows.html#launcher](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Biblioteca venv de Python para entornos virtuales:** [https://docs.python.org/3/library/venv.html](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Instrucciones de instalación de pip en Python:** [https://pip.pypa.io/en/stable/installation/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Opciones de instalación de Django:** [https://docs.djangoproject.com/en/5.2/topics/install/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Notas de la versión de Django 5.2:** [https://docs.djangoproject.com/en/5.2/releases/5.2/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **La herramienta django-upgrade:** [https://github.com/adamchainz/django-upgrade](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **La herramienta pyupgrade:** [https://github.com/asottile/pyupgrade](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Filosofías de diseño de Django:** [https://docs.djangoproject.com/en/5.2/misc/design-philosophies/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Referencia de campos de modelo de Django:** [https://docs.djangoproject.com/en/5.2/ref/models/fields/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Referencia de índices de modelos:** [https://docs.djangoproject.com/en/5.2/ref/models/indexes/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Soporte de Python para enumeraciones:** [https://docs.python.org/3/library/enum.html](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Tipos de enumeración de modelos de Django:** [https://docs.djangoproject.com/en/5.2/ref/models/fields/#enumeration-types](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Referencia de configuración de Django:** [https://docs.djangoproject.com/en/5.2/ref/settings/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Valores predeterminados de base de datos para campos de modelo:** [https://docs.djangoproject.com/en/5.2/ref/models/fields/#django.db.models.Field.db_default](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Funciones de base de datos:** [https://docs.djangoproject.com/en/5.2/ref/models/database-functions/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Sitio de administración de Django:** [https://docs.djangoproject.com/en/5.2/ref/contrib/admin/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Referencia de la API de modelos:** [https://docs.djangoproject.com/en/5.2/ref/models/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Realización de consultas con el ORM de Django:** [https://docs.djangoproject.com/en/5.2/topics/db/queries/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Referencia de la API de QuerySet:** [https://docs.djangoproject.com/en/5.2/ref/models/querysets/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Búsquedas complejas con objetos Q:** [https://docs.djangoproject.com/en/5.2/topics/db/queries/#complex-lookups-with-q-objects](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Despachador de URLs de Django:** [https://docs.djangoproject.com/en/5.2/topics/http/urls/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Lenguaje de plantillas de Django:** [https://docs.djangoproject.com/en/5.2/ref/templates/language/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Etiquetas y filtros de plantilla integrados:** [https://docs.djangoproject.com/en/5.2/ref/templates/builtins/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Comandos de gestión de Django:** [https://docs.djangoproject.com/en/5.2/ref/django-admin/](https://subscription.packtpub.com/book/web-development/9781805125457/1)
- **Archivos estáticos para el código de este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/master/Chapter01/mysite/blog/static](https://subscription.packtpub.com/book/web-development/9781805125457/1)
