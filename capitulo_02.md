# Parte 1: Aplicación de Blog

## Capítulo 2: Mejora de tu blog y adición de funciones sociales

### Introducción

En el capítulo anterior, aprendimos los componentes principales de Django mediante el desarrollo de una aplicación de blog sencilla utilizando vistas, plantillas y URLs. En este capítulo, ampliaremos las funcionalidades de la aplicación de blog con características habituales en muchas plataformas de blogs actuales.

En este capítulo, aprenderás los siguientes temas:

- Uso de URLs canónicas para modelos
- Creación de URLs amigables para SEO en publicaciones
- Adición de paginación a la vista de lista de publicaciones
- Creación de vistas basadas en clases
- Envío de correos electrónicos con Django
- Uso de formularios de Django para compartir publicaciones por correo electrónico
- Adición de comentarios a publicaciones mediante formularios creados a partir de modelos

---

### Visión general funcional

La Figura 2.1 muestra una representación de las vistas, plantillas y funcionalidades que se construirán en este capítulo:

> *Figura 2.1: Diagrama de funcionalidades construidas en el Capítulo 2*

En este capítulo, añadiremos paginación a la página de lista de publicaciones para navegar a través de todas ellas. También aprenderemos a crear vistas basadas en clases con Django y convertiremos la vista `post_list` en una vista basada en clases llamada `PostListView`.

Crearemos la vista `post_share` para compartir publicaciones por correo electrónico. Utilizaremos formularios de Django para compartir artículos y enviar recomendaciones de correo electrónico mediante el protocolo simple de transferencia de correo (SMTP). Para agregar comentarios a las publicaciones, crearemos un modelo `Comment` para almacenar los comentarios y construiremos la vista `post_comment` utilizando formularios para modelos.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter02](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter02).

Todos los paquetes de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente del capítulo. Puedes seguir las instrucciones para instalar cada paquete de Python en las siguientes secciones, o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Uso de URLs canónicas para modelos

Un sitio web puede tener diferentes páginas que muestran el mismo contenido. En nuestra aplicación, la parte inicial del contenido de cada publicación se muestra tanto en la página de lista de publicaciones como en la página de detalle de la publicación. Una URL canónica es la URL preferida para un recurso. Puedes pensar en ella como la URL de la página más representativa de un contenido específico. Puede haber diferentes páginas en tu sitio que muestren publicaciones, pero hay una única URL que utilizas como la URL principal para una publicación.

Las URLs canónicas te permiten especificar la URL para la copia maestra de una página. Django te permite implementar el método `get_absolute_url()` en tus modelos para devolver la URL canónica del objeto.

Utilizaremos la URL `post_detail` definida en los patrones de URL de la aplicación para construir la URL canónica para los objetos `Post`. Django proporciona diferentes funciones de resolución de URLs que te permiten construir URLs dinámicamente utilizando su nombre y cualquier parámetro requerido. Usaremos la función de utilidad `reverse()` del módulo `django.urls`.

Edita el archivo `models.py` de la aplicación `blog` para importar la función `reverse()` y añade el método `get_absolute_url()` al modelo `Post` de la siguiente manera:

```python
from django.conf import settings
from django.db import models
from django.urls import reverse
from django.utils import timezone


class PublishedManager(models.Manager):
    def get_queryset(self):
        return (
            super().get_queryset().filter(status=Post.Status.PUBLISHED)
        )


class Post(models.Model):
    # ...
    class Meta:
        ordering = ['-publish']
        indexes = [
            models.Index(fields=['-publish']),
        ]

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse(
            'blog:post_detail',
            args=[self.id]
        )
```

La función `reverse()` construirá la URL dinámicamente utilizando el nombre de la URL definido en los patrones de URL. Hemos utilizado el espacio de nombres `blog` seguido de dos puntos y el nombre de URL `post_detail`. Recuerda que el espacio de nombres `blog` se define en el archivo `urls.py` principal del proyecto al incluir los patrones de URL de `blog.urls`. La URL `post_detail` se define en el archivo `urls.py` de la aplicación `blog`.

La cadena resultante, `blog:post_detail`, se puede utilizar globalmente en tu proyecto para hacer referencia a la URL de detalle de la publicación. Esta URL tiene un parámetro obligatorio, que es el `id` de la publicación del blog a recuperar. Hemos incluido el `id` del objeto `Post` como un argumento posicional mediante `args=[self.id]`.

Puedes obtener más información sobre las funciones de utilidad de URL en [https://docs.djangoproject.com/en/5.2/ref/urlresolvers/](https://docs.djangoproject.com/en/5.2/ref/urlresolvers/).

Reemplacemos las URLs de detalle de publicación en las plantillas con el nuevo método `get_absolute_url()`.

Edita el archivo `blog/post/list.html` y reemplaza la siguiente línea:

```html
<a href="{% url 'blog:post_detail' post.id %}">
```

Reemplaza la línea anterior con la siguiente línea:

```html
<a href="{{ post.get_absolute_url }}">
```

El archivo `blog/post/list.html` ahora debería verse de la siguiente manera:

```html
{% extends "blog/base.html" %}

{% block title %}My Blog{% endblock %}

{% block content %}
    <h1>My Blog</h1>
    {% for post in posts %}
        <h2>
            <a href="{{ post.get_absolute_url }}">
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

Abre la consola y ejecuta el siguiente comando para iniciar el servidor de desarrollo:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/blog/` en tu navegador. Los enlaces a publicaciones individuales del blog deberían seguir funcionando. Django ahora construye las URLs de las publicaciones utilizando el método `get_absolute_url()` del modelo `Post`.

---

### Creación de URLs amigables para SEO en publicaciones

La URL canónica para la vista de detalle de una publicación de blog actualmente se ve como `/blog/1/`. Modificaremos el patrón de URL para crear URLs optimizadas para SEO en las publicaciones. Utilizaremos los valores de fecha de publicación y slug para construir las URLs de publicaciones individuales. Al combinar fechas, haremos que la URL de detalle de una publicación se vea como `/blog/2024/1/1/who-was-django-reinhardt/`. Proporcionaremos a los motores de búsqueda URLs amigables para indexar, que contengan tanto el título como la fecha de la publicación.

Para recuperar publicaciones individuales con la combinación de fecha de publicación y slug, debemos asegurarnos de que ninguna publicación se pueda almacenar en la base de datos con el mismo slug y fecha de publicación que una publicación existente. Evitaremos que el modelo `Post` almacene publicaciones duplicadas definiendo que los slugs sean únicos para la fecha de publicación del artículo.

Edita el archivo `models.py` y añade el siguiente parámetro `unique_for_date` al campo `slug` del modelo `Post`:

```python
class Post(models.Model):
    # ...
    slug = models.SlugField(
        max_length=250,
        unique_for_date='publish'
    )
    # ...
```

Al utilizar `unique_for_date`, ahora se requiere que el campo `slug` sea único para la fecha almacenada en el campo `publish`. Ten en cuenta que el campo `publish` es una instancia de `DateTimeField`, pero la comprobación de valores únicos se realizará solo sobre la fecha (no sobre la hora). Django te impedirá guardar una nueva publicación con el mismo slug que una publicación existente para una fecha de publicación determinada. Ahora nos hemos asegurado de que los slugs sean únicos para la fecha de publicación, por lo que podemos recuperar publicaciones individuales mediante los campos `publish` y `slug`.

Hemos modificado nuestros modelos, así que creemos las migraciones. Ten en cuenta que `unique_for_date` no se aplica a nivel de base de datos, por lo que no se requiere ninguna migración de base de datos. Sin embargo, Django utiliza migraciones para realizar un seguimiento de todos los cambios de modelos. Crearemos una migración simplemente para mantener las migraciones alineadas con el estado actual del modelo.

Ejecuta el siguiente comando en la consola:

```bash
python manage.py makemigrations blog
```

Deberías obtener la siguiente salida:

```text
Migrations for 'blog':
  blog/migrations/0002_alter_post_slug.py
    - Alter field slug on post
```

Django acaba de crear el archivo `0002_alter_post_slug.py` dentro del directorio `migrations` de la aplicación `blog`.

Ejecuta el siguiente comando en la consola para aplicar las migraciones existentes:

```bash
python manage.py migrate
```

Obtendrás una salida que termina con la siguiente línea:

```text
Applying blog.0002_alter_post_slug... OK
```

Django considerará que se han aplicado todas las migraciones y los modelos están sincronizados. No se realizará ninguna acción en la base de datos porque `unique_for_date` no se aplica a nivel de base de datos.

#### Modificación de los patrones de URL

Modifiquemos los patrones de URL para usar la fecha de publicación y el slug para la URL de detalle de la publicación.

Edita el archivo `urls.py` de la aplicación `blog` y reemplaza la siguiente línea:

```python
path('<int:id>/', views.post_detail, name='post_detail'),
```

Reemplaza la línea anterior con las siguientes líneas:

```python
path(
    '<int:year>/<int:month>/<int:day>/<slug:post>/',
    views.post_detail,
    name='post_detail'
),
```

El archivo `urls.py` ahora debería verse de la siguiente manera:

```python
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    # Post views
    path('', views.post_list, name='post_list'),
    path(
        '<int:year>/<int:month>/<int:day>/<slug:post>/',
        views.post_detail,
        name='post_detail'
    ),
]
```

El patrón de URL para la vista `post_detail` toma los siguientes argumentos:

- `year`: Requiere un número entero
- `month`: Requiere un número entero
- `day`: Requiere un número entero
- `post`: Requiere un slug (una cadena que contiene únicamente letras, números, guiones bajos o guiones)

El convertidor de ruta `int` se utiliza para los parámetros `year`, `month` y `day`, mientras que el convertidor de ruta `slug` se utiliza para el parámetro `post`. Aprendiste sobre los convertidores de ruta en el capítulo anterior. Puedes ver todos los convertidores de ruta proporcionados por Django en [https://docs.djangoproject.com/en/5.2/topics/http/urls/#path-converters](https://docs.djangoproject.com/en/5.2/topics/http/urls/#path-converters).

Nuestras publicaciones ahora tienen una URL optimizada para SEO que se construye con la fecha y el slug de cada publicación. Modifiquemos la vista `post_detail` en consecuencia.

#### Modificación de las vistas

Cambiaremos los parámetros de la vista `post_detail` para que coincidan con los nuevos parámetros de URL y los utilizaremos para recuperar el objeto `Post` correspondiente.

Edita el archivo `views.py` y modifica la vista `post_detail` de la siguiente manera:

```python
def post_detail(request, year, month, day, post):
    post = get_object_or_404(
        Post,
        status=Post.Status.PUBLISHED,
        slug=post,
        publish__year=year,
        publish__month=month,
        publish__day=day
    )
    return render(
        request,
        'blog/post/detail.html',
        {'post': post}
    )
```

Hemos modificado la vista `post_detail` para tomar los argumentos `year`, `month`, `day` y `post`, y recuperar una publicación publicada con el slug y la fecha de publicación dados. Al agregar `unique_for_date='publish'` al campo `slug` del modelo `Post`, nos aseguramos de que solo hubiera una publicación con un slug para una fecha determinada. Por lo tanto, puedes recuperar publicaciones individuales utilizando la fecha y el slug.

#### Modificación de la URL canónica para las publicaciones

También tenemos que modificar los parámetros de la URL canónica para las publicaciones del blog para que coincidan con los nuevos parámetros de URL.

Edita el archivo `models.py` de la aplicación `blog` y modifica el método `get_absolute_url()` de la siguiente manera:

```python
class Post(models.Model):
    # ...
    def get_absolute_url(self):
        return reverse(
            'blog:post_detail',
            args=[
                self.publish.year,
                self.publish.month,
                self.publish.day,
                self.slug
            ]
        )
```

Inicia el servidor de desarrollo escribiendo el siguiente comando en la consola:

```bash
python manage.py runserver
```

A continuación, puedes regresar a tu navegador y hacer clic en uno de los títulos de las publicaciones para echar un vistazo a la vista de detalle de la publicación. Deberías ver algo como esto:

> *Figura 2.2: La página para la vista de detalle de la publicación*

Has diseñado URLs amigables para SEO para las publicaciones del blog. La URL de una publicación ahora se ve como `/blog/2024/1/1/who-was-django-reinhardt/`.

Ahora que has implementado URLs optimizadas para SEO, concentrémonos en implementar la navegación a través de las publicaciones mediante paginación.

---

### Adición de paginación

Cuando comienzas a agregar contenido a tu blog, puedes almacenar fácilmente decenas o cientos de publicaciones en tu base de datos. En lugar de mostrar todas las publicaciones en una sola página, es posible que desees dividir la lista de publicaciones en varias páginas e incluir enlaces de navegación a las diferentes páginas. Esta funcionalidad se llama paginación y puedes encontrarla en casi todas las aplicaciones web que muestran listas largas de elementos.

Por ejemplo, Google utiliza la paginación para dividir los resultados de búsqueda en varias páginas. La Figura 2.3 muestra los enlaces de paginación de Google para las páginas de resultados de búsqueda:

> *Figura 2.3: Enlaces de paginación de Google para páginas de resultados de búsqueda*

Django tiene una clase de paginación integrada que te permite administrar datos paginados fácilmente. Puedes definir la cantidad de objetos que deseas que se devuelvan por página y puedes recuperar las publicaciones que corresponden a la página solicitada por el usuario.

#### Adición de paginación a la vista de lista de publicaciones

Añadiremos paginación a la lista de publicaciones para que los usuarios puedan navegar fácilmente por todas las publicaciones publicadas en el blog.

Edita el archivo `views.py` de la aplicación `blog` para importar la clase `Paginator` de Django y modifica la vista `post_list` de la siguiente manera:

```python
from django.core.paginator import Paginator
from django.shortcuts import get_object_or_404, render
from .models import Post


def post_list(request):
    post_list = Post.published.all()
    # Pagination with 3 posts per page
    paginator = Paginator(post_list, 3)
    page_number = request.GET.get('page', 1)
    posts = paginator.page(page_number)
    return render(
        request,
        'blog/post/list.html',
        {'posts': posts}
    )
```

Revisemos el nuevo código que hemos añadido a la vista:

1. Instanciamos la clase `Paginator` con la cantidad de objetos a devolver por página. Mostraremos tres publicaciones por página.
2. Recuperamos el parámetro HTTP GET `page` y lo almacenamos en la variable `page_number`. Este parámetro contiene el número de página solicitado. Si el parámetro `page` no está en los parámetros GET de la petición, usamos el valor predeterminado `1` para cargar la primera página de resultados.
3. Obtenemos los objetos para la página deseada llamando al método `page()` de `Paginator`. Este método devuelve un objeto `Page` que almacenamos en la variable `posts`.
4. Pasamos el objeto `posts` a la plantilla.

#### Creación de una plantilla de paginación

Necesitamos crear una navegación de página para que los usuarios puedan navegar a través de las diferentes páginas. En esta sección, crearemos una plantilla para mostrar los enlaces de paginación y la haremos genérica para que podamos reutilizar la plantilla para cualquier paginación de objetos en nuestro sitio web.

En el directorio `templates/`, crea un nuevo archivo y nómbralo `pagination.html`. Añade el siguiente código HTML al archivo:

```html
<div class="pagination">
    <span class="step-links">
        {% if page.has_previous %}
            <a href="?page={{ page.previous_page_number }}">Previous</a>
        {% endif %}
        <span class="current">
            Page {{ page.number }} of {{ page.paginator.num_pages }}.
        </span>
        {% if page.has_next %}
            <a href="?page={{ page.next_page_number }}">Next</a>
        {% endif %}
    </span>
</div>
```

Esta es la plantilla de paginación genérica. La plantilla espera tener un objeto `Page` en el contexto para renderizar los enlaces anterior y siguiente, y para mostrar la página actual y el total de páginas de resultados.

> [!NOTE]
> Django 5.1 introdujo la etiqueta de plantilla `{% querystring %}` que simplifica la gestión de parámetros de consulta en URLs. La paginación anterior es simple, ya que solo tiene un parámetro `page`, pero los enlaces podrían cambiarse a:
> ```html
> <a href="{% querystring page=page.next_page_number %}">Next</a>
> ```
> Si hubiera parámetros de consulta adicionales, la etiqueta de plantilla se encargaría de ellos. Para más detalles, consulta el Apéndice.

Regresemos a la plantilla `blog/post/list.html` e incluyamos la plantilla `pagination.html` en la parte inferior del bloque `{% content %}`, de la siguiente manera:

```html
{% extends "blog/base.html" %}

{% block title %}My Blog{% endblock %}

{% block content %}
    <h1>My Blog</h1>
    {% for post in posts %}
        <h2>
            <a href="{{ post.get_absolute_url }}">
                {{ post.title }}
            </a>
        </h2>
        <p class="date">
            Published {{ post.publish }} by {{ post.author }}
        </p>
        {{ post.body|truncatewords:30|linebreaks }}
    {% endfor %}
    {% include "pagination.html" with page=posts %}
{% endblock %}
```

La etiqueta de plantilla `{% include %}` carga la plantilla dada y la renderiza utilizando el contexto de plantilla actual. Usamos `with` para pasar variables de contexto adicionales a la plantilla. La plantilla de paginación utiliza la variable `page` para renderizar, mientras que el objeto `Page` que pasamos de nuestra vista a la plantilla se llama `posts`. Usamos `with page=posts` para pasar la variable esperada por la plantilla de paginación. Puedes seguir este método para usar la plantilla de paginación para cualquier tipo de objeto.

Inicia el servidor de desarrollo escribiendo el siguiente comando en la consola:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/admin/blog/post/` en tu navegador y utiliza el sitio de administración para crear un total de cuatro publicaciones diferentes. Asegúrate de establecer el estado en `Published` para todas ellas.

Ahora, abre `http://127.0.0.1:8000/blog/` en tu navegador. Deberías ver las tres primeras publicaciones en orden cronológico inverso y luego los enlaces de navegación en la parte inferior de la lista de publicaciones de esta forma:

> *Figura 2.4: La página de lista de publicaciones incluyendo paginación*

Si haces clic en **Next**, verás la última publicación. La URL de la segunda página contiene el parámetro GET `?page=2`. La vista utiliza este parámetro para cargar la página de resultados solicitada mediante el paginador.

> *Figura 2.5: La segunda página de resultados*

¡Genial! Los enlaces de paginación funcionan como se esperaba.

#### Gestión de errores de paginación

Ahora que la paginación funciona, podemos agregar control de excepciones para errores de paginación en la vista. El parámetro `page` utilizado por la vista para recuperar la página dada podría utilizarse potencialmente con valores incorrectos, como números de página inexistentes o un valor de cadena que no se puede utilizar como número de página. Implementaremos el manejo de errores apropiado para esos casos.

Abre `http://127.0.0.1:8000/blog/?page=3` en tu navegador. Deberías ver la siguiente página de error:

> *Figura 2.6: La página de error EmptyPage*

El objeto `Paginator` lanza una excepción `EmptyPage` al recuperar la página 3 porque está fuera de rango. No hay resultados para mostrar. Manejemos este error en nuestra vista.

Edita el archivo `views.py` de la aplicación `blog` para añadir las importaciones necesarias y modifica la vista `post_list` de la siguiente manera:

```python
from django.core.paginator import EmptyPage, Paginator
from django.shortcuts import get_object_or_404, render
from .models import Post


def post_list(request):
    post_list = Post.published.all()
    # Pagination with 3 posts per page
    paginator = Paginator(post_list, 3)
    page_number = request.GET.get('page', 1)
    try:
        posts = paginator.page(page_number)
    except EmptyPage:
        # If page_number is out of range get last page of results
        posts = paginator.page(paginator.num_pages)
    return render(
        request,
        'blog/post/list.html',
        {'posts': posts}
    )
```

Hemos añadido un bloque `try` y `except` para gestionar la excepción `EmptyPage` al recuperar una página. Si la página solicitada está fuera de rango, devolvemos la última página de resultados. Obtenemos el número total de páginas con `paginator.num_pages`. El número total de páginas es el mismo que el número de la última página.

Abre `http://127.0.0.1:8000/blog/?page=3` en tu navegador nuevamente. Ahora, la excepción es gestionada por la vista y la última página de resultados se devuelve de la siguiente manera:

> *Figura 2.7: La última página de resultados*

Nuestra vista también debe gestionar el caso en el que se pasa algo diferente a un número entero en el parámetro `page`.

Abre `http://127.0.0.1:8000/blog/?page=asdf` en tu navegador. Deberías ver la siguiente página de error:

> *Figura 2.8: La página de error PageNotAnInteger*

En este caso, el objeto `Paginator` lanza una excepción `PageNotAnInteger` al recuperar la página `asdf` porque los números de página solo pueden ser números enteros. Manejemos este error en nuestra vista.

Edita el archivo `views.py` de la aplicación `blog` para añadir las importaciones necesarias y modifica la vista `post_list` de la siguiente manera:

```python
from django.shortcuts import get_object_or_404, render
from .models import Post
from django.core.paginator import EmptyPage, PageNotAnInteger, Paginator


def post_list(request):
    post_list = Post.published.all()
    # Pagination with 3 posts per page
    paginator = Paginator(post_list, 3)
    page_number = request.GET.get('page')
    try:
        posts = paginator.page(page_number)
    except PageNotAnInteger:
        # If page_number is not an integer get the first page
        posts = paginator.page(1)
    except EmptyPage:
        # If page_number is out of range get last page of results
        posts = paginator.page(paginator.num_pages)
    return render(
        request,
        'blog/post/list.html',
        {'posts': posts}
    )
```

Hemos añadido un nuevo bloque `except` para gestionar la excepción `PageNotAnInteger` al recuperar una página. Si la página solicitada no es un número entero, devolvemos la primera página de resultados.

Abre `http://127.0.0.1:8000/blog/?page=asdf` en tu navegador nuevamente. Ahora, la excepción es gestionada por la vista y la primera página de resultados se devuelve de la siguiente manera:

> *Figura 2.9: La primera página de resultados*

La paginación para las publicaciones del blog ya está completamente implementada.

Puedes obtener más información sobre la clase `Paginator` en [https://docs.djangoproject.com/en/5.2/ref/paginator/](https://docs.djangoproject.com/en/5.2/ref/paginator/).

Habiendo aprendido cómo paginar tu blog, ahora pasaremos a transformar la vista `post_list` en una vista equivalente que se construye utilizando vistas genéricas de Django y paginación integrada.

---

### Creación de vistas basadas en clases

Hemos construido la aplicación de blog utilizando vistas basadas en funciones. Las vistas basadas en funciones son simples y potentes, pero Django también te permite construir vistas utilizando clases.

Las vistas basadas en clases son una forma alternativa de implementar vistas como objetos de Python en lugar de funciones. Dado que una vista es una función que toma una petición web y devuelve una respuesta web, también puedes definir tus vistas como métodos de clase. Django proporciona clases de vista base que puedes utilizar para implementar tus propias vistas. Todas ellas heredan de la clase `View`, que gestiona el despacho de métodos HTTP y otras funcionalidades comunes.

#### Por qué usar vistas basadas en clases

Las vistas basadas en clases ofrecen algunas ventajas sobre las vistas basadas en funciones que son útiles para casos de uso específicos. Las vistas basadas en clases te permiten:

- Organizar el código relacionado con métodos HTTP, como `GET`, `POST` o `PUT`, en métodos separados, en lugar de utilizar bifurcaciones condicionales.
- Utilizar herencia múltiple para crear clases de vista reutilizables (también conocidas como *mixins*).

#### Uso de una vista basada en clases para listar publicaciones

Para comprender cómo escribir vistas basadas en clases, crearemos una nueva vista basada en clases que sea equivalente a la vista `post_list`. Crearemos una clase que heredará de la vista genérica `ListView` que ofrece Django. `ListView` te permite listar cualquier tipo de objeto.

Edita el archivo `views.py` de la aplicación `blog` y agrégale el siguiente código:

```python
from django.views.generic import ListView


class PostListView(ListView):
    """
    Alternative post list view
    """
    queryset = Post.published.all()
    context_object_name = 'posts'
    paginate_by = 3
    template_name = 'blog/post/list.html'
```

La vista `PostListView` es análoga a la vista `post_list` que construimos anteriormente. Hemos implementado una vista basada en clases que hereda de la clase `ListView`. Hemos definido una vista con los siguientes atributos:

- Usamos `queryset` para usar un `QuerySet` personalizado en lugar de recuperar todos los objetos. En lugar de definir un atributo `queryset`, podríamos haber especificado `model = Post` y Django habría creado el `QuerySet` genérico `Post.objects.all()` por nosotros.
- Usamos la variable de contexto `posts` para los resultados de la consulta. La variable predeterminada es `object_list` si no especificas ningún `context_object_name`.
- Definimos la paginación de resultados con `paginate_by`, devolviendo tres objetos por página.
- Usamos una plantilla personalizada para renderizar la página con `template_name`. Si no estableces una plantilla predeterminada, `ListView` utilizará `blog/post_list.html` de forma predeterminada.

Ahora, edita el archivo `urls.py` de la aplicación `blog`, comenta el patrón de URL `post_list` anterior y añade un nuevo patrón de URL utilizando la clase `PostListView`, de la siguiente manera:

```python
urlpatterns = [
    # Post views
    # path('', views.post_list, name='post_list'),
    path('', views.PostListView.as_view(), name='post_list'),
    path(
        '<int:year>/<int:month>/<int:day>/<slug:post>/',
        views.post_detail,
        name='post_detail'
    ),
]
```

Para mantener la paginación en funcionamiento, tenemos que utilizar el objeto de página correcto que se pasa a la plantilla. La vista genérica `ListView` de Django pasa la página solicitada en una variable llamada `page_obj`. Tenemos que editar la plantilla `post/list.html` en consecuencia para incluir el paginador utilizando la variable correcta, de la siguiente manera:

```html
{% extends "blog/base.html" %}

{% block title %}My Blog{% endblock %}

{% block content %}
    <h1>My Blog</h1>
    {% for post in posts %}
        <h2>
            <a href="{{ post.get_absolute_url }}">
                {{ post.title }}
            </a>
        </h2>
        <p class="date">
            Published {{ post.publish }} by {{ post.author }}
        </p>
        {{ post.body|truncatewords:30|linebreaks }}
    {% endfor %}
    {% include "pagination.html" with page=page_obj %}
{% endblock %}
```

Abre `http://127.0.0.1:8000/blog/` en tu navegador y verifica que los enlaces de paginación funcionen como se espera. El comportamiento de los enlaces de paginación debe ser el mismo que con la vista `post_list` anterior.

El manejo de excepciones en este caso es un poco diferente. Si intentas cargar una página fuera de rango o pasar un valor no entero en el parámetro `page`, la vista devolverá una respuesta HTTP con el código de estado 404 (*page not found*) de esta forma:

> *Figura 2.10: Respuesta HTTP 404 Page not found*

El manejo de excepciones que devuelve el código de estado HTTP 404 lo proporciona la vista `ListView`.

Este es un ejemplo sencillo de cómo escribir vistas basadas en clases. Aprenderás más sobre vistas basadas en clases en el Capítulo 13, *Creación de un sistema de gestión de contenidos*, y capítulos sucesivos.

Puedes leer una introducción a las vistas basadas en clases en [https://docs.djangoproject.com/en/5.2/topics/class-based-views/intro/](https://docs.djangoproject.com/en/5.2/topics/class-based-views/intro/).

Después de aprender a usar vistas basadas en clases y utilizar la paginación de objetos integrada, implementaremos la funcionalidad para compartir publicaciones por correo electrónico y captar lectores en tu blog.

---

### Recomendar publicaciones por correo electrónico

Permitiremos a los usuarios compartir publicaciones del blog con otras personas enviando recomendaciones por correo electrónico. Aprenderás a crear formularios en Django, gestionar el envío de datos y enviar correos electrónicos con Django, mejorando tu blog con un toque personal.

Tómate un minuto para pensar en cómo podrías usar vistas, URLs y plantillas para crear esta funcionalidad utilizando lo que aprendiste en el capítulo anterior.

Para permitir que los usuarios compartan publicaciones por correo electrónico, necesitaremos:

1. Crear un formulario para que los usuarios completen su nombre, su dirección de correo electrónico, la dirección de correo electrónico del destinatario y comentarios opcionales.
2. Crear una vista en el archivo `views.py` que gestione los datos enviados y envíe el correo electrónico.
3. Agregar un patrón de URL para la nueva vista en el archivo `urls.py` de la aplicación `blog`.
4. Crear una plantilla para mostrar el formulario.

#### Creación de formularios con Django

Comencemos creando el formulario para compartir publicaciones. Django tiene un framework de formularios integrado que te permite crear formularios fácilmente. El framework de formularios simplifica la definición de los campos del formulario, especifica cómo deben mostrarse e indica cómo deben validar los datos de entrada. El framework de formularios de Django ofrece una forma flexible de renderizar formularios en HTML y manejar datos.

Django viene con dos clases base para construir formularios:

- `Form`: Te permite crear formularios estándar definiendo campos y validaciones.
- `ModelForm`: Te permite crear formularios vinculados a instancias de modelos. Proporciona todas las funcionalidades de la clase base `Form`, pero los campos de formulario se pueden declarar explícitamente o generar automáticamente a partir de los campos del modelo. El formulario se puede utilizar para crear o editar instancias de modelos.

Primero, crea un archivo `forms.py` dentro del directorio de tu aplicación `blog` y agrégale el siguiente código:

```python
from django import forms


class EmailPostForm(forms.Form):
    name = forms.CharField(max_length=25)
    email = forms.EmailField()
    to = forms.EmailField()
    comments = forms.CharField(
        required=False,
        widget=forms.Textarea
    )
```

Hemos definido nuestro primer formulario de Django. El formulario `EmailPostForm` hereda de la clase base `Form`. Utilizamos diferentes tipos de campos para validar los datos en consecuencia.

> [!TIP]
> Los formularios pueden residir en cualquier lugar de tu proyecto de Django. La convención es ubicarlos dentro de un archivo `forms.py` para cada aplicación.

El formulario contiene los siguientes campos:

- `name`: Una instancia de `CharField` con una longitud máxima de 25 caracteres. Lo utilizaremos para el nombre de la persona que envía la publicación.
- `email`: Una instancia de `EmailField`. Utilizaremos el correo electrónico de la persona que envía la recomendación de la publicación.
- `to`: Una instancia de `EmailField`. Utilizaremos la dirección de correo electrónico del destinatario, quien recibirá un correo recomendando la publicación.
- `comments`: Una instancia de `CharField`. Lo utilizaremos para los comentarios que se incluirán en el correo electrónico de recomendación de la publicación. Hemos hecho que este campo sea opcional configurando `required` en `False`, y hemos especificado un widget personalizado para renderizar el campo.

Cada tipo de campo tiene un widget predeterminado que determina cómo se renderiza el campo en HTML. El campo `name` es una instancia de `CharField`. Este tipo de campo se representa como un elemento HTML `<input type="text">`. El widget predeterminado se puede sobrescribir con el atributo `widget`. En el campo `comments`, utilizamos el widget `Textarea` para mostrarlo como un elemento HTML `<textarea>` en lugar del elemento `<input>` predeterminado.

La validación del campo también depende del tipo de campo. Por ejemplo, los campos `email` y `to` son campos `EmailField`. Ambos campos requieren una dirección de correo electrónico válida; de lo contrario, la validación del campo generará una excepción `forms.ValidationError` y el formulario no se validará. También se tienen en cuenta otros parámetros para la validación de campos de formulario, como que el campo `name` tenga una longitud máxima de 25 o que el campo `comments` sea opcional.

Estos son solo algunos de los tipos de campo que Django proporciona para formularios. Puedes encontrar una lista de todos los tipos de campo disponibles en [https://docs.djangoproject.com/en/5.2/ref/forms/fields/](https://docs.djangoproject.com/en/5.2/ref/forms/fields/).

#### Manejo de formularios en vistas

Hemos definido el formulario para recomendar publicaciones por correo electrónico. Ahora, necesitamos una vista para crear una instancia del formulario y gestionar el envío del mismo.

Edita el archivo `views.py` de la aplicación `blog` y agrégale el siguiente código:

```python
from .forms import EmailPostForm


def post_share(request, post_id):
    # Retrieve post by id
    post = get_object_or_404(
        Post,
        id=post_id,
        status=Post.Status.PUBLISHED
    )
    if request.method == 'POST':
        # Form was submitted
        form = EmailPostForm(request.POST)
        if form.is_valid():
            # Form fields passed validation
            cd = form.cleaned_data
            # ... send email
    else:
        form = EmailPostForm()
    return render(
        request,
        'blog/post/share.html',
        {
            'post': post,
            'form': form
        }
    )
```

Hemos definido la vista `post_share` que toma el objeto `request` y la variable `post_id` como parámetros. Usamos el atajo `get_object_or_404()` para recuperar una publicación publicada por su `id`.

Usamos la misma vista tanto para mostrar el formulario inicial como para procesar los datos enviados. El método de petición HTTP nos permite diferenciar si se está enviando el formulario. Una petición GET indicará que se debe mostrar un formulario vacío al usuario y una petición POST indicará que se está enviando el formulario. Usamos `request.method == 'POST'` para diferenciar entre los dos escenarios.

Este es el proceso para mostrar el formulario y gestionar el envío del formulario:

1. Cuando la página se carga por primera vez, la vista recibe una petición GET. En este caso, se crea una nueva instancia de `EmailPostForm` y se almacena en la variable `form`. Esta instancia de formulario se utilizará para mostrar el formulario vacío en la plantilla:
   ```python
   form = EmailPostForm()
   ```
2. Cuando el usuario completa el formulario y lo envía a través de POST, se crea una instancia de formulario utilizando los datos enviados contenidos en `request.POST`:
   ```python
   if request.method == 'POST':
       # Form was submitted
       form = EmailPostForm(request.POST)
   ```
3. Después de esto, los datos enviados se validan mediante el método `is_valid()` del formulario. Este método valida los datos introducidos en el formulario y devuelve `True` si todos los campos contienen datos válidos. Si algún campo contiene datos no válidos, `is_valid()` devuelve `False`. La lista de errores de validación se puede obtener con `form.errors`.
4. Si el formulario no es válido, se vuelve a renderizar en la plantilla, incluyendo los datos enviados. Los errores de validación se mostrarán en la plantilla.
5. Si el formulario es válido, los datos validados se recuperan con `form.cleaned_data`. Este atributo es un diccionario de campos de formulario y sus valores. Los formularios no solo validan los datos sino que también los limpian normalizándolos a un formato consistente.

> [!NOTE]
> Si los datos de tu formulario no se validan, `cleaned_data` contendrá solo los campos válidos.

Hemos implementado la vista para mostrar el formulario y gestionar el envío del formulario. Ahora aprenderemos cómo enviar correos electrónicos con Django y luego agregaremos esa funcionalidad a la vista `post_share`.

#### Envío de correos electrónicos con Django

Enviar correos electrónicos con Django es muy sencillo. Necesitas tener un servidor SMTP local o necesitas acceder a un servidor SMTP externo, como tu proveedor de servicios de correo electrónico.

Las siguientes configuraciones te permiten definir la configuración SMTP para enviar correos electrónicos con Django:

- `EMAIL_HOST`: El host del servidor SMTP; el valor predeterminado es `localhost`.
- `EMAIL_PORT`: El puerto SMTP; el valor predeterminado es `25`.
- `EMAIL_HOST_USER`: El nombre de usuario para el servidor SMTP.
- `EMAIL_HOST_PASSWORD`: La contraseña para el servidor SMTP.
- `EMAIL_USE_TLS`: Si se debe utilizar una conexión segura Transport Layer Security (TLS).
- `EMAIL_USE_SSL`: Si se debe utilizar una conexión segura TLS implícita.

Además, puedes utilizar el ajuste `DEFAULT_FROM_EMAIL` para especificar el remitente predeterminado al enviar correos electrónicos con Django. Para este ejemplo, utilizaremos el servidor SMTP de Google con una cuenta estándar de Gmail.

#### Trabajo con variables de entorno

Agregaremos los ajustes de configuración SMTP al proyecto y cargaremos las credenciales SMTP desde variables de entorno. Al utilizar variables de entorno, evitaremos incrustar credenciales en el código fuente. Hay múltiples razones para mantener la configuración separada del código:

- **Seguridad:** Las credenciales o claves secretas en el código pueden provocar una exposición involuntaria, especialmente si subes el código a repositorios públicos.
- **Flexibilidad:** Mantener la configuración separada te permitirá utilizar la misma base de código en diferentes entornos sin ningún cambio. Aprenderás a construir múltiples entornos en el Capítulo 17, *Puesta en producción*.
- **Mantenibilidad:** Cambiar una configuración no requerirá un cambio de código, lo que garantiza que tu proyecto se mantenga consistente en todas las versiones.

Para facilitar la separación de la configuración del código, vamos a utilizar `python-decouple`. Esta biblioteca simplifica el uso de variables de entorno en tus proyectos. Puedes encontrar información sobre `python-decouple` en [https://github.com/HBNetwork/python-decouple](https://github.com/HBNetwork/python-decouple).

Primero, instala `python-decouple` mediante pip ejecutando el siguiente comando:

```bash
python -m pip install python-decouple==3.8
```

Luego, crea un nuevo archivo dentro del directorio raíz de tu proyecto y nómbralo `.env`. El archivo `.env` contendrá pares clave-valor de variables de entorno. Añade las siguientes líneas al nuevo archivo:

```text
EMAIL_HOST_USER=your_account@gmail.com
EMAIL_HOST_PASSWORD=
DEFAULT_FROM_EMAIL=My Blog <your_account@gmail.com>
```

Si tienes una cuenta de Gmail, reemplaza `your_account@gmail.com` con tu cuenta de Gmail. La variable `EMAIL_HOST_PASSWORD` aún no tiene ningún valor, lo agregaremos más adelante. La variable `DEFAULT_FROM_EMAIL` se utilizará para especificar el remitente predeterminado de nuestros correos electrónicos. Si no tienes una cuenta de Gmail, puedes utilizar las credenciales SMTP de tu proveedor de servicios de correo electrónico.

> [!IMPORTANT]
> Si estás utilizando un repositorio de Git para tu código, asegúrate de incluir `.env` en el archivo `.gitignore` de tu repositorio. Al hacerlo, te aseguras de que las credenciales queden excluidas del repositorio.

Edita el archivo `settings.py` de tu proyecto y agrégale el siguiente código:

```python
from decouple import config

# ...
# Email server configuration
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_HOST_USER = config('EMAIL_HOST_USER')
EMAIL_HOST_PASSWORD = config('EMAIL_HOST_PASSWORD')
EMAIL_PORT = 587
EMAIL_USE_TLS = True
DEFAULT_FROM_EMAIL = config('DEFAULT_FROM_EMAIL')
```

Las configuraciones `EMAIL_HOST_USER`, `EMAIL_HOST_PASSWORD` y `DEFAULT_FROM_EMAIL` ahora se cargan desde variables de entorno definidas en el archivo `.env`.

Las configuraciones `EMAIL_HOST`, `EMAIL_PORT` y `EMAIL_USE_TLS` proporcionadas son para el servidor SMTP de Gmail. Si no tienes una cuenta de Gmail, puedes utilizar la configuración del servidor SMTP de tu proveedor de servicios de correo electrónico.

En lugar de Gmail, también puedes utilizar un servicio de correo electrónico profesional y escalable que te permita enviar correos electrónicos a través de SMTP utilizando tu propio dominio, como SendGrid ([https://sendgrid.com/](https://sendgrid.com/)) o Amazon Simple Email Service (SES) ([https://aws.amazon.com/ses/](https://aws.amazon.com/ses/)). Ambos servicios requerirán que verifiques tu dominio y tus cuentas de correo electrónico de remitente y te proporcionarán credenciales SMTP para enviar correos electrónicos. La aplicación `django-anymail` simplifica la tarea de agregar proveedores de servicios de correo electrónico a tu proyecto como SendGrid o Amazon SES. Puedes encontrar instrucciones de instalación para `django-anymail` en [https://anymail.dev/en/stable/installation/](https://anymail.dev/en/stable/installation/), y la lista de proveedores de servicios de correo electrónico admitidos en [https://anymail.dev/en/stable/esps/](https://anymail.dev/en/stable/esps/).

Si no puedes usar un servidor SMTP, puedes indicarle a Django que escriba correos electrónicos en la consola agregando la siguiente configuración al archivo `settings.py`:

```python
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

Al utilizar esta configuración, Django enviará todos los correos electrónicos a la consola en lugar de enviarlos. Esto es muy útil para probar tu aplicación sin un servidor SMTP.

Para enviar correos electrónicos con el servidor SMTP de Gmail, asegúrate de que la verificación en dos pasos esté activa en tu cuenta de Gmail.

Abre [https://myaccount.google.com/security](https://myaccount.google.com/security) en tu navegador y habilita la verificación en dos pasos para tu cuenta, como se muestra en la Figura 2.11:

> *Figura 2.11: La página de inicio de sesión en Google para cuentas de Google*

Luego, necesitas crear una contraseña de aplicación y utilizarla para tus credenciales SMTP. Una contraseña de aplicación es un código de acceso de 16 dígitos que le otorga permiso a una aplicación o dispositivo menos seguro para acceder a tu cuenta de Google.

Para crear una contraseña de aplicación, abre [https://myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords) en tu navegador. Verás la siguiente pantalla:

> *Figura 2.12: Formulario para generar una nueva contraseña de aplicación de Google*

Si no puedes acceder a Contraseñas de aplicaciones, puede ser que la verificación en dos pasos no esté configurada para tu cuenta, que tu cuenta sea una cuenta de organización en lugar de una cuenta estándar de Gmail o que hayas activado la protección avanzada de Google. Asegúrate de utilizar una cuenta estándar de Gmail y activar la verificación en dos pasos para tu cuenta de Google. Puedes encontrar más información en [https://support.google.com/accounts/answer/185833](https://support.google.com/accounts/answer/185833).

Introduce el nombre `Blog` y haz clic en el botón **Crear**, de la siguiente manera:

> *Figura 2.13: Formulario para generar una nueva contraseña de aplicación de Google*

Se generará una nueva contraseña y se mostrará de esta forma:

> *Figura 2.14: Contraseña de aplicación de Google generada*

Copia la contraseña de aplicación generada.

A continuación, edita el archivo `.env` de tu proyecto y añade la contraseña de la aplicación a la variable `EMAIL_HOST_PASSWORD`, de la siguiente manera:

```text
EMAIL_HOST_USER=your_account@gmail.com
EMAIL_HOST_PASSWORD=xxxxxxxxxxxxxxxx
DEFAULT_FROM_EMAIL=My Blog <your_account@gmail.com>
```

Abre la shell de Python ejecutando el siguiente comando en la consola del sistema:

```bash
python manage.py shell
```

Ejecuta el siguiente código en la shell de Python:

```python
>>> from django.core.mail import send_mail
>>> send_mail('Django mail',
...           'This e-mail was sent with Django.',
...           'your_account@gmail.com',
...           ['your_account@gmail.com'],
...           fail_silently=False)
```

La función `send_mail()` toma el asunto (*subject*), el mensaje (*message*), el remitente (*sender*) y la lista de destinatarios (*recipient_list*) como argumentos obligatorios. Al establecer el argumento opcional `fail_silently=False`, le indicamos que genere una excepción si el correo electrónico no se puede enviar. Si el resultado que ves es `1`, entonces tu correo electrónico se envió correctamente.

> [!NOTE]
> Si obtienes un error `CERTIFICATE_VERIFY_FAILED`, instala el módulo `certifi` con el comando `pip install --upgrade certifi`. Si estás usando macOS, ejecuta el siguiente comando en la consola para instalar `certifi` y permitir que Python acceda a los certificados raíz de macOS:
> ```bash
> /Applications/Python\ 3.12/Install\ Certificates.command
> ```

Revisa tu bandeja de entrada. Deberías haber recibido el correo electrónico como se muestra en la Figura 2.15:

> *Figura 2.15: Correo electrónico de prueba enviado mostrado en Gmail*

¡Acabas de enviar tu primer correo electrónico con Django! Puedes encontrar más información sobre cómo enviar correos electrónicos con Django en [https://docs.djangoproject.com/en/5.2/topics/email/](https://docs.djangoproject.com/en/5.2/topics/email/).

Agreguemos esta funcionalidad a la vista `post_share`.

#### Envío de correos electrónicos en vistas

Edita la vista `post_share` en el archivo `views.py` de la aplicación `blog`, de la siguiente manera:

```python
# ...
from django.core.mail import send_mail
# ...


def post_share(request, post_id):
    # Retrieve post by id
    post = get_object_or_404(
        Post,
        id=post_id,
        status=Post.Status.PUBLISHED
    )
    sent = False
    if request.method == 'POST':
        # Form was submitted
        form = EmailPostForm(request.POST)
        if form.is_valid():
            # Form fields passed validation
            cd = form.cleaned_data
            post_url = request.build_absolute_uri(
                post.get_absolute_url()
            )
            subject = (
                f"{cd['name']} ({cd['email']}) "
                f"recommends you read {post.title}"
            )
            message = (
                f"Read {post.title} at {post_url}\n\n"
                f"{cd['name']}\'s comments: {cd['comments']}"
            )
            send_mail(
                subject=subject,
                message=message,
                from_email=None,
                recipient_list=[cd['to']]
            )
            sent = True
    else:
        form = EmailPostForm()
    return render(
        request,
        'blog/post/share.html',
        {
            'post': post,
            'form': form,
            'sent': sent
        }
    )
```

En el código anterior, hemos declarado una variable `sent` con el valor inicial `False`. Establecemos esta variable en `True` después de que se envía el correo electrónico. Usaremos la variable `sent` más adelante en la plantilla para mostrar un mensaje de éxito cuando el formulario se envíe correctamente.

Dado que tenemos que incluir un enlace a la publicación en el correo electrónico, recuperamos la ruta absoluta de la publicación utilizando su método `get_absolute_url()`. Usamos esta ruta como entrada para `request.build_absolute_uri()` para construir una URL completa, incluyendo el esquema HTTP y el nombre de host.

Creamos el asunto y el cuerpo del mensaje del correo electrónico utilizando los datos limpios del formulario validado. Finalmente, enviamos el correo electrónico a la dirección de correo electrónico contenida en el campo `to` del formulario. En el parámetro `from_email`, pasamos el valor `None`, por lo que se utilizará el valor de la configuración `DEFAULT_FROM_EMAIL` para el remitente.

Ahora que la vista está completa, tenemos que agregar un nuevo patrón de URL para ella.

Abre el archivo `urls.py` de tu aplicación `blog` y añade el patrón de URL `post_share`, de la siguiente manera:

```python
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    # Post views
    # path('', views.post_list, name='post_list'),
    path('', views.PostListView.as_view(), name='post_list'),
    path(
        '<int:year>/<int:month>/<int:day>/<slug:post>/',
        views.post_detail,
        name='post_detail'
    ),
    path('<int:post_id>/share/', views.post_share, name='post_share'),
]
```

#### Renderizado de formularios en plantillas

Después de crear el formulario, programar la vista y agregar el patrón de URL, lo único que falta es la plantilla para la vista.

Crea un nuevo archivo en el directorio `blog/templates/blog/post/` y nómbralo `share.html`.

Añade el siguiente código a la nueva plantilla `share.html`:

```html
{% extends "blog/base.html" %}

{% block title %}Share a post{% endblock %}

{% block content %}
    {% if sent %}
        <h1>E-mail successfully sent</h1>
        <p>
            "{{ post.title }}" was successfully sent to {{ form.cleaned_data.to }}.
        </p>
    {% else %}
        <h1>Share "{{ post.title }}" by e-mail</h1>
        <form method="post">
            {{ form.as_p }}
            {% csrf_token %}
            <input type="submit" value="Send e-mail">
        </form>
    {% endif %}
{% endblock %}
```

Esta es la plantilla que se utiliza tanto para mostrar el formulario para compartir una publicación por correo electrónico como para mostrar un mensaje de éxito cuando se ha enviado el correo electrónico. Diferenciamos entre ambos casos con `{% if sent %}`.

Para mostrar el formulario, hemos definido un elemento de formulario HTML, indicando que debe enviarse mediante el método POST:

```html
<form method="post">
```

Hemos incluido la instancia del formulario con `{{ form.as_p }}`. Le indicamos a Django que renderice los campos del formulario usando elementos de párrafo HTML `<p>` mediante el método `as_p`. También podríamos renderizar el formulario como una lista desordenada con `as_ul` o como una tabla HTML con `as_table`.

Hemos añadido una etiqueta de plantilla `{% csrf_token %}`. Esta etiqueta introduce un campo oculto con un token autogenerado para evitar ataques de falsificación de peticiones en sitios cruzados (CSRF). Estos ataques consisten en que un sitio web o programa malicioso realiza una acción no deseada para un usuario en el sitio. Puedes encontrar más información sobre CSRF en [https://owasp.org/www-community/attacks/csrf](https://owasp.org/www-community/attacks/csrf).

La etiqueta de plantilla `{% csrf_token %}` genera un campo oculto que se renderiza de esta manera:

```html
<input type='hidden' name='csrfmiddlewaretoken' value='26JjKo2lcEtYkGoV9z4XmJIEHLXN5LDR' />
```

> [!IMPORTANT]
> De forma predeterminada, Django busca el token CSRF en todas las peticiones POST. Recuerda incluir la etiqueta `csrf_token` en todos los formularios que se envían a través de POST.

Edita la plantilla `blog/post/detail.html` y haz que se vea de la siguiente manera:

```html
{% extends "blog/base.html" %}

{% block title %}{{ post.title }}{% endblock %}

{% block content %}
    <h1>{{ post.title }}</h1>
    <p class="date">
        Published {{ post.publish }} by {{ post.author }}
    </p>
    {{ post.body|linebreaks }}
    <p>
        <a href="{% url "blog:post_share" post.id %}">
            Share this post
        </a>
    </p>
{% endblock %}
```

Hemos añadido un enlace a la URL `post_share`. La URL se construye dinámicamente con la etiqueta de plantilla `{% url %}` proporcionada por Django. Usamos el espacio de nombres llamado `blog` y la URL llamada `post_share`. Pasamos el `post.id` como parámetro para construir la URL.

Abre la consola y ejecuta el siguiente comando para iniciar el servidor de desarrollo:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/blog/` en tu navegador y haz clic en el título de cualquier publicación para ver la página de detalle de la publicación.

Debajo del cuerpo de la publicación, deberías ver el enlace que acabas de agregar, como se muestra en la Figura 2.16:

> *Figura 2.16: La página de detalle de la publicación, incluyendo un enlace para compartir el artículo*

Haz clic en **Share this post**, y deberías ver la página que incluye el formulario para compartir esta publicación por correo electrónico, de la siguiente manera:

> *Figura 2.17: La página para compartir una publicación por correo electrónico*

Los estilos CSS para el formulario están incluidos en el código de ejemplo en el archivo `static/css/blog.css`. Cuando haces clic en el botón **SEND E-MAIL**, el formulario se envía y se valida. Si todos los campos contienen datos válidos, obtienes un mensaje de éxito, de la siguiente manera:

> *Figura 2.18: Un mensaje de éxito para una publicación compartida por correo electrónico*

Envía una publicación a tu propia dirección de correo electrónico y revisa tu bandeja de entrada. El correo electrónico que recibas debería verse como este:

> *Figura 2.19: Correo electrónico de prueba enviado mostrado en Gmail*

Si envías el formulario con datos no válidos, el formulario se renderizará nuevamente, incluyendo todos los errores de validación:

> *Figura 2.20: El formulario de compartir publicación mostrando errores de datos no válidos*

La mayoría de los navegadores modernos evitarán que envíes un formulario con campos vacíos o erróneos. Esto se debe a que el navegador valida los campos según sus atributos antes de enviar el formulario. En este caso, el formulario no se enviará y el navegador mostrará un mensaje de error para los campos que sean incorrectos. Para probar la validación de formularios de Django usando un navegador moderno, puedes omitir la validación de formularios del navegador agregando el atributo `novalidate` al elemento HTML `<form>`, como `<form method="post" novalidate>`. Puedes agregar este atributo para evitar que el navegador valide campos y probar tu propia validación de formulario. Una vez que hayas terminado la prueba, elimina el atributo `novalidate` para conservar la validación de formularios del navegador.

La funcionalidad para compartir publicaciones por correo electrónico ya está completa. Puedes encontrar más información sobre cómo trabajar con formularios en [https://docs.djangoproject.com/en/5.2/topics/forms/](https://docs.djangoproject.com/en/5.2/topics/forms/).

---

### Creación de un sistema de comentarios

Continuaremos ampliando nuestra aplicación de blog con un sistema de comentarios que permitirá a los usuarios comentar en las publicaciones. Para construir el sistema de comentarios, necesitaremos lo siguiente:

1. Un modelo de comentarios para almacenar los comentarios de los usuarios en las publicaciones.
2. Un formulario de Django que permita a los usuarios enviar comentarios y gestione la validación de datos.
3. Una vista que procese el formulario y guarde un nuevo comentario en la base de datos.
4. Una lista de comentarios y el formulario HTML para agregar un nuevo comentario que se puedan incluir en la plantilla de detalle de la publicación.

#### Creación de un modelo para comentarios

Comencemos creando un modelo para almacenar los comentarios de los usuarios en las publicaciones.

Abre el archivo `models.py` de tu aplicación `blog` y añade el siguiente código:

```python
class Comment(models.Model):
    post = models.ForeignKey(
        Post,
        on_delete=models.CASCADE,
        related_name='comments'
    )
    name = models.CharField(max_length=80)
    email = models.EmailField()
    body = models.TextField()
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)
    active = models.BooleanField(default=True)

    class Meta:
        ordering = ['created']
        indexes = [
            models.Index(fields=['created']),
        ]

    def __str__(self):
        return f'Comment by {self.name} on {self.post}'
```

Este es el modelo `Comment`. Hemos añadido un campo `ForeignKey` para asociar cada comentario con una única publicación. Esta relación de muchos a uno se define en el modelo `Comment` porque cada comentario se realizará en una publicación y cada publicación puede tener varios comentarios.

El atributo `related_name` te permite nombrar el atributo que usas para la relación desde el objeto relacionado de regreso a este. Podemos recuperar la publicación de un objeto de comentario usando `comment.post` y recuperar todos los comentarios asociados con un objeto de publicación usando `post.comments.all()`. Si no defines el atributo `related_name`, Django usará el nombre del modelo en minúsculas, seguido de `_set` (es decir, `comment_set`) para nombrar la relación del objeto relacionado con el objeto del modelo donde se ha definido esta relación.

Puedes aprender más sobre las relaciones de muchos a uno en [https://docs.djangoproject.com/en/5.2/topics/db/examples/many_to_one/](https://docs.djangoproject.com/en/5.2/topics/db/examples/many_to_one/).

Hemos definido el campo booleano `active` para controlar el estado de los comentarios. Este campo nos permitirá desactivar manualmente comentarios inapropiados utilizando el sitio de administración. Usamos `default=True` para indicar que todos los comentarios están activos de forma predeterminada.

Hemos definido el campo `created` para almacenar la fecha y la hora en que se creó el comentario. Al usar `auto_now_add`, la fecha se guardará automáticamente al crear un objeto. En la clase `Meta` del modelo, hemos añadido `ordering = ['created']` para ordenar los comentarios en orden cronológico de forma predeterminada y hemos añadido un índice para el campo `created` en orden ascendente. Esto mejorará el rendimiento de las búsquedas en bases de datos o la ordenación de resultados utilizando el campo `created`.

El modelo `Comment` que hemos construido no está sincronizado con la base de datos. Necesitamos generar una nueva migración de base de datos para crear la tabla correspondiente en la base de datos.

Ejecuta el siguiente comando en la consola:

```bash
python manage.py makemigrations blog
```

Deberías ver la siguiente salida:

```text
Migrations for 'blog':
  blog/migrations/0003_comment.py
    - Create model Comment
```

Django ha generado un archivo `0003_comment.py` dentro del directorio `migrations/` de la aplicación `blog`. Necesitamos crear el esquema de base de datos relacionado y aplicar los cambios a la base de datos.

Ejecuta el siguiente comando para aplicar las migraciones existentes:

```bash
python manage.py migrate
```

Obtendrás una salida que incluye la siguiente línea:

```text
Applying blog.0003_comment... OK
```

La migración se ha aplicado y la tabla `blog_comment` se ha creado en la base de datos.

#### Adición de comentarios al sitio de administración

A continuación, añadiremos el nuevo modelo al sitio de administración para gestionar comentarios a través de una interfaz sencilla.

Abre el archivo `admin.py` de la aplicación `blog`, importa el modelo `Comment` y añade la siguiente clase `ModelAdmin`:

```python
from .models import Comment, Post


@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    list_display = ['name', 'email', 'post', 'created', 'active']
    list_filter = ['active', 'created', 'updated']
    search_fields = ['name', 'email', 'body']
```

Abre la consola y ejecuta el siguiente comando para iniciar el servidor de desarrollo:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/admin/` en tu navegador. Deberías ver el nuevo modelo incluido en la sección BLOG, como se muestra en la Figura 2.21:

> *Figura 2.21: Modelos de la aplicación blog en la página de índice de administración de Django*

El modelo ya está registrado en el sitio de administración.

En la fila **Comments**, haz clic en **Add**. Verás el formulario para agregar un nuevo comentario:

> *Figura 2.22: Formulario para añadir un nuevo comentario en el sitio de administración de Django*

Ahora podemos gestionar instancias de `Comment` utilizando el sitio de administración.

#### Creación de formularios a partir de modelos

Necesitamos crear un formulario para permitir que los usuarios comenten en las publicaciones del blog. Recuerda que Django tiene dos clases base que se pueden usar para crear formularios: `Form` y `ModelForm`. Usamos la clase `Form` para permitir que los usuarios compartan publicaciones por correo electrónico. Ahora, utilizaremos `ModelForm` para aprovechar el modelo `Comment` existente y construir un formulario dinámicamente para él.

Edita el archivo `forms.py` de tu aplicación `blog` y agrégale las siguientes líneas:

```python
from .models import Comment


class CommentBoundField(forms.BoundField):
    comment_class = "comment"

    def css_classes(self, extra_classes=None):
        result = super().css_classes(extra_classes)
        if self.comment_class not in result:
            result += f" {self.comment_class}"
        return result.strip()


class CommentForm(forms.ModelForm):
    bound_field_class = CommentBoundField

    class Meta:
        model = Comment
        fields = ['name', 'email', 'body']
```

> [!NOTE]
> Como novedad en Django 5.2, estilizar tus formularios es más fácil que nunca con la sobrescritura simplificada del `BoundField` de un formulario. Con `CommentBoundField`, los campos del formulario de comentarios ahora están envueltos en un `div` con la clase `comment`. Para más detalles, consulta el Apéndice, al que puedes acceder aquí: [https://packt.link/1g7Af](https://packt.link/1g7Af).

Para crear un formulario a partir de un modelo, simplemente indicamos para qué modelo construir el formulario en la clase `Meta` del formulario. Django inspeccionará el modelo y creará el formulario correspondiente dinámicamente.

Cada tipo de campo de modelo tiene un tipo de campo de formulario predeterminado correspondiente. Los atributos de los campos de modelo se tienen en cuenta para la validación del formulario. De forma predeterminada, Django crea un campo de formulario para cada campo contenido en el modelo. Sin embargo, podemos indicarle explícitamente a Django qué campos incluir en el formulario mediante el atributo `fields` o definir qué campos excluir mediante el atributo `exclude`. En el formulario `CommentForm`, hemos incluido explícitamente los campos `name`, `email` y `body`. Estos son los únicos campos que se incluirán en el formulario.

Puedes encontrar más información sobre la creación de formularios a partir de modelos en [https://docs.djangoproject.com/en/5.2/topics/forms/modelforms/](https://docs.djangoproject.com/en/5.2/topics/forms/modelforms/).

#### Manejo de ModelForms en vistas

Para compartir publicaciones por correo electrónico, utilizamos la misma vista para mostrar el formulario y gestionar su envío. Utilizamos el método HTTP para diferenciar entre ambos casos: GET para mostrar el formulario y POST para enviarlo. En este caso, añadiremos el formulario de comentarios a la página de detalle de la publicación y crearemos una vista independiente para gestionar el envío del formulario. La nueva vista que procesa el formulario permitirá al usuario regresar a la vista de detalle de la publicación una vez que el comentario se haya guardado en la base de datos.

Edita el archivo `views.py` de la aplicación `blog` y agrégale el siguiente código:

```python
from django.core.mail import send_mail
from django.core.paginator import EmptyPage, PageNotAnInteger, Paginator
from django.shortcuts import get_object_or_404, render
from django.views.decorators.http import require_POST
from django.views.generic import ListView
from .forms import CommentForm, EmailPostForm
from .models import Post

# ...


@require_POST
def post_comment(request, post_id):
    post = get_object_or_404(
        Post,
        id=post_id,
        status=Post.Status.PUBLISHED
    )
    comment = None
    # A comment was posted
    form = CommentForm(data=request.POST)
    if form.is_valid():
        # Create a Comment object without saving it to the database
        comment = form.save(commit=False)
        # Assign the post to the comment
        comment.post = post
        # Save the comment to the database
        comment.save()
    return render(
        request,
        'blog/post/comment.html',
        {
            'post': post,
            'form': form,
            'comment': comment
        }
    )
```

Hemos definido la vista `post_comment` que toma el objeto `request` y la variable `post_id` como parámetros. Usaremos esta vista para gestionar el envío del comentario. Esperamos que el formulario se envíe utilizando el método HTTP POST. Usamos el decorador `require_POST` proporcionado por Django para permitir solo peticiones POST para esta vista. Django te permite restringir los métodos HTTP permitidos para las vistas. Django arrojará un error HTTP 405 (*method not allowed*) si intentas acceder a la vista con cualquier otro método HTTP.

En esta vista, hemos implementado las siguientes acciones:

1. Recuperamos una publicación publicada por su `id` mediante el atajo `get_object_or_404()`.
2. Definimos una variable `comment` con el valor inicial `None`. Esta variable se utilizará para almacenar el objeto de comentario cuando se cree.
3. Instanciamos el formulario utilizando los datos POST enviados y lo validamos mediante el método `is_valid()`. Si el formulario no es válido, la plantilla se renderiza con los errores de validación.
4. Si el formulario es válido, creamos un nuevo objeto `Comment` llamando al método `save()` del formulario y lo asignamos a la variable `comment`, de la siguiente manera:
   ```python
   comment = form.save(commit=False)
   ```
   El método `save()` crea una instancia del modelo al que está vinculado el formulario y la guarda en la base de datos. Si lo llamas usando `commit=False`, la instancia del modelo se crea pero no se guarda en la base de datos. Esto nos permite modificar el objeto antes de guardarlo definitivamente.
   El método `save()` está disponible para instancias de `ModelForm` pero no para instancias de `Form`, ya que no están vinculadas a ningún modelo.
5. Asignamos la publicación al comentario que creamos:
   ```python
   comment.post = post
   ```
6. Guardamos el nuevo comentario en la base de datos llamando a su método `save()`:
   ```python
   comment.save()
   ```
7. Renderizamos la plantilla `blog/post/comment.html`, pasando los objetos `post`, `form` y `comment` en el contexto de la plantilla. Esta plantilla aún no existe; la crearemos más adelante.

Creemos un patrón de URL para esta vista.

Edita el archivo `urls.py` de la aplicación `blog` y agrégale el siguiente patrón de URL:

```python
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    # Post views
    # path('', views.post_list, name='post_list'),
    path('', views.PostListView.as_view(), name='post_list'),
    path(
        '<int:year>/<int:month>/<int:day>/<slug:post>/',
        views.post_detail,
        name='post_detail'
    ),
    path('<int:post_id>/share/', views.post_share, name='post_share'),
    path(
        '<int:post_id>/comment/',
        views.post_comment,
        name='post_comment'
    ),
]
```

Hemos implementado la vista para gestionar el envío de comentarios y su correspondiente URL. Creemos las plantillas necesarias.

#### Creación de plantillas para el formulario de comentarios

Crearemos una plantilla para el formulario de comentarios que utilizaremos en dos lugares:

1. En la plantilla de detalle de la publicación asociada con la vista `post_detail` para permitir a los usuarios publicar comentarios.
2. En la plantilla de comentarios de la publicación asociada con la vista `post_comment` para mostrar el formulario nuevamente si hay algún error en el formulario.

Crearemos la plantilla del formulario y utilizaremos la etiqueta de plantilla `{% include %}` para incluirla en las otras dos plantillas.

En el directorio `templates/blog/post/`, crea un nuevo directorio `includes/`. Añade un nuevo archivo dentro de este directorio y nómbralo `comment_form.html`.

La estructura de archivos debería verse de la siguiente manera:

```text
templates/
    blog/
        post/
            includes/
                comment_form.html
            detail.html
            list.html
            share.html
```

Edita la nueva plantilla `blog/post/includes/comment_form.html` y añade el siguiente código:

```html
<h2>Add a new comment</h2>
<form action="{% url "blog:post_comment" post.id %}" method="post">
    {{ form.as_p }}
    {% csrf_token %}
    <p><input type="submit" value="Add comment"></p>
</form>
```

En esta plantilla, construimos la URL de acción del elemento `<form>` HTML dinámicamente utilizando la etiqueta de plantilla `{% url %}`. Construimos la URL de la vista `post_comment` que procesará el formulario. Mostramos el formulario renderizado en párrafos e incluimos `{% csrf_token %}` para la protección CSRF porque este formulario se enviará con el método POST.

Crea un nuevo archivo en el directorio `templates/blog/post/` de la aplicación `blog` y nómbralo `comment.html`.

La estructura de archivos ahora debería verse de la siguiente manera:

```text
templates/
    blog/
        post/
            includes/
                comment_form.html
            comment.html
            detail.html
            list.html
            share.html
```

Edita la nueva plantilla `blog/post/comment.html` y añade el siguiente código:

```html
{% extends "blog/base.html" %}

{% block title %}Add a comment{% endblock %}

{% block content %}
    {% if comment %}
        <h2>Your comment has been added.</h2>
        <p><a href="{{ post.get_absolute_url }}">Back to the post</a></p>
    {% else %}
        {% include "blog/post/includes/comment_form.html" %}
    {% endif %}
{% endblock %}
```

Esta es la plantilla para la vista de comentarios de publicaciones. En esta vista, esperamos que el formulario se envíe mediante el método POST. La plantilla cubre dos escenarios diferentes:

- Si los datos del formulario enviados son válidos, la variable `comment` contendrá el objeto de comentario que se creó y se mostrará un mensaje de éxito.
- Si los datos del formulario enviados no son válidos, la variable `comment` será `None`. En este caso, mostraremos el formulario de comentarios. Usamos la etiqueta de plantilla `{% include %}` para incluir la plantilla `comment_form.html` que hemos creado previamente.

#### Adición de comentarios a la vista de detalle de publicación

Para completar la funcionalidad de comentarios, añadiremos la lista de comentarios y el formulario de comentarios a la vista `post_detail`.

Edita el archivo `views.py` de la aplicación `blog` y modifica la vista `post_detail` de la siguiente manera:

```python
def post_detail(request, year, month, day, post):
    post = get_object_or_404(
        Post,
        status=Post.Status.PUBLISHED,
        slug=post,
        publish__year=year,
        publish__month=month,
        publish__day=day
    )
    # List of active comments for this post
    comments = post.comments.filter(active=True)
    # Form for users to comment
    form = CommentForm()
    return render(
        request,
        'blog/post/detail.html',
        {
            'post': post,
            'comments': comments,
            'form': form
        }
    )
```

Revisemos el código que hemos añadido a la vista `post_detail`:

Hemos añadido un `QuerySet` para recuperar todos los comentarios activos para la publicación, de la siguiente manera:

```python
comments = post.comments.filter(active=True)
```

Este `QuerySet` se construye usando el objeto `post`. En lugar de construir un `QuerySet` para el modelo `Comment` directamente, aprovechamos el objeto `post` para recuperar los objetos `Comment` relacionados. Usamos el manager `comments` para los objetos `Comment` relacionados que definimos previamente en el modelo `Comment`, usando el atributo `related_name` del campo `ForeignKey` al modelo `Post`.

También hemos creado una instancia del formulario de comentarios con `form = CommentForm()`.

#### Adición de comentarios a la plantilla de detalle de publicación

Necesitamos editar la plantilla `blog/post/detail.html` para implementar lo siguiente:

- Mostrar el número total de comentarios de una publicación
- Mostrar la lista de comentarios
- Mostrar el formulario para que los usuarios añadan un nuevo comentario

Comenzaremos agregando el número total de comentarios para una publicación.

Edita la plantilla `blog/post/detail.html` y cámbiala de la siguiente manera:

```html
{% extends "blog/base.html" %}

{% block title %}{{ post.title }}{% endblock %}

{% block content %}
    <h1>{{ post.title }}</h1>
    <p class="date">
        Published {{ post.publish }} by {{ post.author }}
    </p>
    {{ post.body|linebreaks }}
    <p>
        <a href="{% url "blog:post_share" post.id %}">
            Share this post
        </a>
    </p>
    {% with comments.count as total_comments %}
        <h2>
            {{ total_comments }} comment{{ total_comments|pluralize }}
        </h2>
    {% endwith %}
{% endblock %}
```

Usamos el mapeador objeto-relacional (ORM) de Django en la plantilla, ejecutando el QuerySet `comments.count()`. Ten en cuenta que el lenguaje de plantillas de Django no utiliza paréntesis para llamar a métodos. La etiqueta `{% with %}` te permite asignar un valor a una nueva variable que estará disponible en la plantilla hasta la etiqueta `{% endwith %}`.

> [!TIP]
> La etiqueta de plantilla `{% with %}` es útil para evitar consultar la base de datos o acceder a métodos costosos varias veces.

Utilizamos el filtro de plantilla `pluralize` para mostrar un sufijo en plural para la palabra "comment", dependiendo del valor de `total_comments`. Los filtros de plantilla toman el valor de la variable a la que se aplican como entrada y devuelven un valor calculado. Aprenderemos más sobre filtros de plantilla en el Capítulo 3, *Extensión de tu aplicación de blog*.

El filtro de plantilla `pluralize` devuelve una cadena con la letra "s" si el valor es diferente de 1. El texto anterior se representará como 0 comments, 1 comment o N comments, según la cantidad de comentarios activos para la publicación.

Ahora, agreguemos la lista de comentarios activos a la plantilla de detalle de la publicación.

Edita la plantilla `blog/post/detail.html` e implementa los siguientes cambios:

```html
{% extends "blog/base.html" %}

{% block title %}{{ post.title }}{% endblock %}

{% block content %}
    <h1>{{ post.title }}</h1>
    <p class="date">
        Published {{ post.publish }} by {{ post.author }}
    </p>
    {{ post.body|linebreaks }}
    <p>
        <a href="{% url "blog:post_share" post.id %}">
            Share this post
        </a>
    </p>
    {% with comments.count as total_comments %}
        <h2>
            {{ total_comments }} comment{{ total_comments|pluralize }}
        </h2>
    {% endwith %}
    {% for comment in comments %}
        <div class="comment">
            <p class="info">
                Comment {{ forloop.counter }} by {{ comment.name }}
                {{ comment.created }}
            </p>
            {{ comment.body|linebreaks }}
        </div>
    {% empty %}
        <p>There are no comments.</p>
    {% endfor %}
{% endblock %}
```

Hemos añadido una etiqueta de plantilla `{% for %}` para recorrer los comentarios de la publicación. Si la lista de comentarios está vacía, mostramos un mensaje que informa a los usuarios que no hay comentarios para esta publicación. Enumeramos los comentarios con la variable `{{ forloop.counter }}`, que contiene el contador de bucle en cada iteración. Para cada publicación, mostramos el nombre del usuario que la publicó, la fecha y el cuerpo del comentario.

Finalmente, agreguemos el formulario de comentarios a la plantilla.

Edita la plantilla `blog/post/detail.html` e incluye la plantilla del formulario de comentarios de la siguiente manera:

```html
{% extends "blog/base.html" %}

{% block title %}{{ post.title }}{% endblock %}

{% block content %}
    <h1>{{ post.title }}</h1>
    <p class="date">
        Published {{ post.publish }} by {{ post.author }}
    </p>
    {{ post.body|linebreaks }}
    <p>
        <a href="{% url "blog:post_share" post.id %}">
            Share this post
        </a>
    </p>
    {% with comments.count as total_comments %}
        <h2>
            {{ total_comments }} comment{{ total_comments|pluralize }}
        </h2>
    {% endwith %}
    {% for comment in comments %}
        <div class="comment">
            <p class="info">
                Comment {{ forloop.counter }} by {{ comment.name }}
                {{ comment.created }}
            </p>
            {{ comment.body|linebreaks }}
        </div>
    {% empty %}
        <p>There are no comments.</p>
    {% endfor %}
    {% include "blog/post/includes/comment_form.html" %}
{% endblock %}
```

Abre `http://127.0.0.1:8000/blog/` en tu navegador y haz clic en el título de una publicación para ver la página de detalle de la publicación. Verás algo como la Figura 2.23:

> *Figura 2.23: La página de detalle de la publicación, incluyendo el formulario para añadir un comentario*

Completa el formulario de comentarios con datos válidos y haz clic en **Add comment**. Deberías ver la siguiente página:

> *Figura 2.24: La página de éxito de comentario añadido*

Haz clic en el enlace **Back to the post**. Deberías ser redirigido nuevamente a la página de detalle de la publicación y deberías poder ver el comentario que acabas de agregar, de la siguiente manera:

> *Figura 2.25: La página de detalle de la publicación, incluyendo un comentario*

Añade un comentario más a la publicación. Los comentarios deberían aparecer debajo del contenido de la publicación en orden cronológico, de la siguiente manera:

> *Figura 2.26: La lista de comentarios en la página de detalle de la publicación*

Abre `http://127.0.0.1:8000/admin/blog/comment/` en tu navegador. Verás la página de administración con la lista de comentarios que creaste, de esta forma:

> *Figura 2.27: Lista de comentarios en el sitio de administración*

Haz clic en el nombre de una de las publicaciones para editarla. Desmarca la casilla de verificación **Active** de la siguiente manera y haz clic en el botón **Save**:

> *Figura 2.28: Edición de un comentario en el sitio de administración*

Serás redirigido a la lista de comentarios. La columna **Active** mostrará un icono inactivo para el comentario, como se muestra en la Figura 2.29:

> *Figura 2.29: Comentarios activos/inactivos en el sitio de administración*

Si regresas a la vista de detalle de la publicación, notarás que el comentario inactivo ya no se muestra, ni tampoco se cuenta para el número total de comentarios activos de la publicación:

> *Figura 2.30: Un único comentario activo mostrado en la página de detalle de la publicación*

Gracias al campo `active`, puedes desactivar comentarios inapropiados y evitar que se muestren en tus publicaciones.

#### Uso de plantillas simplificadas para el renderizado de formularios

Has utilizado `{{ form.as_p }}` para renderizar los formularios mediante párrafos HTML. Este es un método muy sencillo para renderizar formularios, pero puede haber ocasiones en las que necesites emplear marcado HTML personalizado para renderizar formularios.

Para utilizar HTML personalizado para renderizar campos de formulario, puedes acceder a cada campo de formulario directamente o iterar a través de los campos de formulario, como en el siguiente ejemplo:

```html
{% for field in form %}
    <div class="my-div">
        {{ field.errors }}
        {{ field.label_tag }}
        {{ field }}
        <div class="help-text">{{ field.help_text|safe }}</div>
    </div>
{% endfor %}
```

En este código, usamos `{{ field.errors }}` para renderizar cualquier error de campo del formulario, `{{ field.label_tag }}` para renderizar la etiqueta HTML del formulario, `{{ field }}` para renderizar el campo real y `{{ field.help_text|safe }}` para renderizar el HTML del texto de ayuda del campo.

Este método es útil para personalizar cómo se renderizan los formularios, pero es posible que necesites agregar ciertos elementos HTML para campos específicos o incluir algunos campos en contenedores. Django 5.0 introdujo grupos de campos (*field groups*) y plantillas de grupos de campos. Los grupos de campos simplifican el renderizado de etiquetas, widgets, textos de ayuda y errores de campo. Usemos esta nueva característica para personalizar el formulario de comentarios.

Vamos a utilizar marcado HTML personalizado para reposicionar los campos de formulario de nombre y correo electrónico utilizando elementos HTML adicionales.

Edita la plantilla `blog/post/includes/comment_form.html` y modifícala de la siguiente manera:

```html
<h2>Add a new comment</h2>
<form action="{% url "blog:post_comment" post.id %}" method="post">
    <div class="left">
        {{ form.name.as_field_group }}
    </div>
    <div class="left">
        {{ form.email.as_field_group }}
    </div>
    {{ form.body.as_field_group }}
    {% csrf_token %}
    <p><input type="submit" value="Add comment"></p>
</form>
```

Hemos añadido contenedores `<div>` para los campos de nombre y correo electrónico con una clase CSS personalizada para hacer flotar ambos campos a la izquierda. El método `as_field_group` renderiza cada campo incluyendo texto de ayuda y errores. Este método utiliza la plantilla `django/forms/field.html` de forma predeterminada. Puedes ver el contenido de esta plantilla en [https://github.com/django/django/blob/stable/5.2.x/django/forms/templates/django/forms/field.html](https://github.com/django/django/blob/stable/5.2.x/django/forms/templates/django/forms/field.html). También puedes crear plantillas de campo personalizadas y reutilizarlas añadiendo el atributo `template_name` a cualquier campo del formulario. Puedes leer más sobre plantillas de formulario reutilizables en [https://docs.djangoproject.com/en/5.2/topics/forms/#reusable-field-group-templates](https://docs.djangoproject.com/en/5.2/topics/forms/#reusable-field-group-templates).

Abre una publicación de blog y echa un vistazo al formulario de comentarios. El formulario ahora debería verse como la Figura 2.31:

> *Figura 2.31: El formulario de comentarios con el nuevo marcado HTML*

Los campos de nombre y correo electrónico ahora se muestran uno al lado del otro. Los grupos de campos te permiten personalizar fácilmente el renderizado de formularios.

---

### Resumen

En este capítulo, aprendiste a definir URLs canónicas para modelos. Creaste URLs amigables para SEO para publicaciones de blog e implementaste paginación de objetos para tu lista de publicaciones. También aprendiste a trabajar con formularios y model forms de Django. Creaste un sistema para recomendar publicaciones por correo electrónico y creaste un sistema de comentarios para tu blog.

En el próximo capítulo, crearás un sistema de etiquetas (*tags*) para el blog. Aprenderás a construir QuerySets complejos para recuperar objetos por similitud. Aprenderás a crear etiquetas y filtros de plantilla personalizados. También crearás un sitemap personalizado y un feed para las publicaciones de tu blog, e implementarás la funcionalidad de búsqueda de texto completo para tus publicaciones.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter02](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter02)
- **Funciones de utilidad de URL:** [https://docs.djangoproject.com/en/5.2/ref/urlresolvers/](https://docs.djangoproject.com/en/5.2/ref/urlresolvers/)
- **Convertidores de ruta de URL:** [https://docs.djangoproject.com/en/5.2/topics/http/urls/#path-converters](https://docs.djangoproject.com/en/5.2/topics/http/urls/#path-converters)
- **Clase Paginator de Django:** [https://docs.djangoproject.com/en/5.2/ref/paginator/](https://docs.djangoproject.com/en/5.2/ref/paginator/)
- **Introducción a las vistas basadas en clases:** [https://docs.djangoproject.com/en/5.2/topics/class-based-views/intro/](https://docs.djangoproject.com/en/5.2/topics/class-based-views/intro/)
- **Envío de correos electrónicos con Django:** [https://docs.djangoproject.com/en/5.2/topics/email/](https://docs.djangoproject.com/en/5.2/topics/email/)
- **La biblioteca python-decouple:** [https://github.com/HBNetwork/python-decouple](https://github.com/HBNetwork/python-decouple)
- **La biblioteca django-anymail:** [https://anymail.dev/en/stable/installation/](https://anymail.dev/en/stable/installation/)
- **Proveedores de servicios de correo electrónico compatibles con django-anymail:** [https://anymail.dev/en/stable/esps/](https://anymail.dev/en/stable/esps/)
- **Tipos de campos de formulario de Django:** [https://docs.djangoproject.com/en/5.2/ref/forms/fields/](https://docs.djangoproject.com/en/5.2/ref/forms/fields/)
- **Trabajar con formularios:** [https://docs.djangoproject.com/en/5.2/topics/forms/](https://docs.djangoproject.com/en/5.2/topics/forms/)
- **Creación de formularios a partir de modelos:** [https://docs.djangoproject.com/en/5.2/topics/forms/modelforms/](https://docs.djangoproject.com/en/5.2/topics/forms/modelforms/)
- **Relaciones de modelos muchos a uno:** [https://docs.djangoproject.com/en/5.2/topics/db/examples/many_to_one/](https://docs.djangoproject.com/en/5.2/topics/db/examples/many_to_one/)
- **Plantilla de campo de formulario predeterminada:** [https://github.com/django/django/blob/stable/5.2.x/django/forms/templates/django/forms/field.html](https://github.com/django/django/blob/stable/5.2.x/django/forms/templates/django/forms/field.html)
- **Plantillas de grupos de campos reutilizables:** [https://docs.djangoproject.com/en/5.2/topics/forms/#reusable-field-group-templates](https://docs.djangoproject.com/en/5.2/topics/forms/#reusable-field-group-templates)
