# Parte 4: Creación de una plataforma de E-Learning

## Capítulo 16: Creación de un servidor de chat

### Introducción

En el capítulo anterior, creaste una API RESTful para tu proyecto que proporciona una interfaz programable para tu aplicación.

En este capítulo, desarrollarás un servidor de chat para estudiantes utilizando Django Channels, lo que permitirá a los estudiantes interactuar en mensajería en tiempo real dentro de las salas de chat de los cursos. Aprenderás a crear aplicaciones en tiempo real mediante programación asíncrona con Django Channels. Al servir tu proyecto Django a través de Asynchronous Server Gateway Interface (ASGI) e implementar la comunicación asíncrona, mejorarás la capacidad de respuesta y escalabilidad de tu servidor. Además, persistirás los mensajes de chat en la base de datos, construyendo un historial de chat completo y enriqueciendo la experiencia del usuario y la funcionalidad de la aplicación de chat.

En este capítulo, aprenderás a:

- Añadir Channels a tu proyecto
- Construir un consumidor (*consumer*) WebSocket y el enrutamiento (*routing*) apropiado
- Implementar un cliente WebSocket
- Habilitar una capa de canales (*channel layer*) con Redis
- Hacer que tu consumidor sea completamente asíncrono
- Persistir mensajes de chat en la base de datos

---

### Visión general funcional

La Figura 16.1 muestra una representación de las vistas, plantillas y funcionalidades que se construirán en este capítulo:

> *Figura 16.1: Diagrama de funcionalidades construidas en este capítulo*

En este capítulo, implementarás la vista `course_chat_room` en la aplicación `chat`. Esta vista servirá la plantilla que muestra la sala de chat para un curso determinado. Los últimos mensajes de chat se mostrarán cuando un usuario se una a una sala de chat. Utilizarás JavaScript para establecer una conexión WebSocket en el navegador y crearás el consumidor WebSocket `ChatConsumer` para gestionar las conexiones WebSocket y para intercambiar mensajes. Utilizarás Redis para implementar la capa de canales que permite transmitir mensajes a todos los usuarios en la sala de chat.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter16](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter16).

Todos los módulos de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente que acompaña a este capítulo. Puedes seguir las instrucciones para instalar cada módulo de Python a continuación o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Creación de una aplicación de chat

Vas a implementar un servidor de chat para proporcionar a los estudiantes una sala de chat para cada curso. Los estudiantes inscritos en un curso podrán acceder a la sala de chat del curso e intercambiar mensajes en tiempo real. Utilizarás Channels para construir esta funcionalidad. Channels es una aplicación de Django que extiende Django para manejar protocolos que requieren conexiones de larga duración, como WebSockets, chatbots o MQTT (un transporte de mensajes de publicación/suscripción ligero comúnmente utilizado en proyectos de Internet de las cosas (IoT)).

Con Channels, puedes implementar fácilmente funcionalidades asíncronas o en tiempo real en tu proyecto además de tus vistas síncronas HTTP estándar. Comenzarás agregando una nueva aplicación a tu proyecto. La nueva aplicación contendrá la lógica para el servidor de chat.

Puedes consultar la documentación de Django Channels en [https://channels.readthedocs.io/](https://channels.readthedocs.io/).

Comencemos a implementar el servidor de chat. Ejecuta el siguiente comando desde el directorio `educa` del proyecto para crear la estructura de archivos de la nueva aplicación:

```bash
django-admin startapp chat
```

Edita el archivo `settings.py` del proyecto `educa` y activa la aplicación `chat` en tu proyecto editando la configuración `INSTALLED_APPS`, de la siguiente manera:

```python
INSTALLED_APPS = [
    # ...
    'chat.apps.ChatConfig',
]
```

La nueva aplicación de chat ya está activa en tu proyecto. A continuación, vas a crear una vista para las salas de chat de los cursos.

#### Implementación de la vista de la sala de chat

Proporcionarás a los estudiantes una sala de chat diferente para cada curso. Necesitas crear una vista para que los estudiantes se unan a la sala de chat de un curso determinado. Solo los estudiantes que estén inscritos en un curso podrán acceder a su sala de chat.

Edita el archivo `views.py` de la nueva aplicación `chat` y añade el siguiente código:

```python
from django.contrib.auth.decorators import login_required
from django.http import HttpResponseForbidden
from django.shortcuts import render
from courses.models import Course


@login_required
def course_chat_room(request, course_id):
    try:
        # retrieve course with given id joined by the current user
        course = request.user.courses_joined.get(id=course_id)
    except Course.DoesNotExist:
        # user is not a student of the course or course does not exist
        return HttpResponseForbidden()
    return render(request, 'chat/room.html', {'course': course})
```

Esta es la vista `course_chat_room`. En esta vista, utilizas el decorador `@login_required` para evitar que cualquier usuario no autenticado acceda a la vista. La vista funciona de la siguiente manera:

1. La vista recibe un parámetro requerido `course_id` que se utiliza para recuperar el curso con el `id` dado.
2. Los cursos en los que el usuario está inscrito se recuperan a través de la relación `courses_joined` y el curso con el `id` dado se obtiene de ese subconjunto de cursos. Si el curso con el `id` dado no existe o el usuario no está inscrito en él, se devuelve una respuesta `HttpResponseForbidden`, que se traduce en una respuesta HTTP con estado 403.
3. Si el curso con el `id` dado existe y el usuario está inscrito en él, se renderiza la plantilla `chat/room.html`, pasando el objeto `course` al contexto de la plantilla.

Necesitas añadir un patrón de URL para esta vista. Crea un nuevo archivo dentro del directorio de la aplicación `chat` y nómbralo `urls.py`. Añade el siguiente código:

```python
from django.urls import path
from . import views

app_name = 'chat'

urlpatterns = [
    path(
        'room/<int:course_id>/',
        views.course_chat_room,
        name='course_chat_room'
    ),
]
```

Este es el archivo de patrones de URL inicial para la aplicación `chat`. Defines el patrón de URL `course_chat_room`, incluyendo el parámetro `course_id` con el prefijo `int:`, ya que aquí solo esperas un valor entero.

Incluye los nuevos patrones de URL de la aplicación `chat` en los patrones de URL principales del proyecto. Edita el archivo `urls.py` principal del proyecto `educa` y añade la siguiente línea:

```python
urlpatterns = [
    # ...
    path('chat/', include('chat.urls', namespace='chat')),
]
```

Los patrones de URL para la aplicación de chat se añaden al proyecto bajo la ruta `chat/`.

Necesitas crear una plantilla para la vista `course_chat_room`. Esta plantilla contendrá un área para visualizar los mensajes que se intercambian en el chat y una entrada de texto con un botón de envío para enviar mensajes de texto al chat.

Crea la siguiente estructura de archivos dentro del directorio de la aplicación `chat`:

```text
templates/
    chat/
        room.html
```

Edita la plantilla `chat/room.html` y añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Chat room for "{{ course.title }}"{% endblock %}

{% block content %}
    <div id="chat">
    </div>
    <div id="chat-input">
        <input id="chat-message-input" type="text">
        <input id="chat-message-submit" type="submit" value="Send">
    </div>
{% endblock %}

{% block include_js %}
{% endblock %}

{% block domready %}
{% endblock %}
```

Esta es la plantilla para la sala de chat del curso. En esta plantilla, realizas las siguientes acciones:

1. Extiendes la plantilla `base.html` de tu proyecto y completas su bloque `content`.
2. Defines un elemento HTML `<div>` con el ID `chat` que utilizarás para mostrar los mensajes de chat enviados por el usuario y por otros estudiantes.
3. También defines un segundo elemento `<div>` con una entrada de texto y un botón de envío que permitirá al usuario enviar mensajes.
4. Añades los bloques `include_js` y `domready` definidos en la plantilla `base.html`, que implementarás más adelante, para establecer una conexión con un WebSocket y enviar o recibir mensajes.

Ejecuta el servidor de desarrollo y abre `http://127.0.0.1:8000/chat/room/1/` en tu navegador, reemplazando `1` con el `id` de un curso existente en la base de datos.

Accede a la sala de chat con un usuario que haya iniciado sesión y esté inscrito en el curso. Verás la siguiente pantalla:

> *Figura 16.2: La página de la sala de chat del curso*

Esta es la pantalla de la sala de chat del curso que los estudiantes usarán para discutir temas dentro de un curso.

Has creado la vista base para la sala de chat del curso. Ahora necesitas gestionar los mensajes entre los estudiantes. La siguiente sección presentará el soporte asíncrono con Channels para la comunicación en tiempo real.

---

### Django en tiempo real con Channels

Estás creando un servidor de chat para proporcionar a los estudiantes una sala de chat para cada curso. Los estudiantes inscritos en un curso podrán acceder a la sala de chat del curso e intercambiar mensajes. Esta funcionalidad requiere una comunicación en tiempo real entre el servidor y el cliente.

Un modelo estándar de solicitud/respuesta HTTP no funciona aquí porque necesitas que el navegador reciba notificaciones tan pronto como lleguen nuevos mensajes. Hay varias formas en las que podrías implementar esta función, utilizando *polling* AJAX o *long polling* en combinación con el almacenamiento de los mensajes en tu base de datos o Redis. Sin embargo, no existe una forma eficiente de implementar la comunicación en tiempo real mediante una aplicación web síncrona estándar.

Necesitas comunicación asíncrona, que permite interacciones en tiempo real, donde el servidor puede enviar actualizaciones al cliente tan pronto como llegan nuevos mensajes sin que el cliente necesite solicitar actualizaciones periódicamente. La comunicación asíncrona también incluye otras ventajas, como una latencia reducida, un rendimiento mejorado bajo carga y una mejor experiencia de usuario en general. Vas a construir un servidor de chat utilizando comunicación asíncrona a través de ASGI.

#### Aplicaciones asíncronas utilizando ASGI

Django normalmente se despliega utilizando Web Server Gateway Interface (WSGI), que es la interfaz estándar para que las aplicaciones de Python manejen solicitudes HTTP. Sin embargo, para trabajar con aplicaciones asíncronas, necesitas usar otra interfaz llamada ASGI, que también puede manejar solicitudes WebSocket. ASGI es el estándar emergente de Python para servidores y aplicaciones web asíncronos. Al usar ASGI, permitiremos que Django maneje cada mensaje de forma independiente y en tiempo real, creando una experiencia de chat en vivo fluida para los estudiantes.

Puedes encontrar una introducción a ASGI en [https://asgi.readthedocs.io/en/latest/introduction.html](https://asgi.readthedocs.io/en/latest/introduction.html).

Django viene con soporte para ejecutar Python asíncrono a través de ASGI. La escritura de vistas asíncronas es compatible desde Django 3.1, y Django 4.1 introdujo controladores asíncronos para vistas basadas en clases. Django 5.0 agregó el manejo de eventos de desconexión en vistas asíncronas antes de que se genere la respuesta. También añade funciones asíncronas al framework de autenticación, proporciona soporte para el envío de señales asíncronas y añade soporte asíncrono a múltiples decoradores integrados.

Channels se basa en el soporte nativo de ASGI disponible en Django y proporciona funcionalidades adicionales para manejar protocolos que requieren conexiones de larga duración, como WebSockets, protocolos de IoT y protocolos de chat.

Los WebSockets proporcionan comunicación dúplex completa (*full-duplex*) al establecer una conexión de Protocolo de Control de Transmisión (TCP) persistente, abierta y bidireccional entre servidores y clientes. En lugar de enviar solicitudes HTTP al servidor, estableces una conexión con el servidor; una vez que el canal está abierto, los mensajes se pueden intercambiar en ambas direcciones sin necesidad de establecer una nueva conexión cada vez. Vas a usar WebSockets para implementar tu servidor de chat.

Puedes leer más sobre WebSockets en [https://en.wikipedia.org/wiki/WebSocket](https://en.wikipedia.org/wiki/WebSocket).

Puedes encontrar más información sobre el despliegue de Django con ASGI en [https://docs.djangoproject.com/en/5.2/howto/deployment/asgi/](https://docs.djangoproject.com/en/5.2/howto/deployment/asgi/).

Puedes encontrar más información sobre el soporte de Django para escribir vistas asíncronas en [https://docs.djangoproject.com/en/5.2/topics/async/](https://docs.djangoproject.com/en/5.2/topics/async/) y el soporte de Django para vistas basadas en clases asíncronas en [https://docs.djangoproject.com/en/5.2/topics/class-based-views/#async-class-based-views](https://docs.djangoproject.com/en/5.2/topics/class-based-views/#async-class-based-views).

A continuación, vamos a aprender cómo se altera el ciclo de solicitud/respuesta de Django al usar Channels.

#### El ciclo de solicitud/respuesta utilizando Channels

Es importante comprender las diferencias en un ciclo de solicitud entre un ciclo de solicitud síncrono estándar y una implementación de Channels. El siguiente esquema muestra el ciclo de solicitud de una configuración síncrona de Django:

> *Figura 16.3: El ciclo de solicitud/respuesta de Django*

Cuando el navegador envía una solicitud HTTP al servidor web, Django maneja la solicitud y pasa el objeto `HttpRequest` a la vista correspondiente. La vista procesa la solicitud y devuelve un objeto `HttpResponse` que se envía de vuelta al navegador como una respuesta HTTP. No existe ningún mecanismo para mantener una conexión abierta o enviar datos al navegador sin una solicitud HTTP asociada.

El siguiente esquema muestra el ciclo de solicitud de un proyecto Django usando Channels con WebSockets:

> *Figura 16.4: El ciclo de solicitud/respuesta de Django Channels*

Channels reemplaza el ciclo de solicitud/respuesta de Django con mensajes que se envían a través de canales. Las solicitudes HTTP todavía se enrutan a funciones de vista usando Django, pero se enrutan a través de canales. Esto también permite el manejo de mensajes WebSocket, donde tienes productores y consumidores que intercambian mensajes a través de una capa de canales. Channels preserva la arquitectura síncrona de Django, permitiéndote elegir entre escribir código síncrono y código asíncrono, o una combinación de ambos.

Tus vistas síncronas existentes coexistirán con la funcionalidad WebSocket que implementaremos con Daphne, y atenderás solicitudes tanto HTTP como WebSocket.

A continuación, vas a instalar Channels y agregarlo a tu proyecto.

---

### Instalación de Channels y Daphne

Vas a añadir Channels a tu proyecto y configurar el enrutamiento básico de la aplicación ASGI necesario para administrar las solicitudes HTTP.

Instala Channels en tu entorno virtual con el siguiente comando:

```bash
python -m pip install -U 'channels[daphne]==4.2.0'
```

Esto instalará simultáneamente Channels junto con el servidor de aplicaciones ASGI Daphne. Se necesita un servidor ASGI para manejar solicitudes asíncronas, y elegimos Daphne por su simplicidad y compatibilidad, ya que viene empaquetado con Channels.

Edita el archivo `settings.py` del proyecto `educa` y añade `daphne` al principio de la configuración `INSTALLED_APPS` de la siguiente manera:

```python
INSTALLED_APPS = [
    'daphne',
    # ...
]
```

Cuando se añade `daphne` a la configuración `INSTALLED_APPS`, toma el control sobre el comando `runserver`, reemplazando el servidor de desarrollo estándar de Django. Esto te permitirá atender solicitudes asíncronas durante el desarrollo. Además de manejar el enrutamiento de URLs a las vistas de Django para solicitudes síncronas, Daphne también administra rutas a consumidores WebSocket. Puedes encontrar más información sobre Daphne en [https://github.com/django/daphne](https://github.com/django/daphne).

Channels espera que definas una única aplicación raíz que se ejecutará para todas las solicitudes. Puedes definir la aplicación raíz agregando la configuración `ASGI_APPLICATION` a tu proyecto. Esto es similar a la configuración `ROOT_URLCONF` que apunta a los patrones de URL base de tu proyecto. Puedes colocar la aplicación raíz en cualquier lugar de tu proyecto, pero se recomienda ponerla en un archivo a nivel de proyecto. Puedes agregar tu configuración de enrutamiento raíz directamente al archivo `asgi.py`, donde se definirá la aplicación ASGI.

Edita el archivo `asgi.py` en el directorio del proyecto `educa` y añade el siguiente código:

```python
import os
from channels.routing import ProtocolTypeRouter
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'educa.settings')

django_asgi_app = get_asgi_application()

application = ProtocolTypeRouter({
    'http': django_asgi_app,
})
```

En el código anterior, defines la aplicación ASGI principal que se ejecutará al servir el proyecto Django a través de ASGI. Utilizas la clase `ProtocolTypeRouter` proporcionada por Channels como el punto de entrada principal de tu sistema de enrutamiento. `ProtocolTypeRouter` toma un diccionario que mapea tipos de comunicación como `http` o `websocket` a aplicaciones ASGI. Instancias esta clase con la aplicación predeterminada para el protocolo HTTP. Más adelante, agregarás un protocolo para el WebSocket.

Añade la siguiente línea al archivo `settings.py` de tu proyecto:

```python
ASGI_APPLICATION = 'educa.asgi.application'
```

Channels utiliza la configuración `ASGI_APPLICATION` para ubicar la configuración de enrutamiento raíz.

Inicia el servidor de desarrollo usando el siguiente comando:

```bash
python manage.py runserver
```

Verás una salida similar a la siguiente:

```text
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
June 23, 2025 - 07:29:50
Django version 5.2.3, using settings 'educa.settings'
Starting ASGI/Daphne version 4.2.0 development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

Comprueba que la salida contenga la línea `Starting ASGI/Daphne version 4.2.0 development server`. Esta línea confirma que estás utilizando el servidor de desarrollo Daphne, que es capaz de administrar solicitudes síncronas y asíncronas, en lugar del servidor de desarrollo estándar de Django. Las solicitudes HTTP continúan comportándose igual que antes, pero se enrutan a través de Channels.

Ahora que Channels está instalado en tu proyecto, puedes construir el servidor de chat para cursos. Para implementar el servidor de chat en tu proyecto, deberás seguir los siguientes pasos:

1. **Configurar un consumidor (*consumer*)**: Los consumidores son fragmentos de código individuales que pueden manejar WebSockets de una manera muy similar a las vistas HTTP tradicionales. Construirás un consumidor para leer y escribir mensajes en un canal de comunicación.
2. **Configurar el enrutamiento (*routing*)**: Channels proporciona clases de enrutamiento que te permiten combinar y apilar tus consumidores. Configurarás el enrutamiento de URL para tu consumidor de chat.
3. **Implementar un cliente WebSocket**: Cuando el estudiante acceda a la sala de chat, te conectarás al WebSocket desde el navegador y enviarás o recibirás mensajes usando JavaScript.
4. **Habilitar una capa de canales (*channel layer*)**: Las capas de canales te permiten hablar entre diferentes instancias de una aplicación. Son una parte útil para hacer una aplicación distribuida en tiempo real. Configurarás una capa de canales usando Redis.

Comencemos escribiendo tu propio consumidor para gestionar la conexión a un WebSocket, recibir y enviar mensajes, y desconectarse.

---

### Escritura de un consumer

Los consumidores son el equivalente de las vistas de Django para aplicaciones asíncronas. Como se mencionó, manejan WebSockets de una manera muy similar a cómo las vistas tradicionales manejan solicitudes HTTP. Los consumidores son aplicaciones ASGI que pueden manejar mensajes, notificaciones y otras cosas. A diferencia de las vistas de Django, los consumidores están construidos para comunicaciones de larga duración. Las URLs se mapean a consumidores a través de clases de enrutamiento que te permiten combinar y apilar consumidores.

Implementemos un consumidor básico que pueda aceptar conexiones WebSocket y reenvíe como un eco cada mensaje que recibe del WebSocket de vuelta a él. Esta funcionalidad inicial permitirá al estudiante enviar mensajes al consumidor y recibir de vuelta los mensajes que envía.

Crea un nuevo archivo dentro del directorio de la aplicación `chat` y nómbralo `consumers.py`. Añade el siguiente código:

```python
import json
from channels.generic.websocket import WebsocketConsumer


class ChatConsumer(WebsocketConsumer):
    def connect(self):
        # accept connection
        self.accept()

    def disconnect(self, close_code):
        pass

    # receive message from WebSocket
    def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']
        # send message to WebSocket
        self.send(text_data=json.dumps({'message': message}))
```

Este es el consumidor `ChatConsumer`. Esta clase hereda de la clase `WebsocketConsumer` de Channels para implementar un consumidor WebSocket básico. En este consumidor, implementas los siguientes métodos:

- `connect()`: Se llama cuando se recibe una nueva conexión. Aceptas cualquier conexión con `self.accept()`. También puedes rechazar una conexión llamando a `self.close()`.
- `disconnect()`: Se llama cuando el socket se cierra. Utilizas `pass` porque no necesitas implementar ninguna acción cuando un cliente cierra la conexión.
- `receive()`: Se llama cada vez que se reciben datos del WebSocket. Esperas que el texto se reciba como `text_data` (esto también podría ser `bytes_data` para datos binarios). Tratas los datos de texto recibidos como JSON. Por lo tanto, utilizas `json.loads()` para cargar los datos JSON recibidos en un diccionario de Python. Accedes a la clave `message`, que esperas que esté presente en la estructura JSON recibida. Para hacer eco del mensaje, lo envías de vuelta al WebSocket con `self.send()`, transformándolo nuevamente en formato JSON a través de `json.dumps()`.

La versión inicial de tu consumidor `ChatConsumer` acepta cualquier conexión WebSocket y reenvía al cliente WebSocket cada mensaje que recibe. Ten en cuenta que el consumidor aún no transmite mensajes a otros clientes. Construirás esta funcionalidad implementando una capa de canales más adelante.

Primero, expongamos nuestro consumidor agregándolo a las URLs del proyecto.

---

### Enrutamiento (routing)

Necesitas definir una URL para enrutar las conexiones al consumidor `ChatConsumer` que has implementado. Channels proporciona clases de enrutamiento que te permiten combinar y apilar consumidores para despacharlos según la conexión. Puedes pensar en ellos como el sistema de enrutamiento de URL de Django para aplicaciones asíncronas.

Crea un nuevo archivo dentro del directorio de la aplicación `chat` y nómbralo `routing.py`. Añade el siguiente código:

```python
from django.urls import re_path
from . import consumers

websocket_urlpatterns = [
    re_path(
        r'ws/chat/room/(?P<course_id>\d+)/$',
        consumers.ChatConsumer.as_asgi()
    ),
]
```

En este código, mapeas un patrón de URL con la clase `ChatConsumer` que definiste en el archivo `chat/consumers.py`. Hay algunos detalles que vale la pena revisar:

- Utilizas `re_path()` de Django para definir la ruta con una expresión regular en lugar de `path()`. El enrutamiento de URL de Channels puede no funcionar correctamente con rutas `path()` si los enrutadores internos están envueltos por middleware adicional, por lo que este enfoque ayuda a evitar problemas.
- La URL incluye un parámetro entero llamado `course_id`. Este parámetro estará disponible en el alcance (*scope*) del consumidor y te permitirá identificar la sala de chat del curso a la que se está conectando el usuario.
- Llamas al método `as_asgi()` de la clase del consumidor para obtener una aplicación ASGI que instanciará una instancia del consumidor para cada conexión de usuario. Este comportamiento es similar al método `as_view()` de Django para vistas basadas en clases.
- Es una buena práctica anteponer a las URLs de WebSocket el prefijo `/ws/` para diferenciarlas de las URLs utilizadas para solicitudes HTTP síncronas estándar. Esto también simplifica la configuración de producción cuando un servidor HTTP enruta las solicitudes según la ruta.

Edita el archivo global `asgi.py` ubicado junto al archivo `settings.py` para que se vea así:

```python
import os
from channels.auth import AuthMiddlewareStack
from channels.routing import ProtocolTypeRouter, URLRouter
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'educa.settings')

django_asgi_app = get_asgi_application()

from chat.routing import websocket_urlpatterns

application = ProtocolTypeRouter({
    'http': django_asgi_app,
    'websocket': AuthMiddlewareStack(
        URLRouter(websocket_urlpatterns)
    ),
})
```

En este código, has añadido:

- Una nueva ruta para el protocolo `websocket`. Utilizas `URLRouter` para mapear conexiones websocket a los patrones de URL definidos en la lista `websocket_urlpatterns` del módulo `chat.routing`.
- `AuthMiddlewareStack` como una función contenedora para el enrutador de URL. La clase `AuthMiddlewareStack` proporcionada por Channels admite la autenticación estándar de Django, donde los detalles del usuario se almacenan en la sesión. Más adelante, accederás a la instancia del usuario en el alcance del consumidor para identificar al usuario que envía un mensaje.

Ten en cuenta que la importación de `websocket_urlpatterns` está debajo de la llamada a la función `get_asgi_application()`. Esto es necesario para garantizar que el registro de la aplicación se complete antes de importar código que pueda importar modelos del ORM.

Ahora que tenemos un consumidor WebSocket funcional que está disponible a través de una URL, podemos implementar el cliente WebSocket.

---

### Implementación del cliente WebSocket

Hasta ahora, has creado la vista `course_chat_room` y su plantilla correspondiente para que los estudiantes accedan a la sala de chat del curso. Has implementado un consumidor WebSocket para el servidor de chat y lo has vinculado con el enrutamiento de URL. Ahora, necesitas crear un cliente WebSocket para establecer una conexión con el WebSocket en la plantilla de la sala de chat del curso y poder enviar/recibir mensajes.

Vas a implementar el cliente WebSocket con JavaScript para abrir y mantener una conexión en el navegador, e interactuarás con el Document Object Model (DOM) mediante JavaScript.

Realizarás las siguientes tareas relacionadas con el cliente WebSocket:

- Abrir una conexión WebSocket con el servidor cuando se cargue la página.
- Añadir mensajes a un contenedor HTML cuando se reciban datos a través del WebSocket.
- Adjuntar un detector de eventos al botón de envío para enviar mensajes a través del WebSocket cuando el usuario haga clic en el botón SEND o presione la tecla Enter.

Comencemos abriendo la conexión WebSocket.

Edita la plantilla `chat/room.html` de la aplicación `chat` y modifica los bloques `include_js` y `domready`, de la siguiente manera:

```html
{% block include_js %}
    {{ course.id|json_script:"course-id" }}
{% endblock %}

{% block domready %}
    const courseId = JSON.parse(
        document.getElementById('course-id').textContent
    );
    const url = 'ws://' + window.location.host + '/ws/chat/room/' + courseId + '/';
    const chatSocket = new WebSocket(url);
{% endblock %}
```

En el bloque `include_js`, utilizas el filtro de plantilla `json_script` para usar de forma segura el valor de `course.id` con JavaScript. El filtro de plantilla `json_script` proporcionado por Django genera un objeto de Python como JSON, envuelto en una etiqueta `<script>`, para que puedas usarlo de forma segura con JavaScript. El código `{{ course.id|json_script:"course-id" }}` se renderiza como `<script id="course-id" type="application/json">6</script>`. Este valor se recupera luego en el bloque `domready` analizando el contenido del elemento con `id="course-id"` mediante `JSON.parse()`. Esta es la forma segura de usar objetos de Python en JavaScript.

El filtro de plantilla `json_script` codifica de forma segura objetos de Python como JSON y los incrusta en una etiqueta HTML `<script>`, protegiendo contra ataques de cross-site scripting (XSS) al escapar caracteres potencialmente dañinos.

Puedes encontrar más información sobre el filtro de plantilla `json_script` en [https://docs.djangoproject.com/en/5.2/ref/templates/builtins/#json-script](https://docs.djangoproject.com/en/5.2/ref/templates/builtins/#json-script).

En el bloque `domready`, defines una URL con el protocolo WebSocket, que se parece a `ws://` (o `wss://` para WebSockets seguros, al igual que `https://`). Construyes la URL utilizando la ubicación actual del navegador, que obtienes de `window.location.host`. El resto de la URL se construye con la ruta para el patrón de URL de la sala de chat que definiste en el archivo `routing.py` de la aplicación `chat`.

Escribes la URL en lugar de construirla con un resolver porque Channels no proporciona una forma de invertir URLs (*reverse*). Utilizas el ID del curso actual para generar la URL para el curso actual y almacenas la URL en una nueva constante llamada `url`.

Luego, abres una conexión WebSocket a la URL almacenada usando `new WebSocket(url)`. Asignas el objeto cliente WebSocket instanciado a la nueva constante `chatSocket`.

Has creado un consumidor WebSocket, has incluido el enrutamiento para él y has implementado un cliente WebSocket básico. Probemos la versión inicial de tu chat.

Inicia el servidor de desarrollo usando el siguiente comando:

```bash
python manage.py runserver
```

Abre la URL `http://127.0.0.1:8000/chat/room/1/` en tu navegador, reemplazando `1` con el `id` de un curso existente en la base de datos. Echa un vistazo a la salida de la consola. Además de las solicitudes HTTP GET para la página y sus archivos estáticos, deberías ver dos líneas, incluyendo `WebSocket HANDSHAKING` y `WebSocket CONNECT`, como la siguiente salida:

```text
HTTP GET /chat/room/1/ 200 [0.02, 127.0.0.1:57141]
WebSocket HANDSHAKING /ws/chat/room/1/ [127.0.0.1:57144]
WebSocket CONNECT /ws/chat/room/1/ [127.0.0.1:57144]
```

El servidor Daphne escucha las conexiones de socket entrantes utilizando un socket TCP estándar. El *handshake* es el puente de HTTP a WebSockets. En el handshake, se negocian los detalles de la conexión y cualquiera de las partes puede cerrar la conexión antes de completarla. Recuerda que estás utilizando `self.accept()` para aceptar cualquier conexión en el método `connect()` de la clase `ChatConsumer`, implementada en el archivo `consumers.py` de la aplicación `chat`. La conexión se acepta y, por lo tanto, ves el mensaje `WebSocket CONNECT` en la consola.

Si utilizas las herramientas de desarrollo del navegador para rastrear conexiones de red, también puedes ver información de la conexión WebSocket que se ha establecido.

Debería verse como la Figura 16.5:

> *Figura 16.5: Las herramientas de desarrollo del navegador muestran que se ha establecido la conexión WebSocket*

Ahora que puedes conectarte al WebSocket, es hora de interactuar con él. Implementarás los métodos para manejar eventos comunes, como recibir un mensaje y cerrar la conexión. Edita la plantilla `chat/room.html` de la aplicación `chat` y modifica el bloque `domready`, de la siguiente manera:

```html
{% block domready %}
    const courseId = JSON.parse(
        document.getElementById('course-id').textContent
    );
    const url = 'ws://' + window.location.host + '/ws/chat/room/' + courseId + '/';
    const chatSocket = new WebSocket(url);

    chatSocket.onmessage = function(event) {
        const data = JSON.parse(event.data);
        const chat = document.getElementById('chat');
        chat.innerHTML += '<div class="message">' + data.message + '</div>';
        chat.scrollTop = chat.scrollHeight;
    };

    chatSocket.onclose = function(event) {
        console.error('Chat socket closed unexpectedly');
    };
{% endblock %}
```

En este código, defines los siguientes eventos para el cliente WebSocket:

- `onmessage`: Se dispara cuando se reciben datos a través del WebSocket. Analizas el mensaje, que esperas en formato JSON, y accedes a su atributo `message`. Luego, agregas un nuevo elemento `<div>` con el mensaje recibido al elemento HTML con el ID `chat`. Esto añadirá nuevos mensajes al registro de chat, manteniendo todos los mensajes anteriores que se hayan añadido al registro. Desplazas el `<div>` del registro de chat hacia abajo para garantizar que el nuevo mensaje obtenga visibilidad. Logras esto desplazándote a la altura desplazable total del registro de chat, que se puede obtener accediendo a su atributo `scrollHeight`.
- `onclose`: Se dispara cuando se cierra la conexión con el WebSocket. No esperas cerrar la conexión y, por lo tanto, escribes el error `Chat socket closed unexpectedly` en el registro de la consola si esto sucede.

Has implementado la acción para mostrar el mensaje cuando se recibe un nuevo mensaje. Necesitas implementar la funcionalidad para enviar mensajes al socket también.

Edita la plantilla `chat/room.html` de la aplicación `chat` y añade el siguiente código JavaScript al final del bloque `domready`:

```javascript
    const input = document.getElementById('chat-message-input');
    const submitButton = document.getElementById('chat-message-submit');

    submitButton.addEventListener('click', function(event) {
        const message = input.value;
        if(message) {
            // send message in JSON format
            chatSocket.send(JSON.stringify({'message': message}));
            // clear input
            input.value = '';
            input.focus();
        }
    });
```

En este código, defines un detector de eventos para el evento `click` del botón de envío, que seleccionas por su ID `chat-message-submit`. Cuando se hace clic en el botón, realizas las siguientes acciones:

1. Lees el mensaje ingresado por el usuario desde el valor del elemento de entrada de texto con el ID `chat-message-input`.
2. Verificas si el mensaje tiene algún contenido con `if(message)`.
3. Si el usuario ha ingresado un mensaje, creas contenido JSON como `{'message': 'string ingresada por el usuario'}` usando `JSON.stringify()`.
4. Envías el contenido JSON a través del WebSocket, llamando al método `send()` del cliente `chatSocket`.
5. Borras el contenido de la entrada de texto estableciendo su valor en una cadena vacía con `input.value = ''`.
6. Devuelves el foco a la entrada de texto con `input.focus()` para que el usuario pueda escribir un nuevo mensaje de inmediato.

El usuario ahora puede enviar mensajes utilizando la entrada de texto y haciendo clic en el botón de envío.

Para mejorar la experiencia del usuario, le daremos el foco a la entrada de texto cuando se cargue la página, lo que permitirá a los usuarios comenzar a escribir de inmediato sin necesidad de hacer clic en ella primero. También capturarás eventos de pulsación de teclas del teclado para identificar la tecla Enter y disparar el evento de clic en el botón de envío. Los usuarios podrán hacer clic en el botón o presionar la tecla Enter para enviar un mensaje.

Edita la plantilla `chat/room.html` de la aplicación `chat` y añade el siguiente código JavaScript al final del bloque `domready`:

```javascript
    input.addEventListener('keypress', function(event) {
        if (event.key === 'Enter') {
            // cancel the default action, if needed
            event.preventDefault();
            // trigger click event on button
            submitButton.click();
        }
    });

    input.focus();
```

En este código, también defines una función para el evento `keypress` del elemento de entrada. Para cualquier tecla que presione el usuario, realizas las siguientes acciones:

1. Verificas si su tecla es `Enter`.
2. Si se presiona la tecla Enter: Evitas el comportamiento predeterminado para esta tecla con `event.preventDefault()`.
3. Luego, disparas el evento `click` en el botón de envío para enviar el mensaje al WebSocket.

Fuera del controlador de eventos, en el código JavaScript principal para el bloque `domready`, le das el foco a la entrada de texto con `input.focus()`. Al hacerlo, cuando se cargue el DOM, el foco se establecerá en el elemento de entrada para que el usuario escriba un mensaje.

El bloque `domready` de la plantilla `chat/room.html` ahora debería verse de la siguiente manera:

```html
{% block domready %}
    const courseId = JSON.parse(
        document.getElementById('course-id').textContent
    );
    const url = 'ws://' + window.location.host + '/ws/chat/room/' + courseId + '/';
    const chatSocket = new WebSocket(url);

    chatSocket.onmessage = function(event) {
        const data = JSON.parse(event.data);
        const chat = document.getElementById('chat');
        chat.innerHTML += '<div class="message">' + data.message + '</div>';
        chat.scrollTop = chat.scrollHeight;
    };

    chatSocket.onclose = function(event) {
        console.error('Chat socket closed unexpectedly');
    };

    const input = document.getElementById('chat-message-input');
    const submitButton = document.getElementById('chat-message-submit');

    submitButton.addEventListener('click', function(event) {
        const message = input.value;
        if(message) {
            // send message in JSON format
            chatSocket.send(JSON.stringify({'message': message}));
            // clear input
            input.value = '';
            input.focus();
        }
    });

    input.addEventListener('keypress', function(event) {
        if (event.key === 'Enter') {
            // cancel the default action, if needed
            event.preventDefault();
            // trigger click event on button
            submitButton.click();
        }
    });

    input.focus();
{% endblock %}
```

Abre la URL `http://127.0.0.1:8000/chat/room/1/` en tu navegador, reemplazando `1` con el `id` de un curso existente en la base de datos. Con un usuario que haya iniciado sesión y esté inscrito en el curso, escribe un texto en el campo de entrada y haz clic en el botón SEND o presiona la tecla Enter.

Verás que tu mensaje aparece en el registro de chat:

> *Figura 16.6: La página de la sala de chat, incluyendo los mensajes enviados a través de WebSocket*

¡Genial! El mensaje se ha enviado a través del WebSocket y el consumidor `ChatConsumer` ha recibido el mensaje y lo ha devuelto a través del WebSocket. El cliente `chatSocket` ha recibido un evento de mensaje y se ha ejecutado la función `onmessage`, agregando el mensaje al registro de chat.

Has implementado la funcionalidad con un consumidor WebSocket y un cliente WebSocket para establecer la comunicación cliente/servidor y puedes enviar o recibir eventos. Sin embargo, el servidor de chat no puede transmitir mensajes a otros clientes. Si abres una segunda pestaña del navegador e ingresas un mensaje, el mensaje no aparecerá en la primera pestaña. Para construir la comunicación entre consumidores, debes habilitar una capa de canales (*channel layer*).

---

### Habilitación de una capa de canales (channel layer)

Las capas de canales te permiten comunicarte entre diferentes instancias de una aplicación. Una capa de canales es el mecanismo de transporte que permite que múltiples instancias de consumidores se comuniquen entre sí y con otras partes de Django.

En tu servidor de chat, planeas tener múltiples instancias del consumidor `ChatConsumer` para la misma sala de chat del curso. Cada estudiante que se una a la sala de chat instanciará el cliente WebSocket en su navegador, y eso abrirá una conexión con una instancia del consumidor WebSocket. Necesitas una capa de canales común para distribuir mensajes entre consumidores.

#### Canales y grupos

Las capas de canales proporcionan dos abstracciones para gestionar las comunicaciones: canales y grupos:

- **Canal (*Channel*)**: Puedes pensar en un canal como una bandeja de entrada donde se pueden enviar mensajes o como una cola de tareas. Cada canal tiene un nombre. Cualquiera que conozca el nombre del canal puede enviar mensajes a un canal y luego entregarlos a los consumidores que escuchan en ese canal.
- **Grupo (*Group*)**: Múltiples canales se pueden agrupar en un grupo. Cada grupo tiene un nombre. Cualquiera que conozca el nombre del grupo puede agregar o eliminar un canal de un grupo. Usando el nombre del grupo, también puedes enviar un mensaje a todos los canales del grupo.

Trabajarás con grupos de canales para implementar el servidor de chat. Al crear un grupo de canales para la sala de chat de cada curso, las instancias de `ChatConsumer` podrán comunicarse entre sí.

Añadamos una capa de canales a nuestro proyecto.

#### Configuración de una capa de canales con Redis

Redis es la opción preferida para una capa de canales, aunque Channels tiene soporte para otros tipos de capas de canales. Redis funciona como el almacén de comunicación para la capa de canales. Recuerda que ya utilizaste Redis en el Capítulo 7, Seguimiento de las acciones del usuario, en el Capítulo 10, Extender tu tienda, y en el Capítulo 14, Renderizado y almacenamiento en caché de contenido.

Si aún no has instalado Redis, puedes encontrar las instrucciones de instalación en el Capítulo 7, Seguimiento de las acciones del usuario.

Para usar Redis como una capa de canales, debes instalar el paquete `channels-redis`. Instala `channels-redis` en tu entorno virtual con el siguiente comando:

```bash
python -m pip install channels-redis==4.2.0
```

Edita el archivo `settings.py` del proyecto `educa` y añade el siguiente código:

```python
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            'hosts': [('127.0.0.1', 6379)],
        },
    },
}
```

La configuración `CHANNEL_LAYERS` define los ajustes para las capas de canales disponibles para el proyecto. Defines una capa de canales predeterminada utilizando el backend `RedisChannelLayer` proporcionado por `channels-redis` y especificas el host `127.0.0.1` y el puerto `6379`, en el que se está ejecutando Redis.

Probemos la capa de canales. Inicializa el contenedor Docker de Redis usando el siguiente comando:

```bash
docker run -it --rm --name redis -p 6379:6379 redis:7.2.4
```

Si deseas ejecutar el comando en segundo plano (en modo desconectado), puedes usar la opción `-d`.

Abre la shell de Django usando el siguiente comando desde el directorio del proyecto:

```bash
python manage.py shell
```

Para verificar que la capa de canales puede comunicarse con Redis, escribe el siguiente código para enviar un mensaje a un canal de prueba llamado `test_channel` y recibirlo de vuelta:

```python
>>> import channels.layers
>>> from asgiref.sync import async_to_sync
>>> channel_layer = channels.layers.get_channel_layer()
>>> async_to_sync(channel_layer.send)('test_channel', {'message': 'hello'})
>>> async_to_sync(channel_layer.receive)('test_channel')
```

Deberías obtener el siguiente resultado:

```python
{'message': 'hello'}
```

En el código anterior, envías un mensaje a un canal de prueba a través de la capa de canales y luego lo recuperas de la capa de canales. La capa de canales se está comunicando con éxito con Redis.

A continuación, añadiremos la capa de canales a nuestro proyecto.

#### Actualización del consumer para difundir mensajes

Editemos el consumidor `ChatConsumer` para usar la capa de canales que hemos implementado con Redis. Utilizarás un grupo de canales para la sala de chat de cada curso. Por lo tanto, utilizarás el id del curso para construir el nombre del grupo. Las instancias de `ChatConsumer` conocerán el nombre del grupo y podrán comunicarse entre sí.

Edita el archivo `consumers.py` de la aplicación `chat`, importa la función `async_to_sync()` y modifica el método `connect()` de la clase `ChatConsumer`, de la siguiente manera:

```python
import json
from asgiref.sync import async_to_sync
from channels.generic.websocket import WebsocketConsumer


class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.id = self.scope['url_route']['kwargs']['course_id']
        self.room_group_name = f'chat_{self.id}'
        # join room group
        async_to_sync(self.channel_layer.group_add)(
            self.room_group_name,
            self.channel_name
        )
        # accept connection
        self.accept()
    # ...
```

En este código, importas la función auxiliar `async_to_sync()` para envolver llamadas a métodos asíncronos de la capa de canales. `ChatConsumer` es un consumidor síncrono `WebsocketConsumer`, pero necesita llamar a métodos asíncronos de la capa de canales.

En el nuevo método `connect()`, realizas las siguientes tareas:

1. Recuperas el id del curso del alcance (*scope*) para conocer el curso con el que está asociada la sala de chat. Accedes a `self.scope['url_route']['kwargs']['course_id']` para recuperar el parámetro `course_id` de la URL. Cada consumidor tiene un alcance con información sobre su conexión, los argumentos pasados por la URL y el usuario autenticado, si lo hay.
2. Construyes el nombre del grupo con el id del curso al que corresponde el grupo. Recuerda que tendrás un grupo de canales para la sala de chat de cada curso. Almacenas el nombre del grupo en el atributo `room_group_name` del consumidor.
3. Te unes al grupo agregando el canal actual al grupo. Obtienes el nombre del canal del atributo `channel_name` del consumidor. Utilizas el método `group_add` de la capa de canales para agregar el canal al grupo. Utilizas el contenedor `async_to_sync()` para usar el método asíncrono de la capa de canales.
4. Mantienes la llamada `self.accept()` para aceptar la conexión WebSocket.

Cuando el consumidor `ChatConsumer` recibe una nueva conexión WebSocket, agrega el canal al grupo asociado con el curso en su alcance. El consumidor ahora puede recibir cualquier mensaje enviado al grupo.

En el mismo archivo `consumers.py`, modifica el método `disconnect()` de la clase `ChatConsumer`, de la siguiente manera:

```python
class ChatConsumer(WebsocketConsumer):
    # ...
    def disconnect(self, close_code):
        # leave room group
        async_to_sync(self.channel_layer.group_discard)(
            self.room_group_name,
            self.channel_name
        )
    # ...
```

Cuando se cierra la conexión, llamas al método `group_discard()` de la capa de canales para abandonar el grupo. Utilizas el contenedor `async_to_sync()` para usar el método asíncrono de la capa de canales.

En el mismo archivo `consumers.py`, modifica el método `receive()` de la clase `ChatConsumer`, de la siguiente manera:

```python
class ChatConsumer(WebsocketConsumer):
    # ...
    # receive message from WebSocket
    def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']
        # send message to room group
        async_to_sync(self.channel_layer.group_send)(
            self.room_group_name,
            {
                'type': 'chat_message',
                'message': message,
            }
        )
```

Cuando recibes un mensaje de la conexión WebSocket, en lugar de enviar el mensaje al canal asociado, envías el mensaje al grupo. Haces esto llamando al método `group_send()` de la capa de canales. Utilizas el contenedor `async_to_sync()` para usar el método asíncrono de la capa de canales. Pasas la siguiente información en el evento enviado al grupo:

- `type`: El tipo de evento. Esta es una clave especial que corresponde al nombre del método que debe invocarse en los consumidores que reciben el evento. Puedes implementar un método en el consumidor con el mismo nombre que el tipo de mensaje para que se ejecute cada vez que se reciba un mensaje con ese tipo específico.
- `message`: El mensaje real que estás enviando.

En el mismo archivo `consumers.py`, añade un nuevo método `chat_message()` en la clase `ChatConsumer`, de la siguiente manera:

```python
class ChatConsumer(WebsocketConsumer):
    # ...
    # receive message from room group
    def chat_message(self, event):
        # send message to WebSocket
        self.send(text_data=json.dumps(event))
```

Nombres este método `chat_message()` para que coincida con la clave `type` que se envía al grupo de canales cuando se recibe un mensaje del WebSocket. Cuando se envía un mensaje con tipo `chat_message` al grupo, todos los consumidores suscritos al grupo recibirán el mensaje y ejecutarán el método `chat_message()`. En el método `chat_message()`, envías el mensaje de evento recibido al WebSocket.

El archivo `consumers.py` completo ahora debería verse así:

```python
import json
from asgiref.sync import async_to_sync
from channels.generic.websocket import WebsocketConsumer


class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.id = self.scope['url_route']['kwargs']['course_id']
        self.room_group_name = f'chat_{self.id}'
        # join room group
        async_to_sync(self.channel_layer.group_add)(
            self.room_group_name,
            self.channel_name
        )
        # accept connection
        self.accept()

    def disconnect(self, close_code):
        # leave room group
        async_to_sync(self.channel_layer.group_discard)(
            self.room_group_name,
            self.channel_name
        )

    # receive message from WebSocket
    def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']
        # send message to room group
        async_to_sync(self.channel_layer.group_send)(
            self.room_group_name,
            {
                'type': 'chat_message',
                'message': message,
            }
        )

    # receive message from room group
    def chat_message(self, event):
        # send message to WebSocket
        self.send(text_data=json.dumps(event))
```

Has implementado una capa de canales en `ChatConsumer`, lo que permite a los consumidores transmitir mensajes y comunicarse entre sí.

Ejecuta el servidor de desarrollo con el siguiente comando:

```bash
python manage.py runserver
```

Abre la URL `http://127.0.0.1:8000/chat/room/1/` en tu navegador, reemplazando `1` con el `id` de un curso existente en la base de datos. Escribe un mensaje y envíalo. Luego, abre una segunda ventana del navegador y accede a la misma URL. Envía un mensaje desde cada ventana del navegador.

El resultado debería verse así:

> *Figura 16.7: La página de la sala de chat con mensajes enviados desde diferentes ventanas del navegador*

Verás que el primer mensaje solo se muestra en la primera ventana del navegador. Cuando abres una segunda ventana del navegador, los mensajes enviados en cualquiera de las ventanas del navegador se muestran en ambas. Cuando abres una nueva ventana del navegador y accedes a la URL de la sala de chat, se establece una nueva conexión WebSocket entre el cliente WebSocket de JavaScript en el navegador y el consumidor WebSocket en el servidor. Cada canal se agrega al grupo asociado con el ID del curso y se pasa a través de la URL al consumidor. Los mensajes se envían al grupo y son recibidos por todos los consumidores.

A continuación, vamos a enriquecer los mensajes con contexto adicional.

#### Adición de contexto a los mensajes

Ahora que los mensajes se pueden intercambiar entre todos los usuarios de una sala de chat, es probable que desees mostrar quién envió qué mensaje y cuándo se envió. Añadamos algo de contexto a los mensajes.

Edita el archivo `consumers.py` de la aplicación `chat` e implementa los siguientes cambios:

```python
import json
from asgiref.sync import async_to_sync
from channels.generic.websocket import WebsocketConsumer
from django.utils import timezone


class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.user = self.scope['user']
        self.id = self.scope['url_route']['kwargs']['course_id']
        self.room_group_name = f'chat_{self.id}'
        # join room group
        async_to_sync(self.channel_layer.group_add)(
            self.room_group_name,
            self.channel_name
        )
        # accept connection
        self.accept()

    def disconnect(self, close_code):
        # leave room group
        async_to_sync(self.channel_layer.group_discard)(
            self.room_group_name,
            self.channel_name
        )

    # receive message from WebSocket
    def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']
        now = timezone.now()
        # send message to room group
        async_to_sync(self.channel_layer.group_send)(
            self.room_group_name,
            {
                'type': 'chat_message',
                'message': message,
                'user': self.user.username,
                'datetime': now.isoformat(),
            }
        )

    # receive message from room group
    def chat_message(self, event):
        # send message to WebSocket
        self.send(text_data=json.dumps(event))
```

Ahora importas el módulo `timezone` proporcionado por Django. En el método `connect()` del consumidor, recuperas el usuario actual del alcance con `self.scope['user']` y lo almacenas en un nuevo atributo `user` del consumidor.

Cuando el consumidor recibe un mensaje a través del WebSocket, obtiene la hora actual utilizando `timezone.now()` y pasa el usuario actual y la fecha y hora en formato ISO 8601 junto con el mensaje en el evento enviado al grupo de canales.

Edita la plantilla `chat/room.html` de la aplicación `chat` y añade la siguiente línea al bloque `include_js`:

```html
{% block include_js %}
    {{ course.id|json_script:"course-id" }}
    {{ request.user.username|json_script:"request-user" }}
{% endblock %}
```

Usando la plantilla `json_script`, imprimes de forma segura el nombre de usuario del usuario solicitante para usarlo con JavaScript.

En el bloque `domready` de la plantilla `chat/room.html`, añade las siguientes líneas:

```javascript
{% block domready %}
    const courseId = JSON.parse(
        document.getElementById('course-id').textContent
    );
    const requestUser = JSON.parse(
        document.getElementById('request-user').textContent
    );
    // ...
{% endblock %}
```

En el nuevo código, analizas de forma segura los datos del elemento con el ID `request-user` y los almacenas en la constante `requestUser`.

Luego, en el bloque `domready`, busca las siguientes líneas:

```javascript
    const data = JSON.parse(event.data);
    const chat = document.getElementById('chat');
    chat.innerHTML += '<div class="message">' + data.message + '</div>';
    chat.scrollTop = chat.scrollHeight;
```

Reemplaza esas líneas con el siguiente código:

```javascript
    const data = JSON.parse(event.data);
    const chat = document.getElementById('chat');

    const dateOptions = {hour: 'numeric', minute: 'numeric', hour12: true};
    const datetime = new Date(data.datetime).toLocaleString('en', dateOptions);
    const isMe = data.user === requestUser;
    const source = isMe ? 'me' : 'other';
    const name = isMe ? 'Me' : data.user;

    chat.innerHTML += '<div class="message ' + source + '">' +
                      '<strong>' + name + '</strong> ' +
                      '<span class="date">' + datetime + '</span><br>' +
                      data.message + '</div>';
    chat.scrollTop = chat.scrollHeight;
```

En este código, implementas los siguientes cambios:

1. Conviertes la fecha y hora recibida en el mensaje a un objeto `Date` de JavaScript y le das formato con una configuración regional específica.
2. Comparas el nombre de usuario recibido en el mensaje con dos constantes diferentes como ayudantes para identificar al usuario.
3. La constante `source` obtiene el valor `me` si el usuario que envía el mensaje es el usuario actual, o `other` en caso contrario.
4. La constante `name` obtiene el valor `Me` si el usuario que envía el mensaje es el usuario actual o el nombre del usuario que envía el mensaje en caso contrario. Lo utilizas para mostrar el nombre del usuario que envía el mensaje.
5. Utilizas el valor `source` como una clase del elemento de mensaje `<div>` principal para diferenciar los mensajes enviados por el usuario actual de los mensajes enviados por otros. Se aplican diferentes estilos CSS según el atributo de clase. Estos estilos CSS se declaran en el archivo estático `css/base.css`.
6. Utilizas el nombre de usuario y la fecha y hora en el mensaje que agregas al registro de chat.

Abre la URL `http://127.0.0.1:8000/chat/room/1/` en tu navegador, reemplazando `1` con el `id` de un curso existente en la base de datos. Con un usuario que haya iniciado sesión y esté inscrito en el curso, escribe un mensaje y envíalo.

Luego, abre una segunda ventana del navegador en modo incógnito para evitar el uso de la misma sesión. Inicia sesión con un usuario diferente, también inscrito en el mismo curso, y envía un mensaje.

Podrás intercambiar mensajes utilizando los dos usuarios diferentes y ver el usuario y la hora, con una clara distinción entre los mensajes enviados por el usuario y los mensajes enviados por otros. La conversación entre dos usuarios debería verse similar a la siguiente:

> *Figura 16.8: La página de la sala de chat con mensajes de dos sesiones de usuario diferentes*

¡Excelente! Has creado una aplicación de chat funcional en tiempo real utilizando Channels. A continuación, aprenderás a mejorar el consumidor de chat haciéndolo completamente asíncrono.

---

### Modificación del consumer para ser totalmente asíncrono

El `ChatConsumer` que has implementado hereda de la clase base síncrona `WebsocketConsumer`. Los consumidores síncronos operan de manera que cada solicitud debe procesarse en secuencia, una tras otra. Los consumidores síncronos son convenientes para acceder a los modelos de Django y llamar a funciones de E/S síncronas regulares. Sin embargo, los consumidores asíncronos funcionan mejor debido a su capacidad para realizar operaciones sin bloqueo, pasando a otra tarea sin esperar a que se complete la primera operación. No requieren subprocesos adicionales al manejar solicitudes, lo que reduce los tiempos de espera y aumenta la capacidad de escalar a más usuarios y solicitudes simultáneamente.

Dado que ya estás utilizando las funciones asíncronas de la capa de canales, puedes reescribir sin problemas la clase `ChatConsumer` para hacerla asíncrona.

Edita el archivo `consumers.py` de la aplicación `chat` e implementa los siguientes cambios:

```python
import json
from channels.generic.websocket import AsyncWebsocketConsumer
from django.utils import timezone


class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.user = self.scope['user']
        self.id = self.scope['url_route']['kwargs']['course_id']
        self.room_group_name = 'chat_%s' % self.id
        # join room group
        await self.channel_layer.group_add(
            self.room_group_name,
            self.channel_name
        )
        # accept connection
        await self.accept()

    async def disconnect(self, close_code):
        # leave room group
        await self.channel_layer.group_discard(
            self.room_group_name,
            self.channel_name
        )

    # receive message from WebSocket
    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']
        now = timezone.now()
        # send message to room group
        await self.channel_layer.group_send(
            self.room_group_name,
            {
                'type': 'chat_message',
                'message': message,
                'user': self.user.username,
                'datetime': now.isoformat(),
            }
        )

    # receive message from room group
    async def chat_message(self, event):
        # send message to WebSocket
        await self.send(text_data=json.dumps(event))
```

Has implementado los siguientes cambios:

1. El consumidor `ChatConsumer` ahora hereda de la clase `AsyncWebsocketConsumer` para implementar llamadas asíncronas.
2. Has cambiado la definición de todos los métodos de `def` a `async def`.
3. Utilizas `await` para llamar a funciones asíncronas que realizan operaciones de E/S.
4. Ya no utilizas la función auxiliar `async_to_sync()` al llamar a métodos en la capa de canales.

Abre la URL `http://127.0.0.1:8000/chat/room/1/` con dos ventanas de navegador diferentes nuevamente y verifica que el servidor de chat aún funcione. ¡El servidor de chat ahora es totalmente asíncrono!

A continuación, vamos a implementar un historial de chat almacenando mensajes en la base de datos.

---

### Persistencia de mensajes en la base de datos

Mejoremos la aplicación de chat añadiendo persistencia de mensajes. Desarrollaremos la funcionalidad para almacenar mensajes en la base de datos, lo que nos permitirá presentar un historial de chat a los usuarios cuando se unan a una sala de chat. Esta característica es esencial para las aplicaciones en tiempo real, donde es necesario mostrar tanto los datos actuales como los generados previamente. Por ejemplo, considera una aplicación de negociación de acciones: al iniciar sesión, los usuarios deben ver no solo los valores actuales de las acciones, sino también los valores históricos desde el momento en que se abrió el mercado de valores.

Para implementar la funcionalidad del historial de chat, seguiremos estos pasos:

1. Crearemos un modelo de Django para almacenar mensajes de chat y lo agregaremos al sitio de administración.
2. Modificaremos el consumidor WebSocket para persistir los mensajes.
3. Recuperaremos el historial de chat para mostrar los últimos mensajes cuando los usuarios ingresen a una sala de chat.

Comencemos creando el modelo de mensaje.

#### Creación de un modelo para mensajes de chat

Edita el archivo `models.py` de la aplicación `chat` y añade las siguientes líneas:

```python
from django.conf import settings
from django.db import models


class Message(models.Model):
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.PROTECT,
        related_name='chat_messages'
    )
    course = models.ForeignKey(
        'courses.Course',
        on_delete=models.PROTECT,
        related_name='chat_messages'
    )
    content = models.TextField()
    sent_on = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f'{self.user} on {self.course} at {self.sent_on}'
```

Este es el modelo de datos para persistir mensajes de chat. Echemos un vistazo a los campos del modelo `Message`:

- `user`: El objeto `User` que escribió el mensaje. Este es un campo de clave foránea porque especifica una relación de muchos a uno: un usuario puede enviar múltiples mensajes, pero cada mensaje es enviado por un solo usuario. Al usar `PROTECT` para el parámetro `on_delete`, un objeto `User` no se puede eliminar si existen mensajes relacionados.
- `course`: Una relación con el objeto `Course`. Cada mensaje pertenece a la sala de chat de un curso. Al usar `PROTECT` para el parámetro `on_delete`, un objeto `Course` no se puede eliminar si existen mensajes relacionados.
- `content`: Un `TextField` para almacenar el contenido del mensaje.
- `sent_on`: Un `DateTimeField` para almacenar la fecha y la hora en que el objeto del mensaje se guarda por primera vez.

Ejecuta el siguiente comando en el símbolo del sistema para generar las migraciones de la base de datos para la aplicación `chat`:

```bash
python manage.py makemigrations chat
```

Deberías obtener el siguiente resultado:

```text
Migrations for 'chat':
  chat/migrations/0001_initial.py
    - Create model Message
```

Aplica la migración recién creada a tu base de datos con el siguiente comando:

```bash
python manage.py migrate
```

Obtendrás una salida que termina con la siguiente línea:

```text
Applying chat.0001_initial... OK
```

La base de datos ahora está sincronizada con el nuevo modelo. Añadamos el modelo `Message` al sitio de administración.

#### Adición del modelo de mensaje al sitio de administración

Edita el archivo `admin.py` de la aplicación `chat` y registra el modelo `Message` en el sitio de administración, de la siguiente manera:

```python
from django.contrib import admin
from chat.models import Message


@admin.register(Message)
class MessageAdmin(admin.ModelAdmin):
    list_display = ['sent_on', 'user', 'course', 'content']
    list_filter = ['sent_on', 'course']
    search_fields = ['content']
    raw_id_fields = ['user']
```

Ejecuta el servidor de desarrollo y abre `http://127.0.0.1:8000/admin/` en tu navegador. Deberías ver el bloque CHAT y la sección Messages en el sitio de administración:

> *Figura 16.9: La aplicación Chat y la sección Messages en el sitio de administración*

Continuaremos guardando mensajes en la base de datos cuando los envíen los usuarios.

#### Almacenamiento de mensajes en la base de datos

Modificaremos el consumidor WebSocket para persistir cada mensaje que se reciba a través del WebSocket. Edita el archivo `consumers.py` de la aplicación `chat` y añade el siguiente código:

```python
import json
from channels.generic.websocket import AsyncWebsocketConsumer
from django.utils import timezone
from chat.models import Message


class ChatConsumer(AsyncWebsocketConsumer):
    # ...
    async def persist_message(self, message):
        await Message.objects.acreate(
            user=self.user,
            course_id=self.id,
            content=message
        )

    # receive message from WebSocket
    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']
        now = timezone.now()
        # send message to room group
        await self.channel_layer.group_send(
            self.room_group_name,
            {
                'type': 'chat_message',
                'message': message,
                'user': self.user.username,
                'datetime': now.isoformat(),
            },
        )
        # persist message
        await self.persist_message(message)
    # ...
```

En este código, añadimos el método asíncrono `persist_message()` a la clase `ChatConsumer`. Este método toma un parámetro `message` y crea un objeto `Message` en la base de datos con el mensaje dado, el usuario autenticado relacionado y el `id` del objeto `Course` al que pertenece la sala de chat del grupo. Dado que `ChatConsumer` es completamente asíncrono, utilizamos el método de QuerySet `acreate()`, que es la versión asíncrona de `create()`. Puedes leer más sobre cómo escribir consultas asíncronas con el ORM de Django en [https://docs.djangoproject.com/en/5.2/topics/db/queries/#asynchronous-queries](https://docs.djangoproject.com/en/5.2/topics/db/queries/#asynchronous-queries).

Llamamos al método `persist_message()` de forma asíncrona en el método `receive()` que se ejecuta cuando el consumidor recibe un mensaje a través del WebSocket.

Ejecuta el servidor de desarrollo y abre `http://127.0.0.1:8000/chat/room/1/` en tu navegador, reemplazando `1` con el `id` de un curso existente en la base de datos. Con un usuario que haya iniciado sesión y esté inscrito en el curso, escribe un mensaje y envíalo.

Luego, abre una segunda ventana del navegador en modo incógnito para evitar el uso de la misma sesión. Inicia sesión con un usuario diferente, también inscrito en el mismo curso, y envía algunos mensajes también.

La Figura 16.10 muestra un ejemplo de mensajes enviados por dos usuarios diferentes:

> *Figura 16.10: Ejemplo de sala de chat con mensajes enviados por dos usuarios diferentes*

Abre `http://127.0.0.1:8000/admin/chat/message/` en tu navegador. Los mensajes enviados deben aparecer en el sitio de administración:

> *Figura 16.11: Vista de visualización de lista de administración de mensajes almacenados en la base de datos*

Todos los mensajes ahora están persistidos en la base de datos.

Ten en cuenta que los mensajes pueden contener código malicioso, por ejemplo, fragmentos de JavaScript. No marcamos los mensajes como seguros (*safe*) en nuestra plantilla, lo que proporciona una capa inicial de protección contra contenido malicioso. Sin embargo, para mejorar aún más la seguridad, considera sanear (*sanitize*) los mensajes antes de almacenarlos en la base de datos. Una opción confiable para sanear contenido es el paquete `nh3`. Puedes leer más sobre `nh3` en [https://nh3.readthedocs.io/en/latest/](https://nh3.readthedocs.io/en/latest/). Además, `django-nh3` es una integración de Django disponible que ofrece campos de modelo y campos de formulario personalizados de `nh3`. Hay más información disponible en [https://github.com/marksweb/django-nh3](https://github.com/marksweb/django-nh3).

Ahora que estás almacenando el historial de chat completo en tu base de datos, aprendamos a presentar los últimos mensajes del historial de chat a los usuarios cuando se unan a una sala de chat.

#### Visualización del historial de chat

Cuando los usuarios se unan a la sala de chat de un curso, mostraremos los últimos cinco mensajes del historial de chat. Esto garantizará que los usuarios obtengan un contexto inmediato para las conversaciones en curso.

Edita el archivo `views.py` de la aplicación `chat` y añade el siguiente código a la vista `course_chat_room`:

```python
@login_required
def course_chat_room(request, course_id):
    try:
        # retrieve course with given id joined by the current user
        course = request.user.courses_joined.get(id=course_id)
    except Course.DoesNotExist:
        # user is not a student of the course or course does not exist
        return HttpResponseForbidden()
    # retrieve chat history
    latest_messages = course.chat_messages.select_related(
        'user'
    ).order_by('-id')[:5]
    latest_messages = reversed(latest_messages)
    return render(
        request,
        'chat/room.html',
        {'course': course, 'latest_messages': latest_messages}
    )
```

Recuperamos los mensajes de chat relacionados con el curso y utilizamos `select_related()` para recuperar el usuario relacionado en la misma consulta. Esto evitará la generación de consultas SQL adicionales al acceder al nombre de usuario para mostrarlo junto a cada mensaje. El ORM de Django no admite la indexación negativa, por lo que recuperamos los primeros cinco mensajes en orden cronológico inverso y utilizamos la función `reversed()` para reordenarlos nuevamente en una secuencia cronológica.

Ahora, añadiremos el historial de chat a la plantilla de la sala de chat. Edita la plantilla `chat/room.html` y añade las siguientes líneas:

```html
{% block content %}
    <div id="chat">
        {% for message in latest_messages %}
            <div class="message {% if message.user == request.user %}me{% else %}other{% endif %}">
                <strong>{{ message.user.username }}</strong>
                <span class="date">
                    {{ message.sent_on|date:"Y.m.d H:i A" }}
                </span>
                <br>
                {{ message.content }}
            </div>
        {% endfor %}
    </div>
    <div id="chat-input">
        <input id="chat-message-input" type="text">
        <input id="chat-message-submit" type="submit" value="Send">
    </div>
{% endblock %}
```

Abre `http://127.0.0.1:8000/chat/room/1/` en tu navegador, reemplazando `1` con el `id` de un curso existente en la base de datos. Ahora deberías ver los últimos mensajes:

> *Figura 16.12: Sala de chat mostrando inicialmente los últimos mensajes*

Los usuarios ahora pueden ver los últimos mensajes en el historial de chat al unirse a una sala de chat. A continuación, vamos a añadir un enlace al menú para que los usuarios puedan ingresar a la sala de chat del curso.

---

### Integración de la aplicación de chat con las vistas existentes

El servidor de chat ahora está completamente implementado y los estudiantes inscritos en un curso pueden comunicarse entre sí. Añadamos un enlace para que los estudiantes se unan a la sala de chat de cada curso.

Edita la plantilla `students/course/detail.html` de la aplicación `students` y añade el siguiente código de elemento HTML `<h3>` al final del elemento `<div class="contents">`:

```html
<div class="contents">
    ...
    <h3>
        <a href="{% url "chat:course_chat_room" object.id %}">
            Course chat room
        </a>
    </h3>
</div>
```

Abre el navegador y accede a cualquier curso en el que esté inscrito el estudiante para ver los contenidos del curso. La barra lateral ahora contendrá un enlace a la sala de chat del curso (*Course chat room*) que apunta a la vista de la sala de chat del curso. Si haces clic en él, entrarás en la sala de chat:

> *Figura 16.13: La página de detalles del curso, incluyendo un enlace a la sala de chat del curso*

¡Felicitaciones! Has creado con éxito tu primera aplicación asíncrona utilizando Django Channels.

---

### Resumen

En este capítulo, aprendiste a crear un servidor de chat usando Channels. Implementaste tanto un consumidor WebSocket como un cliente. Al habilitar la comunicación a través de una capa de canales con Redis y modificar el consumidor para que sea completamente asíncrono, mejoraste la capacidad de respuesta y la escalabilidad de tu aplicación. Además, implementaste la persistencia de mensajes de chat, lo que proporciona una experiencia sólida y fácil de usar, y mantiene el historial de chat para los usuarios a lo largo del tiempo. Las habilidades que aprendiste en este capítulo te ayudarán en cualquier implementación futura de funcionalidades asíncronas en tiempo real.

El próximo capítulo te enseñará cómo crear un entorno de producción para tu proyecto Django utilizando NGINX, uWSGI y Daphne con Docker Compose. También aprenderás a implementar middleware personalizado para el procesamiento de solicitudes/respuestas en toda tu aplicación y a desarrollar comandos de administración personalizados, que te permitirán automatizar tareas y ejecutarlas a través de la línea de comandos.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter16](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter16)
- **Introducción a ASGI:** [https://asgi.readthedocs.io/en/latest/introduction.html](https://asgi.readthedocs.io/en/latest/introduction.html)
- **Soporte de Django para vistas asíncronas:** [https://docs.djangoproject.com/en/5.2/topics/async/](https://docs.djangoproject.com/en/5.2/topics/async/)
- **Soporte de Django para vistas basadas en clases asíncronas:** [https://docs.djangoproject.com/en/5.2/topics/class-based-views/#async-class-based-views](https://docs.djangoproject.com/en/5.2/topics/class-based-views/#async-class-based-views)
- **Servidor ASGI Daphne:** [https://github.com/django/daphne](https://github.com/django/daphne)
- **Documentación de Django Channels:** [https://channels.readthedocs.io/](https://channels.readthedocs.io/)
- **Despliegue de Django con ASGI:** [https://docs.djangoproject.com/en/5.2/howto/deployment/asgi/](https://docs.djangoproject.com/en/5.2/howto/deployment/asgi/)
- **Introducción a WebSockets:** [https://en.wikipedia.org/wiki/WebSocket](https://en.wikipedia.org/wiki/WebSocket)
- **Uso del filtro de plantilla json_script:** [https://docs.djangoproject.com/en/5.2/ref/templates/builtins/#json-script](https://docs.djangoproject.com/en/5.2/ref/templates/builtins/#json-script)
- **Consultas asíncronas en el ORM de Django:** [https://docs.djangoproject.com/en/5.2/topics/db/queries/#asynchronous-queries](https://docs.djangoproject.com/en/5.2/topics/db/queries/#asynchronous-queries)
- **Documentación de nh3:** [https://nh3.readthedocs.io/en/latest/](https://nh3.readthedocs.io/en/latest/)
- **Proyecto django-nh3:** [https://github.com/marksweb/django-nh3](https://github.com/marksweb/django-nh3)

