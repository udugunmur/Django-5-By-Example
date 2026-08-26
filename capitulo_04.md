# Parte 2: Creación de un sitio web social

## Capítulo 4: Creación de un sitio web social

### Introducción

En el capítulo anterior, aprendiste a implementar un sistema de etiquetado y a recomendar publicaciones similares. Implementaste etiquetas y filtros de plantilla personalizados. También aprendiste a crear sitemaps y canales de sindicación (feeds) para tu sitio, y creaste un motor de búsqueda de texto completo utilizando PostgreSQL.

En este capítulo, aprenderás a desarrollar funcionalidades de cuentas de usuario para crear un sitio web social, incluyendo el registro de usuarios, la gestión de contraseñas, la edición de perfiles y la autenticación. Implementaremos características sociales en este sitio en los próximos capítulos para permitir a los usuarios compartir imágenes e interactuar entre sí. Los usuarios podrán marcar como favorito cualquier imagen de Internet y compartirla con otros usuarios. También podrán ver la actividad en la plataforma de los usuarios a los que siguen y dar "me gusta" o "ya no me gusta" a las imágenes compartidas por ellos.

Este capítulo cubrirá los siguientes temas:

- Creación de una vista de inicio de sesión
- Uso del framework de autenticación de Django
- Creación de plantillas para las vistas de inicio de sesión, cierre de sesión, cambio de contraseña y restablecimiento de contraseña de Django
- Creación de vistas de registro de usuarios
- Extensión del modelo de usuario con un modelo de perfil personalizado
- Configuración del proyecto para la subida de archivos multimedia

---

### Visión general funcional

La Figura 4.1 muestra una representación de las vistas, plantillas y funcionalidades que se construirán en este capítulo:

> *Figura 4.1: Diagrama de funcionalidades construidas en el Capítulo 4*

En este capítulo, crearás un nuevo proyecto y utilizarás las vistas de inicio de sesión, cierre de sesión, cambio de contraseña y recuperación de contraseña proporcionadas por Django en el paquete `django.contrib.auth`. Crearás plantillas para las vistas de autenticación y crearás una vista de panel de control (*dashboard*) a la que los usuarios tendrán acceso cuando se autentiquen correctamente. Implementarás el registro de usuarios con la vista `register`. Finalmente, extenderás el modelo de usuario con un modelo `Profile` personalizado y crearás la vista `edit` para permitir a los usuarios editar su perfil.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter04](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter04).

Todos los paquetes de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente del capítulo. Puedes seguir las instrucciones para instalar cada paquete de Python en las siguientes secciones, o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Creación de un proyecto de sitio web social

Vamos a crear una aplicación social que permitirá a los usuarios compartir imágenes que encuentren en Internet. Este proyecto es relevante porque te ayudará a comprender cómo incorporar capacidades sociales a tu sitio, así como cómo implementar funcionalidades avanzadas con Django y JavaScript.

Para nuestro sitio web de intercambio de imágenes, necesitaremos construir los siguientes elementos:

- Un sistema de autenticación para que los usuarios se registren, inicien sesión, editen su perfil y cambien o restablezcan su contraseña
- Autenticación social para iniciar sesión con servicios como Google
- Funcionalidad para mostrar imágenes compartidas y un sistema para que los usuarios compartan imágenes desde cualquier sitio web
- Un flujo de actividad (*activity stream*) que permita a los usuarios ver el contenido subido por las personas a las que siguen
- Un sistema de seguimiento para permitir a los usuarios seguirse unos a otros en el sitio web

Este capítulo abordará el primer punto de la lista. El resto de los puntos se cubrirán en los Capítulos 5 a 7.

#### Inicio del proyecto de sitio web social

Comenzaremos configurando el entorno virtual para el proyecto y creando la estructura inicial del proyecto.

Abre la terminal y utiliza los siguientes comandos para crear un entorno virtual para tu proyecto:

```bash
mkdir env
python -m venv env/bookmarks
```

Si estás utilizando Linux o macOS, ejecuta el siguiente comando para activar tu entorno virtual:

```bash
source env/bookmarks/bin/activate
```

Si estás utilizando Windows, utiliza el siguiente comando en su lugar:

```bash
.\env\bookmarks\Scripts\activate
```

El indicador de la consola mostrará tu entorno virtual activo, de la siguiente manera:

```text
(bookmarks)laptop:~ zenx$
```

Instala Django en tu entorno virtual con el siguiente comando:

```bash
python -m pip install Django~=5.2.0
```

Ejecuta el siguiente comando para crear un nuevo proyecto:

```bash
django-admin startproject bookmarks
```

La estructura inicial del proyecto ha sido creada. Utiliza los siguientes comandos para ingresar a tu directorio de proyecto y crear una nueva aplicación llamada `account`:

```bash
cd bookmarks/
django-admin startapp account
```

Recuerda que debes agregar la nueva aplicación a tu proyecto añadiendo el nombre de la aplicación a la configuración `INSTALLED_APPS` en el archivo `settings.py`.

Edita `settings.py` y añade la siguiente línea a la lista `INSTALLED_APPS` antes de cualquiera de las otras aplicaciones instaladas:

```python
INSTALLED_APPS = [
    'account.apps.AccountConfig',
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
```

Django busca plantillas en los directorios de plantillas de las aplicaciones por orden de aparición en la configuración `INSTALLED_APPS`. La aplicación `django.contrib.admin` incluye plantillas de autenticación estándar, que sobrescribiremos en la aplicación `account`. Habitualmente, ubicamos nuestras propias aplicaciones al final de la lista. En este caso, colocamos la aplicación en primer lugar en la configuración `INSTALLED_APPS` para garantizar que se utilicen nuestras plantillas de autenticación personalizadas, en lugar de las plantillas de autenticación contenidas en `django.contrib.admin`.

Ejecuta el siguiente comando para sincronizar la base de datos con los modelos de las aplicaciones predeterminadas incluidas en la configuración `INSTALLED_APPS`:

```bash
python manage.py migrate
```

Verás que se aplican todas las migraciones iniciales de la base de datos de Django. Se han creado las tablas de base de datos correspondientes a los modelos de Django de las aplicaciones instaladas. A continuación, integraremos un sistema de autenticación en nuestro proyecto utilizando el framework de autenticación de Django.

---

### Uso del framework de autenticación de Django

Django viene con un framework de autenticación integrado que puede manejar la autenticación de usuarios, sesiones, permisos y grupos de usuarios. El sistema de autenticación incluye vistas para acciones comunes de los usuarios, como iniciar sesión, cerrar sesión, cambiar la contraseña y restablecer la contraseña.

El framework de autenticación se encuentra en `django.contrib.auth` y es utilizado por otros paquetes contrib de Django. Recuerda que ya utilizamos el framework de autenticación en el Capítulo 1, *Creación de una aplicación de blog*, para crear un superusuario para que la aplicación de blog acceda al sitio de administración.

Cuando creamos un nuevo proyecto Django usando el comando `startproject`, el framework de autenticación se incluye en la configuración predeterminada de nuestro proyecto. Consiste en la aplicación `django.contrib.auth` y las siguientes dos clases de middleware que se encuentran en la configuración `MIDDLEWARE` de nuestro proyecto:

- `AuthenticationMiddleware`: Asocia usuarios con peticiones mediante sesiones
- `SessionMiddleware`: Gestiona la sesión actual a través de las peticiones

Los middleware son clases con métodos que se ejecutan globalmente durante la fase de petición o respuesta. Utilizarás clases de middleware en varias ocasiones a lo largo de este libro y aprenderás a crear middleware personalizado en el Capítulo 17, *Puesta en producción*.

El framework de autenticación también incluye los siguientes modelos que están definidos en `django.contrib.auth.models`:

- `User`: Un modelo de usuario con campos básicos; los campos principales de este modelo son `username`, `password`, `email`, `first_name`, `last_name` e `is_active`
- `Group`: Un modelo de grupo para categorizar usuarios
- `Permission`: Indicadores (*flags*) para que usuarios o grupos realicen ciertas acciones

El framework también incluye vistas y formularios de autenticación predeterminados, que utilizarás más adelante.

#### Creación de una vista de inicio de sesión

Comenzaremos esta sección utilizando el framework de autenticación de Django para permitir a los usuarios iniciar sesión en el sitio web. Crearemos una vista que realizará las siguientes acciones para iniciar la sesión de un usuario:

1. Presentar al usuario un formulario de inicio de sesión
2. Obtener el nombre de usuario y la contraseña proporcionados por el usuario cuando envía el formulario
3. Autenticar al usuario contra los datos almacenados en la base de datos
4. Comprobar si el usuario está activo
5. Iniciar la sesión del usuario en el sitio web y comenzar una sesión autenticada

Comenzaremos creando el formulario de inicio de sesión.

Crea un nuevo archivo `forms.py` en el directorio de la aplicación `account` y añade las siguientes líneas:

```python
from django import forms


class LoginForm(forms.Form):
    username = forms.CharField()
    password = forms.CharField(widget=forms.PasswordInput)
```

> [!NOTE]
> Como novedad en Django 5.2, puedes agregar estilos fácilmente a tus formularios con la sobrescritura simplificada del `BoundField` de un formulario. Consulta el Apéndice para más detalles, al que puedes acceder aquí: [https://packt.link/1g7Af](https://packt.link/1g7Af).

Este formulario se utilizará para autenticar usuarios contra la base de datos. El widget `PasswordInput` se utiliza para renderizar el elemento HTML de la contraseña. Esto incluirá `type="password"` en el HTML para que el navegador lo trate como una entrada de contraseña.

Edita el archivo `views.py` de la aplicación `account` y añade el siguiente código:

```python
from django.contrib.auth import authenticate, login
from django.http import HttpResponse
from django.shortcuts import render
from .forms import LoginForm


def user_login(request):
    if request.method == 'POST':
        form = LoginForm(request.POST)
        if form.is_valid():
            cd = form.cleaned_data
            user = authenticate(
                request,
                username=cd['username'],
                password=cd['password']
            )
            if user is not None:
                if user.is_active:
                    login(request, user)
                    return HttpResponse('Authenticated successfully')
                else:
                    return HttpResponse('Disabled account')
            else:
                return HttpResponse('Invalid login')
    else:
        form = LoginForm()
    return render(request, 'account/login.html', {'form': form})
```

Esto es lo que hace la vista básica de inicio de sesión:

1. Cuando la vista `user_login` se llama con una petición GET, se instancia un nuevo formulario de inicio de sesión con `form = LoginForm()`. Luego, el formulario se pasa a la plantilla.
2. Cuando el usuario envía el formulario a través de POST, se realizan las siguientes acciones:
   - El formulario se instancia con los datos enviados mediante `form = LoginForm(request.POST)`.
   - El formulario se valida con `form.is_valid()`. Si no es válido, los errores del formulario se mostrarán más adelante en la plantilla (por ejemplo, si el usuario no completó uno de los campos).
   - Si los datos enviados son válidos, el usuario se autentica contra la base de datos mediante el método `authenticate()`. Este método toma el objeto `request`, el `username` y los parámetros `password`, y devuelve el objeto `User` si el usuario ha sido autenticado con éxito, o `None` en caso contrario. Si el usuario no ha sido autenticado con éxito, se devuelve una respuesta `HttpResponse` con un mensaje de `Invalid login`.
   - Si el usuario se autentica correctamente, el estado del usuario se verifica accediendo al atributo `is_active`. Este es un atributo del modelo de usuario de Django. Si el usuario no está activo, se devuelve un `HttpResponse` con un mensaje de `Disabled account`.
   - Si el usuario está activo, se inicia la sesión del usuario en el sitio. El usuario se establece en la sesión llamando al método `login()`. Se devuelve un mensaje de `Authenticated successfully`.

> [!NOTE]
> Observa la diferencia entre `authenticate()` y `login()`: `authenticate()` verifica las credenciales del usuario y, tras la validación, devuelve un objeto `User` que representa al usuario autenticado. En cambio, `login()` establece al usuario en la sesión actual incorporando el objeto `User` autenticado en el contexto de la sesión actual.

Ahora, crearemos un patrón de URL para esta vista:

Crea un nuevo archivo `urls.py` en el directorio de la aplicación `account` y añade el siguiente código:

```python
from django.urls import path
from . import views

urlpatterns = [
    path('login/', views.user_login, name='login'),
]
```

Edita el archivo `urls.py` principal ubicado en el directorio de tu proyecto `bookmarks`, importa `include` y añade los patrones de URL de la aplicación `account`, de la siguiente manera:

```python
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('admin/', admin.site.urls),
    path('account/', include('account.urls')),
]
```

La vista de inicio de sesión ahora es accesible mediante una URL.

Creemos una plantilla para esta vista. Dado que aún no hay plantillas en el proyecto, comenzaremos creando una plantilla base que será extendida por la plantilla de inicio de sesión:

Crea los siguientes archivos y directorios dentro del directorio de la aplicación `account`:

```text
templates/
    account/
        login.html
    base.html
```

Edita la plantilla `base.html` y añade el siguiente código:

```html
{% load static %}
<!DOCTYPE html>
<html>
<head>
    <title>{% block title %}{% endblock %}</title>
    <link href="{% static "css/base.css" %}" rel="stylesheet">
</head>
<body>
    <div id="header">
        <span class="logo">Bookmarks</span>
    </div>
    <div id="content">
        {% block content %}
        {% endblock %}
    </div>
</body>
</html>
```

Esta será la plantilla base para el sitio web. Como hiciste en tu proyecto anterior, incluye los estilos CSS en la plantilla principal. Puedes encontrar estos archivos estáticos en el código que acompaña a este capítulo. Copia el directorio `static/` de la aplicación `account` del código fuente del capítulo a la misma ubicación en tu proyecto para poder usar los archivos estáticos. Puedes encontrar el contenido del directorio en [https://github.com/PacktPublishing/Django-5-by-Example/tree/master/Chapter04/bookmarks/account/static](https://github.com/PacktPublishing/Django-5-by-Example/tree/master/Chapter04/bookmarks/account/static).

La plantilla base define un bloque `title` y un bloque `content` que se pueden llenar con contenido mediante las plantillas que se extienden a partir de ella.

Rellenemos la plantilla para tu formulario de inicio de sesión.

Abre la plantilla `account/login.html` y añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Log-in{% endblock %}

{% block content %}
    <h1>Log-in</h1>
    <p>Please, use the following form to log-in:</p>
    <form method="post">
        {{ form.as_p }}
        {% csrf_token %}
        <p><input type="submit" value="Log in"></p>
    </form>
{% endblock %}
```

Esta plantilla incluye el formulario que se instancia en la vista. Dado que tu formulario se enviará a través de POST, incluirás la etiqueta de plantilla `{% csrf_token %}` para la protección contra la falsificación de peticiones en sitios cruzados (CSRF). Aprendiste sobre la protección CSRF en el Capítulo 2, *Mejora de tu blog y adición de funciones sociales*.

Todavía no hay usuarios en la base de datos. Primero deberás crear un superusuario para acceder al sitio de administración y gestionar otros usuarios.

Ejecuta el siguiente comando en la consola del sistema:

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

Ejecuta el servidor de desarrollo mediante el siguiente comando:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/admin/` en tu navegador. Accede al sitio de administración utilizando las credenciales del usuario que acabas de crear. Verás el sitio de administración de Django, incluyendo los modelos `User` y `Group` del framework de autenticación de Django.

Se verá de la siguiente manera:

> *Figura 4.2: La página de índice del sitio de administración de Django incluyendo Usuarios y Grupos*

En la fila **Users**, haz clic en el enlace **Add**.

Crea un nuevo usuario usando el sitio de administración de la siguiente manera:

> *Figura 4.3: El formulario Add user en el sitio de administración de Django*

Introduce los detalles del usuario y haz clic en el botón **SAVE** para guardar el nuevo usuario en la base de datos.

Luego, en **Personal info**, completa los campos **First name**, **Last name** y **Email address** como sigue, y luego haz clic en el botón **SAVE** para guardar los cambios:

> *Figura 4.4: El formulario de edición de usuario en el sitio de administración de Django*

Abre `http://127.0.0.1:8000/account/login/` en tu navegador. Deberías ver la plantilla renderizada, incluyendo el formulario de inicio de sesión:

> *Figura 4.5: La página de inicio de sesión de usuario*

Introduce credenciales no válidas y envía el formulario. Deberías obtener la siguiente respuesta `Invalid login`:

> *Figura 4.6: La respuesta en texto plano de inicio de sesión no válido*

Introduce credenciales válidas; obtendrás la siguiente respuesta `Authenticated successfully`:

> *Figura 4.7: La respuesta en texto plano de autenticación exitosa*

Ahora has aprendido a autenticar usuarios y crear tu propia vista de autenticación. Puedes construir tus propias vistas de autenticación, pero Django incluye vistas de autenticación listas para usar que puedes aprovechar.

#### Uso de las vistas de autenticación integradas de Django

Django incluye varios formularios y vistas en el framework de autenticación que puedes usar de inmediato. La vista de inicio de sesión que hemos creado es un buen ejercicio para comprender el proceso de autenticación de usuarios en Django. Sin embargo, puedes usar las vistas de autenticación predeterminadas de Django en la mayoría de los casos.

Django proporciona las siguientes vistas basadas en clases para gestionar la autenticación. Todas ellas se encuentran en `django.contrib.auth.views`:

- `LoginView`: Gestiona un formulario de inicio de sesión e inicia la sesión de un usuario
- `LogoutView`: Cierra la sesión de un usuario

Django proporciona las siguientes vistas para gestionar los cambios de contraseña:

- `PasswordChangeView`: Gestiona un formulario para cambiar la contraseña del usuario
- `PasswordChangeDoneView`: La vista de éxito a la que se redirige al usuario después de un cambio de contraseña exitoso

Django también incluye las siguientes vistas para permitir a los usuarios restablecer su contraseña:

- `PasswordResetView`: Permite a los usuarios restablecer su contraseña. Genera un enlace de un solo uso con un token y lo envía a la cuenta de correo electrónico del usuario
- `PasswordResetDoneView`: Informa a los usuarios de que se les ha enviado un correo electrónico con un enlace para restablecer su contraseña
- `PasswordResetConfirmView`: Permite a los usuarios establecer una nueva contraseña
- `PasswordResetCompleteView`: La vista de éxito a la que se redirige al usuario después de restablecer con éxito su contraseña

Estas vistas pueden ahorrarte mucho tiempo al crear cualquier aplicación web con cuentas de usuario. Las vistas utilizan valores predeterminados que se pueden sobrescribir, como la ubicación de la plantilla que se va a renderizar o el formulario que utilizará la vista.

Puedes obtener más información sobre las vistas de autenticación integradas en [https://docs.djangoproject.com/en/5.2/topics/auth/default/#all-authentication-views](https://docs.djangoproject.com/en/5.2/topics/auth/default/#all-authentication-views).

#### Vistas de inicio y cierre de sesión

Para aprender a usar las vistas de autenticación de Django, sustituiremos nuestra vista de inicio de sesión personalizada por el equivalente integrado de Django y también integraremos una vista de cierre de sesión.

Edita el archivo `urls.py` de la aplicación `account` y añade el siguiente código:

```python
from django.contrib.auth import views as auth_views
from django.urls import path
from . import views

urlpatterns = [
    # previous login url
    # path('login/', views.user_login, name='login'),
    # login / logout urls
    path('login/', auth_views.LoginView.as_view(), name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
]
```

En el código anterior, hemos comentado el patrón de URL para la vista `user_login` que creamos previamente. Ahora utilizaremos la vista `LoginView` del framework de autenticación de Django. También hemos añadido un patrón de URL para la vista `LogoutView`.

Crea un nuevo directorio dentro del directorio `templates/` de la aplicación `account` y nómbralo `registration`. Esta es la ruta predeterminada donde las vistas de autenticación de Django esperan que estén tus plantillas de autenticación.

El módulo `django.contrib.admin` incluye plantillas de autenticación que se utilizan para el sitio de administración, como la plantilla de inicio de sesión. Al colocar la aplicación `account` en la parte superior de la configuración `INSTALLED_APPS` al configurar el proyecto, nos aseguramos de que Django utilizaría nuestras plantillas de autenticación en lugar de las definidas en cualquier otra aplicación.

Crea un nuevo archivo dentro del directorio `templates/registration/`, nómbralo `login.html` y añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Log-in{% endblock %}

{% block content %}
    <h1>Log-in</h1>
    {% if form.errors %}
        <p>
            Your username and password didn't match.
            Please try again.
        </p>
    {% else %}
        <p>Please, use the following form to log-in:</p>
    {% endif %}
    <div class="login-form">
        <form action="{% url 'login' %}" method="post">
            {{ form.as_p }}
            {% csrf_token %}
            <input type="hidden" name="next" value="{{ next }}" />
            <p><input type="submit" value="Log-in"></p>
        </form>
    </div>
{% endblock %}
```

Esta plantilla de inicio de sesión es bastante similar a la que creamos antes. Django utiliza el formulario `AuthenticationForm` ubicado en `django.contrib.auth.forms` de forma predeterminada. Este formulario intenta autenticar al usuario y genera un error de validación si el inicio de sesión no es exitoso. Usamos `{% if form.errors %}` en la plantilla para comprobar si las credenciales proporcionadas son incorrectas.

Hemos añadido un elemento `<input>` HTML oculto para enviar el valor de una variable llamada `next`. Esta variable se proporciona a la vista de inicio de sesión si pasas un parámetro llamado `next` a la petición, por ejemplo, accediendo a `http://127.0.0.1:8000/account/login/?next=/account/`.

El parámetro `next` tiene que ser una URL. Si se proporciona este parámetro, la vista de inicio de sesión de Django redirigirá al usuario a la URL dada después de un inicio de sesión exitoso.

Ahora, crea una plantilla `logged_out.html` dentro del directorio `templates/registration/` y haz que se vea así:

```html
{% extends "base.html" %}

{% block title %}Logged out{% endblock %}

{% block content %}
    <h1>Logged out</h1>
    <p>
        You have been successfully logged out.
        You can <a href="{% url "login" %}">log-in again</a>.
    </p>
{% endblock %}
```

Esta es la plantilla que Django mostrará después de que el usuario cierre sesión.

Hemos añadido los patrones de URL y las plantillas para las vistas de inicio y cierre de sesión. Los usuarios ahora pueden iniciar y cerrar sesión utilizando las vistas de autenticación de Django.

Ahora, crearemos una nueva vista para mostrar un panel de control (*dashboard*) cuando los usuarios inicien sesión en sus cuentas.

Edita el archivo `views.py` de la aplicación `account` y añade el siguiente código:

```python
from django.contrib.auth.decorators import login_required


@login_required
def dashboard(request):
    return render(
        request,
        'account/dashboard.html',
        {'section': 'dashboard'}
    )
```

Hemos creado la vista `dashboard` y le hemos aplicado el decorador `login_required` del framework de autenticación. El decorador `login_required` comprueba si el usuario actual está autenticado.

Si el usuario está autenticado, ejecuta la vista decorada; si el usuario no está autenticado, redirige al usuario a la URL de inicio de sesión, con la URL solicitada originalmente como un parámetro GET llamado `next`.

Al hacer esto, la vista de inicio de sesión redirige a los usuarios a la URL a la que intentaban acceder después de que hayan iniciado sesión correctamente. Recuerda que agregamos un elemento HTML `<input>` oculto llamado `next` en la plantilla de inicio de sesión para este propósito.

También hemos definido una variable `section`. Usaremos esta variable para resaltar la sección actual en el menú principal del sitio.

A continuación, necesitamos crear una plantilla para la vista del panel de control.

Crea un nuevo archivo dentro del directorio `templates/account/` y nómbralo `dashboard.html`. Añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Dashboard{% endblock %}

{% block content %}
    <h1>Dashboard</h1>
    <p>Welcome to your dashboard.</p>
{% endblock %}
```

Edita el archivo `urls.py` de la aplicación `account` y añade el siguiente patrón de URL para la vista:

```python
urlpatterns = [
    # previous login url
    # path('login/', views.user_login, name='login'),
    # login / logout urls
    path('login/', auth_views.LoginView.as_view(), name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
    path('', views.dashboard, name='dashboard'),
]
```

Edita el archivo `settings.py` del proyecto y añade el siguiente código:

```python
LOGIN_REDIRECT_URL = 'dashboard'
LOGIN_URL = 'login'
LOGOUT_URL = 'logout'
```

Hemos definido las siguientes configuraciones:

- `LOGIN_REDIRECT_URL`: Indica a Django a qué URL redirigir al usuario después de un inicio de sesión exitoso si no hay ningún parámetro `next` presente en la petición
- `LOGIN_URL`: La URL a la que redirigir al usuario para iniciar sesión (por ejemplo, vistas que utilizan el decorador `login_required`)
- `LOGOUT_URL`: La URL a la que redirigir al usuario para cerrar sesión

Hemos utilizado los nombres de las URLs que definimos previamente con el atributo `name` de la función `path()` en los patrones de URL. También se pueden utilizar URLs codificadas de forma fija en lugar de nombres de URL para estas configuraciones.

Resumamos lo que hemos hecho hasta ahora:

1. Hemos añadido las vistas de inicio y cierre de sesión de autenticación integradas de Django al proyecto.
2. Hemos creado plantillas personalizadas para ambas vistas y hemos definido una vista de panel de control simple para redirigir a los usuarios después de que inicien sesión.
3. Finalmente, hemos añadido configuraciones para que Django utilice estas URLs de forma predeterminada.

Ahora, añadiremos un enlace a la URL de inicio de sesión y un botón para cerrar sesión en la plantilla base. Para hacer esto, debemos determinar si el usuario actual ha iniciado sesión o no para mostrar la acción adecuada en cada caso. El usuario actual se establece en el objeto `HttpRequest` mediante el middleware de autenticación. Puedes acceder a él con `request.user`. El objeto `request` contiene un objeto `User` incluso si el usuario no está autenticado. Un usuario no autenticado se establece en la petición como una instancia de `AnonymousUser`. La mejor manera de comprobar si el usuario actual está autenticado es accediendo al atributo de solo lectura `is_authenticated`.

Edita la plantilla `templates/base.html` añadiendo las siguientes líneas:

```html
{% load static %}
<!DOCTYPE html>
<html>
<head>
    <title>{% block title %}{% endblock %}</title>
    <link href="{% static "css/base.css" %}" rel="stylesheet">
</head>
<body>
    <div id="header">
        <span class="logo">Bookmarks</span>
        {% if request.user.is_authenticated %}
            <ul class="menu">
                <li {% if section == "dashboard" %}class="selected"{% endif %}>
                    <a href="{% url "dashboard" %}">My dashboard</a>
                </li>
                <li {% if section == "images" %}class="selected"{% endif %}>
                    <a href="#">Images</a>
                </li>
                <li {% if section == "people" %}class="selected"{% endif %}>
                    <a href="#">People</a>
                </li>
            </ul>
        {% endif %}
        <span class="user">
            {% if request.user.is_authenticated %}
                Hello {{ request.user.first_name|default:request.user.username }},
                <form action="{% url "logout" %}" method="post">
                    <button type="submit">Logout</button>
                    {% csrf_token %}
                </form>
            {% else %}
                <a href="{% url "login" %}">Log-in</a>
            {% endif %}
        </span>
    </div>
    <div id="content">
        {% block content %}
        {% endblock %}
    </div>
</body>
</html>
```

El menú del sitio solo se muestra a los usuarios autenticados. Se comprueba la variable `section` para añadir un atributo de clase `selected` al elemento de lista `<li>` del menú de la sección actual. Al hacerlo, el elemento de menú que corresponde a la sección actual se resaltará mediante CSS. El nombre del usuario y un botón para cerrar sesión se muestran si el usuario está autenticado; en caso contrario, se muestra un enlace para iniciar sesión. Si el nombre del usuario está vacío, se muestra el nombre de usuario en su lugar mediante `request.user.first_name|default:request.user.username`. Ten en cuenta que para la acción de cierre de sesión, utilizamos un formulario con el método POST y un botón para enviar el formulario. Esto se debe a que `LogoutView` requiere peticiones POST.

Abre `http://127.0.0.1:8000/account/login/` en tu navegador. Deberías ver la página de inicio de sesión. Introduce un nombre de usuario y contraseña válidos y haz clic en el botón **Log-in**. Deberías ver la siguiente pantalla:

> *Figura 4.8: La página Dashboard*

El elemento de menú **My dashboard** está resaltado con CSS porque tiene la clase `selected`. Dado que el usuario está autenticado, el nombre del usuario se muestra en el lado derecho del encabezado. Haz clic en el botón **Logout**. Deberías ver la siguiente página:

> *Figura 4.9: La página Logged out*

En esta página, puedes ver que el usuario ha cerrado sesión y, por lo tanto, no se muestra el menú del sitio web. El enlace que se muestra en el lado derecho del encabezado ahora es **Log-in**.

> [!TIP]
> Si ves la página de cierre de sesión del sitio de administración de Django en lugar de tu propia página de cierre de sesión, comprueba la configuración `INSTALLED_APPS` de tu proyecto y asegúrate de que `django.contrib.admin` aparezca después de la aplicación `account`. Ambas aplicaciones contienen plantillas de cierre de sesión ubicadas en la misma ruta relativa. El cargador de plantillas de Django revisará las diferentes aplicaciones en la lista `INSTALLED_APPS` y utilizará la primera plantilla que encuentre.

#### Vistas de cambio de contraseña

Necesitamos que los usuarios puedan cambiar su contraseña después de iniciar sesión en el sitio. Integraremos las vistas de autenticación de Django para cambiar contraseñas.

Abre el archivo `urls.py` de la aplicación `account` y añade los siguientes patrones de URL:

```python
urlpatterns = [
    # previous login url
    # path('login/', views.user_login, name='login'),
    # login / logout urls
    path('login/', auth_views.LoginView.as_view(), name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
    # change password urls
    path(
        'password-change/',
        auth_views.PasswordChangeView.as_view(),
        name='password_change'
    ),
    path(
        'password-change/done/',
        auth_views.PasswordChangeDoneView.as_view(),
        name='password_change_done'
    ),
    path('', views.dashboard, name='dashboard'),
]
```

La vista `PasswordChangeView` gestionará el formulario para cambiar la contraseña y la vista `PasswordChangeDoneView` mostrará un mensaje de éxito después de que el usuario haya cambiado su contraseña correctamente. Creemos una plantilla para cada vista.

Añade un nuevo archivo dentro del directorio `templates/registration/` de la aplicación `account` y nómbralo `password_change_form.html`. Añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Change your password{% endblock %}

{% block content %}
    <h1>Change your password</h1>
    <p>Use the form below to change your password.</p>
    <form method="post">
        {{ form.as_p }}
        <p><input type="submit" value="Change"></p>
        {% csrf_token %}
    </form>
{% endblock %}
```

La plantilla `password_change_form.html` incluye el formulario para cambiar la contraseña.

Ahora, crea otro archivo en el mismo directorio y nómbralo `password_change_done.html`. Añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Password changed{% endblock %}

{% block content %}
    <h1>Password changed</h1>
    <p>Your password has been successfully changed.</p>
{% endblock %}
```

La plantilla `password_change_done.html` solo contiene el mensaje de éxito que se mostrará cuando el usuario haya cambiado su contraseña correctamente.

Abre `http://127.0.0.1:8000/account/password-change/` en tu navegador. Si no has iniciado sesión, el navegador te redirigirá a la página de inicio de sesión. Después de autenticarte con éxito, verás la siguiente página de cambio de contraseña:

> *Figura 4.10: El formulario de cambio de contraseña*

Completa el formulario con tu contraseña actual y tu nueva contraseña, y luego haz clic en el botón **Change**. Verás la siguiente página de éxito:

> *Figura 4.11: La página de cambio de contraseña exitoso*

Cierra sesión e inicia sesión nuevamente usando tu nueva contraseña para verificar que todo funcione como se esperaba.

#### Vistas de restablecimiento de contraseña

Si los usuarios olvidan su contraseña, deberían poder recuperar su cuenta. Implementaremos una función de restablecimiento de contraseña. Esto permitirá a los usuarios recuperar el acceso a su cuenta al recibir un correo electrónico de restablecimiento de contraseña que contiene un enlace seguro, generado con un token único, que les permite crear una nueva contraseña.

Edita el archivo `urls.py` de la aplicación `account` y añade los siguientes patrones de URL:

```python
urlpatterns = [
    # previous login url
    # path('login/', views.user_login, name='login'),
    # login / logout urls
    path('login/', auth_views.LoginView.as_view(), name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
    # change password urls
    path(
        'password-change/',
        auth_views.PasswordChangeView.as_view(),
        name='password_change'
    ),
    path(
        'password-change/done/',
        auth_views.PasswordChangeDoneView.as_view(),
        name='password_change_done'
    ),
    # reset password urls
    path(
        'password-reset/',
        auth_views.PasswordResetView.as_view(),
        name='password_reset'
    ),
    path(
        'password-reset/done/',
        auth_views.PasswordResetDoneView.as_view(),
        name='password_reset_done'
    ),
    path(
        'password-reset/<uidb64>/<token>/',
        auth_views.PasswordResetConfirmView.as_view(),
        name='password_reset_confirm'
    ),
    path(
        'password-reset/complete/',
        auth_views.PasswordResetCompleteView.as_view(),
        name='password_reset_complete'
    ),
    path('', views.dashboard, name='dashboard'),
]
```

Añade un nuevo archivo al directorio `templates/registration/` de la aplicación `account` y nómbralo `password_reset_form.html`. Añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Reset your password{% endblock %}

{% block content %}
    <h1>Forgotten your password?</h1>
    <p>Enter your e-mail address to obtain a new password.</p>
    <form method="post">
        {{ form.as_p }}
        <p><input type="submit" value="Send e-mail"></p>
        {% csrf_token %}
    </form>
{% endblock %}
```

Ahora, crea otro archivo en el mismo directorio y nómbralo `password_reset_email.html`. Añade el siguiente código:

```html
Someone asked for password reset for email {{ email }}. Follow the link below:
{{ protocol }}://{{ domain }}{% url "password_reset_confirm" uidb64=uid token=token %}
Your username, in case you've forgotten: {{ user.get_username }}
```

La plantilla `password_reset_email.html` se utilizará para renderizar el correo electrónico enviado a los usuarios para restablecer su contraseña. Incluye un token de restablecimiento que genera la vista dinámicamente.

Crea otro archivo en el mismo directorio y nómbralo `password_reset_done.html`. Añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Reset your password{% endblock %}

{% block content %}
    <h1>Reset your password</h1>
    <p>We've emailed you instructions for setting your password.</p>
    <p>If you don't receive an email, please make sure you've entered the address you registered with.</p>
{% endblock %}
```

Crea otra plantilla en el mismo directorio y nómbrala `password_reset_confirm.html`. Añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Reset your password{% endblock %}

{% block content %}
    <h1>Reset your password</h1>
    {% if validlink %}
        <p>Please enter your new password twice:</p>
        <form method="post">
            {{ form.as_p }}
            {% csrf_token %}
            <p><input type="submit" value="Change my password" /></p>
        </form>
    {% else %}
        <p>The password reset link was invalid, possibly because it has already been used. Please request a new password reset.</p>
    {% endif %}
{% endblock %}
```

En esta plantilla, confirmamos si el enlace para restablecer la contraseña es válido comprobando la variable `validlink`. La vista `PasswordResetConfirmView` comprueba la validez del token proporcionado en la URL y pasa la variable `validlink` a la plantilla. Si el enlace es válido, se muestra el formulario de restablecimiento de contraseña del usuario. Los usuarios solo pueden establecer una nueva contraseña si tienen un enlace de restablecimiento de contraseña válido.

Crea otra plantilla y nómbrala `password_reset_complete.html`. Introduce el siguiente código en ella:

```html
{% extends "base.html" %}

{% block title %}Password reset{% endblock %}

{% block content %}
    <h1>Password set</h1>
    <p>Your password has been set. You can <a href="{% url "login" %}">log in now</a></p>
{% endblock %}
```

Finalmente, edita la plantilla `registration/login.html` de la aplicación `account` y añade las siguientes líneas:

```html
{% extends "base.html" %}

{% block title %}Log-in{% endblock %}

{% block content %}
    <h1>Log-in</h1>
    {% if form.errors %}
        <p>
            Your username and password didn't match.
            Please try again.
        </p>
    {% else %}
        <p>Please, use the following form to log-in:</p>
    {% endif %}
    <div class="login-form">
        <form action="{% url 'login' %}" method="post">
            {{ form.as_p }}
            {% csrf_token %}
            <input type="hidden" name="next" value="{{ next }}" />
            <p><input type="submit" value="Log-in"></p>
        </form>
        <p>
            <a href="{% url "password_reset" %}">
                Forgotten your password?
            </a>
        </p>
    </div>
{% endblock %}
```

Ahora, abre `http://127.0.0.1:8000/account/login/` en tu navegador. La página de inicio de sesión ahora debería incluir un enlace a la página de restablecimiento de contraseña, de la siguiente manera:

> *Figura 4.12: La página Log-in, incluyendo un enlace a la página de restablecimiento de contraseña*

Haz clic en el enlace **Forgotten your password?**. Deberías ver la siguiente página:

> *Figura 4.13: El formulario de recuperación de contraseña*

En este punto, necesitamos añadir una configuración de Protocolo Simple de Transferencia de Correo (SMTP) al archivo `settings.py` de tu proyecto para que Django pueda enviar correos electrónicos. Aprendiste a añadir configuraciones de correo electrónico a tu proyecto en el Capítulo 2, *Mejora de tu blog y adición de funciones sociales*. Sin embargo, durante el desarrollo, puedes configurar Django para que escriba correos electrónicos en la salida estándar en lugar de enviarlos a través de un servidor SMTP. Django proporciona un backend de correo electrónico para escribir correos electrónicos en la consola.

Edita el archivo `settings.py` de tu proyecto y añádele la siguiente línea:

```python
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

La configuración `EMAIL_BACKEND` indica la clase que se utilizará para enviar correos electrónicos.

Regresa a tu navegador, introduce la dirección de correo electrónico de un usuario existente y haz clic en el botón **SEND E-MAIL**. Deberías ver la siguiente página:

> *Figura 4.14: La página de correo electrónico de restablecimiento de contraseña enviado*

Echa un vistazo a la consola donde estás ejecutando el servidor de desarrollo. Verás el correo electrónico generado, de la siguiente manera:

```text
Content-Type: text/plain; charset="utf-8"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
Subject: Password reset on 127.0.0.1:8000
From: webmaster@localhost
To: test@gmail.com
Date: Mon, 10 Jan 2024 19:05:18 -0000
Message-ID: <162896791878.58862.14771487060402279558@MBP-amele.local>

Someone asked for password reset for email test@gmail.com. Follow the link below:
http://127.0.0.1:8000/account/password-reset/MQ/ardx0u-b4973cfa2c70d652a190e79054bc479a/
Your username, in case you've forgotten: test
```

El correo electrónico se renderiza utilizando la plantilla `password_reset_email.html` que creaste anteriormente. La URL para restablecer la contraseña incluye un token generado dinámicamente por Django.

Copia la URL del correo electrónico, que debería verse similar a `http://127.0.0.1:8000/account/password-reset/MQ/ardx0u-b4973cfa2c70d652a190e79054bc479a/`, y ábrela en tu navegador. Deberías ver la siguiente página:

> *Figura 4.15: El formulario de restablecimiento de contraseña*

La página para establecer una nueva contraseña utiliza la plantilla `password_reset_confirm.html`. Rellena una nueva contraseña y haz clic en el botón **CHANGE MY PASSWORD**.

Django creará una nueva contraseña con hash y la guardará en la base de datos. Verás la siguiente página de éxito:

> *Figura 4.16: La página de restablecimiento de contraseña exitoso*

Ahora, puedes volver a iniciar sesión en la cuenta de usuario utilizando la nueva contraseña.

Cada token para establecer una nueva contraseña solo se puede usar una vez. Si abres el enlace que recibiste nuevamente, recibirás un mensaje indicando que el token no es válido.

Ahora hemos integrado las vistas del framework de autenticación de Django en el proyecto. Estas vistas son adecuadas para la mayoría de los casos. Sin embargo, puedes crear tus propias vistas si necesitas un comportamiento diferente.

Django proporciona patrones de URL para las vistas de autenticación que son equivalentes a los que acabamos de crear. Reemplazaremos los patrones de URL de autenticación por los proporcionados por Django.

Comenta los patrones de URL de autenticación que añadiste al archivo `urls.py` de la aplicación `account` e incluye `django.contrib.auth.urls` en su lugar, de la siguiente manera:

```python
from django.urls import include, path
from django.contrib.auth import views as auth_views
from . import views

urlpatterns = [
    # previous login view
    # path('login/', views.user_login, name='login'),
    # path('login/', auth_views.LoginView.as_view(), name='login'),
    # path('logout/', auth_views.LogoutView.as_view(), name='logout'),
    # change password urls
    # path(
    #     'password-change/',
    #     auth_views.PasswordChangeView.as_view(),
    #     name='password_change'
    # ),
    # path(
    #     'password-change/done/',
    #     auth_views.PasswordChangeDoneView.as_view(),
    #     name='password_change_done'
    # ),
    # reset password urls
    # path(
    #     'password-reset/',
    #     auth_views.PasswordResetView.as_view(),
    #     name='password_reset'
    # ),
    # path(
    #     'password-reset/done/',
    #     auth_views.PasswordResetDoneView.as_view(),
    #     name='password_reset_done'
    # ),
    # path(
    #     'password-reset/<uidb64>/<token>/',
    #     auth_views.PasswordResetConfirmView.as_view(),
    #     name='password_reset_confirm'
    # ),
    # path(
    #     'password-reset/complete/',
    #     auth_views.PasswordResetCompleteView.as_view(),
    #     name='password_reset_complete'
    # ),
    path('', include('django.contrib.auth.urls')),
    path('', views.dashboard, name='dashboard'),
]
```

Puedes ver los patrones de URL de autenticación incluidos en [https://github.com/django/django/blob/stable/5.2.x/django/contrib/auth/urls.py](https://github.com/django/django/blob/stable/5.2.x/django/contrib/auth/urls.py).

Ahora hemos añadido todas las vistas de autenticación necesarias a nuestro proyecto. A continuación, implementaremos el registro de usuarios.

---

### Registro de usuarios y perfiles de usuario

Los usuarios del sitio ahora pueden iniciar sesión, cerrar sesión, cambiar su contraseña y restablecerla. Sin embargo, necesitamos crear una vista que permita a los visitantes crear una cuenta de usuario. Deben poder registrarse y crear un perfil en nuestro sitio. Una vez registrados, los usuarios podrán iniciar sesión en nuestro sitio utilizando sus credenciales.

#### Registro de usuarios

Creemos una vista simple para permitir el registro de usuarios en tu sitio web. Inicialmente, debes crear un formulario para permitir que el usuario introduzca un nombre de usuario, su nombre real y una contraseña.

Edita el archivo `forms.py` ubicado dentro del directorio de la aplicación `account` y añade las siguientes líneas:

```python
from django import forms
from django.contrib.auth import get_user_model


class LoginForm(forms.Form):
    username = forms.CharField()
    password = forms.CharField(widget=forms.PasswordInput)


class UserRegistrationForm(forms.ModelForm):
    password = forms.CharField(
        label='Password',
        widget=forms.PasswordInput
    )
    password2 = forms.CharField(
        label='Repeat password',
        widget=forms.PasswordInput
    )

    class Meta:
        model = get_user_model()
        fields = ['username', 'first_name', 'email']
```

Nuevamente, como novedad en Django 5.2, puedes agregar estilos fácilmente a tus formularios con la sobrescritura simplificada del `BoundField` de un formulario. Consulta el Apéndice para más detalles.

Hemos creado un formulario de modelo para el modelo de usuario. Este formulario incluye los campos `username`, `first_name` y `email` del modelo de usuario. Recuperamos el modelo de usuario dinámicamente mediante la función `get_user_model()` proporcionada por la aplicación `auth`. Esto recupera el modelo de usuario, que podría ser un modelo personalizado en lugar del modelo `User` de autenticación predeterminado, ya que Django te permite definir modelos de usuario personalizados. Estos campos se validarán de acuerdo con las validaciones de sus campos de modelo correspondientes. Por ejemplo, si el usuario elige un nombre de usuario que ya existe, obtendrá un error de validación porque `username` es un campo definido con `unique=True`.

> [!TIP]
> Para mantener tu código genérico, usa el método `get_user_model()` para recuperar el modelo de usuario y la configuración `AUTH_USER_MODEL` para hacer referencia a él al definir la relación de un modelo con él, en lugar de hacer referencia al modelo de usuario de `auth` directamente. Puedes leer más información sobre esto en [https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#django.contrib.auth.get_user_model](https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#django.contrib.auth.get_user_model).

También hemos añadido dos campos adicionales (`password` y `password2`) para que los usuarios establezcan una contraseña y la repitan. Añadamos la validación de campos para comprobar que ambas contraseñas sean iguales.

Edita el archivo `forms.py` en la aplicación `account` y añade el método `clean_password2()` a la clase `UserRegistrationForm`:

```python
class UserRegistrationForm(forms.ModelForm):
    password = forms.CharField(
        label='Password',
        widget=forms.PasswordInput
    )
    password2 = forms.CharField(
        label='Repeat password',
        widget=forms.PasswordInput
    )

    class Meta:
        model = get_user_model()
        fields = ['username', 'first_name', 'email']

    def clean_password2(self):
        cd = self.cleaned_data
        if cd['password'] != cd['password2']:
            raise forms.ValidationError("Passwords don't match.")
        return cd['password2']
```

Hemos definido un método `clean_password2()` para comparar la segunda contraseña con la primera y generar un error de validación si las contraseñas no coinciden. Este método se ejecuta cuando el formulario se valida llamando a su método `is_valid()`. Puedes proporcionar un método `clean_<fieldname>()` a cualquiera de los campos de tu formulario para limpiar el valor o generar errores de validación de formulario para un campo específico. Los formularios también incluyen un método general `clean()` para validar todo el formulario, lo cual es útil para validar campos que dependen unos de otros. En este caso, utilizamos la validación `clean_password2()` específica del campo en lugar de sobrescribir el método `clean()` del formulario. Esto evita sobrescribir otras comprobaciones específicas de campos que el `ModelForm` obtiene de las restricciones establecidas en el modelo (por ejemplo, validar que el nombre de usuario sea único).

Django también proporciona un formulario `UserCreationForm` que reside en `django.contrib.auth.forms` y es muy similar al que hemos creado.

Edita el archivo `views.py` de la aplicación `account` y añade el siguiente código:

```python
from django.contrib.auth import authenticate, login
from django.contrib.auth.decorators import login_required
from django.http import HttpResponse
from django.shortcuts import render
from .forms import LoginForm, UserRegistrationForm

# ...


def register(request):
    if request.method == 'POST':
        user_form = UserRegistrationForm(request.POST)
        if user_form.is_valid():
            # Create a new user object but avoid saving it yet
            new_user = user_form.save(commit=False)
            # Set the chosen password
            new_user.set_password(
                user_form.cleaned_data['password']
            )
            # Save the User object
            new_user.save()
            return render(
                request,
                'account/register_done.html',
                {'new_user': new_user}
            )
    else:
        user_form = UserRegistrationForm()
    return render(
        request,
        'account/register.html',
        {'user_form': user_form}
    )
```

La vista para crear cuentas de usuario es bastante simple. Por razones de seguridad, en lugar de guardar la contraseña en texto plano introducida por el usuario, utilizamos el método `set_password()` del modelo de usuario. Este método gestiona el hash de la contraseña antes de guardarla en la base de datos.

Django no almacena contraseñas en texto claro; en su lugar, almacena contraseñas con hash. El hashing es el proceso de transformar una clave dada en otro valor. Se utiliza una función hash para generar un valor de longitud fija según un algoritmo matemático. Al procesar las contraseñas con algoritmos seguros, Django garantiza que las contraseñas de los usuarios almacenadas en la base de datos requieran cantidades masivas de tiempo de cálculo para ser descifradas.

De forma predeterminada, Django utiliza el algoritmo de hashing PBKDF2 con un hash SHA256 para almacenar todas las contraseñas. Sin embargo, Django no solo admite la comprobación de contraseñas existentes con hash PBKDF2, sino que también admite la comprobación de contraseñas almacenadas con otros algoritmos, como PBKDF2SHA1, argon2, bcrypt y scrypt.

La configuración `PASSWORD_HASHERS` define los generadores de hash de contraseñas que admite el proyecto Django. La siguiente es la lista predeterminada de `PASSWORD_HASHERS`:

```python
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.PBKDF2PasswordHasher',
    'django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher',
    'django.contrib.auth.hashers.Argon2PasswordHasher',
    'django.contrib.auth.hashers.BCryptSHA256PasswordHasher',
    'django.contrib.auth.hashers.ScryptPasswordHasher',
]
```

Django utiliza la primera entrada de la lista (en este caso, `PBKDF2PasswordHasher`) para generar el hash de todas las contraseñas. El resto de los generadores de hash pueden ser utilizados por Django para comprobar contraseñas existentes.

El generador de hash `scrypt` se introdujo en Django 4.0. Es más seguro y se recomienda sobre PBKDF2. Sin embargo, PBKDF2 sigue siendo el generador de hash predeterminado, ya que scrypt requiere OpenSSL 1.1+ y más memoria.

Puedes obtener más información sobre cómo Django almacena contraseñas y sobre los generadores de hash de contraseñas incluidos en [https://docs.djangoproject.com/en/5.2/topics/auth/passwords/](https://docs.djangoproject.com/en/5.2/topics/auth/passwords/).

Ahora, edita el archivo `urls.py` de la aplicación `account` y añade el siguiente patrón de URL:

```python
urlpatterns = [
    # ...
    path('', include('django.contrib.auth.urls')),
    path('', views.dashboard, name='dashboard'),
    path('register/', views.register, name='register'),
]
```

Finalmente, crea una nueva plantilla en el directorio de plantillas `templates/account/` de la aplicación `account`, nómbrala `register.html` y haz que se vea de la siguiente manera:

```html
{% extends "base.html" %}

{% block title %}Create an account{% endblock %}

{% block content %}
    <h1>Create an account</h1>
    <p>Please, sign up using the following form:</p>
    <form method="post">
        {{ user_form.as_p }}
        {% csrf_token %}
        <p><input type="submit" value="Create my account"></p>
    </form>
{% endblock %}
```

Crea un archivo de plantilla adicional en el mismo directorio y nómbralo `register_done.html`. Añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Welcome{% endblock %}

{% block content %}
    <h1>Welcome {{ new_user.first_name }}!</h1>
    <p>
        Your account has been successfully created.
        Now you can <a href="{% url "login" %}">log in</a>.
    </p>
{% endblock %}
```

Abre `http://127.0.0.1:8000/account/register/` en tu navegador. Verás la página de registro que has creado:

> *Figura 4.17: El formulario de creación de cuenta*

Completa los detalles para un nuevo usuario y haz clic en el botón **CREATE MY ACCOUNT**.

Si todos los campos son válidos, el usuario se creará y verás el siguiente mensaje de éxito:

> *Figura 4.18: La página de cuenta creada exitosamente*

Haz clic en el enlace de inicio de sesión e introduce tu nombre de usuario y contraseña para verificar que puedes acceder a tu cuenta recién creada.

Añadamos un enlace para registrarse en la plantilla de inicio de sesión. Edita la plantilla `registration/login.html` y busca la siguiente línea:

```html
<p>Please, use the following form to log-in:</p>
```

Reemplázala con las siguientes líneas:

```html
<p>
    Please, use the following form to log-in. If you don't have an account
    <a href="{% url "register" %}">register here</a>.
</p>
```

Abre `http://127.0.0.1:8000/account/login/` en tu navegador. La página ahora debería verse de la siguiente manera:

> *Figura 4.19: La página Log-in incluyendo un enlace para registrarse*

Ahora hemos hecho que la página de registro sea accesible desde la página de inicio de sesión.

#### Extensión del modelo de usuario

Si bien el modelo de usuario proporcionado por el framework de autenticación de Django es suficiente para la mayoría de los escenarios típicos, tiene un conjunto limitado de campos predefinidos. Si deseas capturar detalles adicionales relevantes para tu aplicación, es posible que desees ampliar el modelo de usuario predeterminado. Por ejemplo, el modelo de usuario predeterminado viene con los campos `first_name` y `last_name`, una estructura que puede no alinearse con las convenciones de nombres en varios países. Además, es posible que desees almacenar más detalles del usuario o construir un perfil de usuario más completo.

Una forma sencilla de extender el modelo de usuario es creando un modelo de perfil que contenga una relación uno a uno con el modelo de usuario de Django y cualquier campo adicional. Una relación uno a uno es similar a un campo `ForeignKey` con el parámetro `unique=True`. El lado inverso de la relación es una relación uno a uno implícita con el modelo relacionado en lugar de un manager para múltiples elementos. Desde cada lado de la relación, accedes a un único objeto relacionado.

Edita el archivo `models.py` de tu aplicación `account` y añade el siguiente código:

```python
from django.db import models
from django.conf import settings


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE
    )
    date_of_birth = models.DateField(blank=True, null=True)
    photo = models.ImageField(
        upload_to='users/%Y/%m/%d/',
        blank=True
    )

    def __str__(self):
        return f'Profile of {self.user.username}'
```

Nuestro perfil de usuario incluirá la fecha de nacimiento del usuario y una imagen del usuario.

El campo uno a uno `user` se utilizará para asociar perfiles con usuarios. Usamos `AUTH_USER_MODEL` para hacer referencia al modelo de usuario en lugar de apuntar directamente al modelo `auth.User`. Esto hace que nuestro código sea más genérico, ya que puede operar con modelos de usuario definidos de forma personalizada. Con `on_delete=models.CASCADE`, forzamos la eliminación del objeto `Profile` relacionado cuando se elimina un objeto `User`.

El campo `date_of_birth` es un `DateField`. Hemos hecho que este campo sea opcional con `blank=True` y permitimos valores nulos con `null=True`.

El campo `photo` es un `ImageField`. Hemos hecho que este campo sea opcional con `blank=True`. Un campo `ImageField` gestiona el almacenamiento de archivos de imagen. Valida que el archivo proporcionado sea una imagen válida, almacena el archivo de imagen en el directorio indicado con el parámetro `upload_to` y almacena la ruta relativa al archivo en el campo de base de datos relacionado. Un campo `ImageField` se traduce a una columna `VARCHAR(100)` en la base de datos de forma predeterminada. Se almacenará una cadena vacía si el valor se deja en blanco.

#### Instalación de Pillow y servicio de archivos multimedia

Necesitamos instalar la biblioteca Pillow para gestionar imágenes. Pillow es la biblioteca estándar de facto para el procesamiento de imágenes en Python. Admite múltiples formatos de imagen y proporciona potentes funciones de procesamiento de imágenes. Pillow es requerida por Django para manejar imágenes con `ImageField`.

Instala Pillow ejecutando el siguiente comando desde la consola:

```bash
python -m pip install Pillow==11.0.0
```

Edita el archivo `settings.py` del proyecto y añade las siguientes líneas:

```python
MEDIA_URL = 'media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

Esto permitirá a Django gestionar las subidas de archivos y servir archivos multimedia. `MEDIA_URL` es la URL base utilizada para servir los archivos multimedia subidos por los usuarios. `MEDIA_ROOT` es la ruta local donde residen. Las rutas y URLs de los archivos se construyen dinámicamente anteponiéndoles la ruta del proyecto o la URL multimedia para mayor portabilidad.

Ahora, edita el archivo `urls.py` principal del proyecto `bookmarks` y modifica el código, de la siguiente manera:

```python
from django.conf import settings
from django.conf.urls.static import static
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('account/', include('account.urls')),
]

if settings.DEBUG:
    urlpatterns += static(
        settings.MEDIA_URL,
        document_root=settings.MEDIA_ROOT
    )
```

Hemos añadido la función auxiliar `static()` para servir archivos multimedia con el servidor de desarrollo de Django durante el desarrollo (es decir, cuando la configuración `DEBUG` se establece en `True`).

> [!CAUTION]
> La función auxiliar `static()` es adecuada para el desarrollo, pero no para uso en producción. Django es muy ineficiente a la hora de servir archivos estáticos. Nunca sirvas tus archivos estáticos con Django en un entorno de producción. Aprenderás a servir archivos estáticos en un entorno de producción en el Capítulo 17, *Puesta en producción*.

#### Creación de migraciones para el modelo de perfil

Creemos la tabla de base de datos para el nuevo modelo `Profile`. Abre la consola y ejecuta el siguiente comando para crear la migración de base de datos para el nuevo modelo:

```bash
python manage.py makemigrations
```

Obtendrás la siguiente salida:

```text
Migrations for 'account':
  account/migrations/0001_initial.py
    - Create model Profile
```

A continuación, sincroniza la base de datos con el siguiente comando en la consola:

```bash
python manage.py migrate
```

Verás una salida que incluye la siguiente línea:

```text
Applying account.0001_initial... OK
```

Edita el archivo `admin.py` de la aplicación `account` y registra el modelo `Profile` en el sitio de administración añadiendo el siguiente código:

```python
from django.contrib import admin
from .models import Profile


@admin.register(Profile)
class ProfileAdmin(admin.ModelAdmin):
    list_display = ['user', 'date_of_birth', 'photo']
    raw_id_fields = ['user']
```

Ejecuta el servidor de desarrollo utilizando el siguiente comando desde la consola:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/admin/` en tu navegador. Ahora deberías poder ver el modelo `Profile` en el sitio de administración de tu proyecto, de la siguiente manera:

> *Figura 4.20: El bloque ACCOUNT en la página de índice del sitio de administración*

Haz clic en el enlace **Add** de la fila **Profiles**. Verás el siguiente formulario para añadir un nuevo perfil:

> *Figura 4.21: El formulario Add profile*

Crea un objeto `Profile` manualmente para cada uno de los usuarios existentes en la base de datos.

A continuación, permitiremos a los usuarios editar sus perfiles en el sitio web.

Edita el archivo `forms.py` de la aplicación `account` y añade las siguientes líneas:

```python
# ...
from .models import Profile

# ...


class UserEditForm(forms.ModelForm):
    class Meta:
        model = get_user_model()
        fields = ['first_name', 'last_name', 'email']


class ProfileEditForm(forms.ModelForm):
    class Meta:
        model = Profile
        fields = ['date_of_birth', 'photo']
```

Estos formularios son los siguientes:

- `UserEditForm`: Esto permitirá a los usuarios editar su nombre, apellido y correo electrónico, que son atributos del modelo de usuario integrado de Django.
- `ProfileEditForm`: Esto permitirá a los usuarios editar los datos de perfil que se guardan en el modelo `Profile` personalizado. Los usuarios podrán editar su fecha de nacimiento y subir una imagen para su foto de perfil.

Edita el archivo `views.py` de la aplicación `account` y añade las siguientes líneas:

```python
# ...
from .models import Profile

# ...


def register(request):
    if request.method == 'POST':
        user_form = UserRegistrationForm(request.POST)
        if user_form.is_valid():
            # Create a new user object but avoid saving it yet
            new_user = user_form.save(commit=False)
            # Set the chosen password
            new_user.set_password(
                user_form.cleaned_data['password']
            )
            # Save the User object
            new_user.save()
            # Create the user profile
            Profile.objects.create(user=new_user)
            return render(
                request,
                'account/register_done.html',
                {'new_user': new_user}
            )
    else:
        user_form = UserRegistrationForm()
    return render(
        request,
        'account/register.html',
        {'user_form': user_form}
    )
```

Cuando los usuarios se registren en el sitio, se creará automáticamente un objeto `Profile` correspondiente y se asociará con el objeto `User` creado. Sin embargo, los usuarios creados a través del sitio de administración no obtendrán automáticamente un objeto `Profile` asociado. Tanto los usuarios con perfil como sin perfil (por ejemplo, los usuarios del personal o *staff*) pueden coexistir.

> [!TIP]
> Si deseas forzar la creación de perfiles para todos los usuarios, puedes utilizar señales (*signals*) de Django para activar la creación de un objeto `Profile` cada vez que se crea un usuario. Aprenderás sobre señales en el Capítulo 7, *Seguimiento de las acciones del usuario*, donde realizarás un ejercicio para implementar esta función en la sección *Ampliación de tu proyecto usando IA*.

Ahora, permitiremos a los usuarios editar sus perfiles.

Edita el archivo `views.py` de la aplicación `account` y añade el siguiente código:

```python
from django.contrib.auth import authenticate, login
from django.contrib.auth.decorators import login_required
from django.http import HttpResponse
from django.shortcuts import render
from .forms import (
    LoginForm,
    UserRegistrationForm,
    UserEditForm,
    ProfileEditForm
)
from .models import Profile

# ...


@login_required
def edit(request):
    if request.method == 'POST':
        user_form = UserEditForm(
            instance=request.user,
            data=request.POST
        )
        profile_form = ProfileEditForm(
            instance=request.user.profile,
            data=request.POST,
            files=request.FILES
        )
        if user_form.is_valid() and profile_form.is_valid():
            user_form.save()
            profile_form.save()
    else:
        user_form = UserEditForm(instance=request.user)
        profile_form = ProfileEditForm(instance=request.user.profile)
    return render(
        request,
        'account/edit.html',
        {
            'user_form': user_form,
            'profile_form': profile_form
        }
    )
```

Hemos añadido la nueva vista `edit` para permitir a los usuarios editar su información personal. Hemos añadido el decorador `login_required` a la vista porque solo los usuarios autenticados podrán editar sus perfiles. Para esta vista, usamos dos formularios de modelo: `UserEditForm` para almacenar los datos del modelo de usuario integrado y `ProfileEditForm` para almacenar los datos personales adicionales en el modelo `Profile` personalizado. Para validar los datos enviados, llamamos al método `is_valid()` de ambos formularios. Si ambos formularios contienen datos válidos, guardamos ambos formularios llamando al método `save()` para actualizar los objetos correspondientes en la base de datos.

Añade el siguiente patrón de URL al archivo `urls.py` de la aplicación `account`:

```python
urlpatterns = [
    #...
    path('', include('django.contrib.auth.urls')),
    path('', views.dashboard, name='dashboard'),
    path('register/', views.register, name='register'),
    path('edit/', views.edit, name='edit'),
]
```

Finalmente, crea una plantilla para esta vista en el directorio `templates/account/` y nómbrala `edit.html`. Añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Edit your account{% endblock %}

{% block content %}
    <h1>Edit your account</h1>
    <p>You can edit your account using the following form:</p>
    <form method="post" enctype="multipart/form-data">
        {{ user_form.as_p }}
        {{ profile_form.as_p }}
        {% csrf_token %}
        <p><input type="submit" value="Save changes"></p>
    </form>
{% endblock %}
```

En el código anterior, hemos añadido `enctype="multipart/form-data"` al elemento HTML `<form>` para permitir la subida de archivos. Usamos un formulario HTML para enviar tanto el formulario `user_form` como `profile_form`.

Abre la URL `http://127.0.0.1:8000/account/register/` y registra un nuevo usuario. Luego, inicia sesión con el nuevo usuario y abre la URL `http://127.0.0.1:8000/account/edit/`. Deberías ver la siguiente página:

> *Figura 4.22: El formulario de edición de perfil*

Ahora puedes añadir la información del perfil y guardar los cambios.

Editaremos la plantilla del panel de control para incluir enlaces a las páginas de edición de perfil y cambio de contraseña.

Abre la plantilla `templates/account/dashboard.html` y añade las siguientes líneas:

```html
{% extends "base.html" %}

{% block title %}Dashboard{% endblock %}

{% block content %}
    <h1>Dashboard</h1>
    <p>
        Welcome to your dashboard. You can
        <a href="{% url "edit" %}">edit your profile</a>
        or
        <a href="{% url "password_change" %}">change your password</a>.
    </p>
{% endblock %}
```

Los usuarios ahora pueden acceder al formulario para editar su perfil desde el panel de control. Abre `http://127.0.0.1:8000/account/` en tu navegador y prueba el nuevo enlace para editar el perfil de un usuario. El panel de control ahora debería verse así:

> *Figura 4.23: Contenido de la página Dashboard, incluyendo enlaces para editar un perfil y cambiar una contraseña*

#### Uso de un modelo de usuario personalizado

Django también ofrece una forma de sustituir el modelo de usuario por un modelo personalizado. La clase `User` debe heredar de la clase `AbstractUser` de Django, que proporciona la implementación completa del usuario predeterminado como un modelo abstracto. Puedes leer más sobre este método en [https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#substituting-a-custom-user-model](https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#substituting-a-custom-user-model).

El uso de un modelo de usuario personalizado te dará más flexibilidad, pero también podría resultar en una integración más compleja con aplicaciones conectables (*pluggable apps*) que interactúan directamente con el modelo de usuario `auth` de Django.

---

### Resumen

En este capítulo, aprendiste a construir un sistema de autenticación para tu sitio. Implementaste todas las vistas necesarias para que los usuarios se registren, inicien sesión, cierren sesión, editen su contraseña y la restablezcan. También creaste un modelo para almacenar perfiles de usuario personalizados.

En el próximo capítulo, mejorarás la experiencia del usuario proporcionando comentarios sobre las acciones del usuario a través del framework de mensajes de Django. También ampliarás el alcance de los métodos de autenticación, permitiendo a los usuarios autenticarse con su dirección de correo electrónico e integrando la autenticación social a través de Google. También aprenderás a servir el servidor de desarrollo a través de HTTPS utilizando Django Extensions, y personalizarás el pipeline de autenticación para crear perfiles de usuario automáticamente.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter04](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter04)
- **Vistas de autenticación integradas:** [https://docs.djangoproject.com/en/5.2/topics/auth/default/#all-authentication-views](https://docs.djangoproject.com/en/5.2/topics/auth/default/#all-authentication-views)
- **Patrones de URL de autenticación:** [https://github.com/django/django/blob/stable/5.2.x/django/contrib/auth/urls.py](https://github.com/django/django/blob/stable/5.2.x/django/contrib/auth/urls.py)
- **Cómo Django gestiona las contraseñas y generadores de hash disponibles:** [https://docs.djangoproject.com/en/5.2/topics/auth/passwords/](https://docs.djangoproject.com/en/5.2/topics/auth/passwords/)
- **Modelo de usuario genérico y el método get_user_model():** [https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#django.contrib.auth.get_user_model](https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#django.contrib.auth.get_user_model)
- **Uso de un modelo de usuario personalizado:** [https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#substituting-a-custom-user-model](https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#substituting-a-custom-user-model)
