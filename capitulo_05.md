# Parte 2: Creación de un sitio web social

## Capítulo 5: Implementación de autenticación social

### Introducción

En el capítulo anterior, integraste el registro de usuarios y la autenticación en tu sitio web. Implementaste funcionalidades de cambio, restablecimiento y recuperación de contraseñas, y aprendiste a crear un modelo de perfil personalizado para tus usuarios.

En este capítulo, añadirás autenticación social a tu sitio usando Google. Utilizarás Python Social Auth para Django para implementar la autenticación social mediante OAuth 2.0, el protocolo estándar de la industria para autorización. También modificarás la canalización (*pipeline*) de autenticación social para crear automáticamente un perfil de usuario para los nuevos usuarios.

Este capítulo cubrirá los siguientes puntos:

- Uso del framework de mensajes
- Creación de un backend de autenticación personalizado
- Evitar que los usuarios utilicen un correo electrónico existente
- Adición de autenticación social con Python Social Auth
- Ejecución del servidor de desarrollo a través de HTTPS utilizando Django Extensions
- Adición de autenticación mediante Google
- Creación de un perfil para usuarios que se registran con autenticación social

---

### Visión general funcional

La Figura 5.1 muestra una representación de las vistas, plantillas y funcionalidades que se construirán en este capítulo:

> *Figura 5.1: Diagrama de funcionalidades construidas en el Capítulo 5*

En este capítulo, generarás mensajes de éxito y error en la vista `edit` utilizando el framework de mensajes de Django. Construirás un nuevo backend de autenticación llamado `EmailAuthBackend` para que los usuarios puedan autenticarse utilizando sus direcciones de correo electrónico. Servirás tu sitio a través de HTTPS durante el desarrollo utilizando Django Extensions, e implementarás la autenticación social con Google en tu sitio utilizando Python Social Auth. Los usuarios serán redirigidos a la vista del panel de control (*dashboard*) tras una autenticación exitosa. Personalizarás el pipeline de autenticación para crear perfiles de usuario automáticamente cuando se cree un nuevo usuario mediante autenticación social.

---

### Requisitos técnicos

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter05](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter05).

Todos los paquetes de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente del capítulo. Puedes seguir las instrucciones para instalar cada paquete de Python en las siguientes secciones, o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Uso del framework de mensajes

Cuando los usuarios interactúan con la plataforma, hay muchos casos en los que desearás informarles sobre el resultado de acciones específicas, como la creación exitosa de un objeto en la base de datos o el envío correcto de un formulario.

Django cuenta con un framework de mensajes integrado que te permite mostrar notificaciones de un solo uso a tus usuarios. Esto mejora la experiencia del usuario proporcionando retroalimentación inmediata sobre sus acciones, haciendo que la interfaz sea más intuitiva y fácil de usar.

El framework de mensajes se encuentra en `django.contrib.messages` y está incluido en la lista `INSTALLED_APPS` predeterminada del archivo `settings.py` cuando creas nuevos proyectos usando `django-admin startproject`. El archivo de configuración también contiene el middleware `django.contrib.messages.middleware.MessageMiddleware` en la configuración `MIDDLEWARE`.

El framework de mensajes proporciona una forma sencilla de agregar mensajes a los usuarios. Los mensajes se almacenan en una cookie de forma predeterminada (con respaldo en el almacenamiento de sesiones) y se muestran y borran en la siguiente petición del usuario. Puedes usar el framework de mensajes en tus vistas importando el módulo `messages` y agregando nuevos mensajes con métodos abreviados simples, de la siguiente manera:

```python
from django.contrib import messages
messages.error(request, 'Something went wrong')
```

Puedes crear nuevos mensajes utilizando el método `add_message()` o cualquiera de los siguientes métodos abreviados:

- `success()`: Los mensajes de éxito se utilizan para mostrar cuándo una acción fue exitosa.
- `info()`: Mensajes informativos.
- `warning()`: Muestra que aún no se ha producido un fallo, pero puede ser inminente.
- `error()`: Muestra que una acción no fue exitosa o que ocurrió un error.
- `debug()`: Muestra mensajes de depuración que se eliminarán o ignorarán en un entorno de producción.

Agreguemos mensajes al proyecto. El framework de mensajes se aplica globalmente al proyecto. Utilizaremos la plantilla base para mostrar los mensajes disponibles al cliente. Esto nos permitirá notificar al cliente sobre los resultados de cualquier acción en cualquier página.

Abre la plantilla `templates/base.html` de la aplicación `account` y añade el siguiente código:

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
        ...
    </div>
    {% if messages %}
        <ul class="messages">
            {% for message in messages %}
                <li class="{{ message.tags }}">
                    {{ message|safe }}
                    <a href="#" class="close">x</a>
                </li>
            {% endfor %}
        </ul>
    {% endif %}
    <div id="content">
        {% block content %}
        {% endblock %}
    </div>
</body>
</html>
```

El framework de mensajes incluye el procesador de contexto `django.contrib.messages.context_processors.messages`, que añade una variable `messages` al contexto de la petición. Puedes encontrarlo en la lista `context_processors` dentro de la configuración `TEMPLATES` de tu proyecto. Puedes usar la variable `messages` en las plantillas para mostrar todos los mensajes existentes al usuario.

Un procesador de contexto es una función de Python que toma el objeto `request` como argumento y devuelve un diccionario que se añade al contexto de la petición. Aprenderás a crear tus propios procesadores de contexto en el Capítulo 8, *Creación de una tienda online*.

Modifiquemos la vista `edit` para utilizar el framework de mensajes.

Edita el archivo `views.py` de la aplicación `account` y añade las siguientes líneas:

```python
from django.contrib import messages

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
            messages.success(
                request,
                'Profile updated successfully'
            )
        else:
            messages.error(request, 'Error updating your profile')
    else:
        user_form = UserEditForm(instance=request.user)
        profile_form = ProfileEditForm(
            instance=request.user.profile
        )
    return render(
        request,
        'account/edit.html',
        {
            'user_form': user_form,
            'profile_form': profile_form
        }
    )
```

Se genera un mensaje de éxito cuando los usuarios actualizan su perfil correctamente. Si alguno de los formularios contiene datos no válidos, se genera un mensaje de error en su lugar.

Abre `http://127.0.0.1:8000/account/edit/` en tu navegador y edita el perfil del usuario. Deberías ver el siguiente mensaje cuando el perfil se actualice correctamente:

> *Figura 5.2: Mensaje de perfil editado con éxito*

Introduce una fecha no válida en el campo **Date of birth** y envía el formulario nuevamente. Deberías ver el siguiente mensaje:

> *Figura 5.3: Mensaje de error al actualizar el perfil*

Generar mensajes para informar a tus usuarios sobre los resultados de sus acciones es muy sencillo. Puedes agregar fácilmente mensajes a otras vistas también.

Puedes obtener más información sobre el framework de mensajes en [https://docs.djangoproject.com/en/5.2/ref/contrib/messages/](https://docs.djangoproject.com/en/5.2/ref/contrib/messages/).

Ahora que hemos construido toda la funcionalidad relacionada con la autenticación de usuarios y la edición de perfiles, profundizaremos en la personalización de la autenticación. Aprenderemos a crear un backend de autenticación personalizado para que los usuarios puedan iniciar sesión en el sitio utilizando sus direcciones de correo electrónico.

---

### Creación de un backend de autenticación personalizado

Django te permite autenticar usuarios contra diferentes fuentes, como el sistema de autenticación integrado de Django, sistemas de autenticación externos como servidores LDAP (Lightweight Directory Access Protocol) o incluso proveedores de terceros. La configuración `AUTHENTICATION_BACKENDS` incluye una lista de backends de autenticación disponibles en el proyecto. Django te permite especificar múltiples backends de autenticación para esquemas de autenticación flexibles. El valor predeterminado de la configuración `AUTHENTICATION_BACKENDS` es el siguiente:

```python
['django.contrib.auth.backends.ModelBackend']
```

El `ModelBackend` predeterminado autentica a los usuarios contra la base de datos utilizando el modelo `User` de `django.contrib.auth`. Esto es adecuado para la mayoría de los proyectos web. Sin embargo, puedes crear backends personalizados para autenticar a tus usuarios contra otras fuentes.

Puedes leer más información sobre la personalización de la autenticación en [https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#other-authentication-sources](https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#other-authentication-sources).

Siempre que se utiliza la función `authenticate()` de `django.contrib.auth`, Django intenta autenticar al usuario contra cada uno de los backends definidos en `AUTHENTICATION_BACKENDS` uno por uno, hasta que uno de ellos autentica al usuario con éxito. Solo si todos los backends fallan en la autenticación, el usuario no será autenticado.

Django proporciona una forma sencilla de definir tus propios backends de autenticación. Un backend de autenticación es una clase que proporciona los dos métodos siguientes:

- `authenticate()`: Toma el objeto `request` y las credenciales del usuario como parámetros. Debe devolver un objeto de usuario que coincida con esas credenciales si las credenciales son válidas, o `None` en caso contrario. El parámetro `request` es un objeto `HttpRequest`, o `None` si no se proporciona a la función `authenticate()`.
- `get_user()`: Toma un parámetro de ID de usuario y debe devolver un objeto de usuario.

Crear un backend de autenticación personalizado es tan sencillo como escribir una clase de Python que implemente ambos métodos. Creemos un backend de autenticación para permitir a los usuarios autenticarse en el sitio utilizando su dirección de correo electrónico en lugar de su nombre de usuario.

Crea un nuevo archivo dentro del directorio de la aplicación `account` y nómbralo `authentication.py`. Añádele el siguiente código:

```python
from django.contrib.auth.models import User


class EmailAuthBackend:
    """
    Authenticate using an e-mail address.
    """
    def authenticate(self, request, username=None, password=None):
        try:
            user = User.objects.get(email=username)
            if user.check_password(password):
                return user
            return None
        except (User.DoesNotExist, User.MultipleObjectsReturned):
            return None

    def get_user(self, user_id):
        try:
            return User.objects.get(pk=user_id)
        except User.DoesNotExist:
            return None
```

El código anterior es un backend de autenticación simple. El método `authenticate()` recibe un objeto `request` y los parámetros opcionales `username` y `password`. Podríamos usar diferentes parámetros, pero usamos `username` y `password` para hacer que nuestro backend funcione de inmediato con las vistas del framework de autenticación. El código anterior funciona de la siguiente manera:

- `authenticate()`: Se recupera el usuario con la dirección de correo electrónico dada y se verifica la contraseña utilizando el método integrado `check_password()` del modelo de usuario. Este método gestiona el hash de la contraseña para comparar la contraseña proporcionada con la contraseña almacenada en la base de datos. Se capturan dos excepciones diferentes de QuerySet: `DoesNotExist` y `MultipleObjectsReturned`. La excepción `DoesNotExist` se genera si no se encuentra ningún usuario con la dirección de correo electrónico proporcionada. La excepción `MultipleObjectsReturned` se genera si se encuentran varios usuarios con la misma dirección de correo electrónico. Modificaremos las vistas de registro y edición más adelante para evitar que los usuarios utilicen una dirección de correo electrónico existente.
- `get_user()`: Obtienes un usuario mediante el ID proporcionado en el parámetro `user_id`. Django utiliza el backend que autenticó al usuario para recuperar el objeto `User` durante la duración de la sesión del usuario. `pk` es una abreviatura de clave primaria (*primary key*), que es un identificador único para cada registro en la base de datos. Cada modelo de Django tiene un campo que sirve como su clave primaria. De forma predeterminada, la clave primaria es el campo `id` generado automáticamente. Puedes encontrar más información sobre los campos de clave primaria automáticos en [https://docs.djangoproject.com/en/5.2/topics/db/models/#automatic-primary-key-fields](https://docs.djangoproject.com/en/5.2/topics/db/models/#automatic-primary-key-fields).

Edita el archivo `settings.py` de tu proyecto y añade el siguiente código:

```python
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'account.authentication.EmailAuthBackend',
]
```

En la configuración anterior, mantenemos el `ModelBackend` predeterminado que se utiliza para autenticar con el nombre de usuario y la contraseña, e incluimos nuestro propio backend de autenticación basado en correo electrónico `EmailAuthBackend`.

Abre `http://127.0.0.1:8000/account/login/` en tu navegador. Recuerda que Django intentará autenticar al usuario contra cada uno de los backends, por lo que ahora deberías poder iniciar sesión sin problemas utilizando tu nombre de usuario o tu cuenta de correo electrónico.

Las credenciales del usuario se verificarán utilizando `ModelBackend` y, si no se devuelve ningún usuario, las credenciales se verificarán utilizando `EmailAuthBackend`.

El orden de los backends enumerados en la configuración `AUTHENTICATION_BACKENDS` es importante. Si las mismas credenciales son válidas para múltiples backends, Django autenticará al usuario utilizando el primer backend de la lista que valide con éxito estas credenciales. Esto significa que Django no procederá a verificar los backends restantes una vez que se encuentre una coincidencia.

#### Evitar que los usuarios utilicen un correo electrónico existente

El modelo `User` del framework de autenticación no impide la creación de usuarios con la misma dirección de correo electrónico. Si dos o más cuentas de usuario comparten la misma dirección de correo electrónico, no podremos discernir qué usuario se está autenticando. Ahora que los usuarios pueden iniciar sesión utilizando su dirección de correo electrónico, debemos evitar que los usuarios se registren con una dirección de correo electrónico existente.

Ahora cambiaremos el formulario de registro de usuarios para evitar que varios usuarios se registren con la misma dirección de correo electrónico.

Edita el archivo `forms.py` de la aplicación `account` y añade las siguientes líneas a la clase `UserRegistrationForm`:

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
        model = User
        fields = ['username', 'first_name', 'email']

    def clean_password2(self):
        cd = self.cleaned_data
        if cd['password'] != cd['password2']:
            raise forms.ValidationError("Passwords don't match.")
        return cd['password2']

    def clean_email(self):
        data = self.cleaned_data['email']
        if User.objects.filter(email=data).exists():
            raise forms.ValidationError('Email already in use.')
        return data
```

Hemos añadido una validación para el campo `email` que evita que los usuarios se registren con una dirección de correo electrónico existente. Construimos un QuerySet para buscar usuarios existentes con la misma dirección de correo electrónico. Comprobamos si hay algún resultado con el método `exists()`. El método `exists()` devuelve `True` si el QuerySet contiene algún resultado y `False` en caso contrario.

Ahora, añade las siguientes líneas a la clase `UserEditForm`:

```python
class UserEditForm(forms.ModelForm):
    class Meta:
        model = User
        fields = ['first_name', 'last_name', 'email']

    def clean_email(self):
        data = self.cleaned_data['email']
        qs = User.objects.exclude(
            id=self.instance.id
        ).filter(
            email=data
        )
        if qs.exists():
            raise forms.ValidationError('Email already in use.')
        return data
```

En este caso, hemos añadido una validación para el campo `email` que evita que los usuarios cambien su dirección de correo electrónico a una dirección de correo electrónico existente de otro usuario. Excluimos al usuario actual del QuerySet. De lo contrario, la dirección de correo electrónico actual del usuario se consideraría una dirección de correo electrónico existente y el formulario no se validaría.

---

### Adición de autenticación social a tu sitio

La autenticación social es una característica ampliamente utilizada que permite a los usuarios autenticarse utilizando su cuenta existente de un proveedor de servicios mediante inicio de sesión único (*Single Sign-On* o SSO). El proceso de autenticación permite a los usuarios acceder al sitio utilizando su cuenta existente de servicios sociales como Google, Facebook o Twitter. En esta sección, añadiremos autenticación social al sitio utilizando Google.

Para implementar la autenticación social, utilizaremos el protocolo estándar de la industria OAuth 2.0 para la autorización. OAuth significa *Open Authorization*. OAuth 2.0 es un estándar diseñado para permitir que un sitio web o aplicación acceda a recursos alojados por otras aplicaciones web en nombre de un usuario. Google utiliza el protocolo OAuth 2.0 para la autenticación y autorización.

Python Social Auth es un módulo de Python que simplifica el proceso de añadir autenticación social a tu sitio web. Utilizando este módulo, puedes permitir que tus usuarios inicien sesión en tu sitio web utilizando sus cuentas de otros servicios. Puedes encontrar el código de este módulo en [https://github.com/python-social-auth/social-app-django](https://github.com/python-social-auth/social-app-django).

Este módulo viene con backends de autenticación para diferentes frameworks de Python, incluido Django.

Ejecuta el siguiente comando en la consola:

```bash
python -m pip install social-auth-app-django==5.4.0
```

Esto instalará Python Social Auth.

Luego, añade `social_django` a la configuración `INSTALLED_APPS` en el archivo `settings.py` del proyecto, de la siguiente manera:

```python
INSTALLED_APPS = [
    # ...
    'social_django',
]
```

Esta es la aplicación predeterminada para añadir Python Social Auth a proyectos Django. Ahora, ejecuta el siguiente comando para sincronizar los modelos de Python Social Auth con tu base de datos:

```bash
python manage.py migrate
```

Deberías ver que se aplican las migraciones para la aplicación predeterminada, de la siguiente manera:

```text
Applying social_django.0001_initial... OK
...
Applying social_django.0015_rename_extra_data_new_usersocialauth_extra_data... OK
```

Python Social Auth incluye backends de autenticación para múltiples servicios. Puedes encontrar la lista con todos los backends disponibles en [https://python-social-auth.readthedocs.io/en/latest/backends/index.html#supported-backends](https://python-social-auth.readthedocs.io/en/latest/backends/index.html#supported-backends).

Añadiremos autenticación social a nuestro proyecto, permitiendo a nuestros usuarios autenticarse con el backend de Google.

Primero, necesitamos añadir los patrones de URL de inicio de sesión social al proyecto.

Abre el archivo `urls.py` principal del proyecto `bookmarks` e incluye los patrones de URL de `social_django`, de la siguiente manera:

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('account/', include('account.urls')),
    path(
        'social-auth/',
        include('social_django.urls', namespace='social')
    ),
]
```

Actualmente, nuestra aplicación web es accesible a través de la IP de localhost, `127.0.0.1`, o utilizando el nombre de host `localhost`. Google permite la redirección de usuarios a `localhost` después de una autenticación exitosa, pero otros servicios sociales esperan un nombre de dominio para la redirección de URL. En este proyecto, simularemos un entorno real sirviendo nuestro sitio bajo un nombre de dominio en nuestra máquina local.

Localiza el archivo `hosts` de tu máquina. Si estás utilizando Linux o macOS, el archivo `hosts` se encuentra en `/etc/hosts`. Si estás utilizando Windows, el archivo `hosts` se encuentra en `C:\Windows\System32\Drivers\etc\hosts`.

Edita el archivo `hosts` de tu máquina y añádele la siguiente línea:

```text
127.0.0.1 mysite.com
```

Esto le indicará a tu computadora que dirija el nombre de host `mysite.com` a tu propia máquina.

Verifiquemos que la asociación del nombre de host haya funcionado. Ejecuta el servidor de desarrollo utilizando el siguiente comando desde la consola:

```bash
python manage.py runserver
```

Abre `http://mysite.com:8000/account/login/` en tu navegador. Verás el siguiente error:

> *Figura 5.4: Mensaje de encabezado de host no válido (Invalid HTTP_HOST header)*

Django controla los hosts que pueden servir la aplicación mediante la configuración `ALLOWED_HOSTS`. Esta es una medida de seguridad para prevenir ataques de falsificación de encabezados de host HTTP (*HTTP host header attacks*). Django solo permitirá que los hosts incluidos en esta lista sirvan la aplicación.

Puedes obtener más información sobre la configuración `ALLOWED_HOSTS` en [https://docs.djangoproject.com/en/5.2/ref/settings/#allowed-hosts](https://docs.djangoproject.com/en/5.2/ref/settings/#allowed-hosts).

Edita el archivo `settings.py` del proyecto y modifica la configuración `ALLOWED_HOSTS` de la siguiente manera:

```python
ALLOWED_HOSTS = ['mysite.com', 'localhost', '127.0.0.1']
```

Además del host `mysite.com`, hemos incluido explícitamente `localhost` y `127.0.0.1`. Esto permite el acceso al sitio a través de `localhost` y `127.0.0.1`, que es el comportamiento predeterminado de Django cuando `DEBUG` es `True` y `ALLOWED_HOSTS` está vacío.

Abre `http://mysite.com:8000/account/login/` nuevamente en tu navegador. Ahora deberías ver la página de inicio de sesión del sitio en lugar de un error.

#### Ejecución del servidor de desarrollo a través de HTTPS

A continuación, vamos a ejecutar el servidor de desarrollo a través de HTTPS para simular un entorno real donde el contenido intercambiado con el navegador está protegido. Esto nos ayudará más adelante en el Capítulo 6, *Compartir contenido en tu sitio web*, para servir nuestro sitio de forma segura y cargar nuestra herramienta de marcadores de imágenes sobre cualquier sitio web seguro. El protocolo Transport Layer Security (TLS) es el estándar para servir sitios web a través de una conexión segura. El predecesor de TLS es Secure Sockets Layer (SSL).

Aunque SSL está actualmente en desuso, encontrarás referencias tanto a los términos TLS como SSL en múltiples bibliotecas y documentación en línea. El servidor de desarrollo de Django no es capaz de servir tu sitio a través de HTTPS, ya que ese no es su propósito previsto. Para probar la funcionalidad de autenticación social sirviendo el sitio a través de HTTPS, vamos a utilizar la extensión `RunServerPlus` del paquete `django-extensions`. Este paquete contiene una colección de herramientas útiles para Django. Ten en cuenta que nunca debes utilizar el servidor de desarrollo para ejecutar tu sitio en un entorno de producción.

Utiliza el siguiente comando para instalar Django Extensions:

```bash
python -m pip install django-extensions==3.2.3
```

Necesitarás instalar Werkzeug, que contiene una capa de depurador requerida por la extensión `RunServerPlus` de Django Extensions. Usa el siguiente comando para instalar Werkzeug:

```bash
python -m pip install werkzeug==3.0.2
```

Finalmente, utiliza el siguiente comando para instalar pyOpenSSL, que es necesario para utilizar la funcionalidad SSL/TLS de `RunServerPlus`:

```bash
python -m pip install pyOpenSSL==24.1.0
```

Edita el archivo `settings.py` de tu proyecto y añade Django Extensions a la configuración `INSTALLED_APPS`, de la siguiente manera:

```python
INSTALLED_APPS = [
    # ...
    'django_extensions',
]
```

Ahora, utiliza el comando de gestión `runserver_plus` proporcionado por Django Extensions para ejecutar el servidor de desarrollo, de la siguiente manera:

```bash
python manage.py runserver_plus --cert-file cert.crt
```

Hemos proporcionado un nombre de archivo al comando `runserver_plus` para el certificado SSL/TLS. Django Extensions generará una clave y un certificado automáticamente.

Abre `https://mysite.com:8000/account/login/` en tu navegador. Ahora estás accediendo a tu sitio a través de HTTPS. Observa que ahora estamos usando `https://` en lugar de `http://`.

Tu navegador mostrará una advertencia de seguridad porque estás utilizando un certificado autogenerado en lugar de un certificado emitido por una autoridad de certificación (CA) de confianza.

Si estás usando Google Chrome, verás la siguiente pantalla:

> *Figura 5.5: Advertencia de seguridad en Google Chrome*

En este caso, haz clic en **Avanzado** y luego haz clic en **Continuar a mysite.com (no seguro)**.

Si estás usando Safari, verás la siguiente pantalla:

> *Figura 5.6: Advertencia de seguridad en Safari*

En este caso, haz clic en **Mostrar detalles** y luego en **visitar este sitio web**.

Si estás usando Microsoft Edge, verás la siguiente pantalla:

> *Figura 5.7: Advertencia de seguridad en Microsoft Edge*

En este caso, haz clic en **Avanzado** y luego en **Continuar a mysite.com (no seguro)**.

Si estás utilizando cualquier otro navegador, accede a la información avanzada mostrada por tu navegador y acepta el certificado autofirmado para que tu navegador confíe en él.

Verás que la URL comienza con `https://` y, en algunos casos, un icono de candado que indica que la conexión es segura. Algunos navegadores pueden mostrar un icono de candado roto porque estás utilizando un certificado autofirmado en lugar de uno confiable. Eso no será un problema para nuestras pruebas:

> *Figura 5.8: La URL con el icono de conexión segura*

Django Extensions incluye muchas otras herramientas y características interesantes. Puedes encontrar más información sobre este paquete en [https://django-extensions.readthedocs.io/en/latest/](https://django-extensions.readthedocs.io/en/latest/).

Ahora puedes servir tu sitio a través de HTTPS durante el desarrollo.

#### Autenticación mediante Google

Google ofrece autenticación social utilizando OAuth 2.0, lo que permite a los usuarios iniciar sesión con cuentas de Google. Puedes leer sobre la implementación de OAuth2 de Google en [https://developers.google.com/identity/protocols/OAuth2](https://developers.google.com/identity/protocols/OAuth2).

Para implementar la autenticación utilizando Google, añade la siguiente línea a la configuración `AUTHENTICATION_BACKENDS` en el archivo `settings.py` de tu proyecto:

```python
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'account.authentication.EmailAuthBackend',
    'social_core.backends.google.GoogleOAuth2',
]
```

Primero, necesitarás crear una clave de API en tu consola de desarrollador de Google. Abre [https://console.cloud.google.com/projectcreate](https://console.cloud.google.com/projectcreate) en tu navegador. Verás la siguiente pantalla:

> *Figura 5.9: Formulario de creación de proyectos de Google*

En **Project name**, introduce `Bookmarks` y haz clic en el botón **CREATE**.

Cuando el nuevo proyecto esté listo, asegúrate de que el proyecto esté seleccionado en la barra de navegación superior de la siguiente manera:

> *Figura 5.10: Barra de navegación superior de Google Developer Console*

Después de crear el proyecto, en **APIs & Services**, haz clic en **Credentials**:

> *Figura 5.11: Menú APIs y servicios de Google*

Verás la siguiente pantalla:

> *Figura 5.12: Creación de credenciales de API de Google*

Luego, haz clic en **CREATE CREDENTIALS** y haz clic en **OAuth client ID**.

Google te pedirá que configures primero la pantalla de consentimiento, de esta manera:

> *Figura 5.13: Alerta para configurar la pantalla de consentimiento de OAuth*

Configuraremos la página que se mostrará a los usuarios para dar su consentimiento para acceder a tu sitio con su cuenta de Google. Haz clic en el botón **CONFIGURE CONSENT SCREEN**. Serás redirigido a la siguiente pantalla:

> *Figura 5.14: Selección del tipo de usuario en la configuración de la pantalla de consentimiento de OAuth de Google*

Elige **External** para **User Type** y haz clic en el botón **CREATE**. Verás la siguiente pantalla:

> *Figura 5.15: Configuración de la pantalla de consentimiento de OAuth de Google*

En **App name**, introduce `Bookmarks` y selecciona tu correo electrónico para **User support email**.

En **Authorised domains**, introduce `mysite.com` de la siguiente manera:

> *Figura 5.16: Dominios autorizados en OAuth de Google*

Introduce tu correo electrónico en **Developer contact information** y haz clic en **SAVE AND CONTINUE**.

En el paso 2, **Scopes**, no cambies nada y haz clic en **SAVE AND CONTINUE**.

En el paso 3, **Test users**, añade tu usuario de Google a **Test users** y haz clic en **SAVE AND CONTINUE** de la siguiente manera:

> *Figura 5.17: Usuarios de prueba en OAuth de Google*

Verás un resumen de la configuración de tu pantalla de consentimiento. Haz clic en **BACK TO DASHBOARD**.

En el menú de la barra lateral izquierda, haz clic en **Credentials**, haz clic de nuevo en **Create credentials** y luego en **OAuth client ID**.

Como siguiente paso, introduce la siguiente información:

- **Application type**: Selecciona **Web application**
- **Name**: Introduce `Bookmarks`
- **Authorised JavaScript origins**: Añade `https://mysite.com:8000`
- **Authorised redirect URIs**: Añade `https://mysite.com:8000/social-auth/complete/google-oauth2/`

El formulario debería verse así:

> *Figura 5.18: Formulario de creación del ID de cliente OAuth de Google*

Haz clic en el botón **CREATE**. Obtendrás el **Client ID** y el **Client secret**:

> *Figura 5.19: OAuth de Google – ID de cliente y Secreto de cliente*

Crea un nuevo archivo dentro del directorio raíz de tu proyecto y nómbralo `.env`. El archivo `.env` contendrá pares clave-valor de variables de entorno. Añade las credenciales de OAuth2 al nuevo archivo, de la siguiente manera:

```text
GOOGLE_OAUTH2_KEY=xxxx
GOOGLE_OAUTH2_SECRET=xxxx
```

Reemplaza `xxxx` con la clave y el secreto de OAuth2 respectivamente.

Para facilitar la separación de la configuración del código, vamos a utilizar `python-decouple`. Ya utilizaste esta biblioteca en el Capítulo 2, *Mejora de tu blog y adición de funciones sociales*.

Instala `python-decouple` mediante pip ejecutando el siguiente comando:

```bash
python -m pip install python-decouple==3.8
```

Edita el archivo `settings.py` de tu proyecto y añádele el siguiente código:

```python
from decouple import config

# ...
SOCIAL_AUTH_GOOGLE_OAUTH2_KEY = config('GOOGLE_OAUTH2_KEY')
SOCIAL_AUTH_GOOGLE_OAUTH2_SECRET = config('GOOGLE_OAUTH2_SECRET')
```

Las configuraciones `SOCIAL_AUTH_GOOGLE_OAUTH2_KEY` y `SOCIAL_AUTH_GOOGLE_OAUTH2_SECRET` se cargan a partir de las variables de entorno definidas en el archivo `.env`.

Edita la plantilla `registration/login.html` de la aplicación `account` y añade el siguiente código al final del bloque `content`:

```html
{% block content %}
    ...
    <div class="social">
        <ul>
            <li class="google">
                <a href="{% url "social:begin" "google-oauth2" %}">
                    Sign in with Google
                </a>
            </li>
        </ul>
    </div>
{% endblock %}
```

Utiliza el comando de gestión `runserver_plus` proporcionado por Django Extensions para ejecutar el servidor de desarrollo, de la siguiente manera:

```bash
python manage.py runserver_plus --cert-file cert.crt
```

Abre `https://mysite.com:8000/account/login/` en tu navegador. La página de inicio de sesión ahora debería verse de la siguiente manera:

> *Figura 5.20: La página de inicio de sesión incluyendo el botón para autenticación con Google*

Haz clic en el botón **Sign in with Google**. Verás la siguiente pantalla:

> *Figura 5.21: La pantalla de autorización de la aplicación de Google*

Haz clic en tu cuenta de Google para autorizar la aplicación. Iniciarás sesión y serás redirigido a la página del panel de control de tu sitio. Recuerda que has establecido esta URL en la configuración `LOGIN_REDIRECT_URL`. Como puedes ver, añadir autenticación social a tu sitio es bastante sencillo.

Ahora has añadido autenticación social a tu proyecto con Google. Puedes implementar fácilmente la autenticación social con otros servicios en línea utilizando Python Social Auth. En la siguiente sección, abordaremos la creación de perfiles de usuario cuando se registran mediante autenticación social.

#### Creación de un perfil para usuarios que se registran con autenticación social

Cuando un usuario se autentica mediante autenticación social, se crea un nuevo objeto `User` si no existe un usuario asociado con ese perfil social. Python Social Auth utiliza un pipeline (*canalización*) que consta de un conjunto de funciones que se ejecutan en un orden específico durante el flujo de autenticación. Estas funciones se encargan de recuperar los detalles del usuario, crear un perfil social en la base de datos y asociarlo con un usuario existente o crear uno nuevo.

Actualmente, no se crea ningún objeto `Profile` cuando se crean nuevos usuarios mediante autenticación social. Añadiremos un nuevo paso al pipeline para crear automáticamente un objeto `Profile` en la base de datos cuando se cree un nuevo usuario.

Añade la siguiente configuración `SOCIAL_AUTH_PIPELINE` al archivo `settings.py` de tu proyecto:

```python
SOCIAL_AUTH_PIPELINE = [
    'social_core.pipeline.social_auth.social_details',
    'social_core.pipeline.social_auth.social_uid',
    'social_core.pipeline.social_auth.auth_allowed',
    'social_core.pipeline.social_auth.social_user',
    'social_core.pipeline.user.get_username',
    'social_core.pipeline.user.create_user',
    'social_core.pipeline.social_auth.associate_user',
    'social_core.pipeline.social_auth.load_extra_data',
    'social_core.pipeline.user.user_details',
]
```

Este es el pipeline de autenticación predeterminado utilizado por Python Social Auth. Consiste en varias funciones que realizan diferentes tareas al autenticar a un usuario. Puedes encontrar más detalles sobre el pipeline de autenticación predeterminado en [https://python-social-auth.readthedocs.io/en/latest/pipeline.html](https://python-social-auth.readthedocs.io/en/latest/pipeline.html).

Construyamos una función que cree un objeto `Profile` en la base de datos siempre que se cree un nuevo usuario. Luego añadiremos esta función al pipeline de autenticación social.

Edita el archivo `account/authentication.py` y añádele el siguiente código:

```python
from account.models import Profile


def create_profile(backend, user, *args, **kwargs):
    """
    Create user profile for social authentication
    """
    Profile.objects.get_or_create(user=user)
```

La función `create_profile` toma dos argumentos obligatorios:

- `backend`: El backend de autenticación social utilizado para la autenticación del usuario. Recuerda que añadiste los backends de autenticación social a la configuración `AUTHENTICATION_BACKENDS` en tu proyecto.
- `user`: La instancia `User` del nuevo usuario o del usuario autenticado existente.

Puedes consultar los diferentes argumentos que se pasan a las funciones del pipeline en [https://python-social-auth.readthedocs.io/en/latest/pipeline.html#extending-the-pipeline](https://python-social-auth.readthedocs.io/en/latest/pipeline.html#extending-the-pipeline).

En la función `create_profile`, comprobamos que haya un objeto de usuario presente y utilizamos el método `get_or_create()` para buscar un objeto `Profile` para el usuario dado, y creamos uno si es necesario.

Ahora, necesitamos añadir la nueva función al pipeline de autenticación. Añade la siguiente línea a la configuración `SOCIAL_AUTH_PIPELINE` en tu archivo `settings.py`:

```python
SOCIAL_AUTH_PIPELINE = [
    'social_core.pipeline.social_auth.social_details',
    'social_core.pipeline.social_auth.social_uid',
    'social_core.pipeline.social_auth.auth_allowed',
    'social_core.pipeline.social_auth.social_user',
    'social_core.pipeline.user.get_username',
    'social_core.pipeline.user.create_user',
    'account.authentication.create_profile',
    'social_core.pipeline.social_auth.associate_user',
    'social_core.pipeline.social_auth.load_extra_data',
    'social_core.pipeline.user.user_details',
]
```

Hemos añadido la función `create_profile` después de `social_core.pipeline.user.create_user`. En este punto, hay una instancia de `User` disponible. El usuario puede ser un usuario existente o uno nuevo creado en este paso del pipeline. La función `create_profile` utiliza la instancia `User` para buscar el objeto `Profile` relacionado y crear uno nuevo si es necesario.

Accede a la lista de usuarios en el sitio de administración en `https://mysite.com:8000/admin/auth/user/`. Elimina cualquier usuario creado a través de la autenticación social.

Luego, abre `https://mysite.com:8000/account/login/` y realiza la autenticación social para el usuario que eliminaste. Se creará un nuevo usuario y ahora también se creará un objeto `Profile`. Accede a `https://mysite.com:8000/admin/account/profile/` para verificar que se haya creado un perfil para el nuevo usuario.

Hemos añadido con éxito la funcionalidad para crear el perfil de usuario automáticamente para la autenticación social.

Python Social Auth también ofrece un mecanismo de pipeline para el flujo de desconexión. Puedes encontrar más detalles en [https://python-social-auth.readthedocs.io/en/latest/pipeline.html#disconnection-pipeline](https://python-social-auth.readthedocs.io/en/latest/pipeline.html#disconnection-pipeline).

---

### Resumen

En este capítulo, mejoraste significativamente las capacidades de autenticación de tu sitio social creando un backend de autenticación basado en correo electrónico y añadiendo autenticación social con Google. También mejoraste la experiencia del usuario proporcionando retroalimentación sobre sus acciones mediante el framework de mensajes de Django. Finalmente, personalizaste el pipeline de autenticación para crear perfiles de usuario para nuevos usuarios automáticamente.

En el próximo capítulo, crearás un sistema de marcadores de imágenes. Aprenderás sobre relaciones de muchos a muchos (*many-to-many*) y la personalización del comportamiento de los formularios. Aprenderás cómo generar miniaturas de imágenes y cómo construir funcionalidades AJAX utilizando JavaScript y Django.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter05](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter05)
- **El framework de mensajes de Django:** [https://docs.djangoproject.com/en/5.2/ref/contrib/messages/](https://docs.djangoproject.com/en/5.2/ref/contrib/messages/)
- **Fuentes de autenticación personalizadas:** [https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#other-authentication-sources](https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#other-authentication-sources)
- **Campos de clave primaria automáticos:** [https://docs.djangoproject.com/en/5.2/topics/db/models/#automatic-primary-key-fields](https://docs.djangoproject.com/en/5.2/topics/db/models/#automatic-primary-key-fields)
- **Python Social Auth:** [https://github.com/python-social-auth](https://github.com/python-social-auth)
- **Backends de autenticación de Python Social Auth:** [https://python-social-auth.readthedocs.io/en/latest/backends/index.html#supported-backends](https://python-social-auth.readthedocs.io/en/latest/backends/index.html#supported-backends)
- **Configuración allowed hosts de Django:** [https://docs.djangoproject.com/en/5.2/ref/settings/#allowed-hosts](https://docs.djangoproject.com/en/5.2/ref/settings/#allowed-hosts)
- **Documentación de Django Extensions:** [https://django-extensions.readthedocs.io/en/latest/](https://django-extensions.readthedocs.io/en/latest/)
- **Implementación de OAuth2 de Google:** [https://developers.google.com/identity/protocols/OAuth2](https://developers.google.com/identity/protocols/OAuth2)
- **Credenciales de API de Google:** [https://console.developers.google.com/apis/credentials](https://console.developers.google.com/apis/credentials)
- **Pipeline de Python Social Auth:** [https://python-social-auth.readthedocs.io/en/latest/pipeline.html](https://python-social-auth.readthedocs.io/en/latest/pipeline.html)
- **Extensión del pipeline de Python Social Auth:** [https://python-social-auth.readthedocs.io/en/latest/pipeline.html#extending-the-pipeline](https://python-social-auth.readthedocs.io/en/latest/pipeline.html#extending-the-pipeline)
- **Pipeline de desconexión de Python Social Auth:** [https://python-social-auth.readthedocs.io/en/latest/pipeline.html#disconnection-pipeline](https://python-social-auth.readthedocs.io/en/latest/pipeline.html#disconnection-pipeline)
