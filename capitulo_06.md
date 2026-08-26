# Parte 2: Creación de un sitio web social

## Capítulo 6: Compartir contenido en tu sitio web

### Introducción

En el capítulo anterior, añadiste mensajes de éxito y error a tu sitio utilizando el framework de mensajes de Django. También creaste un backend de autenticación por correo electrónico y añadiste autenticación social a tu sitio utilizando Google. Aprendiste a ejecutar tu servidor de desarrollo con HTTPS en tu máquina local mediante Django Extensions. Personalizaste el pipeline de autenticación social para crear automáticamente un perfil de usuario para nuevos usuarios.

En este capítulo, aprenderás a crear un bookmarklet de JavaScript para compartir contenido de otros sitios en tu sitio web, e implementarás peticiones asíncronas del navegador en tu proyecto utilizando JavaScript y Django.

Este capítulo cubrirá los siguientes temas:

- Creación de relaciones de muchos a muchos (*many-to-many*)
- Personalización del comportamiento de los formularios
- Uso de JavaScript con Django
- Creación de un bookmarklet de JavaScript
- Generación de miniaturas de imágenes usando `easy-thumbnails`
- Implementación de peticiones HTTP asíncronas con JavaScript y Django
- Creación de paginación con desplazamiento infinito (*infinite scroll*)

En este capítulo, crearás un sistema de marcadores de imágenes. Crearás modelos con relaciones muchos a muchos y personalizarás el comportamiento de los formularios. Aprenderás a generar miniaturas de imágenes y a construir funcionalidades asíncronas del navegador utilizando JavaScript y Django.

---

### Visión general funcional

La Figura 6.1 muestra una representación de las vistas, plantillas y funcionalidades que se construirán en este capítulo:

> *Figura 6.1: Diagrama de funcionalidades construidas en el Capítulo 6*

En este capítulo, implementarás un botón *Bookmark it* que permitirá a los usuarios marcar imágenes de cualquier sitio web como favoritas. Utilizarás JavaScript para mostrar un selector de imágenes superpuesto en cualquier sitio web para que los usuarios elijan una imagen para marcar. Implementarás la vista `image_create` y un formulario para recuperar la imagen de su fuente original y almacenarla en tu sitio web. Construirás la vista `image_detail` para mostrar imágenes individuales y generarás miniaturas de imágenes automáticamente utilizando el paquete `easy-thumbnails`. También implementarás la vista `image_like` para permitir a los usuarios dar "me gusta" o "ya no me gusta" a las imágenes. Esta vista gestionará peticiones HTTP asíncronas realizadas con JavaScript y devolverá una respuesta en formato JSON. Finalmente, crearás la vista `image_list` para mostrar todas las imágenes marcadas e implementarás un desplazamiento infinito (*infinite scroll*) utilizando JavaScript y la paginación de Django.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter06](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter06).

Todos los paquetes de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente del capítulo. Puedes seguir las instrucciones para instalar cada paquete de Python en las siguientes secciones, o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Creación de un sitio web de marcadores de imágenes

Ahora aprenderemos a permitir que los usuarios guarden imágenes que encuentren en otros sitios web y las compartan en nuestro sitio. Para construir esta funcionalidad, necesitaremos los siguientes elementos:

- Un modelo de datos para almacenar imágenes e información relacionada.
- Un formulario y una vista para gestionar las subidas de imágenes.
- Código de bookmarklet de JavaScript que se pueda ejecutar en cualquier sitio web. Este código buscará imágenes en toda la página y permitirá a los usuarios seleccionar la imagen que desean guardar.

Primero, crea una nueva aplicación dentro del directorio de tu proyecto `bookmarks` ejecutando el siguiente comando en la consola:

```bash
django-admin startapp images
```

Añade la nueva aplicación a la configuración `INSTALLED_APPS` en el archivo `settings.py` del proyecto, de la siguiente manera:

```python
INSTALLED_APPS = [
    # ...
    'images.apps.ImagesConfig',
]
```

Hemos activado la aplicación `images` en el proyecto.

#### Creación del modelo de imagen

Edita el archivo `models.py` de la aplicación `images` y añade el siguiente código:

```python
from django.conf import settings
from django.db import models


class Image(models.Model):
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        related_name='images_created',
        on_delete=models.CASCADE
    )
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, blank=True)
    url = models.URLField(max_length=2000)
    image = models.ImageField(upload_to='images/%Y/%m/%d/')
    description = models.TextField(blank=True)
    created = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=['-created']),
        ]
        ordering = ['-created']

    def __str__(self):
        return self.title
```

Este es el modelo que utilizaremos para almacenar imágenes en la plataforma. Echemos un vistazo a los campos de este modelo:

- `user`: Indica el objeto `User` que marcó esta imagen. Este es un campo de clave foránea (*foreign key*) porque especifica una relación de muchos a uno: un usuario puede publicar múltiples imágenes, pero cada imagen es publicada por un único usuario. Hemos utilizado `CASCADE` para el parámetro `on_delete` para que las imágenes relacionadas se eliminen cuando se elimina un usuario.
- `title`: Un título para la imagen.
- `slug`: Una etiqueta corta que contiene solo letras, números, guiones bajos o guiones para ser utilizada en la construcción de URLs atractivas y amigables para SEO.
- `url`: La URL original de esta imagen. Usamos `max_length` para definir una longitud máxima de 2000 caracteres.
- `image`: El archivo de imagen.
- `description`: Una descripción opcional para la imagen.
- `created`: La fecha y hora que indican cuándo se creó el objeto en la base de datos. Hemos añadido `auto_now_add` para establecer automáticamente la fecha y hora actual cuando se crea el objeto.

En la clase `Meta` del modelo, hemos definido un índice de base de datos en orden descendente para el campo `created`. También hemos añadido el atributo `ordering` para indicarle a Django que debe ordenar los resultados por el campo `created` de forma predeterminada. Indicamos el orden descendente usando un guión antes del nombre del campo, como `-created`, para que las nuevas imágenes se muestren primero.

Los índices de base de datos mejoran el rendimiento de las consultas. Considera crear índices para campos que consultas con frecuencia usando `filter()`, `exclude()` o `order_by()`. Los campos `ForeignKey` o campos con `unique=True` implican la creación automática de un índice. Puedes obtener más información sobre los índices de base de datos en [https://docs.djangoproject.com/en/5.2/ref/models/options/#django.db.models.Options.indexes](https://docs.djangoproject.com/en/5.2/ref/models/options/#django.db.models.Options.indexes).

Sobrescribiremos el método `save()` del modelo `Image` para generar automáticamente el campo `slug` en función del valor del campo `title`. Importa la función `slugify()` y añade un método `save()` al modelo `Image`, de la siguiente manera:

```python
from django.utils.text import slugify


class Image(models.Model):
    # ...
    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
```

Cuando se guarda un objeto `Image`, si el campo `slug` no tiene un valor, se utiliza la función `slugify()` para generar automáticamente un slug a partir del campo `title` de la imagen. Luego se guarda el objeto. Al generar slugs automáticamente a partir del título, los usuarios no tendrán que proporcionar un slug cuando compartan imágenes en nuestro sitio web.

#### Creación de relaciones muchos a muchos

A continuación, añadiremos otro campo al modelo `Image` para almacenar los usuarios a los que les gusta una imagen. Necesitaremos una relación de muchos a muchos (*many-to-many*) en este caso porque a un usuario le pueden gustar múltiples imágenes y cada imagen puede gustarle a múltiples usuarios.

Añade el siguiente campo al modelo `Image`:

```python
    users_like = models.ManyToManyField(
        settings.AUTH_USER_MODEL,
        related_name='images_liked',
        blank=True
    )
```

Cuando definimos un campo `ManyToManyField`, Django crea una tabla intermedia de unión utilizando las claves primarias de ambos modelos. La Figura 6.2 muestra la tabla de base de datos que se creará para esta relación:

> *Figura 6.2: Tabla intermedia de base de datos para la relación de muchos a muchos*

La tabla `images_image_users_like` es creada por Django como una tabla intermedia que tiene referencias a la tabla `images_image` (modelo `Image`) y a la tabla `auth_user` (modelo `User`). El campo `ManyToManyField` se puede definir en cualquiera de los dos modelos relacionados.

Al igual que con los campos `ForeignKey`, el atributo `related_name` de `ManyToManyField` te permite nombrar la relación desde el objeto relacionado de vuelta a este. Los campos `ManyToManyField` proporcionan un gestor (*manager*) de muchos a muchos que te permite recuperar objetos relacionados, como `image.users_like.all()`, u obtenerlos desde un objeto de usuario, como `user.images_liked.all()`.

Puedes obtener más información sobre relaciones de muchos a muchos en [https://docs.djangoproject.com/en/5.2/topics/db/examples/many_to_many/](https://docs.djangoproject.com/en/5.2/topics/db/examples/many_to_many/).

Abre la consola y ejecuta el siguiente comando para crear una migración inicial:

```bash
python manage.py makemigrations images
```

La salida debería ser similar a la siguiente:

```text
Migrations for 'images':
  images/migrations/0001_initial.py
    - Create model Image
    - Create index images_imag_created_d57897_idx on field(s) -created of model image
```

Ahora ejecuta el siguiente comando para aplicar tu migración:

```bash
python manage.py migrate images
```

Obtendrás una salida que incluye la siguiente línea:

```text
Applying images.0001_initial... OK
```

El modelo `Image` ahora está sincronizado con la base de datos.

#### Registro del modelo de imagen en el sitio de administración

Edita el archivo `admin.py` de la aplicación `images` y registra el modelo `Image` en el sitio de administración, de la siguiente manera:

```python
from django.contrib import admin
from .models import Image


@admin.register(Image)
class ImageAdmin(admin.ModelAdmin):
    list_display = ['title', 'slug', 'image', 'created']
    list_filter = ['created']
```

Inicia el servidor de desarrollo con el siguiente comando:

```bash
python manage.py runserver_plus --cert-file cert.crt
```

Abre `https://127.0.0.1:8000/admin/` en tu navegador y verás el modelo `Image` en el sitio de administración, así:

> *Figura 6.3: El bloque Images en la página de índice del sitio de administración de Django*

Has completado el modelo para almacenar imágenes. Ahora aprenderás a implementar un formulario para recuperar imágenes por su URL y almacenarlas utilizando el modelo `Image`.

---

### Publicación de contenido desde otros sitios web

Permitiremos a los usuarios marcar imágenes de sitios web externos como favoritas y compartirlas en nuestro sitio. Los usuarios proporcionarán la URL de la imagen, un título y una descripción opcional. Crearemos un formulario y una vista para descargar la imagen y crear un nuevo objeto `Image` en la base de datos.

Comencemos construyendo un formulario para enviar nuevas imágenes.

Crea un nuevo archivo `forms.py` dentro del directorio de la aplicación `images` y añade el siguiente código:

```python
from django import forms
from .models import Image


class ImageCreateForm(forms.ModelForm):
    class Meta:
        model = Image
        fields = ['title', 'url', 'description']
        widgets = {
            'url': forms.HiddenInput,
        }
```

Hemos definido un formulario `ModelForm` a partir del modelo `Image`, incluyendo solo los campos `title`, `url` y `description`. Los usuarios no introducirán la URL de la imagen directamente en el formulario. En su lugar, les proporcionaremos una herramienta de JavaScript para elegir una imagen de un sitio externo, y el formulario recibirá la URL de la imagen como parámetro. Hemos sobrescrito el widget predeterminado del campo `url` para usar un widget `HiddenInput`. Este widget se renderiza como un elemento HTML input con un atributo `type="hidden"`. Usamos este widget porque no queremos que este campo sea visible para los usuarios.

#### Limpieza de campos de formulario

Para verificar que la URL de la imagen proporcionada sea válida, comprobaremos que el nombre del archivo termine con una extensión `.jpg`, `.jpeg` o `.png` para permitir compartir únicamente archivos JPEG y PNG. En el capítulo anterior, utilizamos la convención `clean_<fieldname>()` para implementar la validación de campos. Este método se ejecuta para cada campo, si el campo está presente, cuando llamamos a `is_valid()` en una instancia de formulario. En el método clean, puedes modificar el valor del campo o generar cualquier error de validación para el campo.

En el archivo `forms.py` de la aplicación `images`, añade el siguiente método a la clase `ImageCreateForm`:

```python
    def clean_url(self):
        url = self.cleaned_data['url']
        valid_extensions = ['jpg', 'jpeg', 'png']
        extension = url.rsplit('.', 1)[1].lower()
        if extension not in valid_extensions:
            raise forms.ValidationError(
                'The given URL does not match valid image extensions.'
            )
        return url
```

En el código anterior, hemos definido un método `clean_url()` para limpiar el campo `url`. El código funciona de la siguiente manera:

- El valor del campo `url` se recupera accediendo al diccionario `cleaned_data` de la instancia del formulario.
- La URL se divide para comprobar si el archivo tiene una extensión válida. Si la extensión no es válida, se genera un `ValidationError` y la instancia del formulario no se valida.

Además de validar la URL dada, también necesitamos descargar el archivo de imagen y guardarlo. Podríamos, por ejemplo, utilizar la vista que gestiona el formulario para descargar el archivo de imagen. En su lugar, adoptaremos un enfoque más general sobrescribiendo el método `save()` del formulario de modelo para realizar esta tarea cuando se guarde el formulario.

#### Instalación de la biblioteca Requests

Cuando un usuario marca una imagen, necesitaremos descargar el archivo de imagen mediante su URL. Utilizaremos la biblioteca de Python Requests para este propósito. Requests es la biblioteca HTTP más popular para Python. Abstrae la complejidad de tratar con peticiones HTTP y proporciona una interfaz muy simple para consumir servicios HTTP. Puedes encontrar la documentación de la biblioteca Requests en [https://requests.readthedocs.io/en/master/](https://requests.readthedocs.io/en/master/).

Abre la consola e instala la biblioteca Requests con el siguiente comando:

```bash
python -m pip install requests==2.31.0
```

Ahora sobrescribiremos el método `save()` de `ImageCreateForm` y utilizaremos la biblioteca Requests para recuperar la imagen por su URL.

#### Sobrescritura del método save() de un ModelForm

Como sabes, `ModelForm` proporciona un método `save()` para guardar la instancia de modelo actual en la base de datos y devolver el objeto. Este método recibe un parámetro booleano `commit`, que te permite especificar si el objeto debe persistir en la base de datos. Si `commit` es `False`, el método `save()` devolverá una instancia de modelo pero no la guardará en la base de datos. Sobrescribiremos el método `save()` del formulario para recuperar el archivo de imagen mediante la URL dada y guardarlo en el sistema de archivos.

Añade las siguientes importaciones en la parte superior del archivo `forms.py`:

```python
import requests
from django.core.files.base import ContentFile
from django.utils.text import slugify
```

Luego, añade el siguiente método `save()` al formulario `ImageCreateForm`:

```python
    def save(self, force_insert=False, force_update=False, commit=True):
        image = super().save(commit=False)
        image_url = self.cleaned_data['url']
        name = slugify(image.title)
        extension = image_url.rsplit('.', 1)[1].lower()
        image_name = f'{name}.{extension}'
        # download image from the given URL
        response = requests.get(image_url)
        image.image.save(
            image_name,
            ContentFile(response.content),
            save=False
        )
        if commit:
            image.save()
        return image
```

Hemos sobrescrito el método `save()`, manteniendo los parámetros requeridos por `ModelForm`. El código anterior se puede explicar de la siguiente manera:

- Se crea una nueva instancia de imagen llamando al método `save()` del formulario con `commit=False`.
- La URL de la imagen se recupera del diccionario `cleaned_data` del formulario.
- Se genera un nombre de imagen combinando el slug del título de la imagen con la extensión de archivo original de la imagen.
- Se utiliza la biblioteca de Python Requests para descargar la imagen enviando una petición HTTP GET utilizando la URL de la imagen. La respuesta se almacena en el objeto `response`.
- Se llama al método `save()` del campo de imagen, pasándole un objeto `ContentFile` que se instancia con el contenido del archivo descargado. De esta manera, el archivo se guarda en el directorio `media` del proyecto. El parámetro `save=False` se pasa para evitar que el objeto se guarde en la base de datos.
- Para mantener el mismo comportamiento que el método `save()` original del formulario de modelo, el formulario solo se guarda en la base de datos si el parámetro `commit` es `True`.

Necesitaremos una vista para crear una instancia del formulario y gestionar su envío.

Edita el archivo `views.py` de la aplicación `images` y añádele el siguiente código:

```python
from django.contrib import messages
from django.contrib.auth.decorators import login_required
from django.shortcuts import redirect, render
from .forms import ImageCreateForm


@login_required
def image_create(request):
    if request.method == 'POST':
        # form is sent
        form = ImageCreateForm(data=request.POST)
        if form.is_valid():
            # form data is valid
            cd = form.cleaned_data
            new_image = form.save(commit=False)
            # assign current user to the item
            new_image.user = request.user
            new_image.save()
            messages.success(request, 'Image added successfully')
            # redirect to new created item detail view
            return redirect(new_image.get_absolute_url())
    else:
        # build form with data provided by the bookmarklet via GET
        form = ImageCreateForm(data=request.GET)
    return render(
        request,
        'images/image/create.html',
        {'section': 'images', 'form': form}
    )
```

En el código anterior, hemos creado una vista para almacenar imágenes en el sitio. Hemos añadido el decorador `login_required` a la vista `image_create` para evitar el acceso a usuarios no autenticados. Así es como funciona esta vista:

- Los datos iniciales deben proporcionarse a través de una petición HTTP GET para crear una instancia del formulario. Estos datos consistirán en los atributos `url` y `title` de una imagen de un sitio web externo. Ambos parámetros serán establecidos en la petición GET por el bookmarklet de JavaScript que crearemos más adelante. Por ahora, podemos asumir que estos datos estarán disponibles en la petición.
- Cuando el formulario se envía con una petición HTTP POST, se valida con `form.is_valid()`. Si los datos del formulario son válidos, se crea una nueva instancia de imagen guardando el formulario con `form.save(commit=False)`. La nueva instancia no se guarda en la base de datos debido a `commit=False`.
- Se añade una relación con el usuario actual que realiza la petición a la nueva instancia de imagen con `new_image.user = request.user`. Así es como sabremos quién subió cada imagen.
- El objeto `Image` se guarda en la base de datos.
- Finalmente, se crea un mensaje de éxito utilizando el framework de mensajes de Django y el usuario es redirigido a la URL canónica de la nueva imagen. Todavía no hemos implementado el método `get_absolute_url()` del modelo `Image`; lo haremos más adelante.

Crea un nuevo archivo `urls.py` dentro de la aplicación `images` y añádele el siguiente código:

```python
from django.urls import path
from . import views

app_name = 'images'

urlpatterns = [
    path('create/', views.image_create, name='create'),
]
```

Edita el archivo `urls.py` principal del proyecto `bookmarks` para incluir los patrones para la aplicación `images`, de la siguiente manera:

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('account/', include('account.urls')),
    path(
        'social-auth/',
        include('social_django.urls', namespace='social')
    ),
    path('images/', include('images.urls', namespace='images')),
]
```

Finalmente, necesitamos crear una plantilla para renderizar el formulario. Crea la siguiente estructura de directorios dentro del directorio de la aplicación `images`:

```text
templates/
    images/
        image/
            create.html
```

Edita la nueva plantilla `create.html` y añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Bookmark an image{% endblock %}

{% block content %}
    <h1>Bookmark an image</h1>
    <img src="{{ request.GET.url }}" class="image-preview">
    <form method="post">
        {{ form.as_p }}
        {% csrf_token %}
        <input type="submit" value="Bookmark it!">
    </form>
{% endblock %}
```

Ejecuta el servidor de desarrollo con el siguiente comando en la consola:

```bash
python manage.py runserver_plus --cert-file cert.crt
```

Abre `https://127.0.0.1:8000/images/create/?title=...&url=...` en tu navegador, incluyendo los parámetros GET `title` y `url`, proporcionando una URL de imagen JPEG existente en este último. Por ejemplo, puedes utilizar la siguiente URL: `https://127.0.0.1:8000/images/create/?title=%20Django%20and%20Duke&url=https://upload.wikimedia.org/wikipedia/commons/8/85/Django_Reinhardt_and_Duke_Ellington_%28Gottlieb%29.jpg`.

Verás el formulario con una vista previa de la imagen, como la siguiente:

> *Figura 6.4: La página de marcadores Bookmark an image*

Añade una descripción y haz clic en el botón **BOOKMARK IT!**. Se guardará un nuevo objeto `Image` en tu base de datos. Sin embargo, obtendrás un error que indica que el modelo `Image` no tiene ningún método `get_absolute_url()`, de la siguiente manera:

> *Figura 6.5: Un error que muestra que el objeto Image no tiene el atributo get_absolute_url*

No te preocupes por este error por ahora; implementaremos el método `get_absolute_url` en el modelo `Image` más adelante.

Abre `https://127.0.0.1:8000/admin/images/image/` en tu navegador y verifica que el nuevo objeto `Image` se haya guardado:

> *Figura 6.6: La página de lista de imágenes del sitio de administración mostrando el objeto Image creado*

#### Creación de un bookmarklet con JavaScript

Un bookmarklet es un marcador guardado en un navegador web que contiene código JavaScript para extender la funcionalidad del navegador. Cuando haces clic en el marcador en la barra de marcadores o favoritos de tu navegador, el código JavaScript se ejecuta en el sitio web que se muestra en el navegador. Esto es muy útil para construir herramientas que interactúen con otros sitios web.

Algunos servicios en línea, como Pinterest, implementan su propio bookmarklet para permitir a los usuarios compartir contenido de otros sitios en sus plataformas. El bookmarklet de Pinterest está implementado como una extensión de navegador y está disponible en [https://help.pinterest.com/en/article/save-pins-with-the-pinterest-browser-button](https://help.pinterest.com/en/article/save-pins-with-the-pinterest-browser-button). La extensión Pinterest Save permite a los usuarios guardar imágenes o sitios web en su cuenta de Pinterest con solo un clic en el navegador.

> *Figura 6.7: La extensión Pinterest Save*

Creemos un bookmarklet de manera similar para tu sitio web. Para ello, utilizaremos JavaScript.

Así es como tus usuarios añadirán el bookmarklet a su navegador y lo utilizarán:

- El usuario arrastra un enlace desde tu sitio a la barra de marcadores de su navegador. El enlace contiene código JavaScript en su atributo `href`. Este código se almacenará en el marcador.
- El usuario navega a cualquier sitio web y hace clic en el marcador en la barra de marcadores o favoritos. Se ejecuta el código JavaScript del marcador.

Dado que el código JavaScript se almacenará como un marcador, no podremos actualizarlo después de que el usuario lo haya añadido a su barra de marcadores. Este es un inconveniente importante que puedes resolver implementando un script lanzador (*launcher script*). Los usuarios guardarán el script lanzador como un marcador, y el script lanzador cargará el bookmarklet de JavaScript real desde una URL. Al hacer esto, podrás actualizar el código del bookmarklet en cualquier momento. Este es el enfoque que adoptaremos para construir el bookmarklet. ¡Comencemos!

Crea una nueva plantilla bajo `images/templates/` y nómbrala `bookmarklet_launcher.js`. Este será el script lanzador. Añade el siguiente código JavaScript al nuevo archivo:

```javascript
(function(){
    if(!window.bookmarklet) {
        bookmarklet_js = document.body.appendChild(document.createElement('script'));
        bookmarklet_js.src = '//127.0.0.1:8000/static/js/bookmarklet.js?r='+Math.floor(Math.random()*9999999999999999);
        window.bookmarklet = true;
    }
    else {
        bookmarkletLaunch();
    }
})();
```

El script anterior comprueba si el bookmarklet ya se ha cargado verificando el valor de la variable de ventana `bookmarklet` con `if(!window.bookmarklet)`:

- Si `window.bookmarklet` no está definido o no tiene un valor verdadero (*truthy*), se carga un archivo JavaScript añadiendo un elemento `<script>` al cuerpo del documento HTML cargado en el navegador. El atributo `src` se utiliza para cargar la URL del script `bookmarklet.js` con un parámetro entero aleatorio de 16 dígitos generado con `Math.random()*9999999999999999`. Al utilizar un número aleatorio, evitamos que el navegador cargue el archivo desde la memoria caché del navegador. Si el JavaScript del bookmarklet se ha cargado previamente, el valor diferente del parámetro obligará al navegador a cargar el script desde la URL de origen nuevamente. De esta manera, nos aseguramos de que el bookmarklet siempre ejecute el código JavaScript más actualizado.
- Si `window.bookmarklet` está definido y tiene un valor verdadero, se ejecuta la función `bookmarkletLaunch()`. Definiremos `bookmarkletLaunch()` como una función global en el script `bookmarklet.js`.

Al comprobar la variable de ventana `bookmarklet`, evitamos que el código JavaScript del bookmarklet se cargue más de una vez si los usuarios hacen clic en el bookmarklet repetidamente.

Has creado el código del lanzador del bookmarklet. El código real del bookmarklet residirá en el archivo estático `bookmarklet.js`. El uso de código lanzador te permite actualizar el código del bookmarklet en cualquier momento sin requerir que los usuarios cambien el marcador que añadieron previamente a su navegador.

Añadamos el lanzador del bookmarklet a las páginas del panel de control para que los usuarios puedan añadirlo a la barra de marcadores de su navegador.

Edita la plantilla `account/dashboard.html` de la aplicación `account` y haz que se vea de la siguiente manera:

```html
{% extends "base.html" %}

{% block title %}Dashboard{% endblock %}

{% block content %}
    <h1>Dashboard</h1>
    {% with total_images_created=request.user.images_created.count %}
        <p>Welcome to your dashboard. You have bookmarked {{ total_images_created }} image{{ total_images_created|pluralize }}.</p>
    {% endwith %}
    <p>Drag the following button to your bookmarks toolbar to bookmark images from other websites → <a href="javascript:{% include "bookmarklet_launcher.js" %}" class="button">Bookmark it</a></p>
    <p>You can also <a href="{% url "edit" %}">edit your profile</a> or <a href="{% url "password_change" %}">change your password</a>.</p>
{% endblock %}
```

Asegúrate de que ninguna etiqueta de plantilla se divida en varias líneas; Django no admite etiquetas de varias líneas.

El panel de control ahora muestra el número total de imágenes marcadas por el usuario. Hemos añadido una etiqueta de plantilla `{% with %}` para crear una variable con el número total de imágenes marcadas por el usuario actual. Hemos incluido un enlace con un atributo `href` que contiene el script lanzador del bookmarklet. Este código JavaScript se carga desde la plantilla `bookmarklet_launcher.js`.

Abre `https://127.0.0.1:8000/account/` en tu navegador. Deberías ver la siguiente página:

> *Figura 6.8: La página del panel de control, incluyendo el total de imágenes marcadas y el botón para el bookmarklet*

Ahora crea los siguientes directorios y archivos dentro del directorio de la aplicación `images`:

```text
static/
    js/
        bookmarklet.js
```

Encontrarás un directorio `static/css/` en el directorio de la aplicación `images` en el código que acompaña a este capítulo. Copia el directorio `css/` en el directorio `static/` de tu código. Puedes encontrar el contenido del directorio en [https://github.com/PacktPublishing/Django-5-by-Example/tree/main/Chapter06/bookmarks/images/static](https://github.com/PacktPublishing/Django-5-by-Example/tree/main/Chapter06/bookmarks/images/static).

El archivo `css/bookmarklet.css` proporciona los estilos para el bookmarklet de JavaScript. El directorio `static/` ahora debería contener la siguiente estructura de archivos:

```text
css/
    bookmarklet.css
js/
    bookmarklet.js
```

Edita el archivo estático `bookmarklet.js` y añádele el siguiente código JavaScript:

```javascript
const siteUrl = '//127.0.0.1:8000/';
const styleUrl = siteUrl + 'static/css/bookmarklet.css';
const minWidth = 250;
const minHeight = 250;
```

Has declarado cuatro constantes diferentes que serán utilizadas por el bookmarklet. Estas constantes son:

- `siteUrl` y `styleUrl`: La URL base para el sitio web y la URL base para archivos estáticos.
- `minWidth` y `minHeight`: El ancho y alto mínimos en píxeles para las imágenes que el bookmarklet recopilará del sitio. El bookmarklet identificará imágenes que tengan al menos 250px de ancho y 250px de alto.

Edita el archivo estático `bookmarklet.js` y añade el siguiente código:

```javascript
const siteUrl = '//127.0.0.1:8000/';
const styleUrl = siteUrl + 'static/css/bookmarklet.css';
const minWidth = 250;
const minHeight = 250;

// load CSS
var head = document.getElementsByTagName('head')[0];
var link = document.createElement('link');
link.rel = 'stylesheet';
link.type = 'text/css';
link.href = styleUrl + '?r=' + Math.floor(Math.random()*9999999999999999);
head.appendChild(link);
```

Esta sección carga la hoja de estilos CSS para el bookmarklet. Usamos JavaScript para manipular el Modelo de Objetos del Documento (DOM). El DOM representa un documento HTML en memoria y es creado por el navegador cuando se carga una página web. El DOM se construye como un árbol de objetos que componen la estructura y el contenido del documento HTML.

El código anterior genera un objeto equivalente al siguiente código JavaScript y lo añade al elemento `<head>` de la página HTML:

```html
<link rel="stylesheet" type="text/css" href="//127.0.0.1:8000/static/css/bookmarklet.css?r=1234567890123456">
```

Revisemos cómo se hace esto:

- El elemento `<head>` del sitio se recupera con `document.getElementsByTagName()`. Esta función recupera todos los elementos HTML de la página con la etiqueta dada. Al usar `[0]`, accedemos a la primera instancia encontrada. Accedemos al primer elemento porque todos los documentos HTML deben tener un único elemento `<head>`.
- Se crea un elemento `<link>` con `document.createElement('link')`.
- Se establecen los atributos `rel` y `type` del elemento `<link>`. Esto es equivalente al HTML `<link rel="stylesheet" type="text/css">`.
- El atributo `href` del elemento `<link>` se establece con la URL de la hoja de estilos `bookmarklet.css`. Se utiliza un número aleatorio de 16 dígitos como parámetro de URL para evitar que el navegador cargue el archivo desde la memoria caché.
- El nuevo elemento `<link>` se añade al elemento `<head>` de la página HTML utilizando `head.appendChild(link)`.

Ahora crearemos el elemento HTML para mostrar un contenedor `<div>` en el sitio web donde se ejecuta el bookmarklet. El contenedor HTML se utilizará para mostrar todas las imágenes encontradas en el sitio y permitir a los usuarios elegir la imagen que desean compartir. Utilizará los estilos CSS definidos en la hoja de estilos `bookmarklet.css`.

Edita el archivo estático `bookmarklet.js` y añade el siguiente código:

```javascript
const siteUrl = '//127.0.0.1:8000/';
const styleUrl = siteUrl + 'static/css/bookmarklet.css';
const minWidth = 250;
const minHeight = 250;

// load CSS
var head = document.getElementsByTagName('head')[0];
var link = document.createElement('link');
link.rel = 'stylesheet';
link.type = 'text/css';
link.href = styleUrl + '?r=' + Math.floor(Math.random()*9999999999999999);
head.appendChild(link);

// load HTML
var body = document.getElementsByTagName('body')[0];
boxHtml = ' \
  <div id="bookmarklet"> \
    <a href="#" id="close">&times;</a> \
    <h1>Select an image to bookmark:</h1> \
    <div class="images"></div> \
  </div>';
body.innerHTML += boxHtml;
```

Con este código, se recupera el elemento `<body>` del DOM y se le añade nuevo HTML modificando su propiedad `innerHTML`. Se añade un nuevo elemento `<div>` al cuerpo de la página. El contenedor `<div>` consta de los siguientes elementos:

- Un enlace para cerrar el contenedor definido con `<a href="#" id="close">&times;</a>`.
- Un título definido con `<h1>Select an image to bookmark:</h1>`.
- Un elemento `<div>` para listar las imágenes encontradas en el sitio definido con `<div class="images"></div>`. Este contenedor está inicialmente vacío y se llenará con las imágenes encontradas en el sitio.

El contenedor HTML, incluidos los estilos CSS cargados previamente, se verá como en la Figura 6.9:

> *Figura 6.9: El contenedor de selección de imágenes*

Ahora implementemos una función para lanzar el bookmarklet. Edita el archivo estático `bookmarklet.js` y añade el siguiente código en la parte inferior:

```javascript
function bookmarkletLaunch() {
    bookmarklet = document.getElementById('bookmarklet');
    var imagesFound = bookmarklet.querySelector('.images');
    // clear images found
    imagesFound.innerHTML = '';
    // display bookmarklet
    bookmarklet.style.display = 'block';
    // close event
    bookmarklet.querySelector('#close')
               .addEventListener('click', function(){
                   bookmarklet.style.display = 'none'
               });
}

// launch the bookmkarklet
bookmarkletLaunch();
```

Esta es la función `bookmarkletLaunch()`. Antes de la definición de esta función, se carga el CSS para el bookmarklet y se añade el contenedor HTML al DOM de la página. La función `bookmarkletLaunch()` funciona de la siguiente manera:

- El contenedor principal del bookmarklet se recupera obteniendo el elemento DOM con el ID `bookmarklet` mediante `document.getElementById()`.
- El elemento `bookmarklet` se utiliza para recuperar el elemento secundario con la clase `images`. El método `querySelector()` te permite recuperar elementos del DOM utilizando selectores CSS. Los selectores te permiten encontrar elementos del DOM a los que se aplica un conjunto de reglas CSS. Puedes encontrar una lista de selectores CSS en [https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Selectors](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Selectors) y puedes leer más información sobre cómo localizar elementos del DOM utilizando selectores en [https://developer.mozilla.org/en-US/docs/Web/API/Document_object_model/Locating_DOM_elements_using_selectors](https://developer.mozilla.org/en-US/docs/Web/API/Document_object_model/Locating_DOM_elements_using_selectors).
- El contenedor de imágenes se borra estableciendo su atributo `innerHTML` en una cadena vacía y el bookmarklet se muestra estableciendo la propiedad CSS `display` en `block`.
- El selector `#close` se utiliza para encontrar el elemento del DOM con el ID `close`. Se asocia un evento de clic al elemento con el método `addEventListener()`. Cuando los usuarios hacen clic en el elemento, el contenedor principal del bookmarklet se oculta estableciendo su propiedad `display` en `none`.
- La función `bookmarkletLaunch()` se ejecuta tras su definición.

Después de cargar los estilos CSS y el contenedor HTML del bookmarklet, debes encontrar los elementos de imagen en el DOM del sitio web actual. Las imágenes que tengan la dimensión mínima requerida deben añadirse al contenedor HTML del bookmarklet. Edita el archivo estático `bookmarklet.js` y añade el siguiente código en la parte inferior de la función `bookmarkletLaunch()`:

```javascript
function bookmarkletLaunch() {
    bookmarklet = document.getElementById('bookmarklet');
    var imagesFound = bookmarklet.querySelector('.images');
    // clear images found
    imagesFound.innerHTML = '';
    // display bookmarklet
    bookmarklet.style.display = 'block';
    // close event
    bookmarklet.querySelector('#close')
               .addEventListener('click', function(){
                   bookmarklet.style.display = 'none'
               });

    // find images in the DOM with the minimum dimensions
    images = document.querySelectorAll('img[src$=".jpg"], img[src$=".jpeg"], img[src$=".png"]');
    images.forEach(image => {
        if(image.naturalWidth >= minWidth && image.naturalHeight >= minHeight) {
            var imageFound = document.createElement('img');
            imageFound.src = image.src;
            imagesFound.append(imageFound);
        }
    })
}

// launch the bookmkarklet
bookmarkletLaunch();
```

El código anterior utiliza los selectores `img[src$=".jpg"]`, `img[src$=".jpeg"]` y `img[src$=".png"]` para encontrar todos los elementos `<img>` del DOM cuyo atributo `src` termine con `.jpg`, `.jpeg` o `.png`, respectivamente. El uso de estos selectores con `document.querySelectorAll()` te permite encontrar todas las imágenes en formato JPEG y PNG que se muestran en el sitio web. La iteración sobre los resultados se realiza con el método `forEach()`. Las imágenes pequeñas se descartan porque no las consideramos relevantes. Solo las imágenes con un tamaño superior al especificado con las variables `minWidth` y `minHeight` se utilizan para los resultados. Se crea un nuevo elemento `<img>` para cada imagen encontrada, donde el atributo de URL de origen `src` se copia de la imagen original y se añade al contenedor `imagesFound`.

Por razones de seguridad, tu navegador te impedirá ejecutar el bookmarklet a través de HTTP en un sitio servido mediante HTTPS. Esa es la razón por la que seguimos usando `RunServerPlus` para ejecutar el servidor de desarrollo utilizando un certificado TLS/SSL generado automáticamente. Recuerda que aprendiste a ejecutar el servidor de desarrollo a través de HTTPS en el Capítulo 5, *Implementación de autenticación social*.

En un entorno de producción, se requerirá un certificado TLS/SSL válido. Cuando posees un nombre de dominio, puedes solicitar que una Autoridad de Certificación (CA) de confianza emita un certificado TLS/SSL para él, de modo que los navegadores puedan verificar su identidad. Si deseas obtener un certificado confiable para un dominio real, puedes utilizar el servicio Let's Encrypt. Let's Encrypt es una CA sin fines de lucro que simplifica la obtención y renovación de certificados TLS/SSL de confianza de forma gratuita. Puedes encontrar más información en [https://letsencrypt.org](https://letsencrypt.org/).

Ejecuta el servidor de desarrollo con el siguiente comando desde la consola:

```bash
python manage.py runserver_plus --cert-file cert.crt
```

Abre `https://127.0.0.1:8000/account/` en tu navegador. Inicia sesión con un usuario existente, luego haz clic y arrastra el botón **BOOKMARK IT** a la barra de marcadores de tu navegador, de la siguiente manera:

> *Figura 6.10: Adición del botón BOOKMARK IT a la barra de marcadores*

Abre un sitio web de tu elección en tu navegador y haz clic en el bookmarklet **Bookmark it** en la barra de marcadores. Verás que aparece una nueva superposición blanca en el sitio web, mostrando todas las imágenes JPEG y PNG encontradas con dimensiones superiores a 250×250 píxeles.

La Figura 6.11 muestra el bookmarklet ejecutándose en [https://amazon.com/](https://amazon.com/):

> *Figura 6.11: El bookmarklet cargado en amazon.com*

Si el contenedor HTML no aparece, comprueba el registro de la consola del shell de `RunServer`. Si ves un error de tipo MIME, lo más probable es que tus archivos de mapeo MIME sean incorrectos o deban actualizarse. Puedes aplicar el mapeo correcto para archivos JavaScript y CSS añadiendo las siguientes líneas al archivo `settings.py`:

```python
if DEBUG:
    import mimetypes
    mimetypes.add_type('application/javascript', '.js', True)
    mimetypes.add_type('text/css', '.css', True)
```

El contenedor HTML incluye las imágenes que se pueden marcar. Ahora implementaremos la funcionalidad para que los usuarios hagan clic en la imagen deseada para guardarla.

Edita el archivo estático `js/bookmarklet.js` y añade el siguiente código en la parte inferior de la función `bookmarkletLaunch()`:

```javascript
function bookmarkletLaunch() {
    bookmarklet = document.getElementById('bookmarklet');
    var imagesFound = bookmarklet.querySelector('.images');
    // clear images found
    imagesFound.innerHTML = '';
    // display bookmarklet
    bookmarklet.style.display = 'block';
    // close event
    bookmarklet.querySelector('#close')
               .addEventListener('click', function(){
                   bookmarklet.style.display = 'none'
               });

    // find images in the DOM with the minimum dimensions
    images = document.querySelectorAll('img[src$=".jpg"], img[src$=".jpeg"], img[src$=".png"]');
    images.forEach(image => {
        if(image.naturalWidth >= minWidth && image.naturalHeight >= minHeight) {
            var imageFound = document.createElement('img');
            imageFound.src = image.src;
            imagesFound.append(imageFound);
        }
    })

    // select image event
    imagesFound.querySelectorAll('img').forEach(image => {
        image.addEventListener('click', function(event){
            imageSelected = event.target;
            bookmarklet.style.display = 'none';
            window.open(siteUrl + 'images/create/?url='
                        + encodeURIComponent(imageSelected.src)
                        + '&title='
                        + encodeURIComponent(document.title),
                        '_blank');
        })
    })
}

// launch the bookmkarklet
bookmarkletLaunch();
```

El código anterior funciona de la siguiente manera:

- Se asocia un evento `click` a cada elemento de imagen dentro del contenedor `imagesFound`.
- Cuando el usuario hace clic en cualquiera de las imágenes, el elemento de imagen en el que se hizo clic se almacena en la variable `imageSelected`.
- Luego, el bookmarklet se oculta estableciendo su propiedad `display` en `none`.
- Se abre una nueva ventana del navegador con la URL para marcar una nueva imagen en el sitio. El contenido del elemento `<title>` del sitio web se pasa a la URL en el parámetro GET `title` y la URL de la imagen seleccionada se pasa en el parámetro `url`.

Abre una nueva URL con tu navegador, por ejemplo, [https://commons.wikimedia.org/](https://commons.wikimedia.org/), de la siguiente manera:

> *Figura 6.12: El sitio web Wikimedia Commons (Imagen: Fila de grullas (Grus grus) en el Valle de Hula, norte de Israel por Tomere, Licencia: Creative Commons Attribution-Share Alike 4.0 International)*

Haz clic en el bookmarklet **Bookmark it** para mostrar la superposición de selección de imágenes. Verás la superposición de selección de imágenes de esta manera:

> *Figura 6.13: El bookmarklet cargado en un sitio web externo*

Si haces clic en una imagen, serás redirigido a la página de creación de imágenes, pasando el título del sitio web y la URL de la imagen seleccionada como parámetros GET. La página se verá de la siguiente manera:

> *Figura 6.14: El formulario para guardar una imagen*

¡Felicitaciones! Este es tu primer bookmarklet de JavaScript y está completamente integrado en tu proyecto Django. A continuación, crearemos la vista de detalle para imágenes e implementaremos la URL canónica para imágenes.

---

### Creación de la vista de detalle para imágenes

Ahora creemos una vista de detalle simple para mostrar las imágenes que se han guardado en el sitio. Abre el archivo `views.py` de la aplicación `images` y añádele el siguiente código:

```python
from django.shortcuts import get_object_or_404
from .models import Image


def image_detail(request, id, slug):
    image = get_object_or_404(Image, id=id, slug=slug)
    return render(
        request,
        'images/image/detail.html',
        {'section': 'images', 'image': image}
    )
```

Esta es una vista simple para mostrar una imagen. Edita el archivo `urls.py` de la aplicación `images` y añade el siguiente patrón de URL:

```python
urlpatterns = [
    path('create/', views.image_create, name='create'),
    path(
        'detail/<int:id>/<slug:slug>/',
        views.image_detail,
        name='detail'
    ),
]
```

Edita el archivo `models.py` de la aplicación `images` y añade el método `get_absolute_url()` al modelo `Image`, de la siguiente manera:

```python
from django.urls import reverse


class Image(models.Model):
    # ...
    def get_absolute_url(self):
        return reverse('images:detail', args=[self.id, self.slug])
```

Recuerda que el patrón común para proporcionar URLs canónicas para objetos es definir un método `get_absolute_url()` en el modelo.

Finalmente, crea una plantilla dentro del directorio de plantillas `templates/images/image/` para la aplicación `images` y nómbrala `detail.html`. Añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}{{ image.title }}{% endblock %}

{% block content %}
    <h1>{{ image.title }}</h1>
    <img src="{{ image.image.url }}" class="image-detail">
    {% with total_likes=image.users_like.count %}
        <div class="image-info">
            <div>
                <span class="count">
                    {{ total_likes }} like{{ total_likes|pluralize }}
                </span>
            </div>
            {{ image.description|linebreaks }}
        </div>
        <div class="image-likes">
            {% for user in image.users_like.all %}
                <div>
                    {% if user.profile.photo %}
                        <img src="{{ user.profile.photo.url }}">
                    {% endif %}
                    <p>{{ user.first_name }}</p>
                </div>
            {% empty %}
                Nobody likes this image yet.
            {% endfor %}
        </div>
    {% endwith %}
{% endblock %}
```

Esta es la plantilla para mostrar la vista de detalle de una imagen guardada. Hemos utilizado la etiqueta `{% with %}` para crear la variable `total_likes` con el resultado de un QuerySet que cuenta todos los "me gusta" de los usuarios. Al hacerlo, evitamos evaluar el mismo QuerySet dos veces (primero para mostrar el número total de "me gusta", luego para usar el filtro de plantilla `pluralize`). También hemos incluido la descripción de la imagen y hemos añadido un bucle `{% for %}` para iterar sobre `image.users_like.all` para mostrar a todos los usuarios a los que les gusta esta imagen.

> [!TIP]
> Siempre que necesites repetir una consulta en tu plantilla, utiliza la etiqueta de plantilla `{% with %}` para evitar consultas adicionales a la base de datos.

Ahora, abre una URL externa en tu navegador y utiliza el bookmarklet para guardar una nueva imagen. Serás redirigido a la página de detalle de la imagen después de publicar la imagen. La página incluirá un mensaje de éxito, de la siguiente manera:

> *Figura 6.15: La página de detalle de la imagen para el marcador de imagen*

¡Genial! Has completado la funcionalidad del bookmarklet. A continuación, aprenderás a crear miniaturas (*thumbnails*) para imágenes.

---

### Creación de miniaturas de imágenes usando easy-thumbnails

Estamos mostrando la imagen original en la página de detalle, pero las dimensiones de las diferentes imágenes pueden variar considerablemente. El tamaño de archivo de algunas imágenes puede ser muy grande y cargarlas puede llevar demasiado tiempo. La mejor manera de mostrar imágenes optimizadas de manera uniforme es generar miniaturas (*thumbnails*). Una miniatura es una representación pequeña de una imagen más grande. Las miniaturas se cargarán más rápido en el navegador y son una excelente manera de homogeneizar imágenes de tamaños muy diferentes. Utilizaremos una aplicación de Django llamada `easy-thumbnails` para generar miniaturas para las imágenes guardadas por los usuarios.

Abre la consola e instala `easy-thumbnails` usando el siguiente comando:

```bash
python -m pip install easy-thumbnails==2.8.5
```

Edita el archivo `settings.py` del proyecto `bookmarks` y añade `easy_thumbnails` a la configuración `INSTALLED_APPS`, de la siguiente manera:

```python
INSTALLED_APPS = [
    # ...
    'easy_thumbnails',
]
```

Luego, ejecuta el siguiente comando para sincronizar la aplicación con tu base de datos:

```bash
python manage.py migrate
```

Verás una salida que incluye las siguientes líneas:

```text
Applying easy_thumbnails.0001_initial... OK
Applying easy_thumbnails.0002_thumbnaildimensions... OK
```

La aplicación `easy-thumbnails` te ofrece diferentes formas de definir miniaturas de imágenes. La aplicación proporciona una etiqueta de plantilla `{% thumbnail %}` para generar miniaturas en plantillas y un `ImageField` personalizado si deseas definir miniaturas en tus modelos. Usemos el enfoque de la etiqueta de plantilla.

Edita la plantilla `images/image/detail.html` y considera la siguiente línea:

```html
<img src="{{ image.image.url }}" class="image-detail">
```

Reemplázala con las siguientes líneas:

```html
{% load thumbnail %}
<a href="{{ image.image.url }}">
    <img src="{% thumbnail image.image 300x0 %}" class="image-detail">
</a>
```

Hemos definido una miniatura con un ancho fijo de 300 píxeles y una altura flexible para mantener la relación de aspecto utilizando el valor `0`. La primera vez que un usuario cargue esta página, se creará una imagen en miniatura. La miniatura se almacena en el mismo directorio que el archivo original. La ubicación está definida por la configuración `MEDIA_ROOT` y el atributo `upload_to` del campo de imagen del modelo `Image`. La miniatura generada se servirá luego en las siguientes peticiones.

Ejecuta el servidor de desarrollo con el siguiente comando desde la consola:

```bash
python manage.py runserver_plus --cert-file cert.crt
```

Accede a la página de detalle de la imagen para una imagen existente. La miniatura se generará y se mostrará en el sitio. Haz clic con el botón derecho en la imagen y ábrela en una nueva pestaña del navegador, de la siguiente manera:

> *Figura 6.16: Apertura de la imagen en una nueva pestaña del navegador*

Comprueba la URL de la imagen generada en tu navegador. Debería verse de la siguiente manera:

> *Figura 6.17: La URL de la imagen generada*

El nombre de archivo original va seguido de detalles adicionales de la configuración utilizada para crear la miniatura. Para una imagen JPEG, verás un nombre de archivo como `filename.jpg.300x0_q85.jpg`, donde `300x0` indica los parámetros de tamaño utilizados para generar la miniatura, y `85` es el valor de la calidad JPEG predeterminada utilizada por la biblioteca para generar la miniatura.

Puedes usar un valor de calidad diferente usando el parámetro `quality`. Para establecer la calidad JPEG más alta, puedes usar el valor 100, así: `{% thumbnail image.image 300x0 quality=100 %}`. Una mayor calidad implicará un tamaño de archivo más grande.

La aplicación `easy-thumbnails` ofrece varias opciones para personalizar tus miniaturas, incluidos algoritmos de recorte y diferentes efectos que se pueden aplicar. Si tienes algún problema al generar miniaturas, puedes añadir `THUMBNAIL_DEBUG = True` al archivo `settings.py` para obtener información de depuración. Puedes leer la documentación completa de `easy-thumbnails` en [https://easy-thumbnails.readthedocs.io/](https://easy-thumbnails.readthedocs.io/).

---

### Adición de acciones asíncronas con JavaScript

Vamos a añadir un botón de "me gusta" a la página de detalle de la imagen para que los usuarios puedan hacer clic en él y dar "me gusta" a una imagen. Cuando los usuarios hagan clic en el botón de "me gusta", enviaremos una petición HTTP al servidor web mediante JavaScript. Esto realizará la acción de dar "me gusta" sin recargar toda la página. Para esta funcionalidad, implementaremos una vista que permita a los usuarios dar "me gusta" o "ya no me gusta" a las imágenes.

La API Fetch de JavaScript es la forma integrada de realizar peticiones HTTP asíncronas a servidores web desde navegadores web. Al utilizar la API Fetch, puedes enviar y recuperar datos del servidor web sin necesidad de actualizar toda la página. La API Fetch se lanzó como un sucesor moderno del objeto `XMLHttpRequest` (XHR) que está integrado en el navegador, utilizado para realizar peticiones HTTP sin recargar la página. El conjunto de técnicas de desarrollo web para enviar y recuperar datos de un servidor web de forma asíncrona sin recargar la página también se conoce como AJAX, que significa *Asynchronous JavaScript and XML*. AJAX es un nombre engañoso porque las peticiones AJAX pueden intercambiar datos no solo en formato XML, sino también en formatos como JSON, HTML y texto plano. Podrías encontrar referencias a la API Fetch y a AJAX indistintamente en Internet.

Puedes encontrar información sobre la API Fetch en [https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch).

Comenzaremos implementando la vista para realizar las acciones de "me gusta" y "ya no me gusta", y luego añadiremos el código JavaScript a la plantilla relacionada para realizar peticiones HTTP asíncronas.

Edita el archivo `views.py` de la aplicación `images` y añádele el siguiente código:

```python
from django.http import JsonResponse
from django.views.decorators.http import require_POST


@login_required
@require_POST
def image_like(request):
    image_id = request.POST.get('id')
    action = request.POST.get('action')
    if image_id and action:
        try:
            image = Image.objects.get(id=image_id)
            if action == 'like':
                image.users_like.add(request.user)
            else:
                image.users_like.remove(request.user)
            return JsonResponse({'status': 'ok'})
        except Image.DoesNotExist:
            pass
    return JsonResponse({'status': 'error'})
```

Hemos utilizado dos decoradores para la nueva vista. El decorador `login_required` evita que los usuarios que no han iniciado sesión accedan a esta vista. El decorador `require_POST` devuelve un objeto `HttpResponseNotAllowed` (código de estado 405) si la petición HTTP no se realiza a través de POST. De esta manera, solo permites peticiones POST para esta vista.

Django también proporciona un decorador `require_GET` para permitir solo peticiones GET y un decorador `require_http_methods` al que puedes pasar una lista de métodos permitidos como argumento.

Esta vista espera los siguientes parámetros POST:

- `image_id`: El ID del objeto `Image` en el que el usuario está realizando la acción.
- `action`: La acción que el usuario desea realizar, que debe ser una cadena con el valor `like` o `unlike`.

Hemos utilizado el gestor proporcionado por Django para el campo de muchos a muchos `users_like` del modelo `Image` con el fin de añadir o eliminar objetos de la relación utilizando los métodos `add()` o `remove()`. Si se llama al método `add()` pasando un objeto que ya está presente en el conjunto de objetos relacionados, no se duplicará. Si se llama al método `remove()` con un objeto que no está en el conjunto de objetos relacionados, no pasará nada. Otro método útil de los gestores de muchos a muchos es `clear()`, que elimina todos los objetos del conjunto de objetos relacionados.

Para generar la respuesta de la vista, hemos utilizado la clase `JsonResponse` proporcionada por Django, que devuelve una respuesta HTTP con un tipo de contenido `application/json`, convirtiendo el objeto dado en una salida JSON.

Edita el archivo `urls.py` de la aplicación `images` y añade el siguiente patrón de URL:

```python
urlpatterns = [
    path('create/', views.image_create, name='create'),
    path(
        'detail/<int:id>/<slug:slug>/',
        views.image_detail,
        name='detail'
    ),
    path('like/', views.image_like, name='like'),
]
```

#### Carga de JavaScript en el DOM

Necesitamos añadir código JavaScript a la plantilla de detalle de la imagen. Para usar JavaScript en nuestras plantillas, añadiremos primero un contenedor base en la plantilla `base.html` del proyecto.

Edita la plantilla `base.html` de la aplicación `account` e incluye el siguiente código antes de la etiqueta HTML de cierre `</body>`:

```html
<!DOCTYPE html>
<html>
<head>
    ...
</head>
<body>
    ...
    <script>
        document.addEventListener('DOMContentLoaded', (event) => {
            // DOM loaded
            {% block domready %}
            {% endblock %}
        })
    </script>
</body>
</html>
```

Hemos añadido una etiqueta `<script>` para incluir código JavaScript. El método `document.addEventListener()` se utiliza para definir una función que se llamará cuando se active el evento dado. Pasamos el nombre del evento `DOMContentLoaded`, que se activa cuando el documento HTML inicial se ha cargado por completo y la jerarquía del DOM se ha construido completamente. Al utilizar este evento, nos aseguramos de que el DOM esté completamente construido antes de interactuar con cualquier elemento HTML y manipular el DOM. El código dentro de la función solo se ejecutará una vez que el DOM esté listo.

Dentro del controlador document-ready, hemos incluido un bloque de plantilla de Django llamado `domready`. Cualquier plantilla que extienda la plantilla `base.html` puede usar este bloque para incluir código JavaScript específico para ejecutar cuando el DOM esté listo.

> [!NOTE]
> No te confundas con el código JavaScript y las etiquetas de plantilla de Django. El lenguaje de plantillas de Django se renderiza en el lado del servidor para generar el documento HTML, y JavaScript se ejecuta en el navegador en el lado del cliente. En algunos casos, es útil generar código JavaScript dinámicamente usando Django para usar los resultados de QuerySets o cálculos del lado del servidor para definir variables en JavaScript.
> 
> Los ejemplos de este capítulo incluyen código JavaScript en plantillas de Django. El método preferido para añadir código JavaScript a tus plantillas es cargando archivos `.js`, que se sirven como archivos estáticos, especialmente si estás utilizando scripts grandes.

#### Falsificación de peticiones en sitios cruzados (CSRF) para peticiones HTTP en JavaScript

Aprendiste sobre la falsificación de peticiones en sitios cruzados (CSRF) en el Capítulo 2, *Mejora de tu blog y adición de funciones sociales*. Con la protección CSRF activa, Django busca un token CSRF en todas las peticiones POST. Cuando envías formularios, puedes usar la etiqueta de plantilla `{% csrf_token %}` para enviar el token junto con el formulario. Las peticiones HTTP realizadas en JavaScript también deben pasar el token CSRF en cada petición POST.

Django te permite establecer un encabezado personalizado `X-CSRFToken` en tus peticiones HTTP con el valor del token CSRF.

Para incluir el token en peticiones HTTP que se originan desde JavaScript, necesitaremos recuperar el token CSRF de la cookie `csrftoken`, que es establecida por Django si la protección CSRF está activa. Para gestionar cookies, utilizaremos JavaScript Cookie. JavaScript Cookie es una API de JavaScript ligera para manejar cookies. Puedes obtener más información al respecto en [https://github.com/js-cookie/js-cookie](https://github.com/js-cookie/js-cookie).

Edita la plantilla `base.html` de la aplicación `account` y añade el siguiente código en la parte inferior del elemento `<body>`:

```html
<!DOCTYPE html>
<html>
<head>
    ...
</head>
<body>
    ...
    <script src="//cdn.jsdelivr.net/npm/js-cookie@3.0.5/dist/js.cookie.min.js"></script>
    <script>
        const csrftoken = Cookies.get('csrftoken');
        document.addEventListener('DOMContentLoaded', (event) => {
            // DOM loaded
            {% block domready %}
            {% endblock %}
        })
    </script>
</body>
</html>
```

Hemos implementado la siguiente funcionalidad:

- El plugin JavaScript Cookie se carga desde una Red de Entrega de Contenidos (CDN) pública.
- El valor de la cookie `csrftoken` se recupera con `Cookies.get()` y se almacena en la constante de JavaScript `csrftoken`.

Tenemos que incluir el token CSRF en todas las peticiones fetch de JavaScript que utilicen métodos HTTP no seguros, como POST o PUT. Más adelante incluiremos la constante `csrftoken` en un encabezado HTTP personalizado llamado `X-CSRFToken` al enviar peticiones HTTP POST.

Puedes encontrar más información sobre la protección CSRF de Django y AJAX en [https://docs.djangoproject.com/en/5.2/ref/csrf/#ajax](https://docs.djangoproject.com/en/5.2/ref/csrf/#ajax).

A continuación, implementaremos el código HTML y JavaScript para que los usuarios den "me gusta" o "ya no me gusta" a las imágenes.

#### Realización de peticiones HTTP con JavaScript

Edita la plantilla `images/image/detail.html` y añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}{{ image.title }}{% endblock %}

{% block content %}
    <h1>{{ image.title }}</h1>
    {% load thumbnail %}
    <a href="{{ image.image.url }}">
        <img src="{% thumbnail image.image 300x0 %}" class="image-detail">
    </a>
    {% with total_likes=image.users_like.count users_like=image.users_like.all %}
        <div class="image-info">
            <div>
                <span class="count">
                    <span class="total">{{ total_likes }}</span>
                    like{{ total_likes|pluralize }}
                </span>
                <a href="#" data-id="{{ image.id }}" data-action="{% if request.user in users_like %}un{% endif %}like" class="like button">
                    {% if request.user not in users_like %}
                        Like
                    {% else %}
                        Unlike
                    {% endif %}
                </a>
            </div>
            {{ image.description|linebreaks }}
        </div>
        <div class="image-likes">
            {% for user in users_like %}
                <div>
                    {% if user.profile.photo %}
                        <img src="{{ user.profile.photo.url }}">
                    {% endif %}
                    <p>{{ user.first_name }}</p>
                </div>
            {% empty %}
                Nobody likes this image yet.
            {% endfor %}
        </div>
    {% endwith %}
{% endblock %}
```

En el código anterior, hemos añadido otra variable a la etiqueta de plantilla `{% with %}` para almacenar los resultados de la consulta `image.users_like.all` y evitar que la consulta se ejecute contra la base de datos varias veces. Esta variable se utiliza para comprobar si el usuario actual está en esta lista con `{% if request.user in users_like %}` y luego con `{% if request.user not in users_like %}`. La misma variable se utiliza luego para iterar sobre los usuarios a los que les gusta esta imagen con `{% for user in users_like %}`.

Hemos añadido a esta página el número total de usuarios a los que les gusta la imagen y hemos incluido un enlace para que el usuario dé "me gusta" o "ya no me gusta" a la imagen. El conjunto de objetos relacionados, `users_like`, se utiliza para verificar si `request.user` está contenido en el conjunto de objetos relacionados, para mostrar el texto *Like* o *Unlike* en función de la relación actual entre el usuario y esta imagen. Hemos añadido los siguientes atributos al elemento de enlace HTML `<a>`:

- `data-id`: El ID de la imagen mostrada.
- `data-action`: La acción a realizar cuando el usuario hace clic en el enlace. Esto puede ser `like` o `unlike`.

Cualquier atributo en cualquier elemento HTML con un nombre que comience con `data-` es un atributo de datos. Los atributos de datos se utilizan para almacenar datos personalizados para tu aplicación.

Enviaremos el valor de los atributos `data-id` y `data-action` en la petición HTTP a la vista `image_like`. Cuando un usuario haga clic en el enlace like/unlike, necesitaremos realizar las siguientes acciones en el navegador:

1. Enviar una petición HTTP POST a la vista `image_like`, pasándole los parámetros `id` y `action` de la imagen.
2. Si la petición HTTP es exitosa, actualizar el atributo `data-action` del elemento HTML `<a>` con la acción opuesta (like / unlike), y modificar su texto visible en consecuencia.
3. Actualizar el número total de "me gusta" mostrados en la página.

Añade el siguiente bloque `domready` en la parte inferior de la plantilla `images/image/detail.html`:

```html
{% block domready %}
    const url = '{% url "images:like" %}';
    var options = {
        method: 'POST',
        headers: {'X-CSRFToken': csrftoken},
        mode: 'same-origin'
    }

    document.querySelector('a.like')
            .addEventListener('click', function(e){
                e.preventDefault();
                var likeButton = this;
            });
{% endblock %}
```

El código anterior funciona de la siguiente manera:

- La etiqueta de plantilla `{% url %}` se utiliza para construir la URL `images:like`. La URL generada se almacena en la constante de JavaScript `url`.
- Se crea un objeto `options` con las opciones que se pasarán a la petición HTTP con la API Fetch. Estas son:
  - `method`: El método HTTP a utilizar. En este caso, es POST.
  - `headers`: Encabezados HTTP adicionales a incluir en la petición. Incluimos el encabezado `X-CSRFToken` con el valor de la constante `csrftoken` que definimos en la plantilla `base.html`.
  - `mode`: El modo de la petición HTTP. Usamos `same-origin` para indicar que la petición se realiza al mismo origen. Puedes encontrar más información sobre los modos en [https://developer.mozilla.org/en-US/docs/Web/API/Request/mode](https://developer.mozilla.org/en-US/docs/Web/API/Request/mode).
- El selector `a.like` se utiliza para encontrar todos los elementos `<a>` del documento HTML con la clase `like` mediante `document.querySelector()`.
- Se define un escucha de eventos (*event listener*) para el evento `click` en los elementos seleccionados. Esta función se ejecuta cada vez que el usuario hace clic en el enlace like/unlike.
- Dentro de la función controladora, se utiliza `e.preventDefault()` para evitar el comportamiento predeterminado del elemento `<a>`. Esto evitará que el enlace navegue a una URL.
- Se utiliza una variable `likeButton` para almacenar la referencia a `this`, el elemento en el que se activó el evento.

Ahora necesitamos enviar la petición HTTP utilizando la API Fetch. Edita el bloque `domready` de la plantilla `images/image/detail.html` y añade el siguiente código:

```html
{% block domready %}
    const url = '{% url "images:like" %}';
    var options = {
        method: 'POST',
        headers: {'X-CSRFToken': csrftoken},
        mode: 'same-origin'
    }

    document.querySelector('a.like')
            .addEventListener('click', function(e){
                e.preventDefault();
                var likeButton = this;

                // add request body
                var formData = new FormData();
                formData.append('id', likeButton.dataset.id);
                formData.append('action', likeButton.dataset.action);
                options['body'] = formData;

                // send HTTP request
                fetch(url, options)
                .then(response => response.json())
                .then(data => {
                    if (data['status'] === 'ok') {
                    }
                })
            });
{% endblock %}
```

El nuevo código funciona de la siguiente manera:

- Se crea un objeto `FormData` para construir un conjunto de pares clave/valor que representan los campos del formulario y sus valores. El objeto se almacena en la variable `formData`.
- Los parámetros `id` y `action` esperados por la vista de Django `image_like` se añaden al objeto `formData`. Los valores para estos parámetros se recuperan del elemento `likeButton` pulsado. Se accede a los atributos `data-id` y `data-action` con `dataset.id` y `dataset.action`.
- Se añade una nueva clave `body` al objeto `options` que se utilizará para la petición HTTP. El valor de esta clave es el objeto `formData`.
- La API Fetch se utiliza llamando a la función `fetch()`. La variable `url` definida previamente se pasa como la URL de la petición, y el objeto `options` se pasa como las opciones de la petición.
- La función `fetch()` devuelve una promesa que se resuelve con un objeto `Response`, que es una representación de la respuesta HTTP. El método `.then()` se utiliza para definir un controlador para la promesa. Para extraer el contenido del cuerpo JSON, utilizamos `response.json()`. Puedes obtener más información sobre el objeto Response en [https://developer.mozilla.org/en-US/docs/Web/API/Response](https://developer.mozilla.org/en-US/docs/Web/API/Response).
- El método `.then()` se utiliza de nuevo para definir un controlador para los datos extraídos a JSON. En este controlador, el atributo `status` de los datos recibidos se utiliza para comprobar si su valor es `ok`.

Has añadido la funcionalidad para enviar la petición HTTP y gestionar la respuesta. Después de una petición exitosa, debes cambiar el botón y su acción relacionada por la opuesta: de like a unlike, o de unlike a like. Al hacerlo, los usuarios podrán deshacer su acción.

Edita el bloque `domready` de la plantilla `images/image/detail.html` y añade el siguiente código:

```html
{% block domready %}
    var url = '{% url "images:like" %}';
    var options = {
        method: 'POST',
        headers: {'X-CSRFToken': csrftoken},
        mode: 'same-origin'
    }

    document.querySelector('a.like')
            .addEventListener('click', function(e){
                e.preventDefault();
                var likeButton = this;

                // add request body
                var formData = new FormData();
                formData.append('id', likeButton.dataset.id);
                formData.append('action', likeButton.dataset.action);
                options['body'] = formData;

                // send HTTP request
                fetch(url, options)
                .then(response => response.json())
                .then(data => {
                    if (data['status'] === 'ok') {
                        var previousAction = likeButton.dataset.action;

                        // toggle button text and data-action
                        var action = previousAction === 'like' ? 'unlike' : 'like';
                        likeButton.dataset.action = action;
                        likeButton.innerHTML = action;

                        // update like count
                        var likeCount = document.querySelector('span.count .total');
                        var totalLikes = parseInt(likeCount.innerHTML);
                        likeCount.innerHTML = previousAction === 'like' ? totalLikes + 1 : totalLikes - 1;
                    }
                })
            });
{% endblock %}
```

El código anterior funciona de la siguiente manera:

- La acción anterior del botón se recupera del atributo `data-action` del enlace y se almacena en la variable `previousAction`.
- El atributo `data-action` del enlace y el texto del enlace se alternan. Esto permite a los usuarios deshacer sus acciones.
- El recuento total de "me gusta" se recupera del DOM utilizando el selector `span.count .total` y el valor se analiza a un entero con `parseInt()`. El recuento total de "me gusta" se incrementa o decrementa según la acción realizada (like o unlike).

Abre la página de detalle de la imagen en tu navegador para una imagen que hayas subido. Deberías poder ver el recuento inicial de "me gusta" y el botón **LIKE**, de la siguiente manera:

> *Figura 6.18: El recuento de "me gusta" y el botón LIKE en la plantilla de detalle de la imagen*

Haz clic en el botón **LIKE**. Notarás que el recuento total de "me gusta" aumenta en uno y el texto del botón cambia a **UNLIKE**, de la siguiente manera:

> *Figura 6.19: El recuento de "me gusta" y el botón después de hacer clic en el botón LIKE*

Si haces clic en el botón **UNLIKE**, se realiza la acción y luego el texto del botón vuelve a cambiar a **LIKE** y el recuento total cambia en consecuencia.

Al programar en JavaScript, especialmente al realizar peticiones AJAX, se recomienda utilizar una herramienta para depurar JavaScript y peticiones HTTP. La mayoría de los navegadores modernos incluyen herramientas de desarrollo para depurar JavaScript. Por lo general, puedes hacer clic derecho en cualquier lugar del sitio web para abrir el menú contextual y hacer clic en **Inspeccionar** o **Inspeccionar elemento** para acceder a las herramientas de desarrollo web de tu navegador.

En la siguiente sección, aprenderás a usar peticiones HTTP asíncronas con JavaScript y Django para implementar la paginación con desplazamiento infinito (*infinite scroll*).

#### Adición de paginación con desplazamiento infinito a la lista de imágenes

A continuación, necesitamos listar todas las imágenes guardadas en el sitio web. Utilizaremos peticiones de JavaScript para construir una funcionalidad de desplazamiento infinito (*infinite scroll*). El desplazamiento infinito se logra cargando los siguientes resultados automáticamente cuando el usuario se desplaza hacia la parte inferior de la página.

Implementemos una vista de lista de imágenes que gestionará tanto las peticiones estándar del navegador como las peticiones que se originan desde JavaScript. Cuando el usuario cargue inicialmente la página de lista de imágenes, mostraremos la primera página de imágenes. Cuando se desplacen hacia la parte inferior de la página, recuperaremos la siguiente página de elementos con JavaScript y la añadiremos al final de la página principal.

La misma vista gestionará tanto la paginación estándar como la paginación de desplazamiento infinito con AJAX. Edita el archivo `views.py` de la aplicación `images` y añade el siguiente código:

```python
from django.core.paginator import EmptyPage, PageNotAnInteger, Paginator
from django.http import HttpResponse

# ...


@login_required
def image_list(request):
    images = Image.objects.all()
    paginator = Paginator(images, 8)
    page = request.GET.get('page')
    images_only = request.GET.get('images_only')
    try:
        images = paginator.page(page)
    except PageNotAnInteger:
        # If page is not an integer deliver the first page
        images = paginator.page(1)
    except EmptyPage:
        if images_only:
            # If AJAX request and page out of range
            # return an empty page
            return HttpResponse('')
        # If page out of range return last page of results
        images = paginator.page(paginator.num_pages)
    if images_only:
        return render(
            request,
            'images/image/list_images.html',
            {'section': 'images', 'images': images}
        )
    return render(
        request,
        'images/image/list.html',
        {'section': 'images', 'images': images}
    )
```

En esta vista, se crea un QuerySet para recuperar todas las imágenes de la base de datos. Luego, se crea un objeto `Paginator` para paginar sobre los resultados, recuperando ocho imágenes por página. El parámetro HTTP GET `page` se recupera para obtener el número de página solicitado. El parámetro HTTP GET `images_only` se recupera para saber si se debe renderizar toda la página o solo las nuevas imágenes.

Renderizaremos toda la página cuando sea solicitada por el navegador. Sin embargo, solo renderizaremos el HTML con las nuevas imágenes para las peticiones de la API Fetch, ya que las añadiremos a la página HTML existente.

Se activará una excepción `EmptyPage` si la página solicitada está fuera de rango. Si este es el caso y solo se deben renderizar imágenes, se devolverá un `HttpResponse` vacío. Esto te permitirá detener la paginación AJAX en el lado del cliente al llegar a la última página. Los resultados se renderizan utilizando dos plantillas diferentes:

- Para peticiones HTTP de JavaScript, que incluirán el parámetro `images_only`, se renderizará la plantilla `list_images.html`. Esta plantilla solo contendrá las imágenes de la página solicitada.
- Para peticiones del navegador, se renderizará la plantilla `list.html`. Esta plantilla extenderá la plantilla `base.html` para mostrar toda la página e incluirá la plantilla `list_images.html` para incluir la lista de imágenes.

Edita el archivo `urls.py` de la aplicación `images` y añade el siguiente patrón de URL:

```python
urlpatterns = [
    path('create/', views.image_create, name='create'),
    path(
        'detail/<int:id>/<slug:slug>/',
        views.image_detail,
        name='detail'
    ),
    path('like/', views.image_like, name='like'),
    path('', views.image_list, name='list'),
]
```

Finalmente, necesitas crear las plantillas mencionadas aquí. Dentro del directorio de plantillas `images/image/`, crea una nueva plantilla y nómbrala `list_images.html`. Añade el siguiente código:

```html
{% load thumbnail %}

{% for image in images %}
    <div class="image">
        <a href="{{ image.get_absolute_url }}">
            {% thumbnail image.image 300x300 crop="smart" as im %}
            <a href="{{ image.get_absolute_url }}">
                <img src="{{ im.url }}">
            </a>
        </a>
        <div class="info">
            <a href="{{ image.get_absolute_url }}" class="title">
                {{ image.title }}
            </a>
        </div>
    </div>
{% endfor %}
```

La plantilla anterior muestra la lista de imágenes. La utilizarás para devolver resultados para peticiones AJAX. En este código, iteras sobre `images` y generas una miniatura cuadrada para cada imagen. Normalizas el tamaño de las miniaturas a 300x300 píxeles. También utilizas la opción de recorte inteligente (`smart`). Esta opción indica que la imagen debe recortarse incrementalmente hasta el tamaño solicitado eliminando cortes de los bordes con menor entropía.

Crea otra plantilla en el mismo directorio y nómbrala `list.html`. Añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Images bookmarked{% endblock %}

{% block content %}
    <h1>Images bookmarked</h1>
    <div id="image-list">
        {% include "images/image/list_images.html" %}
    </div>
{% endblock %}
```

La plantilla de lista extiende la plantilla `base.html`. Para evitar repetir código, incluyes la plantilla `images/image/list_images.html` para mostrar las imágenes. La plantilla `images/image/list.html` contendrá el código JavaScript para cargar páginas adicionales al desplazarse hacia la parte inferior de la página.

Edita la plantilla `images/image/list.html` y añade el siguiente bloque `domready`:

```html
{% extends "base.html" %}

{% block title %}Images bookmarked{% endblock %}

{% block content %}
    <h1>Images bookmarked</h1>
    <div id="image-list">
        {% include "images/image/list_images.html" %}
    </div>
{% endblock %}

{% block domready %}
    var page = 1;
    var emptyPage = false;
    var blockRequest = false;

    window.addEventListener('scroll', function(e) {
        var margin = document.body.clientHeight - window.innerHeight - 200;
        if(window.pageYOffset > margin && !emptyPage && !blockRequest) {
            blockRequest = true;
            page += 1;

            fetch('?images_only=1&page=' + page)
            .then(response => response.text())
            .then(html => {
                if (html === '') {
                    emptyPage = true;
                }
                else {
                    var imageList = document.getElementById('image-list');
                    imageList.insertAdjacentHTML('beforeEnd', html);
                    blockRequest = false;
                }
            })
        }
    });

    // Launch scroll event
    const scrollEvent = new Event('scroll');
    window.dispatchEvent(scrollEvent);
{% endblock %}
```

El código anterior proporciona la funcionalidad de desplazamiento infinito. Incluyes el código JavaScript en el bloque `domready` que definiste en la plantilla `base.html`. El código es el siguiente:

- Defines las siguientes variables:
  - `page`: Almacena el número de página actual.
  - `emptyPage`: Te permite saber si el usuario está en la última página y recupera una página vacía. Tan pronto como obtengas una página vacía, dejarás de enviar peticiones HTTP adicionales porque asumirás que no hay más resultados.
  - `blockRequest`: Evita que envíes peticiones adicionales mientras una petición HTTP está en curso.
- Utilizas `window.addEventListener()` para capturar el evento `scroll` y definir una función controladora para él.
- Calculas la variable `margin` para obtener la diferencia entre la altura total del documento y la altura interior de la ventana, porque esa es la altura del contenido restante para que el usuario se desplace. Restas un valor de 200 del resultado para cargar la página siguiente cuando el usuario esté a menos de 200 píxeles de la parte inferior de la página.
- Antes de enviar una petición HTTP, compruebas que:
  - `window.pageYOffset` sea mayor que el margen calculado.
  - El usuario no haya llegado a la última página de resultados (`emptyPage` debe ser `false`).
  - No haya otra petición HTTP en curso (`blockRequest` debe ser `false`).
- Si se cumplen las condiciones anteriores, estableces `blockRequest` en `true` para evitar que el evento de desplazamiento active peticiones HTTP adicionales, e incrementas el contador de páginas en 1 para recuperar la página siguiente.
- Usas `fetch()` para enviar una petición HTTP GET, estableciendo los parámetros de URL `images_only=1` para recuperar solo el HTML de las imágenes en lugar de toda la página HTML, y `page` para el número de página solicitado.
- El contenido del cuerpo se extrae de la respuesta HTTP con `response.text()` y el HTML devuelto se trata en consecuencia:
  - Si la respuesta no tiene contenido: Has llegado al final de los resultados y no hay más páginas para cargar. Estableces `emptyPage` en `true` para evitar peticiones HTTP adicionales.
  - Si la respuesta contiene datos: Añades los datos al elemento HTML con el ID `image-list`. El contenido de la página se expande verticalmente, añadiendo resultados cuando el usuario se acerca a la parte inferior de la página. Quitas el bloqueo para peticiones HTTP adicionales estableciendo `blockRequest` en `false`.
- Debajo del escucha de eventos, simulas un evento de desplazamiento inicial cuando se carga la página. Creas el evento creando un nuevo objeto `Event`, y luego lo lanzas con `window.dispatchEvent()`. Al hacer esto, te aseguras de que el evento se active si el contenido inicial cabe en la ventana y no tiene barra de desplazamiento.

Abre `https://127.0.0.1:8000/images/` en tu navegador. Verás la lista de imágenes que has guardado hasta ahora. Debería verse similar a esto:

> *Figura 6.20: La página de lista de imágenes con paginación de desplazamiento infinito (Atribuciones de imágenes: Chick Corea por ataelw, Licencia: CC BY 2.0; Al Jarreau - Düsseldorf 1981 por Eddi Laumanns aka RX-Guru, Licencia: CC BY 3.0; Al Jarreau por Kingkongphoto y celebrity-photos.com, Licencia: CC BY-SA 2.0)*

Desplázate hasta la parte inferior de la página para cargar páginas adicionales. Asegúrate de haber marcado más de ocho imágenes usando el bookmarklet, porque esa es la cantidad de imágenes que estás mostrando por página.

Puedes usar las herramientas de desarrollo de tu navegador para rastrear las peticiones AJAX. Por lo general, puedes hacer clic derecho en cualquier lugar del sitio web para abrir el menú contextual y hacer clic en **Inspeccionar** o **Inspeccionar elemento** para acceder a las herramientas de desarrollo web de tu navegador. Busca el panel de peticiones de red (*Network*).

Recarga la página y desplázate hacia abajo para cargar nuevas páginas. Verás la petición de la primera página y las peticiones AJAX para páginas adicionales, como en la Figura 6.21:

> *Figura 6.21: Peticiones HTTP registradas en las herramientas de desarrollo del navegador*

En la consola donde estás ejecutando Django, también verás las peticiones:

```text
[04/Jan/2024 08:14:20] "GET /images/ HTTP/1.1" 200
[04/Jan/2024 08:14:25] "GET /images/?images_only=1&page=2 HTTP/1.1" 200
[04/Jan/2024 08:14:26] "GET /images/?images_only=1&page=3 HTTP/1.1" 200
[04/Jan/2024 08:14:26] "GET /images/?images_only=1&page=4 HTTP/1.1" 200
```

Finalmente, edita la plantilla `base.html` de la aplicación `account` y añade la URL para el elemento de imágenes:

```html
<ul class="menu">
    ...
    <li {% if section == "images" %}class="selected"{% endif %}>
        <a href="{% url "images:list" %}">Images</a>
    </li>
    ...
</ul>
```

Ahora puedes acceder a la lista de imágenes desde el menú principal.

---

### Resumen

En este capítulo, creaste modelos con relaciones de muchos a muchos y aprendiste a personalizar el comportamiento de los formularios. Construiste un bookmarklet de JavaScript para compartir imágenes de otros sitios web en tu sitio. Este capítulo también ha cubierto la creación de miniaturas de imágenes utilizando la aplicación `easy-thumbnails`.

Finalmente, implementaste vistas AJAX utilizando la API Fetch de JavaScript y añadiste paginación con desplazamiento infinito a la vista de lista de imágenes.

En el próximo capítulo, aprenderás a construir un sistema de seguimiento de usuarios y un flujo de actividad (*activity stream*). Trabajarás con relaciones genéricas, señales (*signals*) y desnormalización. También aprenderás a usar Redis con Django para contar visitas a imágenes y generar una clasificación (*ranking*) de imágenes.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter06](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter06)
- **Índices de base de datos:** [https://docs.djangoproject.com/en/5.2/ref/models/options/#django.db.models.Options.indexes](https://docs.djangoproject.com/en/5.2/ref/models/options/#django.db.models.Options.indexes)
- **Relaciones de muchos a muchos:** [https://docs.djangoproject.com/en/5.2/topics/db/examples/many_to_many/](https://docs.djangoproject.com/en/5.2/topics/db/examples/many_to_many/)
- **Biblioteca HTTP Requests para Python:** [https://requests.readthedocs.io/en/master/](https://requests.readthedocs.io/en/master/)
- **Extensión Pinterest Save:** [https://help.pinterest.com/en/article/save-pins-with-the-pinterest-browser-button](https://help.pinterest.com/en/article/save-pins-with-the-pinterest-browser-button)
- **Contenido estático para la aplicación images:** [https://github.com/PacktPublishing/Django-5-by-Example/tree/main/Chapter06/bookmarks/images/static](https://github.com/PacktPublishing/Django-5-by-Example/tree/main/Chapter06/bookmarks/images/static)
- **Selectores CSS:** [https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Selectors](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Selectors)
- **Localización de elementos del DOM usando selectores CSS:** [https://developer.mozilla.org/en-US/docs/Web/API/Document_object_model/Locating_DOM_elements_using_selectors](https://developer.mozilla.org/en-US/docs/Web/API/Document_object_model/Locating_DOM_elements_using_selectors)
- **Autoridad de certificación automatizada y gratuita Let's Encrypt:** [https://letsencrypt.org](https://letsencrypt.org/)
- **Aplicación Django easy-thumbnails:** [https://easy-thumbnails.readthedocs.io/](https://easy-thumbnails.readthedocs.io/)
- **Uso de la API Fetch de JavaScript:** [https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
- **Biblioteca JavaScript Cookie:** [https://github.com/js-cookie/js-cookie](https://github.com/js-cookie/js-cookie)
- **Protección CSRF de Django y AJAX:** [https://docs.djangoproject.com/en/5.2/ref/csrf/#ajax](https://docs.djangoproject.com/en/5.2/ref/csrf/#ajax)
- **Modo de petición en la API Fetch de JavaScript:** [https://developer.mozilla.org/en-US/docs/Web/API/Request/mode](https://developer.mozilla.org/en-US/docs/Web/API/Request/mode)
- **Objeto Response en la API Fetch de JavaScript:** [https://developer.mozilla.org/en-US/docs/Web/API/Response](https://developer.mozilla.org/en-US/docs/Web/API/Response)
