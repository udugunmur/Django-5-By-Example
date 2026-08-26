# Parte 1: Aplicación de Blog

## Capítulo 3: Extensión de tu aplicación de blog

### Introducción

El capítulo anterior abordó los conceptos básicos de los formularios y la creación de un sistema de comentarios. También aprendiste a enviar correos electrónicos con Django. En este capítulo, ampliarás tu aplicación de blog con otras características populares utilizadas en las plataformas de blogs, como el etiquetado, la recomendación de publicaciones similares, la provisión de un canal RSS para los lectores y la posibilidad de que busquen publicaciones. Aprenderás sobre nuevos componentes y funcionalidades de Django mediante la construcción de estas características.

Este capítulo cubrirá los siguientes temas:

- Implementación del sistema de etiquetas utilizando `django-taggit`
- Recuperación de publicaciones por similitud
- Creación de etiquetas y filtros de plantilla personalizados para mostrar las últimas publicaciones y las más comentadas
- Adición de un sitemap al sitio
- Creación de canales de sindicación (feeds) para las publicaciones del blog
- Instalación de PostgreSQL
- Uso de fixtures para volcar y cargar datos en la base de datos
- Implementación de un motor de búsqueda de texto completo con Django y PostgreSQL

---

### Visión general funcional

La Figura 3.1 muestra una representación de las vistas, plantillas y funcionalidades que se construirán en este capítulo:

> *Figura 3.1: Diagrama de funcionalidades construidas en el Capítulo 3*

En este capítulo, crearemos la funcionalidad para añadir etiquetas a las publicaciones. Ampliaremos la vista `post_list` para filtrar publicaciones por etiqueta. Al cargar una sola publicación en la vista `post_detail`, recuperaremos publicaciones similares basadas en etiquetas comunes. También crearemos etiquetas de plantilla personalizadas para mostrar una barra lateral con el número total de publicaciones, las últimas publicaciones publicadas y las publicaciones más comentadas.

Añadiremos soporte para escribir publicaciones con sintaxis Markdown y convertir el contenido a HTML. Crearemos un sitemap para el blog con la clase `PostSitemap` e implementaremos un feed RSS con las últimas publicaciones en la clase `LatestPostsFeed`. Finalmente, implementaremos un motor de búsqueda con la vista `post_search` y utilizaremos las capacidades de búsqueda de texto completo de PostgreSQL.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter03](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter03).

Todos los paquetes de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente del capítulo. Puedes seguir las instrucciones para instalar cada paquete de Python en las siguientes secciones, o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Implementación del sistema de etiquetas con django-taggit

Una funcionalidad muy común en los blogs es la categorización de publicaciones mediante etiquetas (*tags*). Las etiquetas te permiten categorizar el contenido de forma no jerárquica, utilizando palabras clave simples. Una etiqueta es simplemente una etiqueta o palabra clave que se puede asignar a las publicaciones. Crearemos un sistema de etiquetado integrando una aplicación de etiquetado de Django de terceros en el proyecto.

`django-taggit` es una aplicación reutilizable que ofrece principalmente un modelo `Tag` y un manager para añadir etiquetas fácilmente a cualquier modelo. Puedes ver su código fuente en [https://github.com/jazzband/django-taggit](https://github.com/jazzband/django-taggit).

Añadamos el etiquetado a nuestro blog. Primero, necesitas instalar `django-taggit` a través de pip ejecutando el siguiente comando:

```bash
python -m pip install django-taggit==5.0.1
```

Luego, abre el archivo `settings.py` del proyecto `mysite` y añade `taggit` a tu configuración `INSTALLED_APPS`, de la siguiente manera:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'taggit',
    'blog.apps.BlogConfig',
]
```

Es una buena práctica mantener los paquetes propios de Django en la parte superior, los paquetes de terceros en el medio y las aplicaciones locales al final de `INSTALLED_APPS`.

Abre el archivo `models.py` de tu aplicación `blog` y añade el manager `TaggableManager` proporcionado por `django-taggit` al modelo `Post` utilizando el siguiente código:

```python
from taggit.managers import TaggableManager


class Post(models.Model):
    # ...
    tags = TaggableManager()
```

El manager `tags` te permitirá añadir, recuperar y eliminar etiquetas de los objetos `Post`.

El siguiente esquema muestra los modelos de datos definidos por `django-taggit` para crear etiquetas y almacenar objetos etiquetados relacionados:

> *Figura 3.2: Modelos de etiquetas de django-taggit*

El modelo `Tag` se utiliza para almacenar etiquetas. Contiene los campos `name` y `slug`.

El modelo `TaggedItem` se utiliza para almacenar los objetos etiquetados relacionados. Tiene un campo `ForeignKey` para el objeto `Tag` relacionado. Contiene un `ForeignKey` a un objeto `ContentType` y un `IntegerField` para almacenar el `id` relacionado del objeto etiquetado. Los campos `content_type` y `object_id` combinados forman una relación genérica con cualquier modelo de tu proyecto. Esto te permite crear relaciones entre una instancia de `Tag` y cualquier otra instancia de modelo de tus aplicaciones. Aprenderás sobre relaciones genéricas en el Capítulo 7, *Seguimiento de las acciones del usuario*.

Ejecuta el siguiente comando en la consola para crear una migración para los cambios de tu modelo:

```bash
python manage.py makemigrations blog
```

Deberías obtener la siguiente salida:

```text
Migrations for 'blog':
  blog/migrations/0004_post_tags.py
    - Add field tags to post
```

Ahora, ejecuta el siguiente comando para crear las tablas de base de datos requeridas para los modelos de `django-taggit` y para sincronizar los cambios de tu modelo:

```bash
python manage.py migrate
```

Verás una salida que indica que las migraciones se han aplicado, de la siguiente manera:

```text
Applying taggit.0001_initial... OK
Applying taggit.0002_auto_20150616_2121... OK
Applying taggit.0003_taggeditem_add_unique_index... OK
Applying taggit.0004_alter_taggeditem_content_type_alter_taggeditem_tag... OK
Applying taggit.0005_auto_20220424_2025... OK
Applying taggit.0006_rename_taggeditem_content_type_object_id_taggit_tagg_content_8fc721_idx... OK
Applying blog.0004_post_tags... OK
```

La base de datos ahora está sincronizada con los modelos de `taggit` y podemos empezar a utilizar las funcionalidades de `django-taggit`.

Exploremos ahora cómo utilizar el manager de etiquetas.

Abre la shell de Django ejecutando el siguiente comando en la consola del sistema:

```bash
python manage.py shell
```

Ejecuta el siguiente código para recuperar una de las publicaciones (la que tiene el ID 1):

```python
>>> from blog.models import Post
>>> post = Post.objects.get(id=1)
```

Luego, añádele algunas etiquetas y recupera sus etiquetas para comprobar si se añadieron correctamente:

```python
>>> post.tags.add('music', 'jazz', 'django')
>>> post.tags.all()
<QuerySet [<Tag: jazz>, <Tag: music>, <Tag: django>]>
```

Finalmente, elimina una etiqueta y revisa la lista de etiquetas nuevamente:

```python
>>> post.tags.remove('django')
>>> post.tags.all()
<QuerySet [<Tag: jazz>, <Tag: music>]>
```

Es realmente fácil añadir, recuperar o eliminar etiquetas de un modelo utilizando el manager que hemos definido.

Inicia el servidor de desarrollo desde la consola con el siguiente comando:

```bash
python manage.py runserver
```

Abre [http://127.0.0.1:8000/admin/taggit/tag/](http://127.0.0.1:8000/admin/taggit/tag/) en tu navegador.

Verás la página de administración con la lista de objetos `Tag` de la aplicación `taggit`:

> *Figura 3.3: Vista de lista de etiquetas en el sitio de administración de Django*

Haz clic en la etiqueta **jazz**. Verás lo siguiente:

> *Figura 3.4: Vista de edición de etiquetas en el sitio de administración de Django*

Navega a `http://127.0.0.1:8000/admin/blog/post/1/change/` para editar la publicación con ID 1.

Verás que las publicaciones ahora incluyen un nuevo campo **Tags**, de la siguiente manera, donde puedes editar etiquetas fácilmente:

> *Figura 3.5: El campo Tags relacionado de un objeto Post*

Ahora, necesitas editar tus publicaciones de blog para mostrar las etiquetas.

Abre la plantilla `blog/post/list.html` y añade el siguiente código HTML:

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
        <p class="tags">Tags: {{ post.tags.all|join:", " }}</p>
        <p class="date">
            Published {{ post.publish }} by {{ post.author }}
        </p>
        {{ post.body|truncatewords:30|linebreaks }}
    {% endfor %}
    {% include "pagination.html" with page=page_obj %}
{% endblock %}
```

El filtro de plantilla `join` funciona de manera análoga al método `join()` de cadenas de Python. Puedes concatenar una lista de elementos en una cadena, utilizando un carácter o cadena específicos para separar cada elemento. Por ejemplo, una lista de etiquetas como `['music', 'jazz', 'piano']` se convierte en una sola cadena, `'music, jazz, piano'`, uniéndolas con `', '` como separador.

Abre `http://127.0.0.1:8000/blog/` en tu navegador. Deberías poder ver la lista de etiquetas debajo de cada título de publicación:

> *Figura 3.6: Elemento de lista de publicaciones, incluyendo las etiquetas relacionadas*

A continuación, editaremos la vista `post_list` para permitir a los usuarios listar todas las publicaciones etiquetadas con una etiqueta específica.

Abre el archivo `views.py` de tu aplicación `blog`, importa el modelo `Tag` de `django-taggit` y modifica la vista `post_list` para filtrar opcionalmente las publicaciones por una etiqueta, de la siguiente manera:

```python
from taggit.models import Tag


def post_list(request, tag_slug=None):
    post_list = Post.published.all()
    tag = None
    if tag_slug:
        tag = get_object_or_404(Tag, slug=tag_slug)
        post_list = post_list.filter(tags__in=[tag])
    # Pagination with 3 posts per page
    paginator = Paginator(post_list, 3)
    page_number = request.GET.get('page', 1)
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
        {
            'posts': posts,
            'tag': tag
        }
    )
```

La vista `post_list` ahora funciona de la siguiente manera:

1. Toma un parámetro opcional `tag_slug` que tiene un valor predeterminado `None`. Este parámetro se pasará en la URL.
2. Dentro de la vista, construimos el `QuerySet` inicial recuperando todas las publicaciones publicadas y, si hay un slug de etiqueta dado, obtenemos el objeto `Tag` con el slug correspondiente utilizando el atajo `get_object_or_404()`.
3. Luego, filtramos la lista de publicaciones por aquellas que contienen la etiqueta dada. Dado que se trata de una relación de muchos a muchos, tenemos que filtrar las publicaciones por etiquetas contenidas en una lista dada, que, en este caso, contiene solo un elemento. Usamos la búsqueda de campo `__in`. Las relaciones de muchos a muchos ocurren cuando múltiples objetos de un modelo están asociados con múltiples objetos de otro modelo. En nuestra aplicación, una publicación puede tener múltiples etiquetas y una etiqueta puede estar relacionada con múltiples publicaciones.

Aprenderás a crear relaciones de muchos a muchos en el Capítulo 6, *Compartir contenido en tu sitio web*. También puedes descubrir más sobre las relaciones de muchos a muchos en [https://docs.djangoproject.com/en/5.2/topics/db/examples/many_to_many/](https://docs.djangoproject.com/en/5.2/topics/db/examples/many_to_many/).

Finalmente, la función `render()` ahora pasa la nueva variable `tag` a la plantilla.

Recuerda que los QuerySets son perezosos (*lazy*). Los QuerySets para recuperar publicaciones solo se evaluarán cuando recorras `post_list` al renderizar la plantilla.

Abre el archivo `urls.py` de tu aplicación `blog`, comenta el patrón de URL `PostListView` basado en clases y descomenta la vista `post_list`, así:

```python
path('', views.post_list, name='post_list'),
# path('', views.PostListView.as_view(), name='post_list'),
```

Añade el siguiente patrón de URL adicional para listar publicaciones por etiqueta:

```python
path(
    'tag/<slug:tag_slug>/',
    views.post_list,
    name='post_list_by_tag'
),
```

Como puedes ver, ambos patrones apuntan a la misma vista, pero tienen nombres diferentes. El primer patrón llamará a la vista `post_list` sin ningún parámetro opcional, mientras que el segundo patrón llamará a la vista con el parámetro `tag_slug`. Utilizas un convertidor de ruta `slug` para hacer coincidir el parámetro como una cadena en minúsculas con letras ASCII o números, además de los caracteres de guion y guion bajo.

El archivo `urls.py` de la aplicación `blog` ahora debería verse así:

```python
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    # Post views
    path('', views.post_list, name='post_list'),
    # path('', views.PostListView.as_view(), name='post_list'),
    path(
        'tag/<slug:tag_slug>/',
        views.post_list,
        name='post_list_by_tag'
    ),
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

Dado que estás utilizando la vista `post_list`, edita la plantilla `blog/post/list.html` y modifica la paginación para usar el objeto `posts`:

```html
{% include "pagination.html" with page=posts %}
```

Añade las siguientes líneas a la plantilla `blog/post/list.html`:

```html
{% extends "blog/base.html" %}

{% block title %}My Blog{% endblock %}

{% block content %}
    <h1>My Blog</h1>
    {% if tag %}
        <h2>Posts tagged with "{{ tag.name }}"</h2>
    {% endif %}
    {% for post in posts %}
        <h2>
            <a href="{{ post.get_absolute_url }}">
                {{ post.title }}
            </a>
        </h2>
        <p class="tags">Tags: {{ post.tags.all|join:", " }}</p>
        <p class="date">
            Published {{ post.publish }} by {{ post.author }}
        </p>
        {{ post.body|truncatewords:30|linebreaks }}
    {% endfor %}
    {% include "pagination.html" with page=posts %}
{% endblock %}
```

Si un usuario accede al blog, verá la lista de todas las publicaciones. Si filtra por publicaciones etiquetadas con una etiqueta específica, verá la etiqueta por la que está filtrando.

Ahora, edita la plantilla `blog/post/list.html` y cambia la forma en que se muestran las etiquetas, de la siguiente manera:

```html
{% extends "blog/base.html" %}

{% block title %}My Blog{% endblock %}

{% block content %}
    <h1>My Blog</h1>
    {% if tag %}
        <h2>Posts tagged with "{{ tag.name }}"</h2>
    {% endif %}
    {% for post in posts %}
        <h2>
            <a href="{{ post.get_absolute_url }}">
                {{ post.title }}
            </a>
        </h2>
        <p class="tags">
            Tags:
            {% for tag in post.tags.all %}
                <a href="{% url "blog:post_list_by_tag" tag.slug %}">
                    {{ tag.name }}
                </a>
                {% if not forloop.last %}, {% endif %}
            {% endfor %}
        </p>
        <p class="date">
            Published {{ post.publish }} by {{ post.author }}
        </p>
        {{ post.body|truncatewords:30|linebreaks }}
    {% endfor %}
    {% include "pagination.html" with page=posts %}
{% endblock %}
```

En el código anterior, recorremos todas las etiquetas de una publicación mostrando un enlace personalizado a la URL para filtrar publicaciones por esa etiqueta. Construimos la URL con `{% url "blog:post_list_by_tag" tag.slug %}`, utilizando el nombre de la URL y el slug de la etiqueta como su parámetro. Separamos las etiquetas con comas.

Abre `http://127.0.0.1:8000/blog/tag/jazz/` en tu navegador. Verás la lista de publicaciones filtradas por esa etiqueta, así:

> *Figura 3.7: Una publicación filtrada por la etiqueta "jazz"*

---

### Recuperación de publicaciones por similitud

Ahora que hemos implementado el etiquetado para las publicaciones del blog, puedes hacer muchas cosas interesantes con las etiquetas. Las etiquetas te permiten categorizar publicaciones de forma no jerárquica. Las publicaciones sobre temas similares tendrán varias etiquetas en común. Construiremos una funcionalidad para mostrar publicaciones similares según el número de etiquetas que comparten. De esta manera, cuando un usuario lea una publicación, podremos sugerirle que lea otras publicaciones relacionadas.

Para recuperar publicaciones similares para una publicación específica, debes realizar los siguientes pasos:

1. Recuperar todas las etiquetas de la publicación actual.
2. Obtener todas las publicaciones que estén etiquetadas con cualquiera de esas etiquetas.
3. Excluir la publicación actual de esa lista para evitar recomendar la misma publicación.
4. Ordenar los resultados por el número de etiquetas compartidas con la publicación actual.
5. En el caso de dos o más publicaciones con el mismo número de etiquetas, recomendar la publicación más reciente.
6. Limitar la consulta al número de publicaciones que deseas recomendar.

Estos pasos se traducen en un `QuerySet` complejo. Editemos la vista `post_detail` para incorporar estas sugerencias de publicaciones basadas en similitud.

Abre el archivo `views.py` de tu aplicación `blog` y añade la siguiente importación en la parte superior:

```python
from django.db.models import Count
```

Esta es la función de agregación `Count` del ORM de Django. Esta función te permitirá realizar recuentos agregados de etiquetas. `django.db.models` incluye las siguientes funciones de agregación:

- `Avg`: El valor promedio
- `Max`: El valor máximo
- `Min`: El valor mínimo
- `Count`: El número total de objetos

Puedes aprender sobre agregación en [https://docs.djangoproject.com/en/5.2/topics/db/aggregation/](https://docs.djangoproject.com/en/5.2/topics/db/aggregation/).

Abre el archivo `views.py` de tu aplicación `blog` y añade las siguientes líneas a la vista `post_detail`:

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
    # List of similar posts
    post_tags_ids = post.tags.values_list('id', flat=True)
    similar_posts = Post.published.filter(
        tags__in=post_tags_ids
    ).exclude(id=post.id)
    similar_posts = similar_posts.annotate(
        same_tags=Count('tags')
    ).order_by('-same_tags', '-publish')[:4]
    return render(
        request,
        'blog/post/detail.html',
        {
            'post': post,
            'comments': comments,
            'form': form,
            'similar_posts': similar_posts
        }
    )
```

El código anterior realiza lo siguiente:

1. Recuperas una lista de Python de IDs para las etiquetas de la publicación actual. El QuerySet `values_list()` devuelve tuplas con los valores para los campos dados. Le pasas `flat=True` para obtener valores individuales como `[1, 2, 3, ...]` en lugar de tuplas como `[(1,), (2,), (3,) ...]`.
2. Obtienes todas las publicaciones que contienen cualquiera de estas etiquetas, excluyendo la propia publicación actual.
3. Utilizas la función de agregación `Count` para generar un campo calculado (`same_tags`) que contiene el número de etiquetas compartidas con todas las etiquetas consultadas.
4. Ordenas el resultado por el número de etiquetas compartidas (orden descendente) y por `publish` para mostrar las publicaciones recientes primero para aquellas publicaciones con el mismo número de etiquetas compartidas. Limitas el resultado para recuperar solo las primeras cuatro publicaciones mediante slicing (`[:4]`).
5. Pasas el objeto `similar_posts` al diccionario de contexto para la función `render()`.

Ahora, edita la plantilla `blog/post/detail.html` y añade el siguiente código:

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
    <h2>Similar posts</h2>
    {% for post in similar_posts %}
        <p>
            <a href="{{ post.get_absolute_url }}">{{ post.title }}</a>
        </p>
    {% empty %}
        There are no similar posts yet.
    {% endfor %}
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
        <p>There are no comments yet.</p>
    {% endfor %}
    {% include "blog/post/includes/comment_form.html" %}
{% endblock %}
```

La página de detalle de la publicación debería verse así:

> *Figura 3.8: La página de detalle de la publicación, incluyendo una lista de publicaciones similares*

Abre `http://127.0.0.1:8000/admin/blog/post/` en tu navegador, edita una publicación que no tenga etiquetas y añádele las etiquetas **music** y **jazz**, de la siguiente manera:

> *Figura 3.9: Añadir las etiquetas "jazz" y "music" a una publicación*

Edita otra publicación y añade la etiqueta **jazz**, de la siguiente manera:

> *Figura 3.10: Añadir la etiqueta "jazz" a una publicación*

La página de detalle de la publicación para la primera publicación ahora debería verse así:

> *Figura 3.11: La página de detalle de la publicación, incluyendo una lista de publicaciones similares*

Las publicaciones recomendadas en la sección **Similar posts** de la página aparecen en orden descendente según la cantidad de etiquetas compartidas con la publicación original.

Ahora podemos recomendar con éxito publicaciones similares a los lectores. `django-taggit` también incluye un manager `similar_objects()` que puedes utilizar para recuperar objetos por etiquetas compartidas. Puedes consultar todos los managers de `django-taggit` en [https://django-taggit.readthedocs.io/en/stable/api.html](https://django-taggit.readthedocs.io/en/stable/api.html).

También puedes añadir la lista de etiquetas a tu plantilla de detalle de publicación de la misma manera que lo hiciste en la plantilla `blog/post/list.html`.

---

### Creación de etiquetas y filtros de plantilla personalizados

Django ofrece una variedad de etiquetas de plantilla integradas, como `{% if %}` o `{% block %}`. Utilizaste diferentes etiquetas de plantilla en el Capítulo 1, *Creación de una aplicación de blog*, y en el Capítulo 2, *Mejora de tu blog y adición de funciones sociales*. También puedes encontrar una referencia completa de las etiquetas y filtros de plantilla integrados en [https://docs.djangoproject.com/en/5.2/ref/templates/builtins/](https://docs.djangoproject.com/en/5.2/ref/templates/builtins/).

Django también te permite crear tus propias etiquetas de plantilla para realizar acciones personalizadas. Las etiquetas de plantilla personalizadas resultan muy útiles cuando necesitas añadir una funcionalidad a tus plantillas que no está cubierta por el conjunto básico de etiquetas de plantilla de Django. Esto puede ser una etiqueta para ejecutar un QuerySet o cualquier procesamiento del lado del servidor que desees reutilizar en todas las plantillas. Por ejemplo, podríamos construir una etiqueta de plantilla para mostrar una lista de las últimas publicaciones publicadas en el blog. Podríamos incluir esta lista en la barra lateral para que siempre sea visible, independientemente de la vista que procese la petición.

#### Implementación de etiquetas de plantilla personalizadas

Django proporciona las siguientes funciones auxiliares que te permiten crear fácilmente etiquetas de plantilla:

- `simple_tag`: Procesa los datos proporcionados y devuelve una cadena
- `inclusion_tag`: Procesa los datos proporcionados y devuelve una plantilla renderizada

Las etiquetas de plantilla deben residir dentro de las aplicaciones de Django.

Dentro del directorio de tu aplicación `blog`, crea un nuevo directorio, nómbralo `templatetags` y añade un archivo `__init__.py` vacío en él. Crea otro archivo en la misma carpeta y nómbralo `blog_tags.py`. La estructura de archivos de la aplicación `blog` debería verse de la siguiente manera:

```text
blog/
    __init__.py
    models.py
    ...
    templatetags/
        __init__.py
        blog_tags.py
```

La forma en que nombras el archivo es importante porque utilizarás el nombre de este módulo para cargar etiquetas en las plantillas.

#### Creación de una etiqueta de plantilla simple

Comencemos creando una etiqueta simple para recuperar el total de publicaciones que se han publicado en el blog.

Edita el archivo `templatetags/blog_tags.py` que acabas de crear y añade el siguiente código:

```python
from django import template
from ..models import Post

register = template.Library()


@register.simple_tag
def total_posts():
    return Post.published.count()
```

Hemos creado una etiqueta de plantilla simple que devuelve el número de publicaciones publicadas en el blog.

Cada módulo que contiene etiquetas de plantilla necesita definir una variable llamada `register` para ser una biblioteca de etiquetas válida. Esta variable es una instancia de `template.Library` y se utiliza para registrar las etiquetas y filtros de plantilla de la aplicación.

En el código anterior, hemos definido una etiqueta llamada `total_posts` con una función simple de Python. Hemos añadido el decorador `@register.simple_tag` a la función para registrarla como una etiqueta simple. Django utilizará el nombre de la función como el nombre de la etiqueta.

Si deseas registrarla con un nombre diferente, puedes hacerlo especificando un atributo de nombre, como `@register.simple_tag(name='my_tag')`.

Después de añadir un nuevo módulo de etiquetas de plantilla, deberás reiniciar el servidor de desarrollo de Django para poder utilizar las nuevas etiquetas y filtros en las plantillas.

Antes de usar etiquetas de plantilla personalizadas, debemos hacerlas disponibles para la plantilla mediante la etiqueta `{% load %}`. Como se mencionó anteriormente, debemos utilizar el nombre del módulo de Python que contiene nuestras etiquetas y filtros de plantilla.

Edita la plantilla `blog/templates/base.html` y añade `{% load blog_tags %}` en la parte superior para cargar tu módulo de etiquetas de plantilla. Luego, usa la etiqueta que creaste para mostrar el total de publicaciones, de la siguiente manera:

```html
{% load blog_tags %}
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
        <p>
            This is my blog.
            I've written {% total_posts %} posts so far.
        </p>
    </div>
</body>
</html>
```

Tendrás que reiniciar el servidor para registrar los nuevos archivos añadidos al proyecto. Detén el servidor de desarrollo con `Ctrl + C` y ejecútalo nuevamente con el siguiente comando:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/blog/` en tu navegador. Deberías ver el número total de publicaciones en la barra lateral del sitio, de la siguiente manera:

> *Figura 3.12: El total de publicaciones publicadas incluido en la barra lateral*

Si ves el siguiente mensaje de error, es muy probable que no hayas reiniciado el servidor de desarrollo:

> *Figura 3.13: El mensaje de error cuando una biblioteca de etiquetas de plantilla no está registrada*

Las etiquetas de plantilla te permiten procesar cualquier dato y añadirlo a cualquier plantilla, independientemente de la vista ejecutada. Puedes ejecutar QuerySets o procesar cualquier dato para mostrar resultados en tus plantillas.

#### Creación de una etiqueta de plantilla de inclusión

Crearemos otra etiqueta para mostrar las últimas publicaciones en la barra lateral del blog. Esta vez, implementaremos una etiqueta de inclusión (*inclusion tag*). Al utilizar una etiqueta de inclusión, puedes renderizar una plantilla con variables de contexto devueltas por tu etiqueta de plantilla.

Edita el archivo `templatetags/blog_tags.py` y añade el siguiente código:

```python
@register.inclusion_tag('blog/post/latest_posts.html')
def show_latest_posts(count=5):
    latest_posts = Post.published.order_by('-publish')[:count]
    return {'latest_posts': latest_posts}
```

En el código anterior, hemos registrado la etiqueta de plantilla utilizando el decorador `@register.inclusion_tag`. Hemos especificado la plantilla que se renderizará con los valores devueltos utilizando `blog/post/latest_posts.html`. La etiqueta de plantilla aceptará un parámetro opcional `count` cuyo valor predeterminado es 5. Este parámetro nos permitirá especificar la cantidad de publicaciones a mostrar. Usamos esta variable para limitar los resultados de la consulta `Post.published.order_by('-publish')[:count]`.

Ten en cuenta que la función devuelve un diccionario de variables en lugar de un valor simple. Las etiquetas de inclusión deben devolver un diccionario de valores, que se utiliza como contexto para renderizar la plantilla especificada. La etiqueta de plantilla que acabamos de crear nos permite especificar el número opcional de publicaciones a mostrar como `{% show_latest_posts 3 %}`.

Ahora, crea un nuevo archivo de plantilla bajo `blog/post/` y nómbralo `latest_posts.html`.

Edita la nueva plantilla `blog/post/latest_posts.html` y añádele el siguiente código:

```html
<ul>
    {% for post in latest_posts %}
        <li>
            <a href="{{ post.get_absolute_url }}">{{ post.title }}</a>
        </li>
    {% endfor %}
</ul>
```

En el código anterior, has añadido una lista desordenada de publicaciones utilizando la variable `latest_posts` devuelta por tu etiqueta de plantilla. Ahora, edita la plantilla `blog/base.html` y añade la nueva etiqueta de plantilla para mostrar las últimas tres publicaciones, de la siguiente manera:

```html
{% load blog_tags %}
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
        <p>
            This is my blog.
            I've written {% total_posts %} posts so far.
        </p>
        <h3>Latest posts</h3>
        {% show_latest_posts 3 %}
    </div>
</body>
</html>
```

Se llama a la etiqueta de plantilla pasando el número de publicaciones a mostrar y la plantilla se renderiza en su lugar con el contexto dado.

A continuación, regresa a tu navegador y recarga la página. La barra lateral ahora debería verse así:

> *Figura 3.14: La barra lateral del blog, incluyendo las últimas publicaciones publicadas*

#### Creación de una etiqueta de plantilla que devuelve un QuerySet

Finalmente, crearemos una etiqueta de plantilla simple que devuelve un valor. Almacenaremos el resultado en una variable que se pueda reutilizar, en lugar de mostrarlo directamente. Crearemos una etiqueta para mostrar las publicaciones más comentadas.

Edita el archivo `templatetags/blog_tags.py` y añádele la siguiente importación y etiqueta de plantilla:

```python
from django.db.models import Count


@register.simple_tag
def get_most_commented_posts(count=5):
    return Post.published.annotate(
        total_comments=Count('comments')
    ).order_by('-total_comments')[:count]
```

En la etiqueta de plantilla anterior, construyes un `QuerySet` utilizando la función `annotate()` para agregar el número total de comentarios de cada publicación. Utilizas la función de agregación `Count` para almacenar el número de comentarios en el campo calculado `total_comments` para cada objeto `Post`. Ordenas el `QuerySet` por el campo calculado en orden descendente. También proporcionas una variable opcional `count` para limitar el número total de objetos devueltos.

Además de `Count`, Django ofrece las funciones de agregación `Avg`, `Max`, `Min` y `Sum`. Puedes leer más sobre las funciones de agregación en [https://docs.djangoproject.com/en/5.2/topics/db/aggregation/](https://docs.djangoproject.com/en/5.2/topics/db/aggregation/).

A continuación, edita la plantilla `blog/base.html` y añade el siguiente código:

```html
{% load blog_tags %}
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
        <p>
            This is my blog.
            I've written {% total_posts %} posts so far.
        </p>
        <h3>Latest posts</h3>
        {% show_latest_posts 3 %}
        <h3>Most commented posts</h3>
        {% get_most_commented_posts as most_commented_posts %}
        <ul>
            {% for post in most_commented_posts %}
                <li>
                    <a href="{{ post.get_absolute_url }}">{{ post.title }}</a>
                </li>
            {% endfor %}
        </ul>
    </div>
</body>
</html>
```

En el código anterior, almacenamos el resultado en una variable personalizada utilizando el argumento `as` seguido del nombre de la variable. Para la etiqueta de plantilla, usamos `{% get_most_commented_posts as most_commented_posts %}` para almacenar el resultado de la etiqueta de plantilla en una nueva variable llamada `most_commented_posts`. Luego, mostramos las publicaciones devueltas utilizando un elemento de lista desordenada HTML.

Ahora abre tu navegador y actualiza la página para ver el resultado final. Debería verse de la siguiente manera:

> *Figura 3.15: La vista de lista de publicaciones, incluyendo la barra lateral completa con las últimas publicaciones y las más comentadas*

Ahora tienes una idea clara de cómo crear etiquetas de plantilla personalizadas. Puedes leer más sobre ellas en [https://docs.djangoproject.com/en/5.2/howto/custom-template-tags/](https://docs.djangoproject.com/en/5.2/howto/custom-template-tags/).

#### Implementación de filtros de plantilla personalizados

Django tiene una variedad de filtros de plantilla integrados que te permiten alterar variables en plantillas. Estas son funciones de Python que toman uno o dos parámetros: el valor de la variable a la que se aplica el filtro y un argumento opcional. Devuelven un valor que se puede mostrar o tratar mediante otro filtro.

Un filtro se escribe como `{{ variable|my_filter }}`. Los filtros con un argumento se escriben como `{{ variable|my_filter:"foo" }}`. Por ejemplo, puedes utilizar el filtro `capfirst` para poner en mayúscula el primer carácter del valor, como `{{ value|capfirst }}`. Si `value` es `django`, la salida será `Django`. Puedes aplicar tantos filtros como desees a una variable, por ejemplo, `{{ variable|filter1|filter2 }}`, y cada filtro se aplicará a la salida generada por el filtro anterior.

Puedes encontrar la lista de filtros de plantilla integrados de Django en [https://docs.djangoproject.com/en/5.2/ref/templates/builtins/#built-in-filter-reference](https://docs.djangoproject.com/en/5.2/ref/templates/builtins/#built-in-filter-reference).

#### Creación de un filtro de plantilla para admitir la sintaxis Markdown

Crearemos un filtro personalizado para permitirte utilizar la sintaxis Markdown en tus publicaciones de blog y luego convertir el cuerpo de la publicación a HTML en las plantillas.

Markdown es una sintaxis de formato de texto plano que es muy fácil de usar y está diseñada para convertirse en HTML. Puedes escribir publicaciones utilizando una sintaxis Markdown simple y hacer que el contenido se convierta automáticamente en código HTML. Aprender la sintaxis Markdown es mucho más fácil que aprender HTML. Al usar Markdown, puedes hacer que otros colaboradores no técnicos escriban publicaciones fácilmente para tu blog. Puedes aprender los conceptos básicos del formato Markdown en [https://daringfireball.net/projects/markdown/basics](https://daringfireball.net/projects/markdown/basics).

Primero, instala el módulo `markdown` de Python a través de pip ejecutando el siguiente comando en la consola:

```bash
python -m pip install markdown==3.6
```

Luego, edita el archivo `templatetags/blog_tags.py` e incluye el siguiente código:

```python
import markdown
from django.utils.safestring import mark_safe


@register.filter(name='markdown')
def markdown_format(text):
    return mark_safe(markdown.markdown(text))
```

Registramos los filtros de plantilla de la misma manera que las etiquetas de plantilla. Para evitar un conflicto de nombres entre el nombre de la función y el módulo `markdown`, nombramos la función `markdown_format` y nombramos el filtro `markdown` para su uso en plantillas, como `{{ variable|markdown }}`.

Django escapa el código HTML generado por los filtros; los caracteres de las entidades HTML se reemplazan con sus caracteres codificados en HTML. Por ejemplo, `<p>` se convierte en `&lt;p&gt;` (símbolo menor que, carácter p, símbolo mayor que).

Usamos la función `mark_safe` proporcionada por Django para marcar el resultado como HTML seguro para ser renderizado en la plantilla. De forma predeterminada, Django no confiará en ningún código HTML y lo escapará antes de colocarlo en la salida. Las únicas excepciones son las variables que están marcadas como seguras contra el escape. Este comportamiento evita que Django genere HTML potencialmente peligroso y te permite crear excepciones para devolver HTML seguro.

> [!CAUTION]
> En Django, el contenido HTML se escapa de forma predeterminada por seguridad. Usa `mark_safe` con precaución, solo en contenido que controles. Evita usar `mark_safe` en cualquier contenido enviado por usuarios que no pertenezcan al personal para evitar vulnerabilidades de seguridad.

Edita la plantilla `blog/post/detail.html` y añade el siguiente código:

```html
{% extends "blog/base.html" %}
{% load blog_tags %}

{% block title %}{{ post.title }}{% endblock %}

{% block content %}
    <h1>{{ post.title }}</h1>
    <p class="date">
        Published {{ post.publish }} by {{ post.author }}
    </p>
    {{ post.body|markdown }}
    <p>
        <a href="{% url "blog:post_share" post.id %}">
            Share this post
        </a>
    </p>
    <h2>Similar posts</h2>
    {% for post in similar_posts %}
        <p>
            <a href="{{ post.get_absolute_url }}">{{ post.title }}</a>
        </p>
    {% empty %}
        There are no similar posts yet.
    {% endfor %}
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
        <p>There are no comments yet.</p>
    {% endfor %}
    {% include "blog/post/includes/comment_form.html" %}
{% endblock %}
```

Hemos reemplazado el filtro `linebreaks` de la variable de plantilla `{{ post.body }}` por el filtro `markdown`. Este filtro no solo transformará los saltos de línea en etiquetas `<p>`, sino que también transformará el formato Markdown en HTML.

Almacenar texto en formato Markdown en la base de datos, en lugar de HTML, es una estrategia de seguridad inteligente. Markdown limita el potencial de inyectar contenido malicioso. Este enfoque garantiza que cualquier formato de texto se convierta de forma segura a HTML solo en el momento de renderizar la plantilla.

Edita la plantilla `blog/post/list.html` y añade el siguiente código:

```html
{% extends "blog/base.html" %}
{% load blog_tags %}

{% block title %}My Blog{% endblock %}

{% block content %}
    <h1>My Blog</h1>
    {% if tag %}
        <h2>Posts tagged with "{{ tag.name }}"</h2>
    {% endif %}
    {% for post in posts %}
        <h2>
            <a href="{{ post.get_absolute_url }}">
                {{ post.title }}
            </a>
        </h2>
        <p class="tags">
            Tags:
            {% for tag in post.tags.all %}
                <a href="{% url "blog:post_list_by_tag" tag.slug %}">
                    {{ tag.name }}
                </a>
                {% if not forloop.last %}, {% endif %}
            {% endfor %}
        </p>
        <p class="date">
            Published {{ post.publish }} by {{ post.author }}
        </p>
        {{ post.body|markdown|truncatewords_html:30 }}
    {% endfor %}
    {% include "pagination.html" with page=posts %}
{% endblock %}
```

Hemos añadido el nuevo filtro `markdown` a la variable de plantilla `{{ post.body }}`. Este filtro transformará el contenido Markdown en HTML.

Por lo tanto, hemos reemplazado el filtro `truncatewords` anterior con el filtro `truncatewords_html`. Este filtro trunca una cadena después de un cierto número de palabras, evitando etiquetas HTML sin cerrar.

Ahora abre `http://127.0.0.1:8000/admin/blog/post/add/` en tu navegador y crea una nueva publicación con el siguiente cuerpo:

```markdown
This is a post formatted with markdown
--------------------------------------

*This is emphasized* and **this is more emphasized**.

Here is a list:

* One
* Two
* Three

And a [link to the Django website](https://www.djangoproject.com/).
```

El formulario debería verse así:

> *Figura 3.16: La publicación con contenido Markdown renderizado como HTML*

Abre `http://127.0.0.1:8000/blog/` en tu navegador y echa un vistazo a cómo se renderiza la nueva publicación. Deberías ver la siguiente salida:

> *Figura 3.17: La publicación con contenido Markdown renderizado como HTML*

Como puedes ver en la Figura 3.17, los filtros de plantilla personalizados son muy útiles para personalizar el formato. Puedes encontrar más información sobre filtros personalizados en [https://docs.djangoproject.com/en/5.2/howto/custom-template-tags/#writing-custom-template-filters](https://docs.djangoproject.com/en/5.2/howto/custom-template-tags/#writing-custom-template-filters).

---

### Adición de un sitemap al sitio

Django viene con un framework de sitemaps, que te permite generar sitemaps para tu sitio dinámicamente. Un sitemap es un archivo XML que indica a los motores de búsqueda las páginas de tu sitio web, su relevancia y la frecuencia con la que se actualizan. El uso de un sitemap hará que tu sitio sea más visible en las clasificaciones de los motores de búsqueda porque ayuda a los rastreadores a indexar el contenido de tu sitio web.

El framework de sitemaps de Django depende de `django.contrib.sites`, que te permite asociar objetos a sitios web particulares que se ejecutan con tu proyecto. Esto resulta útil cuando deseas ejecutar múltiples sitios utilizando un único proyecto de Django. Para instalar el framework de sitemaps, necesitaremos activar tanto la aplicación `sites` como la aplicación `sitemaps` en tu proyecto. Vamos a construir un sitemap para el blog que incluya los enlaces a todas las publicaciones publicadas.

Edita el archivo `settings.py` del proyecto y añade `django.contrib.sites` y `django.contrib.sitemaps` a la configuración `INSTALLED_APPS`. Además, define una nueva configuración para el ID del sitio, de la siguiente manera:

```python
# ...
SITE_ID = 1

# Application definition
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.sites',
    'django.contrib.sitemaps',
    'django.contrib.staticfiles',
    'taggit',
    'blog.apps.BlogConfig',
]
```

Ahora, ejecuta el siguiente comando desde la consola para crear las tablas de la aplicación `sites` de Django en la base de datos:

```bash
python manage.py migrate
```

Deberías ver una salida que contiene las siguientes líneas:

```text
Applying sites.0001_initial... OK
Applying sites.0002_alter_domain_unique... OK
```

La aplicación `sites` ya está sincronizada con la base de datos.

A continuación, crea un nuevo archivo dentro del directorio de tu aplicación `blog` y nómbralo `sitemaps.py`. Abre el archivo y añade el siguiente código:

```python
from django.contrib.sitemaps import Sitemap
from .models import Post


class PostSitemap(Sitemap):
    changefreq = 'weekly'
    priority = 0.9

    def items(self):
        return Post.published.all()

    def lastmod(self, obj):
        return obj.updated
```

Hemos definido un sitemap personalizado heredando de la clase `Sitemap` del módulo `sitemaps`. Los atributos `changefreq` y `priority` indican la frecuencia de cambio de las páginas de tus publicaciones y su relevancia en tu sitio web (el valor máximo es 1).

El método `items()` devuelve el `QuerySet` de objetos a incluir en este sitemap. De forma predeterminada, Django llama al método `get_absolute_url()` en cada objeto para recuperar su URL. Recuerda que implementamos este método en el Capítulo 2, *Mejora de tu blog y adición de funciones sociales*, para definir la URL canónica de las publicaciones. Si deseas especificar la URL para cada objeto, puedes añadir un método `location` a tu clase de sitemap.

El método `lastmod` recibe cada objeto devuelto por `items()` y devuelve la última vez que se modificó el objeto.

Tanto `changefreq` como `priority` pueden ser métodos o atributos. Puedes consultar la referencia completa de sitemaps en la documentación oficial de Django ubicada en [https://docs.djangoproject.com/en/5.2/ref/contrib/sitemaps/](https://docs.djangoproject.com/en/5.2/ref/contrib/sitemaps/).

Hemos creado el sitemap. Ahora solo necesitamos crear una URL para él.

Edita el archivo `urls.py` principal del proyecto `mysite` y añade el sitemap, de la siguiente manera:

```python
from django.contrib import admin
from django.contrib.sitemaps.views import sitemap
from django.urls import include, path
from blog.sitemaps import PostSitemap

sitemaps = {
    'posts': PostSitemap,
}

urlpatterns = [
    path('admin/', admin.site.urls),
    path('blog/', include('blog.urls', namespace='blog')),
    path(
        'sitemap.xml',
        sitemap,
        {'sitemaps': sitemaps},
        name='django.contrib.sitemaps.views.sitemap'
    ),
]
```

En el código anterior, hemos incluido las importaciones requeridas y definido un diccionario `sitemaps`. Se pueden definir múltiples sitemaps para el sitio. Hemos definido un patrón de URL que coincide con el patrón `sitemap.xml` y utiliza la vista `sitemap` proporcionada por Django. El diccionario `sitemaps` se pasa a la vista `sitemap`.

Inicia el servidor de desarrollo desde la consola con el siguiente comando:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/sitemap.xml` en tu navegador. Verás una salida XML que incluye todas las publicaciones publicadas, así:

```xml
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9" xmlns:xhtml="http://www.w3.org/1999/xhtml">
    <url>
        <loc>http://example.com/blog/2024/1/2/markdown-post/</loc>
        <lastmod>2024-01-02</lastmod>
        <changefreq>weekly</changefreq>
        <priority>0.9</priority>
    </url>
    <url>
        <loc>http://example.com/blog/2024/1/2/notes-on-duke-ellington/</loc>
        <lastmod>2024-01-02</lastmod>
        <changefreq>weekly</changefreq>
        <priority>0.9</priority>
    </url>
    <url>
        <loc>http://example.com/blog/2024/1/2/who-was-miles-davis/</loc>
        <lastmod>2024-01-02</lastmod>
        <changefreq>weekly</changefreq>
        <priority>0.9</priority>
    </url>
    <url>
        <loc>http://example.com/blog/2024/1/1/who-was-django-reinhardt/</loc>
        <lastmod>2024-01-01</lastmod>
        <changefreq>weekly</changefreq>
        <priority>0.9</priority>
    </url>
    <url>
        <loc>http://example.com/blog/2024/1/1/another-post/</loc>
        <lastmod>2024-01-01</lastmod>
        <changefreq>weekly</changefreq>
        <priority>0.9</priority>
    </url>
</urlset>
```

La URL para cada objeto `Post` se construye llamando a su método `get_absolute_url()`.

El atributo `lastmod` corresponde al campo de fecha de actualización de la publicación, tal como especificaste en tu sitemap, y los atributos `changefreq` y `priority` también se toman de la clase `PostSitemap`.

El dominio utilizado para construir las URLs es `example.com`. Este dominio proviene de un objeto `Site` almacenado en la base de datos. Este objeto predeterminado se creó cuando sincronizaste el framework de sitios con tu base de datos. Puedes leer más sobre el framework de sitios en [https://docs.djangoproject.com/en/5.2/ref/contrib/sites/](https://docs.djangoproject.com/en/5.2/ref/contrib/sites/).

Abre `http://127.0.0.1:8000/admin/sites/site/` en tu navegador. Deberías ver algo como esto:

> *Figura 3.18: La vista de lista de administración de Django para el modelo Site del framework de sitios*

La Figura 3.18 contiene la vista de administración de lista para el framework de sitios. Aquí puedes establecer el dominio o host que utilizarán el framework de sitios y las aplicaciones que dependen de él. Para generar URLs que existan en tu entorno local, cambia el nombre de dominio a `localhost:8000`, como se muestra en la Figura 3.19, y guárdalo:

> *Figura 3.19: Vista de edición de administración de Django para el modelo Site del framework de sitios*

Abre `http://127.0.0.1:8000/sitemap.xml` en tu navegador nuevamente. Las URLs mostradas en tu sitemap ahora utilizarán el nuevo hostname y se verán como `http://localhost:8000/blog/2024/1/22/markdown-post/`. Los enlaces ahora son accesibles en tu entorno local. En un entorno de producción, tendrás que utilizar el dominio de tu sitio web para generar URLs absolutas.

---

### Creación de feeds para publicaciones de blog

Django tiene un framework de feeds de sindicación integrado que puedes utilizar para generar dinámicamente feeds RSS o Atom de manera similar a la creación de sitemaps utilizando el framework de sitios. Un feed web es un formato de datos (generalmente XML) que proporciona a los usuarios el contenido actualizado más recientemente. Los usuarios pueden suscribirse al feed utilizando un agregador de feeds, un software que se utiliza para leer feeds y recibir notificaciones de contenido nuevo.

Crea un nuevo archivo en el directorio de tu aplicación `blog` y nómbralo `feeds.py`. Añade las siguientes líneas a él:

```python
import markdown
from django.contrib.syndication.views import Feed
from django.template.defaultfilters import truncatewords_html
from django.urls import reverse_lazy
from .models import Post


class LatestPostsFeed(Feed):
    title = 'My blog'
    link = reverse_lazy('blog:post_list')
    description = 'New posts of my blog.'

    def items(self):
        return Post.published.all()[:5]

    def item_title(self, item):
        return item.title

    def item_description(self, item):
        return truncatewords_html(markdown.markdown(item.body), 30)

    def item_pubdate(self, item):
        return item.publish
```

En el código anterior, hemos definido un feed creando una subclase de la clase `Feed` del framework de sindicación. Los atributos `title`, `link` y `description` corresponden a los elementos RSS `<title>`, `<link>` y `<description>`, respectivamente.

Usamos `reverse_lazy()` para generar la URL para el atributo `link`. El método `reverse()` te permite construir URLs por su nombre y pasar parámetros opcionales. Usamos `reverse()` en el Capítulo 2, *Mejora de tu blog y adición de funciones sociales*.

La función de utilidad `reverse_lazy()` es una versión evaluada perezosamente de `reverse()`. Te permite utilizar la resolución inversa de URLs antes de que se cargue la configuración de URL del proyecto.

El método `items()` recupera los objetos que se incluirán en el feed. Recuperamos las últimas cinco publicaciones publicadas para incluirlas en el feed.

Los métodos `item_title()`, `item_description()` e `item_pubdate()` recibirán cada objeto devuelto por `items()` y devolverán el título, la descripción y la fecha de publicación de cada elemento.

En el método `item_description()`, usamos la función `markdown()` para convertir el contenido de Markdown a HTML y la función de filtro de plantilla `truncatewords_html()` para cortar la descripción de las publicaciones después de 30 palabras, evitando etiquetas HTML sin cerrar.

Ahora, edita el archivo `blog/urls.py`, importa la clase `LatestPostsFeed` e instancia el feed en un nuevo patrón de URL, de la siguiente manera:

```python
from django.urls import path
from . import views
from .feeds import LatestPostsFeed

app_name = 'blog'

urlpatterns = [
    # Post views
    path('', views.post_list, name='post_list'),
    # path('', views.PostListView.as_view(), name='post_list'),
    path(
        'tag/<slug:tag_slug>/',
        views.post_list,
        name='post_list_by_tag'
    ),
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
    path('feed/', LatestPostsFeed(), name='post_feed'),
]
```

Navega a `http://127.0.0.1:8000/blog/feed/` en tu navegador. Ahora deberías ver el feed RSS, incluyendo las últimas cinco publicaciones del blog:

```xml
<?xml version="1.0" encoding="utf-8"?>
<rss xmlns:atom="http://www.w3.org/2005/Atom" version="2.0">
    <channel>
        <title>My blog</title>
        <link>http://localhost:8000/blog/</link>
        <description>New posts of my blog.</description>
        <atom:link href="http://localhost:8000/blog/feed/" rel="self"/>
        <language>en-us</language>
        <lastBuildDate>Tue, 02 Jan 2024 16:30:00 +0000</lastBuildDate>
        <item>
            <title>Markdown post</title>
            <link>http://localhost:8000/blog/2024/1/2/markdown-post/</link>
            <description>This is a post formatted with ...</description>
            <guid>http://localhost:8000/blog/2024/1/2/markdown-post/</guid>
        </item>
        ...
    </channel>
</rss>
```

Si usas Chrome, verás el código XML. Si usas Safari, te pedirá que instales un lector de feeds RSS.

Instalemos un cliente de escritorio RSS para ver el feed RSS con una interfaz fácil de usar. Usaremos Fluent Reader, que es un lector RSS multiplataforma.

Descarga Fluent Reader para Linux, macOS o Windows desde [https://github.com/yang991178/fluent-reader/releases](https://github.com/yang991178/fluent-reader/releases).

Instala Fluent Reader y ábrelo. Verás la siguiente pantalla:

> *Figura 3.20: Fluent Reader sin fuentes de feeds RSS*

Haz clic en el icono de configuración en la esquina superior derecha de la ventana. Verás una pantalla para añadir fuentes de feeds RSS como la siguiente:

> *Figura 3.21: Añadir un feed RSS en Fluent Reader*

Introduce `http://127.0.0.1:8000/blog/feed/` en el campo **Add source** y haz clic en el botón **Add**.

Verás una nueva entrada con el feed RSS del blog en la tabla debajo del formulario, así:

> *Figura 3.22: Fuentes de feeds RSS en Fluent Reader*

Ahora, regresa a la pantalla principal de Fluent Reader. Deberías poder ver las publicaciones incluidas en el feed RSS del blog, de la siguiente manera:

> *Figura 3.23: Feed RSS del blog en Fluent Reader*

Haz clic en una publicación para ver su descripción:

> *Figura 3.24: La descripción de la publicación en Fluent Reader*

Haz clic en el tercer icono en la esquina superior derecha de la ventana para cargar el contenido completo de la página de la publicación:

> *Figura 3.25: El contenido completo de una publicación en Fluent Reader*

El paso final es añadir un enlace de suscripción al feed RSS a la barra lateral del blog.

Abre la plantilla `blog/base.html` y añade el siguiente código:

```html
{% load blog_tags %}
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
        <p>
            This is my blog.
            I've written {% total_posts %} posts so far.
        </p>
        <p>
            <a href="{% url "blog:post_feed" %}">
                Subscribe to my RSS feed
            </a>
        </p>
        <h3>Latest posts</h3>
        {% show_latest_posts 3 %}
        <h3>Most commented posts</h3>
        {% get_most_commented_posts as most_commented_posts %}
        <ul>
            {% for post in most_commented_posts %}
                <li>
                    <a href="{{ post.get_absolute_url }}">{{ post.title }}</a>
                </li>
            {% endfor %}
        </ul>
    </div>
</body>
</html>
```

Ahora abre `http://127.0.0.1:8000/blog/` en tu navegador y echa un vistazo a la barra lateral. El nuevo enlace llevará a los usuarios al feed del blog:

> *Figura 3.26: El enlace de suscripción al feed RSS añadido a la barra lateral*

Puedes leer más sobre el framework de feeds de sindicación de Django en [https://docs.djangoproject.com/en/5.2/ref/contrib/syndication/](https://docs.djangoproject.com/en/5.2/ref/contrib/syndication/).

---

### Adición de búsqueda de texto completo al blog

A continuación, añadiremos capacidades de búsqueda al blog. Buscar datos en la base de datos con la entrada del usuario es una tarea común para las aplicaciones web. El ORM de Django te permite realizar operaciones de coincidencia simples utilizando, por ejemplo, el filtro `contains` (o su versión insensible a mayúsculas y minúsculas, `icontains`). Puedes utilizar la siguiente consulta para encontrar publicaciones que contengan la palabra `framework` en su cuerpo:

```python
from blog.models import Post

Post.objects.filter(body__contains='framework')
```

Sin embargo, si deseas realizar búsquedas complejas, recuperar resultados por similitud o ponderar términos según la frecuencia con la que aparecen en el texto o la importancia de diferentes campos (por ejemplo, la relevancia de que el término aparezca en el título frente al cuerpo), necesitarás utilizar un motor de búsqueda de texto completo. Cuando consideras grandes bloques de texto, construir consultas con operaciones sobre cadenas de caracteres no es suficiente. Una búsqueda de texto completo examina las palabras reales contra el contenido almacenado a medida que intenta coincidir con los criterios de búsqueda.

Django proporciona una potente funcionalidad de búsqueda construida sobre las características de búsqueda de texto completo de la base de datos PostgreSQL. El módulo `django.contrib.postgres` proporciona funcionalidades ofrecidas por PostgreSQL que no comparten las otras bases de datos compatibles con Django. Puedes obtener más información sobre el soporte de búsqueda de texto completo de PostgreSQL en [https://www.postgresql.org/docs/16/textsearch.html](https://www.postgresql.org/docs/16/textsearch.html).

Aunque Django es un framework web independiente de la base de datos, proporciona un módulo que admite parte del rico conjunto de características que ofrece PostgreSQL, que no ofrecen otras bases de datos compatibles con Django.

Actualmente estamos utilizando una base de datos SQLite para el proyecto `mysite`. El soporte de SQLite para búsqueda de texto completo es limitado y Django no lo admite de fábrica. Sin embargo, PostgreSQL se adapta mucho mejor a la búsqueda de texto completo y podemos usar el módulo `django.contrib.postgres` para utilizar las capacidades de búsqueda de texto completo de PostgreSQL. Migraremos nuestros datos de SQLite a PostgreSQL para beneficiarnos de sus funciones de búsqueda de texto completo.

SQLite es suficiente para fines de desarrollo. Sin embargo, para un entorno de producción, necesitarás una base de datos más potente, como PostgreSQL, MariaDB, MySQL u Oracle.

PostgreSQL proporciona una imagen de Docker que hace que sea muy fácil desplegar un servidor PostgreSQL con una configuración estándar.

#### Instalación de Docker

Docker es una popular plataforma de contenedorización de código abierto. Permite a los desarrolladores empaquetar aplicaciones en contenedores, simplificando el proceso de creación, ejecución, administración y distribución de aplicaciones.

Primero, descarga e instala Docker para tu sistema operativo. Encontrarás instrucciones para descargar e instalar Docker en Linux, macOS y Windows en [https://docs.docker.com/get-docker/](https://docs.docker.com/get-docker/). La instalación incluye tanto Docker Desktop como las herramientas de interfaz de línea de comandos de Docker.

#### Instalación de PostgreSQL

Después de instalar Docker en tu máquina Linux, macOS o Windows, puedes descargar fácilmente la imagen de Docker de PostgreSQL. Ejecuta el siguiente comando desde la consola:

```bash
docker pull postgres:16.2
```

Esto descargará la imagen de Docker de PostgreSQL a tu máquina local. Puedes encontrar información sobre la imagen oficial de Docker de PostgreSQL en [https://hub.docker.com/_/postgres](https://hub.docker.com/_/postgres). Puedes encontrar otros paquetes e instaladores de PostgreSQL en [https://www.postgresql.org/download/](https://www.postgresql.org/download/).

Ejecuta el siguiente comando en la consola para iniciar el contenedor Docker de PostgreSQL:

```bash
docker run --name=blog_db -e POSRGRES_DB=blog -e POSTGRES_USER=blog -e POSTGRES_PASSWORD=xxxxx -p 5432:5432 -d postgres:16.2
```

Reemplaza `xxxxx` con la contraseña deseada para tu usuario de base de datos.

Este comando inicia una instancia de PostgreSQL. La opción `--name` se utiliza para asignar un nombre al contenedor, en este caso, `blog_db`. La opción `-e` sirve para definir variables de entorno para la instancia. Establecemos las siguientes variables de entorno:

- `POSTGRES_DB`: Nombre de la base de datos de PostgreSQL. Si no se define, se utiliza el valor de `POSTGRES_USER` para el nombre de la base de datos.
- `POSTGRES_USER`: Se utiliza junto con `POSTGRES_PASSWORD` para definir un nombre de usuario y una contraseña. El usuario se crea con privilegios de superusuario.
- `POSTGRES_PASSWORD`: Establece la contraseña de superusuario para PostgreSQL.

La opción `-p` se utiliza para publicar el puerto 5432, en el que se ejecuta PostgreSQL, en el mismo puerto de interfaz del host. Esto permite que las aplicaciones externas accedan a la base de datos. La opción `-d` es para el modo desconectado (*detached mode*), que ejecuta el contenedor Docker en segundo plano.

Abre la aplicación Docker Desktop. Deberías ver el nuevo contenedor ejecutándose, como en la Figura 3.27:

> *Figura 3.27: Instancia de PostgreSQL ejecutándose en Docker Desktop*

Verás el contenedor `blog_db` recién creado, con el estado **Running**. En **Actions**, puedes detener o reiniciar el servicio. También puedes eliminar el contenedor. Ten en cuenta que eliminar el contenedor también eliminará la base de datos y todos los datos que contiene. Aprenderás a persistir los datos de PostgreSQL en el sistema de archivos local utilizando Docker en el Capítulo 17, *Puesta en producción*.

También necesitas instalar el adaptador de PostgreSQL `psycopg` para Python. Ejecuta el siguiente comando en la consola para instalarlo:

```bash
python -m pip install psycopg==3.1.18
```

A continuación, migraremos los datos existentes en la base de datos SQLite a la nueva instancia de PostgreSQL.

#### Volcado de datos existentes

Antes de cambiar la base de datos en el proyecto de Django, necesitamos volcar los datos existentes de la base de datos SQLite. Exportaremos los datos, cambiaremos la base de datos del proyecto a PostgreSQL e importaremos los datos a la nueva base de datos.

Django incluye una forma sencilla de cargar y volcar datos de la base de datos en archivos llamados *fixtures*. Django admite fixtures en formato JSON, XML o YAML. Vamos a crear un fixture con todos los datos contenidos en la base de datos.

El comando `dumpdata` vuelca datos de la base de datos a la salida estándar, serializados en formato JSON de forma predeterminada. La estructura de datos resultante incluye información sobre el modelo y sus campos para que Django pueda cargarla en la base de datos.

Puedes limitar la salida a los modelos de una aplicación proporcionando los nombres de las aplicaciones al comando, o especificando modelos individuales para generar datos utilizando el formato `app.Model`. También puedes especificar el formato utilizando el indicador `--format`. De forma predeterminada, `dumpdata` envía los datos serializados a la salida estándar. Sin embargo, puedes indicar un archivo de salida mediante el indicador `--output`. El indicador `--indent` te permite especificar la sangría. Para obtener más información sobre los parámetros de `dumpdata`, ejecuta `python manage.py dumpdata --help`.

Ejecuta el siguiente comando desde la consola:

```bash
python manage.py dumpdata --indent=2 --output=mysite_data.json
```

Todos los datos existentes se han exportado en formato JSON a un nuevo archivo llamado `mysite_data.json`. Puedes ver el contenido del archivo para observar la estructura JSON que incluye todos los diferentes objetos de datos para los diferentes modelos de tus aplicaciones instaladas. Si obtienes un error de codificación al ejecutar el comando, incluye el indicador `-Xutf8` de la siguiente manera para activar el modo UTF-8 de Python:

```bash
python -Xutf8 manage.py dumpdata --indent=2 --output=mysite_data.json
```

Ahora cambiaremos la base de datos en el proyecto de Django y luego importaremos los datos a la nueva base de datos.

#### Cambio de base de datos en el proyecto

Ahora añadirás la configuración de la base de datos PostgreSQL a los ajustes de tu proyecto.

Edita el archivo `settings.py` de tu proyecto y modifica la configuración `DATABASES` para que se vea de la siguiente manera:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME'),
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': config('DB_HOST'),
    }
}
```

> [!NOTE]
> Como novedad en Django 5.1, se encuentran las agrupaciones de conexiones (*connection pools*) para PostgreSQL. Establecer una conexión a la base de datos introduce latencia, por lo que mantener las conexiones abiertas puede resultar un beneficio para el rendimiento. Al crear un grupo de conexiones, Django puede solicitar una conexión del grupo cuando necesita acceder a la base de datos. Toma prestada la conexión del grupo y la devuelve una vez que el código termina con la conexión. Esto significa que las conexiones no se cierran cuando el código termina con ellas.
>
> Para habilitar grupos de conexiones, se debe añadir el diccionario `OPTIONS` a la configuración de la base de datos. La clave para el diccionario es `pool` y se le puede pasar `True` para usar los valores predeterminados, o se le pueden pasar valores específicos:
> ```python
> "OPTIONS": {
>     "pool": {
>         "min_size": 2,
>         "max_size": 4,
>         "timeout": 5
>     }
> }
> ```

El motor de la base de datos ahora es `postgresql`. Las credenciales de la base de datos ahora se cargan desde variables de entorno utilizando `python-decouple`.

Añadamos valores a las variables de entorno. Edita el archivo `.env` de tu proyecto y añade las siguientes líneas:

```text
EMAIL_HOST_USER=your_account@gmail.com
EMAIL_HOST_PASSWORD=xxxxxxxxxxxx
DEFAULT_FROM_EMAIL=My Blog <your_account@gmail.com>
DB_NAME=blog
DB_USER=blog
DB_PASSWORD=xxxxx
DB_HOST=localhost
```

Reemplaza `xxxxx` con la contraseña que utilizaste al iniciar el contenedor de PostgreSQL. La nueva base de datos está vacía.

Ejecuta el siguiente comando para aplicar todas las migraciones de bases de datos a la nueva base de datos PostgreSQL:

```bash
python manage.py migrate
```

Verás una salida que incluye todas las migraciones que se han aplicado, así:

```text
Operations to perform:
  Apply all migrations: admin, auth, blog, contenttypes, sessions, sites, taggit
Running migrations:
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
  Applying taggit.0001_initial... OK
  Applying taggit.0002_auto_20150616_2121... OK
  Applying taggit.0003_taggeditem_add_unique_index... OK
  Applying taggit.0004_alter_taggeditem_content_type_alter_taggeditem_tag... OK
  Applying taggit.0005_auto_20220424_2025... OK
  Applying taggit.0006_rename_taggeditem_content_type_object_id_taggit_tagg_content_8fc721_idx... OK
  Applying blog.0001_initial... OK
  Applying blog.0002_alter_post_slug... OK
  Applying blog.0003_comment... OK
  Applying blog.0004_post_tags... OK
  Applying sessions.0001_initial... OK
  Applying sites.0001_initial... OK
  Applying sites.0002_alter_domain_unique... OK
```

La base de datos PostgreSQL ahora está sincronizada con tus modelos de datos y puedes ejecutar tu proyecto Django apuntando a la nueva base de datos. Llevemos la base de datos al mismo estado cargando los datos que exportamos previamente de SQLite.

#### Carga de datos en la nueva base de datos

Vamos a cargar los fixtures de datos que generamos previamente en nuestra nueva base de datos PostgreSQL.

Ejecuta el siguiente comando para cargar los datos previamente exportados en la base de datos PostgreSQL:

```bash
python manage.py loaddata mysite_data.json
```

Verás la siguiente salida:

```text
Installed 104 object(s) from 1 fixture(s)
```

La cantidad de objetos puede diferir según los usuarios, publicaciones, comentarios y otros objetos que se hayan creado en la base de datos.

Inicia el servidor de desarrollo desde la consola con el siguiente comando:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/admin/blog/post/` en tu navegador para verificar que todas las publicaciones se hayan cargado en la nueva base de datos. Deberías ver todas las publicaciones, de la siguiente manera:

> *Figura 3.28: La lista de publicaciones en el sitio de administración*

#### Búsquedas simples

Habiendo habilitado PostgreSQL en nuestro proyecto, ahora podemos construir un potente motor de búsqueda aprovechando las capacidades de búsqueda de texto completo de PostgreSQL. Comenzaremos con búsquedas básicas e incorporaremos progresivamente características más sofisticadas, como lematización (*stemming*), clasificación (*ranking*) o ponderación de consultas (*weighting*), para construir un motor de búsqueda de texto completo integral.

Edita el archivo `settings.py` de tu proyecto y añade `django.contrib.postgres` a la configuración `INSTALLED_APPS`, de la siguiente manera:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.sites',
    'django.contrib.sitemaps',
    'django.contrib.staticfiles',
    'django.contrib.postgres',
    'taggit',
    'blog.apps.BlogConfig',
]
```

Abre la shell de Django ejecutando el siguiente comando en la consola del sistema:

```bash
python manage.py shell
```

Ahora puedes buscar en un solo campo utilizando la búsqueda de QuerySet `search`.

Ejecuta el siguiente código en la shell de Python:

```python
>>> from blog.models import Post
>>> Post.objects.filter(title__search='django')
<QuerySet [<Post: Who was Django Reinhardt?>]>
```

Esta consulta utiliza PostgreSQL para crear un vector de búsqueda para el campo `title` y una consulta de búsqueda a partir del término `django`. Los resultados se obtienen haciendo coincidir la consulta con el vector.

#### Búsqueda en múltiples campos

Es posible que desees buscar en varios campos. En este caso, deberás definir un objeto `SearchVector`. Construyamos un vector que te permita buscar en los campos `title` y `body` del modelo `Post`.

Ejecuta el siguiente código en la shell de Python:

```python
>>> from django.contrib.postgres.search import SearchVector
>>> from blog.models import Post
>>> 
>>> Post.objects.annotate(
...     search=SearchVector('title', 'body'),
... ).filter(search='django')
<QuerySet [<Post: Markdown post>, <Post: Who was Django Reinhardt?>]>
```

Utilizando `annotate` y definiendo `SearchVector` con ambos campos, proporcionas la funcionalidad para hacer coincidir la consulta tanto con el título como con el cuerpo de las publicaciones.

> [!TIP]
> La búsqueda de texto completo es un proceso intensivo. Si buscas en más de unos cientos de filas, debes definir un índice funcional que coincida con el vector de búsqueda que estás utilizando. Django proporciona un campo `SearchVectorField` para tus modelos. Puedes leer más sobre esto en [https://docs.djangoproject.com/en/5.2/ref/contrib/postgres/search/#performance](https://docs.djangoproject.com/en/5.2/ref/contrib/postgres/search/#performance).

#### Creación de una vista de búsqueda

Ahora, crearás una vista personalizada para permitir a tus usuarios buscar publicaciones. Primero, necesitarás un formulario de búsqueda. Edita el archivo `forms.py` de la aplicación `blog` y añade el siguiente formulario:

```python
class SearchForm(forms.Form):
    query = forms.CharField()
```

Utilizarás el campo `query` para permitir a los usuarios introducir términos de búsqueda. Edita el archivo `views.py` de la aplicación `blog` y añádele el siguiente código:

```python
# ...
from django.contrib.postgres.search import SearchVector
from .forms import CommentForm, EmailPostForm, SearchForm

# ...


def post_search(request):
    form = SearchForm()
    query = None
    results = []
    if 'query' in request.GET:
        form = SearchForm(request.GET)
        if form.is_valid():
            query = form.cleaned_data['query']
            results = (
                Post.published.annotate(
                    search=SearchVector('title', 'body'),
                ).filter(search=query)
            )
    return render(
        request,
        'blog/post/search.html',
        {
            'form': form,
            'query': query,
            'results': results
        }
    )
```

En la vista anterior, primero instanciamos el formulario `SearchForm`. Para comprobar si el formulario se ha enviado, buscamos el parámetro `query` en el diccionario `request.GET`. Enviamos el formulario mediante el método GET en lugar de POST para que la URL resultante incluya el parámetro `query` y sea fácil de compartir. Cuando se envía el formulario, lo instanciamos con los datos GET enviados y verificamos que los datos del formulario sean válidos. Si el formulario es válido, buscamos publicaciones publicadas con una instancia personalizada de `SearchVector` construida con los campos `title` y `body`.

La vista de búsqueda ya está lista. Necesitamos crear una plantilla para mostrar el formulario y los resultados cuando el usuario realiza una búsqueda.

Crea un nuevo archivo dentro del directorio `templates/blog/post/`, nómbralo `search.html` y añade el siguiente código:

```html
{% extends "blog/base.html" %}
{% load blog_tags %}

{% block title %}Search{% endblock %}

{% block content %}
    {% if query %}
        <h1>Posts containing "{{ query }}"</h1>
        <h3>
            {% with results.count as total_results %}
                Found {{ total_results }} result{{ total_results|pluralize }}
            {% endwith %}
        </h3>
        {% for post in results %}
            <h4>
                <a href="{{ post.get_absolute_url }}">
                    {{ post.title }}
                </a>
            </h4>
            {{ post.body|markdown|truncatewords_html:12 }}
        {% empty %}
            <p>There are no results for your query.</p>
        {% endfor %}
        <p><a href="{% url "blog:post_search" %}">Search again</a></p>
    {% else %}
        <h1>Search for posts</h1>
        <form method="get">
            {{ form.as_p }}
            <input type="submit" value="Search">
        </form>
    {% endif %}
{% endblock %}
```

Al igual que en la vista de búsqueda, distinguimos si el formulario se ha enviado por la presencia del parámetro `query`. Antes de que se envíe la consulta, mostramos el formulario y un botón de envío. Cuando se envía el formulario de búsqueda, mostramos la consulta realizada, el número total de resultados y la lista de publicaciones que coinciden con la consulta de búsqueda.

Finalmente, edita el archivo `urls.py` de la aplicación `blog` y añade el siguiente patrón de URL:

```python
urlpatterns = [
    # Post views
    path('', views.post_list, name='post_list'),
    # path('', views.PostListView.as_view(), name='post_list'),
    path(
        'tag/<slug:tag_slug>/',
        views.post_list,
        name='post_list_by_tag'
    ),
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
    path('feed/', LatestPostsFeed(), name='post_feed'),
    path('search/', views.post_search, name='post_search'),
]
```

A continuación, abre `http://127.0.0.1:8000/blog/search/` en tu navegador. Deberías ver el siguiente formulario de búsqueda:

> *Figura 3.29: El formulario con el campo query para buscar publicaciones*

Introduce una consulta y haz clic en el botón **Search**. Verás los resultados de la consulta de búsqueda, de la siguiente manera:

> *Figura 3.30: Resultados de búsqueda para el término "jazz"*

¡Felicidades! Has creado un motor de búsqueda básico para tu blog.

#### Lematización (stemming) y clasificación de resultados

La lematización (*stemming*) es el proceso de reducir las palabras a su raíz, base o forma raíz. Los motores de búsqueda utilizan la lematización para reducir las palabras indexadas a su raíz y poder hacer coincidir palabras flexionadas o derivadas. Por ejemplo, las palabras "music", "musical" y "musicality" pueden ser consideradas palabras similares por un motor de búsqueda. El proceso de lematización normaliza cada token de búsqueda en un lexema, una unidad de significado léxico que subyace a un conjunto de palabras relacionadas a través de la flexión. Las palabras "music", "musical" y "musicality" se convertirían en "music" al crear una consulta de búsqueda.

Django proporciona una clase `SearchQuery` para traducir términos en un objeto de consulta de búsqueda. De forma predeterminada, los términos se pasan a través de algoritmos de lematización, lo que te ayuda a obtener mejores coincidencias.

El motor de búsqueda de PostgreSQL también elimina las palabras vacías (*stop words*), como "a", "the", "on" y "of". Las palabras vacías son un conjunto de palabras de uso común en un idioma. Se eliminan al crear una consulta de búsqueda porque aparecen con demasiada frecuencia para ser relevantes en las búsquedas. Puedes encontrar la lista de palabras vacías utilizadas por PostgreSQL para el idioma inglés en [https://github.com/postgres/postgres/blob/master/src/backend/snowball/stopwords/english.stop](https://github.com/postgres/postgres/blob/master/src/backend/snowball/stopwords/english.stop).

También queremos ordenar los resultados por relevancia. PostgreSQL proporciona una función de clasificación que ordena los resultados según la frecuencia con la que aparecen los términos de búsqueda y lo cerca que están entre sí.

Edita el archivo `views.py` de la aplicación `blog` y añade las siguientes importaciones:

```python
from django.contrib.postgres.search import (
    SearchVector,
    SearchQuery,
    SearchRank
)
```

Luego, edita la vista `post_search`, de la siguiente manera:

```python
def post_search(request):
    form = SearchForm()
    query = None
    results = []
    if 'query' in request.GET:
        form = SearchForm(request.GET)
        if form.is_valid():
            query = form.cleaned_data['query']
            search_vector = SearchVector('title', 'body')
            search_query = SearchQuery(query)
            results = (
                Post.published.annotate(
                    search=search_vector,
                    rank=SearchRank(search_vector, search_query)
                )
                .filter(search=search_query)
                .order_by('-rank')
            )
    return render(
        request,
        'blog/post/search.html',
        {
            'form': form,
            'query': query,
            'results': results
        }
    )
```

En el código anterior, creamos un objeto `SearchQuery`, filtramos los resultados por él y usamos `SearchRank` para ordenar los resultados por relevancia.

Puedes abrir `http://127.0.0.1:8000/blog/search/` en tu navegador y probar diferentes búsquedas para comprobar la lematización y la clasificación. El siguiente es un ejemplo de clasificación por el número de apariciones de la palabra `django` en el título y el cuerpo de las publicaciones:

> *Figura 3.31: Resultados de búsqueda para el término "django"*

#### Lematización y eliminación de palabras vacías en diferentes idiomas

Podemos configurar `SearchVector` y `SearchQuery` para ejecutar la lematización y eliminar palabras vacías en cualquier idioma. Podemos pasar un atributo `config` a `SearchVector` y `SearchQuery` para utilizar una configuración de búsqueda diferente. Esto nos permite utilizar diferentes analizadores y diccionarios de idiomas. El siguiente ejemplo ejecuta la lematización y elimina las palabras vacías en español:

```python
search_vector = SearchVector('title', 'body', config='spanish')
search_query = SearchQuery(query, config='spanish')
results = (
    Post.published.annotate(
        search=search_vector,
        rank=SearchRank(search_vector, search_query)
    )
    .filter(search=search_query)
    .order_by('-rank')
)
```

Puedes encontrar el diccionario de palabras vacías en español utilizado por PostgreSQL en [https://github.com/postgres/postgres/blob/master/src/backend/snowball/stopwords/spanish.stop](https://github.com/postgres/postgres/blob/master/src/backend/snowball/stopwords/spanish.stop).

#### Ponderación de consultas

Podemos potenciar vectores específicos para que se les atribuya más peso al ordenar los resultados por relevancia. Por ejemplo, podemos usar esto para dar más relevancia a las publicaciones que coinciden por título en lugar de por contenido.

Edita el archivo `views.py` de la aplicación `blog` y modifica la vista `post_search` de la siguiente manera:

```python
def post_search(request):
    form = SearchForm()
    query = None
    results = []
    if 'query' in request.GET:
        form = SearchForm(request.GET)
        if form.is_valid():
            query = form.cleaned_data['query']
            search_vector = SearchVector(
                'title', weight='A'
            ) + SearchVector('body', weight='B')
            search_query = SearchQuery(query)
            results = (
                Post.published.annotate(
                    search=search_vector,
                    rank=SearchRank(search_vector, search_query)
                )
                .filter(rank__gte=0.3)
                .order_by('-rank')
            )
    return render(
        request,
        'blog/post/search.html',
        {
            'form': form,
            'query': query,
            'results': results
        }
    )
```

En el código anterior, aplicamos diferentes pesos a los vectores de búsqueda construidos utilizando los campos `title` y `body`. Los pesos predeterminados son `D`, `C`, `B` y `A`, y se refieren a los números `0.1`, `0.2`, `0.4` y `1.0`, respectivamente. Aplicamos un peso de `1.0` al vector de búsqueda de títulos (`A`) y un peso de `0.4` al vector del cuerpo (`B`). Las coincidencias de títulos prevalecerán sobre las coincidencias de contenido del cuerpo. Filtramos los resultados para mostrar solo aquellos con un rango superior a `0.3`.

#### Búsqueda con similitud de trigramas

Otro enfoque de búsqueda es la similitud de trigramas (*trigram similarity*). Un trigrama es un grupo de tres caracteres consecutivos. Puedes medir la similitud de dos cadenas contando el número de trigramas que comparten. Este enfoque resulta muy eficaz para medir la similitud de palabras en muchos idiomas.

Para usar trigramas en PostgreSQL, primero deberás instalar la extensión de base de datos `pg_trgm`. Django proporciona operaciones de migración de bases de datos para crear extensiones de PostgreSQL. Añadamos una migración que cree la extensión en la base de datos.

Primero, ejecuta el siguiente comando en la consola para crear una migración vacía:

```bash
python manage.py makemigrations --name=trigram_ext --empty blog
```

Esto creará una migración vacía para la aplicación `blog`. Verás la siguiente salida:

```text
Migrations for 'blog':
  blog/migrations/0005_trigram_ext.py
```

Edita el archivo `blog/migrations/0005_trigram_ext.py` y añade las siguientes líneas:

```python
from django.contrib.postgres.operations import TrigramExtension
from django.db import migrations


class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0004_post_tags'),
    ]

    operations = [
        TrigramExtension()
    ]
```

Has añadido la operación `TrigramExtension` a la migración de base de datos. Esta operación ejecuta la sentencia SQL `CREATE EXTENSION pg_trgm` para crear la extensión en PostgreSQL.

Puedes encontrar más información sobre las operaciones de migración de bases de datos en [https://docs.djangoproject.com/en/5.2/ref/contrib/postgres/operations/](https://docs.djangoproject.com/en/5.2/ref/contrib/postgres/operations/).

Ahora ejecuta la migración con el siguiente comando:

```bash
python manage.py migrate blog
```

Verás la siguiente salida:

```text
Running migrations:
  Applying blog.0005_trigram_ext... OK
```

La extensión `pg_trgm` se ha creado en la base de datos. Modifiquemos `post_search` para buscar por trigramas.

Edita el archivo `views.py` de tu aplicación `blog` y añade la siguiente importación:

```python
from django.contrib.postgres.search import TrigramSimilarity
```

Luego, modifica la vista `post_search` de la siguiente manera:

```python
def post_search(request):
    form = SearchForm()
    query = None
    results = []
    if 'query' in request.GET:
        form = SearchForm(request.GET)
        if form.is_valid():
            query = form.cleaned_data['query']
            results = (
                Post.published.annotate(
                    similarity=TrigramSimilarity('title', query),
                )
                .filter(similarity__gt=0.1)
                .order_by('-similarity')
            )
    return render(
        request,
        'blog/post/search.html',
        {
            'form': form,
            'query': query,
            'results': results
        }
    )
```

Abre `http://127.0.0.1:8000/blog/search/` en tu navegador y prueba diferentes búsquedas de trigramas. El siguiente ejemplo muestra un hipotético error tipográfico en el término `django`, mostrando resultados de búsqueda para `yango`:

> *Figura 3.32: Resultados de búsqueda para el término "yango"*

Hemos añadido un potente motor de búsqueda a la aplicación de blog.

Puedes encontrar más información sobre la búsqueda de texto completo con Django y PostgreSQL en [https://docs.djangoproject.com/en/5.2/ref/contrib/postgres/search/](https://docs.djangoproject.com/en/5.2/ref/contrib/postgres/search/).

---

### Resumen

En este capítulo, implementaste un sistema de etiquetado integrando una aplicación de terceros en tu proyecto. Generaste recomendaciones de publicaciones utilizando QuerySets complejos. También aprendiste a crear etiquetas y filtros de plantilla personalizados de Django para proporcionar a las plantillas funcionalidades a medida. Además, creaste un sitemap para que los motores de búsqueda rastreen tu sitio y un feed RSS para que los usuarios se suscriban a tu blog. Luego, creaste un motor de búsqueda para tu blog utilizando el motor de búsqueda de texto completo de PostgreSQL.

En el próximo capítulo, aprenderás cómo construir un sitio web social utilizando el framework de autenticación de Django y cómo implementar funcionalidades de cuenta de usuario y perfiles de usuario personalizados.

---

### Ampliación de tu proyecto usando IA

Habiendo completado la aplicación de blog, es probable que tengas numerosas ideas para añadir nuevas funcionalidades a tu blog. Esta sección tiene como objetivo proporcionar algunas ideas para explorar nuevas funcionalidades para incorporar a tu proyecto con la ayuda de ChatGPT. ChatGPT es un modelo de lenguaje grande (LLM) de IA sofisticado creado por OpenAI que genera respuestas similares a las humanas basadas en las instrucciones que recibe. En esta sección, se te presentará una tarea para extender tu proyecto, acompañada de un prompt de muestra para que ChatGPT te ayude.

Interactúa con ChatGPT en [https://chat.openai.com/](https://chat.openai.com/). Encontrarás orientación similar después de completar cada proyecto de Django dentro de este libro, en el Capítulo 7, *Seguimiento de las acciones del usuario*, Capítulo 11, *Adición de internacionalización a tu tienda*, y Capítulo 17, *Puesta en producción*.

Mejoremos aún más tu blog con la ayuda de ChatGPT. Tu blog actualmente permite filtrar publicaciones por etiquetas. Añadir estas etiquetas a nuestro sitemap podría mejorar significativamente la optimización SEO del blog. Utiliza el prompt proporcionado en [https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter03/prompts/task.md](https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter03/prompts/task.md) para añadir páginas de etiquetas al sitemap. Este desafío es una excelente oportunidad para perfeccionar tu proyecto y profundizar tu comprensión de Django, mientras aprendes a interactuar con ChatGPT.

ChatGPT está listo para ayudarte con problemas de código. Simplemente comparte tu código junto con los errores a los que te enfrentas, y ChatGPT puede ayudarte a identificar y resolver los problemas.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter03](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter03)
- **django-taggit:** [https://github.com/jazzband/django-taggit](https://github.com/jazzband/django-taggit)
- **Managers de ORM de django-taggit:** [https://django-taggit.readthedocs.io/en/latest/api.html](https://django-taggit.readthedocs.io/en/latest/api.html)
- **Relaciones de muchos a muchos:** [https://docs.djangoproject.com/en/5.2/topics/db/examples/many_to_many/](https://docs.djangoproject.com/en/5.2/topics/db/examples/many_to_many/)
- **Funciones de agregación de Django:** [https://docs.djangoproject.com/en/5.2/topics/db/aggregation/](https://docs.djangoproject.com/en/5.2/topics/db/aggregation/)
- **Etiquetas y filtros de plantilla integrados:** [https://docs.djangoproject.com/en/5.2/ref/templates/builtins/](https://docs.djangoproject.com/en/5.2/ref/templates/builtins/)
- **Escritura de etiquetas de plantilla personalizadas:** [https://docs.djangoproject.com/en/5.2/howto/custom-template-tags/](https://docs.djangoproject.com/en/5.2/howto/custom-template-tags/)
- **Referencia del formato Markdown:** [https://daringfireball.net/projects/markdown/basics](https://daringfireball.net/projects/markdown/basics)
- **Framework de sitemaps de Django:** [https://docs.djangoproject.com/en/5.2/ref/contrib/sitemaps/](https://docs.djangoproject.com/en/5.2/ref/contrib/sitemaps/)
- **Framework de sitios de Django:** [https://docs.djangoproject.com/en/5.2/ref/contrib/sites/](https://docs.djangoproject.com/en/5.2/ref/contrib/sites/)
- **Framework de feeds de sindicación de Django:** [https://docs.djangoproject.com/en/5.2/ref/contrib/syndication/](https://docs.djangoproject.com/en/5.2/ref/contrib/syndication/)
- **Instrucciones de descarga e instalación de Docker:** [https://docs.docker.com/get-docker/](https://docs.docker.com/get-docker/)
- **Imagen de Docker de PostgreSQL:** [https://hub.docker.com/_/postgres](https://hub.docker.com/_/postgres)
- **Descargas de PostgreSQL:** [https://www.postgresql.org/download/](https://www.postgresql.org/download/)
- **Capacidades de búsqueda de texto completo de PostgreSQL:** [https://www.postgresql.org/docs/16/textsearch.html](https://www.postgresql.org/docs/16/textsearch.html)
- **Operaciones de migración de bases de datos:** [https://docs.djangoproject.com/en/5.2/ref/contrib/postgres/operations/](https://docs.djangoproject.com/en/5.2/ref/contrib/postgres/operations/)
- **Soporte de Django para búsqueda de texto completo de PostgreSQL:** [https://docs.djangoproject.com/en/5.2/ref/contrib/postgres/search/](https://docs.djangoproject.com/en/5.2/ref/contrib/postgres/search/)
- **Interfaz de ChatGPT:** [https://chat.openai.com/](https://chat.openai.com/)
- **Prompt de muestra de ChatGPT para añadir páginas de etiquetas al sitemap:** [https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter03/prompts/task.md](https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter03/prompts/task.md)
