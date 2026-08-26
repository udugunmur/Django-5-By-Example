# Parte 2: Creación de un sitio web social

## Capítulo 7: Seguimiento de acciones de usuario

### Introducción

En el capítulo anterior, construiste un bookmarklet de JavaScript para compartir contenido de otros sitios web en tu plataforma. También implementaste acciones asíncronas con JavaScript en tu proyecto y creaste un desplazamiento infinito (*infinite scroll*).

En este capítulo, aprenderás a construir un sistema de seguimiento (*follow*) y crear un flujo de actividad de usuario (*activity stream*). También descubrirás cómo funcionan las señales (*signals*) de Django e integrarás el almacenamiento de E/S rápida de Redis en tu proyecto para registrar las visualizaciones de elementos.

Este capítulo cubrirá los siguientes puntos:

- Creación de un sistema de seguimiento
- Creación de relaciones de muchos a muchos con un modelo intermedio
- Creación de una aplicación de flujo de actividad
- Adición de relaciones genéricas a los modelos
- Optimización de QuerySets para objetos relacionados
- Uso de señales para desnormalizar conteos
- Uso de Django Debug Toolbar para obtener información relevante de depuración
- Conteo de visualizaciones de imágenes con Redis
- Creación de un ranking de las imágenes más vistas con Redis

---

### Visión general funcional

La Figura 7.1 muestra una representación de las vistas, plantillas y funcionalidades que se construirán en este capítulo:

> *Figura 7.1: Diagrama de funcionalidades construidas en el Capítulo 7*

En este capítulo, construirás la vista `user_list` para listar todos los usuarios y la vista `user_detail` para mostrar el perfil de un usuario individual. Implementarás un sistema de seguimiento con JavaScript, utilizando la vista `user_follow` para almacenar los seguimientos de usuarios. Crearás un sistema para almacenar las acciones de los usuarios e implementarás las acciones para crear una cuenta, seguir a un usuario, crear una imagen y dar "me gusta" a una imagen. Utilizarás este sistema para mostrar un flujo de actividad en la vista del panel de control (*dashboard*) con las acciones más recientes. También utilizarás Redis para almacenar una visualización cada vez que se cargue la vista `image_detail` y crearás la vista `image_ranking` para mostrar una clasificación de las imágenes más vistas.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter07](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter07).

Todos los paquetes de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente del capítulo. Puedes seguir las instrucciones para instalar cada paquete de Python en las siguientes secciones, o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Creación de un sistema de seguimiento

Construyamos un sistema de seguimiento en tu proyecto. Esto significa que tus usuarios podrán seguirse unos a otros y rastrear lo que otros usuarios comparten en la plataforma. La relación entre usuarios es una relación de muchos a muchos (*many-to-many*); esto significa que un usuario puede seguir a múltiples usuarios y estos, a su vez, pueden ser seguidos por múltiples usuarios.

#### Creación de relaciones muchos a muchos con un modelo intermedio

En capítulos anteriores, creaste relaciones de muchos a muchos añadiendo el campo `ManyToManyField` a uno de los modelos relacionados y dejando que Django creara la tabla de base de datos para la relación. Esto es adecuado para la mayoría de los casos, pero a veces es posible que necesites crear un modelo intermedio para la relación. Crear un modelo intermedio es necesario cuando deseas almacenar información adicional sobre la relación, por ejemplo, la fecha en que se creó la relación o un campo que describa la naturaleza de la relación.

Creemos un modelo intermedio para construir relaciones entre usuarios. Hay dos razones para usar un modelo intermedio:

1. Estás usando el modelo `User` proporcionado por Django y deseas evitar alterarlo.
2. Deseas almacenar el momento en que se creó la relación.

Edita el archivo `models.py` de la aplicación `account` y añádele el siguiente código:

```python
class Contact(models.Model):
    user_from = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        related_name='rel_from_set',
        on_delete=models.CASCADE
    )
    user_to = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        related_name='rel_to_set',
        on_delete=models.CASCADE
    )
    created = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=['-created']),
        ]
        ordering = ['-created']

    def __str__(self):
        return f'{self.user_from} follows {self.user_to}'
```

El código anterior muestra el modelo `Contact` que utilizarás para las relaciones de usuario. Contiene los siguientes campos:

- `user_from`: Un `ForeignKey` para el usuario que crea la relación.
- `user_to`: Un `ForeignKey` para el usuario que está siendo seguido.
- `created`: Un campo `DateTimeField` con `auto_now_add=True` para almacenar el momento en que se creó la relación.

> [!NOTE]
> En Django 5.2, un modelo intermedio (*through model*) como este es candidato para el nuevo campo `CompositePrimaryKey` utilizando `user_from` y `user_to` para formar la clave. Sin embargo, vale la pena considerar si un proyecto necesita utilizar esta función; esto se debe a que aún no es compatible con `django-admin` y añade complejidad cuando un ID entero suele ser suficiente.

Se crea automáticamente un índice de base de datos en los campos `ForeignKey`. En la clase `Meta` del modelo, hemos definido un índice de base de datos en orden descendente para el campo `created`. También hemos añadido el atributo `ordering` para indicarle a Django que debe ordenar los resultados por el campo `created` de forma predeterminada. Indicamos el orden descendente usando un guión antes del nombre del campo, como `-created`.

La Figura 7.2 muestra el modelo intermedio `Contact` y su tabla de base de datos correspondiente:

> *Figura 7.2: El modelo intermedio Contact y su tabla de base de datos*

Utilizando el ORM, podrías crear una relación para un usuario, `user1`, siguiendo a otro usuario, `user2`, de esta manera:

```python
user1 = User.objects.get(id=1)
user2 = User.objects.get(id=2)
Contact.objects.create(user_from=user1, user_to=user2)
```

Los gestores relacionados, `rel_from_set` y `rel_to_set`, devolverán un QuerySet para el modelo `Contact`. Para acceder al lado final de la relación desde el modelo `User`, sería deseable que `User` contuviera un `ManyToManyField`, de la siguiente manera:

```python
following = models.ManyToManyField(
    'self',
    through=Contact,
    related_name='followers',
    symmetrical=False
)
```

En el ejemplo anterior, le indicas a Django que use tu modelo intermedio personalizado para la relación añadiendo `through=Contact` a `ManyToManyField`. Esta es una relación de muchos a muchos desde el modelo `User` hacia sí mismo; haces referencia a `'self'` en `ManyToManyField` para crear una relación con el mismo modelo.

Cuando necesites campos adicionales en una relación de muchos a muchos, crea un modelo personalizado con un `ForeignKey` para cada lado de la relación. Puedes gestionar la relación utilizando el modelo intermedio, o puedes añadir un campo `ManyToManyField` en uno de los modelos relacionados e indicarle a Django que debe usar tu modelo intermedio incluyéndolo en el parámetro `through`.

Si el modelo `User` fuera parte de tu aplicación, podrías añadir el campo anterior al modelo. Sin embargo, no puedes alterar la clase `User` directamente porque pertenece a la aplicación `django.contrib.auth`. Adoptemos un enfoque ligeramente diferente añadiendo este campo dinámicamente al modelo `User`.

Edita el archivo `models.py` de la aplicación `account` y añade las siguientes líneas:

```python
from django.contrib.auth import get_user_model

# ...

# Add the following field to User dynamically
user_model = get_user_model()
user_model.add_to_class(
    'following',
    models.ManyToManyField(
        'self',
        through=Contact,
        related_name='followers',
        symmetrical=False
    )
)
```

En el código anterior, recuperas el modelo de usuario con la función genérica `get_user_model()` proporcionada por Django. Utilizas el método `add_to_class()` de los modelos de Django para modificar (*monkey patch*) el modelo `User`.

Ten en cuenta que usar `add_to_class()` no es la forma recomendada de añadir campos a los modelos. Sin embargo, puedes aprovechar su uso en este caso para evitar crear un modelo de usuario personalizado, manteniendo todas las ventajas del modelo `User` integrado de Django.

También simplificas la forma en que recuperas objetos relacionados usando el ORM de Django con `user.followers.all()` y `user.following.all()`. Utilizas el modelo intermedio `Contact` y evitas consultas complejas que implicarían uniones de base de datos adicionales, como habría sido el caso si hubieras definido la relación en tu modelo personalizado `Profile`. La tabla para esta relación de muchos a muchos se creará utilizando el modelo `Contact`. Por lo tanto, el `ManyToManyField`, añadido dinámicamente, no implicará ningún cambio en la base de datos para el modelo `User` de Django.

> [!TIP]
> Ten en cuenta que, en la mayoría de los casos, es preferible añadir campos al modelo `Profile` que creaste antes, en lugar de alterar el modelo `User`. Idealmente, no deberías alterar el modelo `User` existente de Django. Django te permite usar modelos de usuario personalizados. Si deseas usar un modelo de usuario personalizado, consulta la documentación en [https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#specifying-a-custom-user-model](https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#specifying-a-custom-user-model).
> 
> Cuando comienzas un nuevo proyecto, es muy recomendable que crees un modelo de usuario personalizado, incluso si el modelo `User` predeterminado es suficiente para ti. Esto se debe a que obtienes la opción de personalizar el modelo en el futuro.

Ten en cuenta que la relación incluye `symmetrical=False`. Cuando defines un `ManyToManyField` en el modelo creando una relación consigo mismo, Django fuerza que la relación sea simétrica. En este caso, estás estableciendo `symmetrical=False` para definir una relación no simétrica (si yo te sigo, no significa que tú me sigas automáticamente).

Cuando utilizas un modelo intermedio para relaciones de muchos a muchos, algunos de los métodos del gestor relacionado están desactivados, como `add()`, `create()` o `remove()`. En su lugar, debes crear o eliminar instancias del modelo intermedio.

Ejecuta el siguiente comando para generar las migraciones iniciales para la aplicación `account`:

```bash
python manage.py makemigrations account
```

Obtendrás una salida como la siguiente:

```text
Migrations for 'account':
  account/migrations/0002_contact.py
    - Create model Contact
```

Ahora, ejecuta el siguiente comando para sincronizar la aplicación con la base de datos:

```bash
python manage.py migrate account
```

Deberías ver una salida que incluye la siguiente línea:

```text
Applying account.0002_contact... OK
```

El modelo `Contact` ahora está sincronizado con la base de datos y puedes crear relaciones entre usuarios. Sin embargo, tu sitio aún no ofrece una forma de explorar usuarios ni de ver el perfil de un usuario en particular. Construyamos las vistas de lista y detalle para el modelo `User`.

#### Creación de vistas de lista y detalle para perfiles de usuario

Abre el archivo `views.py` de la aplicación `account` y añade el siguiente código:

```python
from django.contrib.auth import authenticate, get_user_model, login
from django.shortcuts import get_object_or_404, render

# ...

User = get_user_model()


@login_required
def user_list(request):
    users = User.objects.filter(is_active=True)
    return render(
        request,
        'account/user/list.html',
        {'section': 'people', 'users': users}
    )


@login_required
def user_detail(request, username):
    user = get_object_or_404(User, username=username, is_active=True)
    return render(
        request,
        'account/user/detail.html',
        {'section': 'people', 'user': user}
    )
```

Estas son vistas simples de lista y detalle para objetos `User`. Recuperamos el modelo `User` dinámicamente usando la función `get_user_model()`. La vista `user_list` obtiene todos los usuarios activos. El modelo `User` de Django contiene un indicador `is_active` para designar si la cuenta de usuario se considera activa. Filtras la consulta por `is_active=True` para devolver solo usuarios activos. Esta vista devuelve todos los resultados, pero puedes mejorarla añadiendo paginación de la misma manera que lo hiciste para la vista `image_list`.

La vista `user_detail` utiliza el atajo `get_object_or_404()` para recuperar el usuario activo con el nombre de usuario dado. La vista devuelve una respuesta HTTP 404 si no se encuentra ningún usuario activo con el nombre de usuario dado.

Edita el archivo `urls.py` de la aplicación `account` y añade un patrón de URL para cada vista, de la siguiente manera:

```python
urlpatterns = [
    # ...
    path('', include('django.contrib.auth.urls')),
    path('', views.dashboard, name='dashboard'),
    path('register/', views.register, name='register'),
    path('edit/', views.edit, name='edit'),
    path('users/', views.user_list, name='user_list'),
    path('users/<username>/', views.user_detail, name='user_detail'),
]
```

Utilizarás el patrón de URL `user_detail` para generar la URL canónica para los usuarios. Ya has definido un método `get_absolute_url()` en un modelo para devolver la URL canónica para cada objeto. Otra forma de especificar la URL para un modelo es añadiendo la configuración `ABSOLUTE_URL_OVERRIDES` a tu proyecto.

Al usar el nombre de usuario en lugar del ID de usuario en el patrón de URL `user_detail`, mejoras tanto la usabilidad como la seguridad. Los nombres de usuario, a diferencia de los IDs secuenciales, frustran los ataques de enumeración al ocultar la estructura de tus datos. Esto dificulta que los atacantes predigan URLs y formulen vectores de ataque.

Edita el archivo `settings.py` de tu proyecto y añade el siguiente código:

```python
from django.urls import reverse_lazy

# ...

ABSOLUTE_URL_OVERRIDES = {
    'auth.user': lambda u: reverse_lazy('user_detail', args=[u.username])
}
```

Django añade un método `get_absolute_url()` dinámicamente a cualquier modelo que aparezca en la configuración `ABSOLUTE_URL_OVERRIDES`. Este método devuelve la URL correspondiente para el modelo dado especificado en la configuración. En la sección de código anterior, generas la URL `user_detail` para el usuario dado para el objeto `auth.user`. Ahora, puedes usar `get_absolute_url()` en una instancia de `User` para recuperar su URL correspondiente.

Abre el shell de Python con el siguiente comando:

```bash
python manage.py shell
```

> [!NOTE]
> Como novedad en Django 5.2, el shell ahora importa automáticamente todos los modelos de tus `INSTALLED_APPS`. También puedes crear una subclase del comando shell de Django para importar automáticamente los módulos de tu elección sobrescribiendo el método `get_auto_imports` y devolviendo una lista de rutas de módulos de Python que se importarán. Por ejemplo:
> 
> ```python
> from django.core.management.commands import shell
> 
> 
> class Command(shell.Command):
>     def get_auto_imports(self):
>         return super().get_auto_imports() + [
>             "django.conf.settings"
>         ]
> ```

Luego ejecuta el siguiente código para probarlo:

```text
>>> from django.contrib.auth.models import User
>>> user = User.objects.latest('id')
>>> str(user.get_absolute_url())
'/account/users/ellington/'
```

La URL devuelta sigue el formato esperado, `/account/users/<username>/`.

Necesitarás crear plantillas para las vistas que acabas de construir. Añade el siguiente directorio y archivos al directorio `templates/account/` de la aplicación `account`:

```text
user/
    detail.html
    list.html
```

Edita la plantilla `account/user/list.html` y añade el siguiente código:

```html
{% extends "base.html" %}
{% load thumbnail %}

{% block title %}People{% endblock %}

{% block content %}
    <h1>People</h1>
    <div id="people-list">
        {% for user in users %}
            <div class="user">
                <a href="{{ user.get_absolute_url }}">
                    <img src="{% thumbnail user.profile.photo 180x180 %}">
                </a>
                <div class="info">
                    <a href="{{ user.get_absolute_url }}" class="title">
                        {{ user.get_full_name }}
                    </a>
                </div>
            </div>
        {% endfor %}
    </div>
{% endblock %}
```

La plantilla anterior te permite listar todos los usuarios activos en el sitio. Iteras sobre los usuarios dados y utilizas la etiqueta de plantilla `{% thumbnail %}` de `easy-thumbnails` para generar miniaturas de fotos de perfil.

Ten en cuenta que los usuarios deben tener una foto de perfil. Para usar una imagen predeterminada para los usuarios que no tienen una foto de perfil, puedes añadir una instrucción if/else para verificar si el usuario tiene una foto de perfil, como `{% if user.profile.photo %} {# photo thumbnail #} {% else %} {# default image #} {% endif %}`.

Abre la plantilla `base.html` de tu proyecto e incluye la URL `user_list` en el atributo `href` del siguiente elemento de menú:

```html
<ul class="menu">
    ...
    <li {% if section == "people" %}class="selected"{% endif %}>
        <a href="{% url "user_list" %}">People</a>
    </li>
</ul>
```

Inicia el servidor de desarrollo con el siguiente comando:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/account/users/` en tu navegador. Deberías ver una lista de usuarios como la siguiente:

> *Figura 7.3: La página de lista de usuarios con miniaturas de fotos de perfil*

Recuerda que si tienes alguna dificultad para generar miniaturas, puedes añadir `THUMBNAIL_DEBUG = True` a tu archivo `settings.py` para obtener información de depuración en la consola.

Edita la plantilla `account/user/detail.html` de la aplicación `account` y añade el siguiente código:

```html
{% extends "base.html" %}
{% load thumbnail %}

{% block title %}{{ user.get_full_name }}{% endblock %}

{% block content %}
    <h1>{{ user.get_full_name }}</h1>
    <div class="profile-info">
        <img src="{% thumbnail user.profile.photo 180x180 %}" class="user-detail">
    </div>
    {% with total_followers=user.followers.count %}
        <span class="count">
            <span class="total">{{ total_followers }}</span>
            follower{{ total_followers|pluralize }}
        </span>
        <a href="#" data-id="{{ user.id }}" data-action="{% if request.user in user.followers.all %}un{% endif %}follow" class="follow button">
            {% if request.user not in user.followers.all %}
                Follow
            {% else %}
                Unfollow
            {% endif %}
        </a>
        <div id="image-list" class="image-container">
            {% include "images/image/list_images.html" with images=user.images_created.all %}
        </div>
    {% endwith %}
{% endblock %}
```

Asegúrate de que ninguna etiqueta de plantilla se divida en varias líneas; Django no admite etiquetas de varias líneas.

En la plantilla de detalle, se muestra el perfil del usuario y se utiliza la etiqueta de plantilla `{% thumbnail %}` para mostrar la foto de perfil. El número total de seguidores se presenta junto con un enlace para seguir o dejar de seguir al usuario. Este enlace se utilizará para seguir/dejar de seguir a un usuario en particular. Los atributos `data-id` y `data-action` del elemento HTML `<a>` contienen el ID del usuario y la acción inicial a realizar cuando se hace clic en el elemento de enlace: follow o unfollow. La acción inicial (seguir o dejar de seguir) depende de si el usuario que solicita la página ya es seguidor del usuario. Las imágenes guardadas por el usuario se muestran incluyendo la plantilla `images/image/list_images.html`.

Abre tu navegador de nuevo y haz clic en un usuario que haya guardado algunas imágenes. La página del usuario se verá de la siguiente manera:

> *Figura 7.4: La página de detalle de usuario (Imagen de Chick Corea por ataelw, Licencia: CC BY 2.0)*

#### Adición de acciones de seguimiento y dejar de seguir con JavaScript

Añadamos funcionalidad para seguir/dejar de seguir a usuarios. Crearemos una nueva vista para seguir/dejar de seguir a usuarios e implementaremos una petición HTTP asíncrona con JavaScript para la acción de seguir/dejar de seguir.

Edita el archivo `views.py` de la aplicación `account` y añade el siguiente código:

```python
from django.http import HttpResponse, JsonResponse
from django.views.decorators.http import require_POST
from .models import Contact, Profile

# ...


@require_POST
@login_required
def user_follow(request):
    user_id = request.POST.get('id')
    action = request.POST.get('action')
    if user_id and action:
        try:
            user = User.objects.get(id=user_id)
            if action == 'follow':
                Contact.objects.get_or_create(
                    user_from=request.user,
                    user_to=user
                )
            else:
                Contact.objects.filter(
                    user_from=request.user,
                    user_to=user
                ).delete()
            return JsonResponse({'status': 'ok'})
        except User.DoesNotExist:
            return JsonResponse({'status': 'error'})
    return JsonResponse({'status': 'error'})
```

La vista `user_follow` es bastante similar a la vista `image_like` que creaste en el Capítulo 6, *Compartir contenido en tu sitio web*. Dado que estás utilizando un modelo intermedio personalizado para la relación de muchos a muchos de los usuarios, los métodos predeterminados `add()` y `remove()` del gestor automático de `ManyToManyField` no están disponibles. En su lugar, se utiliza el modelo intermedio `Contact` para crear o eliminar relaciones entre usuarios.

Edita el archivo `urls.py` de la aplicación `account` y añade el siguiente patrón de URL:

```python
urlpatterns = [
    path('', include('django.contrib.auth.urls')),
    path('', views.dashboard, name='dashboard'),
    path('register/', views.register, name='register'),
    path('edit/', views.edit, name='edit'),
    path('users/', views.user_list, name='user_list'),
    path('users/follow/', views.user_follow, name='user_follow'),
    path('users/<username>/', views.user_detail, name='user_detail'),
]
```

Asegúrate de colocar el patrón anterior antes del patrón de URL `user_detail`. De lo contrario, cualquier petición a `/users/follow/` coincidirá con la expresión regular del patrón `user_detail` y esa vista se ejecutará en su lugar. Recuerda que en cada petición HTTP, Django comprueba la URL solicitada contra cada patrón en orden de aparición y se detiene en la primera coincidencia.

Edita la plantilla `user/detail.html` de la aplicación `account` y añade el siguiente código:

```html
{% block domready %}
    const url = '{% url "user_follow" %}';
    var options = {
        method: 'POST',
        headers: {'X-CSRFToken': csrftoken},
        mode: 'same-origin'
    }

    document.querySelector('a.follow')
            .addEventListener('click', function(e){
                e.preventDefault();
                var followButton = this;

                // add request body
                var formData = new FormData();
                formData.append('id', followButton.dataset.id);
                formData.append('action', followButton.dataset.action);
                options['body'] = formData;

                // send HTTP request
                fetch(url, options)
                .then(response => response.json())
                .then(data => {
                    if (data['status'] === 'ok') {
                        var previousAction = followButton.dataset.action;

                        // toggle button text and data-action
                        var action = previousAction === 'follow' ? 'unfollow' : 'follow';
                        followButton.dataset.action = action;
                        followButton.innerHTML = action;

                        // update follower count
                        var followerCount = document.querySelector('span.count .total');
                        var totalFollowers = parseInt(followerCount.innerHTML);
                        followerCount.innerHTML = previousAction === 'follow' ? totalFollowers + 1 : totalFollowers - 1;
                    }
                })
            });
{% endblock %}
```

El bloque de plantilla anterior contiene el código JavaScript para realizar la petición HTTP asíncrona para seguir o dejar de seguir a un usuario en particular y también para alternar el enlace follow/unfollow.

La API Fetch se utiliza para realizar la petición AJAX y establecer tanto el atributo `data-action` como el texto del elemento HTML `<a>` en función de su valor anterior. Cuando se completa la acción, también se actualiza el número total de seguidores que se muestra en la página.

Abre la página de detalle de un usuario existente y haz clic en el enlace **FOLLOW** para probar la funcionalidad que acabas de construir. Verás que el recuento de seguidores aumenta:

> *Figura 7.5: El recuento de seguidores y el botón follow/unfollow*

El sistema de seguimiento ahora está completo y los usuarios pueden seguirse unos a otros. A continuación, crearemos un flujo de actividad para generar contenido relevante para cada usuario en función de las personas a las que sigue.

---

### Creación de una aplicación de flujo de actividad

Muchos sitios web sociales muestran un flujo de actividad (*activity stream*) a sus usuarios para que puedan rastrear lo que hacen otros usuarios en la plataforma. Un flujo de actividad es una lista de actividades recientes realizadas por un usuario o un grupo de usuarios. Por ejemplo, el News Feed de Facebook es un flujo de actividad. Las acciones de ejemplo pueden ser *el usuario X guardó la imagen Y* o *el usuario X ahora sigue al usuario Y*.

Vas a construir una aplicación de flujo de actividad para que cada usuario pueda ver las interacciones recientes de los usuarios a los que sigue. Para hacerlo, necesitarás un modelo para guardar las acciones realizadas por los usuarios en el sitio web y una forma sencilla de añadir acciones al feed.

Crea una nueva aplicación llamada `actions` dentro de tu proyecto con el siguiente comando:

```bash
python manage.py startapp actions
```

Añade la nueva aplicación a `INSTALLED_APPS` en el archivo `settings.py` de tu proyecto para activarla:

```python
INSTALLED_APPS = [
    # ...
    'actions.apps.ActionsConfig',
]
```

Edita el archivo `models.py` de la aplicación `actions` y añade el siguiente código:

```python
from django.conf import settings
from django.db import models


class Action(models.Model):
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        related_name='actions',
        on_delete=models.CASCADE
    )
    verb = models.CharField(max_length=255)
    created = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=['-created']),
        ]
        ordering = ['-created']
```

El código anterior muestra el modelo `Action` que se utilizará para almacenar las actividades del usuario. Los campos de este modelo son los siguientes:

- `user`: El usuario que realizó la acción; este es un `ForeignKey` a `AUTH_USER_MODEL`, de forma predeterminada, el modelo `User` de Django.
- `verb`: El verbo que describe la acción que el usuario ha realizado.
- `created`: La fecha y hora en que se creó esta acción. Usamos `auto_now_add=True` para establecer esto automáticamente en la fecha y hora actual cuando el objeto se guarda por primera vez en la base de datos.

En la clase `Meta` del modelo, hemos definido un índice de base de datos en orden descendente para el campo `created`. También hemos añadido el atributo `ordering` para indicarle a Django que debe ordenar los resultados por el campo `created` en orden descendente de forma predeterminada.

Con este modelo básico, solo puedes almacenar acciones como *el usuario X hizo algo*. Necesitas un campo `ForeignKey` adicional para guardar acciones que involucran un objeto de destino (*target object*), como *el usuario X guardó la imagen Y* o *el usuario X ahora sigue al usuario Y*. Como ya sabes, un `ForeignKey` normal solo puede apuntar a un modelo. En su lugar, necesitarás una forma de que el objeto de destino de la acción sea una instancia de un modelo existente. Esto es lo que el framework `contenttypes` de Django te ayudará a hacer.

#### Uso del framework contenttypes

Django incluye un framework `contenttypes` ubicado en `django.contrib.contenttypes`. Esta aplicación puede rastrear todos los modelos instalados en tu proyecto y proporciona una interfaz genérica para interactuar con tus modelos.

La aplicación `django.contrib.contenttypes` se incluye en la configuración `INSTALLED_APPS` de forma predeterminada cuando creas un nuevo proyecto utilizando el comando `startproject`. Es utilizada por otros paquetes contrib, como el framework de autenticación y la aplicación de administración.

La aplicación `contenttypes` contiene un modelo `ContentType`. Las instancias de este modelo representan los modelos reales de tu aplicación, y las nuevas instancias de `ContentType` se crean automáticamente cuando se instalan nuevos modelos en tu proyecto. El modelo `ContentType` tiene los siguientes campos:

- `app_label`: Indica el nombre de la aplicación a la que pertenece el modelo. Esto se toma automáticamente del atributo `app_label` de las opciones `Meta` del modelo. Por ejemplo, tu modelo `Image` pertenece a la aplicación `images`.
- `model`: El nombre de la clase del modelo.
- `name`: Esta es una propiedad que indica el nombre legible por humanos del modelo, generado automáticamente a partir del atributo `verbose_name` de las opciones `Meta` del modelo.

Echemos un vistazo a cómo puedes interactuar con objetos `ContentType`. Abre el shell usando el siguiente comando:

```bash
python manage.py shell
```

Puedes obtener el objeto `ContentType` correspondiente a un modelo específico realizando una consulta con los atributos `app_label` y `model`, de la siguiente manera:

```text
>>> from django.contrib.contenttypes.models import ContentType
>>> image_type = ContentType.objects.get(app_label='images', model='image')
>>> image_type
<ContentType: images | image>
```

También puedes recuperar la clase de modelo de un objeto `ContentType` llamando a su método `model_class()`:

```text
>>> image_type.model_class()
<class 'images.models.Image'>
```

También es común obtener el objeto `ContentType` para una clase de modelo en particular, de la siguiente manera:

```text
>>> from images.models import Image
>>> ContentType.objects.get_for_model(Image)
<ContentType: images | image>
```

Estos son solo algunos ejemplos de uso de contenttypes. Puedes encontrar la documentación oficial para el framework `contenttypes` en [https://docs.djangoproject.com/en/5.2/ref/contrib/contenttypes/](https://docs.djangoproject.com/en/5.2/ref/contrib/contenttypes/).

#### Adición de relaciones genéricas a tus modelos

En las relaciones genéricas, los objetos `ContentType` desempeñan el papel de apuntar al modelo utilizado para la relación. Necesitarás tres campos para configurar una relación genérica en un modelo:

1. Un campo `ForeignKey` a `ContentType`: Esto te dirá cuál es el modelo para la relación.
2. Un campo para almacenar la clave primaria del objeto relacionado: Por lo general, será un `PositiveIntegerField` para que coincida con los campos de clave primaria automáticos de Django.
3. Un campo para definir y gestionar la relación genérica utilizando los dos campos anteriores: El framework `contenttypes` ofrece un campo `GenericForeignKey` para este propósito.

Edita el archivo `models.py` de la aplicación `actions` y añade el siguiente código:

```python
from django.conf import settings
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType
from django.db import models


class Action(models.Model):
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        related_name='actions',
        on_delete=models.CASCADE
    )
    verb = models.CharField(max_length=255)
    created = models.DateTimeField(auto_now_add=True)
    target_ct = models.ForeignKey(
        ContentType,
        blank=True,
        null=True,
        related_name='target_obj',
        on_delete=models.CASCADE
    )
    target_id = models.PositiveIntegerField(null=True, blank=True)
    target = GenericForeignKey('target_ct', 'target_id')

    class Meta:
        indexes = [
            models.Index(fields=['-created']),
            models.Index(fields=['target_ct', 'target_id']),
        ]
        ordering = ['-created']
```

Hemos añadido los siguientes campos al modelo `Action`:

- `target_ct`: Un campo `ForeignKey` que apunta al modelo `ContentType`.
- `target_id`: Un `PositiveIntegerField` para almacenar la clave primaria del objeto relacionado.
- `target`: Un campo `GenericForeignKey` al objeto relacionado basado en la combinación de los dos campos anteriores.

También hemos añadido un índice de varios campos que incluye los campos `target_ct` y `target_id`.

Django no crea campos `GenericForeignKey` en la base de datos. Los únicos campos que se asignan a campos de base de datos son `target_ct` y `target_id`. Ambos campos tienen atributos `blank=True` y `null=True` para que no se requiera un objeto de destino al guardar objetos `Action`.

Puedes hacer que tus aplicaciones sean más flexibles utilizando relaciones genéricas en lugar de claves foráneas convencionales. Las relaciones genéricas te permiten asociar modelos de manera no exclusiva, permitiendo que un solo modelo se relacione con múltiples otros modelos.

La Figura 7.6 muestra el modelo `Action`, incluida la relación con el modelo `ContentType`:

> *Figura 7.6: El modelo Action y el modelo ContentType*

Ejecuta el siguiente comando para crear las migraciones iniciales para esta aplicación:

```bash
python manage.py makemigrations actions
```

Deberías ver la siguiente salida:

```text
Migrations for 'actions':
  actions/migrations/0001_initial.py
    - Create model Action
    - Create index actions_act_created_64f10d_idx on field(s) -created of model action
    - Create index actions_act_target__f20513_idx on field(s) target_ct, target_id of model action
```

Luego, ejecuta el siguiente comando para sincronizar la aplicación con la base de datos:

```bash
python manage.py migrate
```

La salida del comando debería indicar que se han aplicado las nuevas migraciones:

```text
Applying actions.0001_initial... OK
```

Añadamos el modelo `Action` al sitio de administración. Edita el archivo `admin.py` de la aplicación `actions` y añade el siguiente código:

```python
from django.contrib import admin
from .models import Action


@admin.register(Action)
class ActionAdmin(admin.ModelAdmin):
    list_display = ['user', 'verb', 'target', 'created']
    list_filter = ['created']
    search_fields = ['verb']
```

Acabas de registrar el modelo `Action` en el sitio de administración.

Inicia el servidor de desarrollo con el siguiente comando:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/admin/actions/action/add/` en tu navegador. Deberías ver la página para crear un nuevo objeto `Action`, de la siguiente manera:

> *Figura 7.7: La página Add action en el sitio de administración de Django*

Como notarás, solo se muestran los campos `target_ct` y `target_id` que están asignados a campos reales de la base de datos. El campo `GenericForeignKey` no aparece en el formulario. El campo `target_ct` te permite seleccionar cualquiera de los modelos registrados de tu proyecto Django.

Crea un nuevo archivo dentro del directorio de la aplicación `actions` y nómbralo `utils.py`. Necesitas definir una función abreviada que te permita crear nuevos objetos `Action` de forma sencilla. Edita el nuevo archivo `utils.py` y añade el siguiente código:

```python
from django.contrib.contenttypes.models import ContentType
from .models import Action


def create_action(user, verb, target=None):
    action = Action(user=user, verb=verb, target=target)
    action.save()
```

La función `create_action()` te permite crear acciones que incluyen opcionalmente un objeto de destino. Puedes usar esta función en cualquier parte de tu código como un atajo para añadir nuevas acciones al flujo de actividad.

#### Evitar acciones duplicadas en el flujo de actividad

A veces, tus usuarios pueden hacer clic varias veces en el botón Like o Unlike o realizar la misma acción varias veces en un período corto de tiempo. Esto conducirá fácilmente a almacenar y mostrar acciones duplicadas. Para evitar esto, mejoremos la función `create_action()` para omitir acciones duplicadas obvias.

Edita el archivo `utils.py` de la aplicación `actions`, de la siguiente manera:

```python
import datetime
from django.contrib.contenttypes.models import ContentType
from django.utils import timezone
from .models import Action


def create_action(user, verb, target=None):
    # check for any similar action made in the last minute
    now = timezone.now()
    last_minute = now - datetime.timedelta(seconds=60)
    similar_actions = Action.objects.filter(
        user_id=user.id,
        verb=verb,
        created__gte=last_minute
    )
    if target:
        target_ct = ContentType.objects.get_for_model(target)
        similar_actions = similar_actions.filter(
            target_ct=target_ct,
            target_id=target.id
        )
    if not similar_actions:
        # no existing actions found
        action = Action(user=user, verb=verb, target=target)
        action.save()
        return True
    return False
```

Has modificado la función `create_action()` para evitar guardar acciones duplicadas y devolver un valor booleano que indica si la acción se guardó. Así es como evitas duplicados:

- Primero, obtienes la hora actual utilizando el método `timezone.now()` proporcionado por Django. Este método hace lo mismo que `datetime.datetime.now()` pero devuelve un objeto que tiene en cuenta la zona horaria (*timezone-aware*). Django proporciona una configuración llamada `USE_TZ` para habilitar o deshabilitar la compatibilidad con zonas horarias. El archivo `settings.py` predeterminado incluye `USE_TZ = True`.
- Utilizas la variable `last_minute` para almacenar la fecha y hora de hace un minuto y recuperar cualquier acción idéntica realizada por el usuario desde entonces.
- Creas un objeto `Action` si no existe ninguna acción idéntica en el último minuto. Devuelves `True` si se creó un objeto `Action` o `False` en caso contrario.

#### Adición de acciones de usuario al flujo de actividad

Es hora de añadir algunas acciones a tus vistas para construir el flujo de actividad para tus usuarios. Almacenarás una acción para cada una de las siguientes interacciones:

- Un usuario guarda una imagen como favorita
- A un usuario le gusta una imagen
- Un usuario crea una cuenta
- Un usuario comienza a seguir a otro usuario

Edita el archivo `views.py` de la aplicación `images` y añade la siguiente importación:

```python
from actions.utils import create_action
```

En la vista `image_create`, añade `create_action()` después de guardar la imagen, de la siguiente manera:

```python
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
            create_action(request.user, 'bookmarked image', new_image)
            messages.success(request, 'Image added successfully')
            # redirect to new created image detail view
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

En la vista `image_like`, añade `create_action()` después de añadir el usuario a la relación `users_like`, de la siguiente manera:

```python
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
                create_action(request.user, 'likes', image)
            else:
                image.users_like.remove(request.user)
            return JsonResponse({'status': 'ok'})
        except Image.DoesNotExist:
            pass
    return JsonResponse({'status': 'error'})
```

Ahora, edita el archivo `views.py` de la aplicación `account` y añade la siguiente importación:

```python
from actions.utils import create_action
```

En la vista `register`, añade `create_action()` después de crear el objeto `Profile`, de la siguiente manera:

```python
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
            create_action(new_user, 'has created an account')
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

En la vista `user_follow`, añade `create_action()`, de la siguiente manera:

```python
@require_POST
@login_required
def user_follow(request):
    user_id = request.POST.get('id')
    action = request.POST.get('action')
    if user_id and action:
        try:
            user = User.objects.get(id=user_id)
            if action == 'follow':
                Contact.objects.get_or_create(
                    user_from=request.user,
                    user_to=user
                )
                create_action(request.user, 'is following', user)
            else:
                Contact.objects.filter(
                    user_from=request.user,
                    user_to=user
                ).delete()
            return JsonResponse({'status': 'ok'})
        except User.DoesNotExist:
            return JsonResponse({'status': 'error'})
    return JsonResponse({'status': 'error'})
```

Como puedes ver en el código anterior, gracias al modelo `Action` y a la función auxiliar, es muy fácil guardar nuevas acciones en el flujo de actividad.

#### Visualización del flujo de actividad

Finalmente, necesitas una forma de mostrar el flujo de actividad para cada usuario. Incluirás el flujo de actividad en el panel de control del usuario. Edita el archivo `views.py` de la aplicación `account`. Importa el modelo `Action` y modifica la vista `dashboard`, de la siguiente manera:

```python
from actions.models import Action

# ...


@login_required
def dashboard(request):
    # Display all actions by default
    actions = Action.objects.exclude(user=request.user)
    following_ids = request.user.following.values_list(
        'id',
        flat=True
    )
    if following_ids:
        # If user is following others, retrieve only their actions
        actions = actions.filter(user_id__in=following_ids)
    actions = actions[:10]
    return render(
        request,
        'account/dashboard.html',
        {'section': 'dashboard', 'actions': actions}
    )
```

En la vista anterior, recuperas todas las acciones de la base de datos, excluyendo las realizadas por el usuario actual. De forma predeterminada, recuperas las últimas acciones realizadas por todos los usuarios de la plataforma.

Si el usuario sigue a otros usuarios, restringes la consulta para recuperar solo las acciones realizadas por los usuarios a los que sigue. Finalmente, limitas el resultado a las primeras 10 acciones devueltas. No utilizas `order_by()` en el QuerySet porque confías en el orden predeterminado que proporcionaste en las opciones `Meta` del modelo `Action`. Las acciones recientes aparecerán primero ya que estableciste `ordering = ['-created']` en el modelo `Action`.

#### Optimización de QuerySets que involucran objetos relacionados

Cada vez que recuperes un objeto `Action`, normalmente accederás a su objeto `User` relacionado y al objeto `Profile` relacionado del usuario. El ORM de Django ofrece una forma sencilla de recuperar objetos relacionados al mismo tiempo, evitando así consultas adicionales a la base de datos.

##### Uso de select_related()

Django ofrece un método de QuerySet llamado `select_related()` que te permite recuperar objetos relacionados para relaciones de uno a muchos. Esto se traduce en un único QuerySet más complejo, pero evitas consultas adicionales al acceder a los objetos relacionados.

El método `select_related` es para campos `ForeignKey` y `OneToOne`. Funciona realizando un `JOIN` de SQL e incluyendo los campos del objeto relacionado en la instrucción `SELECT`.

Para aprovechar `select_related()`, edita la vista `dashboard` en el archivo `views.py` de la aplicación `account`, de la siguiente manera:

```python
@login_required
def dashboard(request):
    # Display all actions by default
    actions = Action.objects.exclude(user=request.user)
    following_ids = request.user.following.values_list(
        'id',
        flat=True
    )
    if following_ids:
        # If user is following others, retrieve only their actions
        actions = actions.filter(user_id__in=following_ids)
    actions = actions.select_related(
        'user', 'user__profile'
    )[:10]
    return render(
        request,
        'account/dashboard.html',
        {'section': 'dashboard', 'actions': actions}
    )
```

Utilizas `user__profile` para unir la tabla `Profile` en una sola consulta SQL. Si llamas a `select_related()` sin pasar ningún argumento, recuperará objetos de todas las relaciones `ForeignKey`. Limita siempre `select_related()` a las relaciones a las que se accederá después.

> [!TIP]
> El uso cuidadoso de `select_related()` puede mejorar enormemente el tiempo de ejecución.

##### Uso de prefetch_related()

`select_related()` te ayudará a mejorar el rendimiento al recuperar objetos relacionados en relaciones de uno a muchos. Sin embargo, `select_related()` no funciona para relaciones de muchos a muchos o de muchos a uno (`ManyToMany` o campos `ForeignKey` inversos). Django ofrece un método de QuerySet diferente llamado `prefetch_related` que funciona para relaciones de muchos a muchos y de muchos a uno, además de las relaciones admitidas por `select_related()`. El método `prefetch_related()` realiza una búsqueda separada para cada relación y une los resultados utilizando Python. Este método también admite la precarga de `GenericRelation` y `GenericForeignKey`.

Edita el archivo `views.py` de la aplicación `account` y completa tu consulta añadiendo `prefetch_related()` para el campo `GenericForeignKey` `target`, de la siguiente manera:

```python
@login_required
def dashboard(request):
    # Display all actions by default
    actions = Action.objects.exclude(user=request.user)
    following_ids = request.user.following.values_list(
        'id',
        flat=True
    )
    if following_ids:
        # If user is following others, retrieve only their actions
        actions = actions.filter(user_id__in=following_ids)
    actions = actions.select_related(
        'user', 'user__profile'
    ).prefetch_related('target')[:10]
    return render(
        request,
        'account/dashboard.html',
        {'section': 'dashboard', 'actions': actions}
    )
```

Esta consulta ahora está optimizada para recuperar las acciones del usuario, incluidos los objetos relacionados.

#### Creación de plantillas para acciones

Ahora creemos la plantilla para mostrar un objeto `Action` en particular. Crea un nuevo directorio dentro del directorio de la aplicación `actions` y nómbralo `templates`. Añade la siguiente estructura de archivos:

```text
actions/
    action/
        detail.html
```

Edita el archivo de plantilla `actions/action/detail.html` y añade las siguientes líneas:

```html
{% load thumbnail %}

{% with user=action.user profile=action.user.profile %}
    <div class="action">
        <div class="images">
            {% if profile.photo %}
                {% thumbnail user.profile.photo "80x80" crop="100%" as im %}
                <a href="{{ user.get_absolute_url }}">
                    <img src="{{ im.url }}" alt="{{ user.get_full_name }}" class="item-img">
                </a>
            {% endif %}
            {% if action.target %}
                {% with target=action.target %}
                    {% if target.image %}
                        {% thumbnail target.image "80x80" crop="100%" as im %}
                        <a href="{{ target.get_absolute_url }}">
                            <img src="{{ im.url }}" class="item-img">
                        </a>
                    {% endif %}
                {% endwith %}
            {% endif %}
        </div>
        <div class="info">
            <p>
                <span class="date">{{ action.created|timesince }} ago</span>
                <br />
                <a href="{{ user.get_absolute_url }}">
                    {{ user.first_name }}
                </a>
                {{ action.verb }}
                {% if action.target %}
                    {% with target=action.target %}
                        <a href="{{ target.get_absolute_url }}">{{ target }}</a>
                    {% endwith %}
                {% endif %}
            </p>
        </div>
    </div>
{% endwith %}
```

Esta es la plantilla utilizada para mostrar un objeto `Action`. Primero, utilizas la etiqueta de plantilla `{% with %}` para recuperar el usuario que realiza la acción y el objeto `Profile` relacionado. Luego, muestras la imagen del objeto de destino si el objeto `Action` tiene un objeto de destino relacionado. Finalmente, muestras el enlace al usuario que realizó la acción, el verbo y el objeto de destino, si existe.

Edita la plantilla `account/dashboard.html` de la aplicación `account` y añade el siguiente código en la parte inferior del bloque `content`:

```html
{% extends "base.html" %}

{% block title %}Dashboard{% endblock %}

{% block content %}
    ...
    <h2>What's happening</h2>
    <div id="action-list">
        {% for action in actions %}
            {% include "actions/action/detail.html" %}
        {% endfor %}
    </div>
{% endblock %}
```

Abre `http://127.0.0.1:8000/account/` en tu navegador. Inicia sesión como un usuario existente y realiza varias acciones para que se almacenen en la base de datos. Luego, inicia sesión con otro usuario, sigue al usuario anterior y echa un vistazo al flujo de acciones generado en la página del panel de control.

Debería verse de la siguiente manera:

> *Figura 7.8: El flujo de actividad para el usuario actual (Atribuciones de imágenes: Motor de inducción de Tesla por Ctac, CC BY-SA 3.0; Modelo de máquina de Turing Davey 2012 por Rocky Acosta, CC BY 3.0; Chick Corea por ataelw, CC BY 2.0)*

Acabas de crear un flujo de actividad completo para tus usuarios y puedes añadirle fácilmente nuevas acciones de usuario. También puedes añadir la funcionalidad de desplazamiento infinito al flujo de actividad implementando el mismo paginador AJAX que utilizaste para la vista `image_list`. A continuación, aprenderás a usar las señales de Django para desnormalizar los conteos de acciones.

---

### Uso de señales para desnormalizar conteos

Hay algunos casos en los que es posible que desees desnormalizar tus datos. La desnormalización consiste en hacer que los datos sean redundantes de tal manera que se optimice el rendimiento de lectura. Por ejemplo, podrías estar copiando datos relacionados a un objeto para evitar costosas consultas de lectura a la base de datos al recuperar los datos relacionados. Debes tener cuidado con la desnormalización y solo comenzar a usarla cuando realmente la necesites. El mayor problema que encontrarás con la desnormalización es que es difícil mantener actualizados los datos desnormalizados.

Echemos un vistazo a un ejemplo de cómo mejorar tus consultas desnormalizando conteos. Desnormalizarás los datos de tu modelo `Image` y utilizarás señales de Django para mantener los datos actualizados.

#### Trabajo con señales

Django viene con un despachador de señales (*signal dispatcher*) que permite que las funciones receptoras (*receivers*) reciban notificaciones cuando ocurren ciertas acciones. Las señales son muy útiles cuando necesitas que tu código haga algo cada vez que sucede otra cosa. Las señales te permiten desacoplar la lógica: puedes capturar una determinada acción, independientemente de la aplicación o código que activó esa acción, e implementar la lógica que se ejecuta siempre que ocurre esa acción. Por ejemplo, puedes construir una función receptora de señales que se ejecute cada vez que se guarde un objeto `User`. También puedes crear tus propias señales para que otros puedan recibir notificaciones cuando ocurra un evento.

Django proporciona varias señales para modelos ubicadas en `django.db.models.signals`. Algunas de estas señales son:

- `pre_save` y `post_save`: Se envían antes o después de llamar al método `save()` de un modelo.
- `pre_delete` y `post_delete`: Se envían antes o después de llamar al método `delete()` de un modelo o QuerySet.
- `m2m_changed`: Se envía cuando se modifica un `ManyToManyField` en un modelo.

Estos son solo un subconjunto de las señales proporcionadas por Django. Puedes encontrar una lista de todas las señales integradas en [https://docs.djangoproject.com/en/5.2/ref/signals/](https://docs.djangoproject.com/en/5.2/ref/signals/).

Supongamos que deseas recuperar imágenes por popularidad. Puedes usar las funciones de agregación de Django para recuperar imágenes ordenadas por la cantidad de usuarios a los que les gustan. Recuerda que utilizaste funciones de agregación de Django en el Capítulo 3, *Extensión de tu aplicación de blog*. El siguiente ejemplo de código recuperará imágenes según su cantidad de "me gusta":

```python
from django.db.models import Count
from images.models import Image

images_by_popularity = Image.objects.annotate(
    total_likes=Count('users_like')
).order_by('-total_likes')
```

Sin embargo, ordenar imágenes contando el total de sus "me gusta" es más costoso en términos de rendimiento que ordenarlas por un campo que almacene los conteos totales. Puedes añadir un campo al modelo `Image` para desnormalizar el número total de "me gusta" y mejorar el rendimiento en las consultas que involucran este campo. El problema es cómo mantener este campo actualizado.

Edita el archivo `models.py` de la aplicación `images` y añade el siguiente campo `total_likes` al modelo `Image`:

```python
class Image(models.Model):
    # ...
    total_likes = models.PositiveIntegerField(default=0)

    class Meta:
        indexes = [
            models.Index(fields=['-created']),
            models.Index(fields=['-total_likes']),
        ]
        ordering = ['-created']
```

El campo `total_likes` te permitirá almacenar el recuento total de usuarios a los que les gusta cada imagen. Desnormalizar conteos es útil cuando deseas filtrar u ordenar QuerySets por ellos. Hemos añadido un índice de base de datos para el campo `total_likes` en orden descendente porque planeamos recuperar imágenes ordenadas por sus "me gusta" totales en orden descendente.

> [!NOTE]
> Hay varias formas de mejorar el rendimiento que debes tener en cuenta antes de desnormalizar campos. Considera los índices de base de datos, la optimización de consultas y el almacenamiento en caché antes de comenzar a desnormalizar tus datos.

Ejecuta el siguiente comando para crear las migraciones para añadir el nuevo campo a la tabla de la base de datos:

```bash
python manage.py makemigrations images
```

Deberías ver la siguiente salida:

```text
Migrations for 'images':
  images/migrations/0002_image_total_likes_and_more.py
    - Add field total_likes to image
    - Create index images_imag_total_l_0bcd7e_idx on field(s) -total_likes of model image
```

Luego, ejecuta el siguiente comando para aplicar la migración:

```bash
python manage.py migrate images
```

La salida debería incluir la siguiente línea:

```text
Applying images.0002_image_total_likes_and_more... OK
```

Necesitas adjuntar una función receptora a la señal `m2m_changed`.

Crea un nuevo archivo dentro del directorio de la aplicación `images` y nómbralo `signals.py`. Añádele el siguiente código:

```python
from django.db.models.signals import m2m_changed
from django.dispatch import receiver
from .models import Image


@receiver(m2m_changed, sender=Image.users_like.through)
def users_like_changed(sender, instance, **kwargs):
    instance.total_likes = instance.users_like.count()
    instance.save()
```

Primero, registras la función `users_like_changed` como una función receptora utilizando el decorador `receiver()`. La adjuntas a la señal `m2m_changed`. Luego, conectas la función a `Image.users_like.through` para que la función solo se llame si este emisor ha lanzado la señal `m2m_changed`. Hay un método alternativo para registrar una función receptora; consiste en utilizar el método `connect()` del objeto `Signal`.

> [!NOTE]
> Las señales de Django son síncronas y bloqueantes. No confundas señales con tareas asíncronas. Sin embargo, puedes combinar ambas para lanzar tareas asíncronas cuando tu código recibe una notificación de una señal. Aprenderás a crear tareas asíncronas con Celery en el Capítulo 8, *Creación de una tienda online*.

Debes conectar tu función receptora a una señal para que se llame cada vez que se envíe la señal. El método recomendado para registrar tus señales es importándolas en el método `ready()` de tu clase de configuración de la aplicación. Django proporciona un registro de aplicaciones que te permite configurar e introspeccionar tus aplicaciones.

##### Clases de configuración de aplicaciones

Django te permite especificar clases de configuración para tus aplicaciones. Cuando creas una aplicación utilizando el comando `startapp`, Django añade un archivo `apps.py` al directorio de la aplicación, incluyendo una configuración básica de la aplicación que hereda de la clase `AppConfig`.

La clase de configuración de la aplicación te permite almacenar metadatos y la configuración de la aplicación, y proporciona introspección para la aplicación. Puedes encontrar más información sobre las configuraciones de aplicaciones en [https://docs.djangoproject.com/en/5.2/ref/applications/](https://docs.djangoproject.com/en/5.2/ref/applications/).

Para registrar tus funciones receptoras de señales, cuando usas el decorador `receiver()`, solo necesitas importar el módulo `signals` de tu aplicación dentro del método `ready()` de la clase de configuración de la aplicación. Este método se llama tan pronto como el registro de la aplicación está completamente poblado. Cualquier otra inicialización para tu aplicación también debe incluirse en este método.

Edita el archivo `apps.py` de la aplicación `images` y añade el siguiente código:

```python
from django.apps import AppConfig


class ImagesConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'images'

    def ready(self):
        # import signal handlers
        import images.signals
```

Importas las señales para esta aplicación en el método `ready()` para que se importen cuando se cargue la aplicación `images`.

Ejecuta el servidor de desarrollo con el siguiente comando:

```bash
python manage.py runserver
```

Abre tu navegador para ver la página de detalle de una imagen y haz clic en el botón Like.

Ve al sitio de administración, navega a la URL de edición de la imagen (por ejemplo, `http://127.0.0.1:8000/admin/images/image/1/change/`) y echa un vistazo al atributo `total_likes`. Deberías ver que el atributo `total_likes` se actualiza con el número total de usuarios a los que les gusta la imagen:

> *Figura 7.9: La página de edición de imagen en el sitio de administración, incluida la desnormalización para el total de likes*

Ahora, puedes usar el atributo `total_likes` para ordenar las imágenes por popularidad o mostrar el valor en cualquier lugar, evitando usar consultas complejas para calcularlo.

Considera la siguiente consulta para obtener imágenes ordenadas por su recuento de "me gusta" en orden descendente:

```python
from django.db.models import Count
images_by_popularity = Image.objects.annotate(
    likes=Count('users_like')
).order_by('-likes')
```

La consulta anterior ahora se puede escribir de la siguiente manera:

```python
images_by_popularity = Image.objects.order_by('-total_likes')
```

Esto da como resultado una consulta SQL menos costosa gracias a la desnormalización del total de "me gusta" para las imágenes. También has aprendido a utilizar las señales de Django.

> [!WARNING]
> Usa las señales con precaución ya que dificultan el seguimiento del flujo de control. En muchos casos, puedes evitar el uso de señales si sabes qué receptores necesitan ser notificados.

Deberás establecer recuentos iniciales para el resto de los objetos `Image` para que coincidan con el estado actual de la base de datos.

Abre el shell con el siguiente comando:

```bash
python manage.py shell
```

Ejecuta el siguiente código en el shell:

```text
>>> from images.models import Image
>>> for image in Image.objects.all():
...     image.total_likes = image.users_like.count()
...     image.save()
```

Has actualizado manualmente el recuento de "me gusta" para las imágenes existentes en la base de datos. A partir de ahora, la función receptora de señales `users_like_changed` se encargará de actualizar el campo `total_likes` siempre que cambien los objetos relacionados de muchos a muchos.

A continuación, aprenderás a usar Django Debug Toolbar para obtener información relevante de depuración para las peticiones, incluido el tiempo de ejecución, las consultas SQL ejecutadas, las plantillas renderizadas, las señales registradas y mucho más.

---

### Uso de Django Debug Toolbar

Llegados a este punto, ya estarás familiarizado con la página de depuración de Django. A lo largo de los capítulos anteriores, has visto la distintiva página de depuración amarilla y gris de Django varias veces.

La página de depuración de Django proporciona información de depuración útil. Sin embargo, existe una aplicación de Django que incluye información de depuración más detallada y puede ser realmente útil durante el desarrollo.

Django Debug Toolbar es una aplicación externa de Django que te permite ver información de depuración relevante sobre el ciclo de petición/respuesta actual. La información se divide en múltiples paneles que muestran diferentes datos, incluidos datos de petición/respuesta, versiones de paquetes de Python utilizados, tiempo de ejecución, configuraciones, encabezados, consultas SQL, plantillas utilizadas, caché, señales y registro de eventos (*logging*).

Puedes encontrar la documentación de Django Debug Toolbar en [https://django-debug-toolbar.readthedocs.io/](https://django-debug-toolbar.readthedocs.io/).

#### Instalación de Django Debug Toolbar

Instala `django-debug-toolbar` mediante pip usando el siguiente comando:

```bash
python -m pip install django-debug-toolbar==5.1.0
```

Edita el archivo `settings.py` de tu proyecto y añade `debug_toolbar` a la configuración `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    # ...
    'debug_toolbar',
    # ...
]
```

En el mismo archivo, añade la siguiente línea a la configuración `MIDDLEWARE`:

```python
MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

Django Debug Toolbar se implementa principalmente como middleware. El orden de `MIDDLEWARE` es importante. `DebugToolbarMiddleware` debe colocarse antes de cualquier otro middleware, excepto los middleware que codifican el contenido de la respuesta, como `GZipMiddleware`, que, si está presente, debe ir primero.

Añade las siguientes líneas al final del archivo `settings.py`:

```python
INTERNAL_IPS = [
    '127.0.0.1',
]
```

Django Debug Toolbar solo se mostrará si tu dirección IP coincide con una entrada en la configuración `INTERNAL_IPS`. Para evitar mostrar información de depuración en producción, Django Debug Toolbar comprueba que la configuración `DEBUG` sea `True`.

Edita el archivo `urls.py` principal de tu proyecto y añade el siguiente patrón de URL a `urlpatterns`:

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('account/', include('account.urls')),
    path(
        'social-auth/',
        include('social_django.urls', namespace='social')
    ),
    path('images/', include('images.urls', namespace='images')),
    path('__debug__/', include('debug_toolbar.urls')),
]
```

Django Debug Toolbar ahora está instalado en tu proyecto. ¡Probémoslo!

Ejecuta el servidor de desarrollo con el siguiente comando:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/images/` con tu navegador. Ahora deberías ver una barra lateral plegable a la derecha:

> *Figura 7.10: La barra lateral de Django Debug Toolbar (Atribuciones de imágenes: Chick Corea por ataelw, CC BY 2.0; Al Jarreau Düsseldorf 1981 por Eddi Laumanns aka RX-Guru, CC BY 3.0; Al Jarreau por Kingkongphoto y celebrity-photos.com, CC BY-SA 2.0)*

Si la barra de herramientas de depuración no aparece, comprueba el registro de la consola del shell de `RunServer`. Si ves un error de tipo MIME, lo más probable es que tus archivos de mapeo MIME sean incorrectos o deban actualizarse. Puedes aplicar el mapeo correcto para archivos JavaScript y CSS añadiendo las siguientes líneas al archivo `settings.py`:

```python
if DEBUG:
    import mimetypes
    mimetypes.add_type('application/javascript', '.js', True)
    mimetypes.add_type('text/css', '.css', True)
```

#### Paneles de Django Debug Toolbar

Django Debug Toolbar cuenta con múltiples paneles que organizan la información de depuración para el ciclo de petición/respuesta. La barra lateral contiene enlaces a cada panel y puedes usar la casilla de verificación de cualquier panel para activarlo o desactivarlo. El cambio se aplicará a la siguiente petición. Esto es útil cuando no estamos interesados en un panel específico, pero su cálculo añade demasiada sobrecarga a la petición.

- **Time**: El panel Time incluye un temporizador para las diferentes fases del ciclo de petición/respuesta. También muestra la CPU, el tiempo transcurrido y la cantidad de cambios de contexto.
  > *Figura 7.11: Panel Time – Django Debug Toolbar*
- **SQL**: Aquí puedes ver las diferentes consultas SQL que se han ejecutado. Esta información puede ayudarte a identificar consultas innecesarias, consultas duplicadas que se pueden reutilizar o consultas de larga duración que se pueden optimizar.
  > *Figura 7.12: Panel SQL – Django Debug Toolbar*
- **Templates**: Este panel muestra las diferentes plantillas utilizadas al renderizar el contenido, las rutas de las plantillas y el contexto utilizado. También puedes ver los diferentes procesadores de contexto utilizados.
  > *Figura 7.13: Panel Templates – Django Debug Toolbar*
- **Signals**: En este panel, puedes ver todas las señales registradas en tu proyecto y las funciones receptoras adjuntas a cada señal.
  > *Figura 7.14: Panel Signals – Django Debug Toolbar*

Además de los paneles integrados, puedes encontrar paneles de terceros adicionales que puedes descargar y usar en [https://django-debug-toolbar.readthedocs.io/en/latest/panels.html#third-party-panels](https://django-debug-toolbar.readthedocs.io/en/latest/panels.html#third-party-panels).

#### Comandos de Django Debug Toolbar

Además de los paneles de depuración de petición/respuesta, Django Debug Toolbar proporciona un comando de gestión para depurar SQL para llamadas de ORM. El comando de gestión `debugsqlshell` replica el comando shell de Django pero muestra instrucciones SQL para consultas realizadas con el ORM de Django.

Abre el shell con el siguiente comando:

```bash
python manage.py debugsqlshell
```

Ejecuta el siguiente código:

```text
>>> from images.models import Image
>>> Image.objects.get(id=1)
```

Verás la siguiente salida:

```text
SELECT "images_image"."id",
       "images_image"."user_id",
       "images_image"."title",
       "images_image"."slug",
       "images_image"."url",
       "images_image"."image",
       "images_image"."description",
       "images_image"."created",
       "images_image"."total_likes"
FROM "images_image"
WHERE "images_image"."id" = 1
LIMIT 21 [0.44ms]
<Image: Django and Duke>
```

Puedes usar este comando para probar consultas ORM antes de añadirlas a tus vistas. Puedes comprobar la instrucción SQL resultante y el tiempo de ejecución de cada llamada al ORM.

En la siguiente sección, aprenderás a contar visualizaciones de imágenes usando Redis, una base de datos en memoria que proporciona baja latencia y acceso a datos de alto rendimiento.

---

### Conteo de visualizaciones de imágenes con Redis

Redis es una base de datos avanzada de clave/valor que te permite guardar diferentes tipos de datos. También tiene operaciones de E/S extremadamente rápidas. Redis almacena todo en memoria, pero los datos se pueden persistir volcando el conjunto de datos en el disco de vez en cuando o añadiendo cada comando a un registro. Redis es muy versátil en comparación con otros almacenes de clave/valor: proporciona un conjunto de comandos potentes y admite diversas estructuras de datos, como cadenas, hashes, listas, conjuntos, conjuntos ordenados e incluso métodos de mapa de bits o HyperLogLog.

Aunque SQL es más adecuado para el almacenamiento de datos persistentes definidos por esquemas, Redis ofrece numerosas ventajas cuando se trata de datos que cambian rápidamente, almacenamiento volátil o cuando se necesita una memoria caché rápida. Echemos un vistazo a cómo se puede usar Redis para construir nuevas funcionalidades en tu proyecto.

Puedes encontrar más información sobre Redis en su página de inicio en [https://redis.io/](https://redis.io/).

Redis proporciona una imagen de Docker que hace que sea muy fácil desplegar un servidor Redis con una configuración estándar.

#### Instalación de Redis

Para instalar la imagen de Docker de Redis, asegúrate de que Docker esté instalado en tu máquina. Aprendiste a instalar Docker en el Capítulo 3, *Extensión de tu aplicación de blog*.

Ejecuta el siguiente comando desde la consola:

```bash
docker pull redis:7.2.4
```

Esto descargará la imagen de Docker de Redis a tu máquina local. Puedes encontrar información sobre la imagen oficial de Docker de Redis en [https://hub.docker.com/_/redis](https://hub.docker.com/_/redis). Puedes encontrar otros métodos alternativos para instalar Redis en [https://redis.io/download/](https://redis.io/download/).

Ejecuta el siguiente comando en la consola para iniciar el contenedor Docker de Redis:

```bash
docker run -it --rm --name redis -p 6379:6379 redis:7.2.4
```

Con este comando, ejecutamos Redis en un contenedor Docker. La opción `-it` le indica a Docker que te lleve directamente dentro del contenedor para entrada interactiva. La opción `--rm` le dice a Docker que limpie automáticamente el contenedor y elimine el sistema de archivos cuando el contenedor se detenga. La opción `--name` se utiliza para asignar un nombre al contenedor. La opción `-p` se utiliza para publicar el puerto 6379 en el que se ejecuta Redis en el mismo puerto de interfaz del host. 6379 es el puerto predeterminado para Redis.

Deberías ver una salida que finaliza con las siguientes líneas:

```text
# Server initialized
* Ready to accept connections
```

Mantén el servidor Redis ejecutándose en el puerto 6379 y abre otra consola. Inicia el cliente Redis con el siguiente comando:

```bash
docker exec -it redis sh
```

Verás una línea con el símbolo de almohadilla:

```text
#
```

Inicia el cliente Redis con el siguiente comando:

```bash
redis-cli
```

Verás el indicador de la consola del cliente Redis, así:

```text
127.0.0.1:6379>
```

El cliente Redis te permite ejecutar comandos de Redis directamente desde la consola. Probemos algunos comandos. Introduce el comando `SET` en la consola de Redis para almacenar un valor en una clave:

```text
127.0.0.1:6379> SET name "Peter"
OK
```

El comando anterior crea una clave `name` con el valor de cadena `"Peter"` en la base de datos de Redis. La salida `OK` indica que la clave se ha guardado correctamente.

A continuación, recupera el valor utilizando el comando `GET`, de la siguiente manera:

```text
127.0.0.1:6379> GET name
"Peter"
```

También puedes comprobar si una clave existe utilizando el comando `EXISTS`. Este comando devuelve 1 si la clave dada existe y 0 en caso contrario:

```text
127.0.0.1:6379> EXISTS name
(integer) 1
```

Puedes establecer el tiempo para que expire una clave usando el comando `EXPIRE`, que te permite establecer el tiempo de vida en segundos. Otra opción es utilizar el comando `EXPIREAT`, que espera una marca de tiempo Unix. La caducidad de claves es útil para usar Redis como caché o para almacenar datos volátiles:

```text
127.0.0.1:6379> GET name
"Peter"
127.0.0.1:6379> EXPIRE name 2
(integer) 1
```

Espera más de dos segundos e intenta obtener la misma clave de nuevo:

```text
127.0.0.1:6379> GET name
(nil)
```

La respuesta `(nil)` es una respuesta nula y significa que no se ha encontrado ninguna clave. También puedes eliminar cualquier clave usando el comando `DEL`, de la siguiente manera:

```text
127.0.0.1:6379> SET total 1
OK
127.0.0.1:6379> DEL total
(integer) 1
127.0.0.1:6379> GET total
(nil)
```

Estos son solo comandos básicos para operaciones con claves. Puedes encontrar todos los comandos de Redis en [https://redis.io/commands/](https://redis.io/commands/) y todos los tipos de datos de Redis en [https://redis.io/docs/manual/data-types/](https://redis.io/docs/manual/data-types/).

#### Uso de Redis con Python

Necesitarás enlaces de Python para Redis. Instala `redis-py` a través de pip usando el siguiente comando:

```bash
python -m pip install redis==5.2.1
```

Puedes encontrar la documentación de `redis-py` en [https://redis-py.readthedocs.io/](https://redis-py.readthedocs.io/).

El paquete `redis-py` interactúa con Redis, proporcionando una interfaz de Python que sigue la sintaxis de los comandos de Redis. Abre el shell de Python con el siguiente comando:

```bash
python manage.py shell
```

Ejecuta el siguiente código:

```python
import redis
r = redis.Redis(host='localhost', port=6379, db=0)
```

El código anterior crea una conexión con la base de datos de Redis. En Redis, las bases de datos se identifican mediante un índice entero en lugar de un nombre de base de datos. De forma predeterminada, un cliente está conectado a la base de datos 0. El número de bases de datos de Redis disponibles se establece en 16, pero puedes cambiarlo en el archivo de configuración `redis.conf`.

A continuación, establece una clave utilizando el shell de Python:

```python
r.set('foo', 'bar')
```

El comando devuelve `True`, lo que indica que la clave se ha creado correctamente. Ahora puedes recuperar la clave usando el comando `get()`:

```python
r.get('foo')
```

Devolverá `b'bar'`. Como notarás en el código anterior, los métodos de Redis siguen la sintaxis de los comandos de Redis.

Integremos Redis en tu proyecto. Edita el archivo `settings.py` del proyecto `bookmarks` y añade las siguientes configuraciones:

```python
REDIS_HOST = 'localhost'
REDIS_PORT = 6379
REDIS_DB = 0
```

Estas son las configuraciones para el servidor Redis y la base de datos que utilizarás para tu proyecto.

#### Almacenamiento de vistas de imágenes en Redis

Busquemos una forma de almacenar el número total de veces que se ha visto una imagen. Si implementas esto utilizando el ORM de Django, implicará una consulta SQL `UPDATE` cada vez que se muestre una imagen.

Si usas Redis en su lugar, solo necesitas incrementar un contador almacenado en memoria, lo que resulta en un rendimiento mucho mejor y menos sobrecarga.

Edita el archivo `views.py` de la aplicación `images` y añade el siguiente código después de las sentencias de importación existentes:

```python
import redis
from django.conf import settings

# connect to redis
r = redis.Redis(
    host=settings.REDIS_HOST,
    port=settings.REDIS_PORT,
    db=settings.REDIS_DB
)
```

Con el código anterior, estableces la conexión con Redis para usarla en tus vistas. Edita el archivo `views.py` de la aplicación `images` y modifica la vista `image_detail`, de la siguiente manera:

```python
def image_detail(request, id, slug):
    image = get_object_or_404(Image, id=id, slug=slug)
    # increment total image views by 1
    total_views = r.incr(f'image:{image.id}:views')
    return render(
        request,
        'images/image/detail.html',
        {
            'section': 'images',
            'image': image,
            'total_views': total_views
        }
    )
```

En esta vista, utilizas el comando `incr`, que incrementa el valor de una clave dada en 1. Si la clave no existe, el comando `incr` la crea. El método `incr()` devuelve el valor final de la clave después de realizar la operación. Almacenas el valor en la variable `total_views` y lo pasas al contexto de la plantilla. Construyes la clave de Redis utilizando una notación como `object-type:id:field` (por ejemplo, `image:33:views`).

> [!TIP]
> La convención para nombrar claves de Redis es usar el signo de dos puntos como separador para crear claves con espacio de nombres (*namespaced keys*). Al hacerlo, los nombres de las claves son especialmente detallados y las claves relacionadas comparten parte del mismo esquema en sus nombres.

Edita la plantilla `images/image/detail.html` de la aplicación `images` y añade el siguiente código:

```html
<div class="image-info">
    <div>
        <span class="count">
            <span class="total">{{ total_likes }}</span>
            like{{ total_likes|pluralize }}
        </span>
        <span class="count">
            {{ total_views }} view{{ total_views|pluralize }}
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
```

Ejecuta el servidor de desarrollo con el siguiente comando:

```bash
python manage.py runserver
```

Abre la página de detalle de una imagen en tu navegador y recárgala varias veces. Verás que cada vez que se procesa la vista, el total de vistas mostrado se incrementa en 1:

> *Figura 7.15: La página de detalle de imagen, incluyendo el recuento de likes y vistas*

¡Genial! Has integrado con éxito Redis en tu proyecto para contar las vistas de imágenes. En la siguiente sección, aprenderás a construir una clasificación de las imágenes más vistas con Redis.

#### Almacenamiento de un ranking en Redis

Ahora crearemos algo más complejo con Redis. Usaremos Redis para almacenar un ranking de las imágenes más vistas en la plataforma. Usaremos conjuntos ordenados (*sorted sets*) de Redis para esto. Un conjunto ordenado es una colección no repetitiva de cadenas en la que cada miembro está asociado con una puntuación (*score*). Los elementos se ordenan por su puntuación.

Edita el archivo `views.py` de la aplicación `images` y añade el siguiente código a la vista `image_detail`:

```python
def image_detail(request, id, slug):
    image = get_object_or_404(Image, id=id, slug=slug)
    # increment total image views by 1
    total_views = r.incr(f'image:{image.id}:views')
    # increment image ranking by 1
    r.zincrby('image_ranking', 1, image.id)
    return render(
        request,
        'images/image/detail.html',
        {
            'section': 'images',
            'image': image,
            'total_views': total_views
        }
    )
```

Utilizas el comando `zincrby()` para almacenar visualizaciones de imágenes en un conjunto ordenado con la clave `image_ranking`. Almacenarás el `id` de la imagen y una puntuación relacionada de 1, que se sumará a la puntuación total de este elemento en el conjunto ordenado. Esto te permitirá realizar un seguimiento de todas las vistas de imágenes globalmente y tener un conjunto ordenado clasificado por el número total de visualizaciones.

Ahora, crea una nueva vista para mostrar la clasificación de las imágenes más vistas. Añade el siguiente código al archivo `views.py` de la aplicación `images`:

```python
@login_required
def image_ranking(request):
    # get image ranking dictionary
    image_ranking = r.zrange(
        'image_ranking', 0, -1,
        desc=True
    )[:10]
    image_ranking_ids = [int(id) for id in image_ranking]
    # get most viewed images
    most_viewed = list(
        Image.objects.filter(
            id__in=image_ranking_ids
        )
    )
    most_viewed.sort(key=lambda x: image_ranking_ids.index(x.id))
    return render(
        request,
        'images/image/ranking.html',
        {'section': 'images', 'most_viewed': most_viewed}
    )
```

La vista `image_ranking` funciona de la siguiente manera:

- Utilizas el comando `zrange()` para obtener los elementos en el conjunto ordenado. Este comando espera un rango personalizado según las puntuaciones más bajas y más altas. Al usar 0 como la puntuación más baja y -1 como la más alta, le estás diciendo a Redis que devuelva todos los elementos del conjunto ordenado. También especificas `desc=True` para recuperar los elementos ordenados por puntuación descendente. Finalmente, divides los resultados usando `[:10]` para obtener los primeros 10 elementos con la puntuación más alta.
- Construyes una lista de IDs de imagen devueltos y la almacenas en la variable `image_ranking_ids` como una lista de enteros. Recuperas los objetos `Image` para esos IDs y fuerzas la ejecución de la consulta utilizando la función `list()`. Es importante forzar la ejecución de QuerySet porque utilizarás el método `sort()` en él (en este punto, necesitas una lista de objetos en lugar de un QuerySet).
- Ordenas los objetos `Image` por su índice de aparición en el ranking de imágenes. Ahora puedes usar la lista `most_viewed` en tu plantilla para mostrar las 10 imágenes más vistas.

Crea una nueva plantilla `ranking.html` dentro del directorio de plantillas `images/image/` de la aplicación `images` y añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Images ranking{% endblock %}

{% block content %}
    <h1>Images ranking</h1>
    <ol>
        {% for image in most_viewed %}
            <li>
                <a href="{{ image.get_absolute_url }}">
                    {{ image.title }}
                </a>
            </li>
        {% endfor %}
    </ol>
{% endblock %}
```

La plantilla es bastante sencilla. Iteras sobre los objetos `Image` contenidos en la lista `most_viewed` y muestras sus nombres, incluyendo un enlace a la página de detalle de la imagen.

Finalmente, necesitas crear un patrón de URL para la nueva vista. Edita el archivo `urls.py` de la aplicación `images` y añade el siguiente patrón de URL:

```python
urlpatterns = [
    path('create/', views.image_create, name='create'),
    path('detail/<int:id>/<slug:slug>/', views.image_detail, name='detail'),
    path('like/', views.image_like, name='like'),
    path('', views.image_list, name='list'),
    path('ranking/', views.image_ranking, name='ranking'),
]
```

Ejecuta el servidor de desarrollo, accede a tu sitio en tu navegador web y carga la página de detalle de la imagen varias veces para diferentes imágenes. Luego, accede a `http://127.0.0.1:8000/images/ranking/` desde tu navegador. Deberías poder ver un ranking de imágenes, de la siguiente manera:

> *Figura 7.16: La página de ranking construida con datos recuperados de Redis*

¡Genial! Acabas de crear un ranking con Redis.

#### Próximos pasos con Redis

Redis no reemplaza tu base de datos SQL, pero ofrece un almacenamiento rápido en memoria que es más adecuado para ciertas tareas. Añádelo a tu infraestructura tecnológica y úsalo cuando realmente sientas que es necesario. Los siguientes son algunos escenarios en los que Redis podría ser útil:

- **Conteo:** Como has visto, es muy fácil gestionar contadores con Redis. Puedes usar `incr()` e `incrby()` para contar elementos.
- **Almacenamiento de los elementos más recientes:** Puedes añadir elementos al inicio o final de una lista usando `lpush()` y `rpush()`. Elimina y devuelve el primer o último elemento usando `lpop()`/`rpop()`. Puedes recortar la longitud de la lista usando `ltrim()` para mantener su tamaño.
- **Colas:** Además de los comandos push y pop, Redis ofrece comandos de colas bloqueantes.
- **Almacenamiento en caché:** Usar `expire()` y `expireat()` te permite usar Redis como caché. También puedes encontrar backends de caché de Redis de terceros para Django.
- **Pub/Sub (Publicación/Suscripción):** Redis proporciona comandos para suscribirse/desuscribirse y enviar mensajes a canales.
- **Rankings y tablas de clasificación:** Los conjuntos ordenados de Redis con puntuaciones hacen que sea muy fácil crear tablas de clasificación.
- **Seguimiento en tiempo real:** La rápida velocidad de E/S de Redis lo hace perfecto para escenarios en tiempo real.

---

### Resumen

En este capítulo, construiste un sistema de seguimiento utilizando relaciones de muchos a muchos con un modelo intermedio. También creaste un flujo de actividad utilizando relaciones genéricas y optimizaste los QuerySets para recuperar objetos relacionados. Luego, este capítulo te introdujo a las señales de Django y creaste una función receptora de señales para desnormalizar los conteos de objetos relacionados. Cubrimos las clases de configuración de aplicaciones, que utilizaste para cargar tus manejadores de señales. Añadiste Django Debug Toolbar a tu proyecto. También aprendiste a instalar y configurar Redis en tu proyecto Django. Finalmente, utilizaste Redis en tu proyecto para almacenar visualizaciones de elementos y construiste un ranking de imágenes con Redis.

En el próximo capítulo, aprenderás a construir una tienda online. Crearás un catálogo de productos y construirás un carrito de compras usando sesiones. Aprenderás a crear procesadores de contexto personalizados. También gestionarás pedidos de clientes y enviarás notificaciones asíncronas utilizando Celery y RabbitMQ.

---

### Ampliación de tu proyecto mediante IA

En esta sección, se te presenta una tarea para ampliar tu proyecto, acompañada de un prompt de muestra para ChatGPT que te ayudará. Para interactuar con ChatGPT, visita [https://chatgpt.com/](https://chatgpt.com/). Si esta es tu primera interacción con ChatGPT, puedes revisar la sección *Ampliación de tu proyecto mediante IA* en el Capítulo 3, *Extensión de tu aplicación de blog*.

En este ejemplo de proyecto, has aprendido a utilizar las señales de Django y has implementado con éxito un receptor de señales para actualizar el número total de "me gusta" de las imágenes cada vez que hay un cambio en el recuento de "me gusta". Ahora, aprovechemos ChatGPT para explorar la implementación de un receptor de señales que genere automáticamente un objeto `Profile` relacionado cada vez que se cree un objeto `User`. Puedes utilizar el prompt proporcionado en [https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter07/prompts/task.md](https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter07/prompts/task.md).

Después de implementar con éxito el receptor de señales, puedes eliminar los pasos de creación manual de perfiles incluidos previamente en la vista `register` de la aplicación `account` y del pipeline de autenticación social. Con la función receptora ahora vinculada a la señal `post_save` del modelo `User`, los perfiles se crearán automáticamente para los nuevos usuarios.

Si tienes problemas para comprender un concepto o tema en particular del libro, pídele a ChatGPT que te proporcione ejemplos adicionales o que te explique el concepto de una manera diferente. Este enfoque personalizado puede reforzar tu aprendizaje y garantizar que comprendas temas complejos.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter07](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter07)
- **Modelos de usuario personalizados:** [https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#specifying-a-custom-user-model](https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#specifying-a-custom-user-model)
- **El framework contenttypes:** [https://docs.djangoproject.com/en/5.2/ref/contrib/contenttypes/](https://docs.djangoproject.com/en/5.2/ref/contrib/contenttypes/)
- **Señales integradas de Django:** [https://docs.djangoproject.com/en/5.2/ref/signals/](https://docs.djangoproject.com/en/5.2/ref/signals/)
- **Clases de configuración de aplicaciones:** [https://docs.djangoproject.com/en/5.2/ref/applications/](https://docs.djangoproject.com/en/5.2/ref/applications/)
- **Documentación de Django Debug Toolbar:** [https://django-debug-toolbar.readthedocs.io/](https://django-debug-toolbar.readthedocs.io/)
- **Paneles de terceros para Django Debug Toolbar:** [https://django-debug-toolbar.readthedocs.io/en/latest/panels.html#third-party-panels](https://django-debug-toolbar.readthedocs.io/en/latest/panels.html#third-party-panels)
- **Almacén de datos en memoria Redis:** [https://redis.io/](https://redis.io/)
- **Imagen oficial de Redis en Docker:** [https://hub.docker.com/_/redis](https://hub.docker.com/_/redis)
- **Opciones de descarga de Redis:** [https://redis.io/download/](https://redis.io/download/)
- **Comandos de Redis:** [https://redis.io/commands/](https://redis.io/commands/)
- **Tipos de datos de Redis:** [https://redis.io/docs/manual/data-types/](https://redis.io/docs/manual/data-types/)
- **Documentación de redis-py:** [https://redis-py.readthedocs.io/](https://redis-py.readthedocs.io/)
