# Parte 3: Creación de una tienda online

## Capítulo 8: Creación de una tienda online

### Introducción

En el capítulo anterior, creaste un sistema de seguimiento (*follow*) y construiste un flujo de actividad de usuario (*activity stream*). También aprendiste cómo funcionan las señales (*signals*) de Django e integraste Redis en tu proyecto para contar las visualizaciones de imágenes.

En este capítulo, comenzarás un nuevo proyecto de Django que consiste en una tienda online completa. Este capítulo y los dos siguientes te mostrarán cómo construir las funcionalidades esenciales de una plataforma de comercio electrónico. Tu tienda online permitirá a los clientes explorar productos, añadirlos al carrito, aplicar códigos de descuento, pasar por el proceso de compra (*checkout*), pagar con tarjeta de crédito y obtener una factura. También implementarás un motor de recomendaciones para recomendar productos a tus clientes y utilizarás la internacionalización para ofrecer tu sitio en múltiples idiomas.

En este capítulo, aprenderás a:

- Crear un catálogo de productos
- Construir un carrito de compras usando sesiones de Django
- Crear procesadores de contexto de plantilla personalizados
- Gestionar pedidos de clientes
- Configurar Celery en tu proyecto con RabbitMQ como broker de mensajes
- Enviar notificaciones asíncronas a los clientes utilizando Celery
- Monitorizar Celery utilizando Flower

---

### Visión general funcional

La Figura 8.1 muestra una representación de las vistas, plantillas y funcionalidades principales que se construirán en este capítulo:

> *Figura 8.1: Diagrama de funcionalidades construidas en el Capítulo 8*

En este capítulo, implementarás la vista `product_list` para listar todos los productos y la vista `product_detail` para mostrar un solo producto. Permitirás filtrar productos por categoría en la vista `product_list` utilizando el parámetro `category_slug`. Implementarás un carrito de compras usando sesiones y construirás la vista `cart_detail` para mostrar los elementos del carrito. Crearás la vista `cart_add` para añadir productos al carrito y actualizar cantidades, y la vista `cart_remove` para eliminar productos del carrito. Implementarás el procesador de contexto de plantilla del carrito para mostrar la cantidad de artículos en el carrito y el costo total en el encabezado del sitio. También crearás la vista `order_create` para realizar pedidos y utilizarás Celery para implementar la tarea asíncrona `order_created` que envía una confirmación por correo electrónico a los clientes cuando realizan un pedido. Este capítulo te proporcionará el conocimiento para implementar sesiones de usuario en tu aplicación y te mostrará cómo trabajar con tareas asíncronas. Ambos son casos de uso muy comunes que puedes aplicar a casi cualquier proyecto.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter08](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter08).

Todos los módulos de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente que acompaña a este capítulo. Puedes seguir las instrucciones para instalar cada módulo de Python a continuación o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Creación de un proyecto de tienda online

Comencemos con un nuevo proyecto de Django para construir una tienda online. Tus usuarios podrán navegar a través de un catálogo de productos y añadir productos a un carrito de compras. Finalmente, podrán tramitar el pedido (*checkout*) y realizar una compra. Este capítulo cubrirá las siguientes funcionalidades de una tienda online:

- Creación de los modelos del catálogo de productos, su adición al sitio de administración y la creación de las vistas básicas para mostrar el catálogo.
- Creación de un sistema de carrito de compras usando sesiones de Django para permitir a los usuarios conservar los productos seleccionados mientras navegan por el sitio.
- Creación del formulario y la funcionalidad para realizar pedidos en el sitio.
- Envío de una confirmación asíncrona por correo electrónico a los usuarios cuando realizan un pedido.

Abre una consola y utiliza el siguiente comando para crear un nuevo entorno virtual para este proyecto dentro del directorio `env/`:

```bash
python -m venv env/myshop
```

Si estás utilizando Linux o macOS, ejecuta el siguiente comando para activar tu entorno virtual:

```bash
source env/myshop/bin/activate
```

Si estás utilizando Windows, utiliza el siguiente comando en su lugar:

```cmd
.\env\myshop\Scripts\activate
```

El indicador de la consola mostrará tu entorno virtual activo, de la siguiente manera:

```text
(myshop)laptop:~ zenx$
```

Instala Django en tu entorno virtual con el siguiente comando:

```bash
python -m pip install Django~=5.2.0
```

Inicia un nuevo proyecto llamado `myshop` con una aplicación llamada `shop` abriendo una consola y ejecutando el siguiente comando:

```bash
django-admin startproject myshop
```

Se ha creado la estructura inicial del proyecto. Utiliza los siguientes comandos para entrar en el directorio de tu proyecto y crear una nueva aplicación llamada `shop`:

```bash
cd myshop/
django-admin startapp shop
```

Edita `settings.py` y añade la siguiente línea a la lista `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'shop.apps.ShopConfig',
]
```

Tu aplicación ahora está activa para este proyecto. Definamos los modelos para el catálogo de productos.

#### Creación de los modelos del catálogo de productos

El catálogo de tu tienda constará de productos organizados en diferentes categorías. Cada producto tendrá un nombre, una descripción opcional, una imagen opcional, un precio y su disponibilidad.

Edita el archivo `models.py` de la aplicación `shop` que acabas de crear y añade el siguiente código:

```python
from django.db import models


class Category(models.Model):
    name = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)

    class Meta:
        ordering = ['name']
        indexes = [
            models.Index(fields=['name']),
        ]
        verbose_name = 'category'
        verbose_name_plural = 'categories'

    def __str__(self):
        return self.name


class Product(models.Model):
    category = models.ForeignKey(
        Category,
        related_name='products',
        on_delete=models.CASCADE
    )
    name = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200)
    image = models.ImageField(
        upload_to='products/%Y/%m/%d',
        blank=True
    )
    description = models.TextField(blank=True)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    available = models.BooleanField(default=True)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['name']
        indexes = [
            models.Index(fields=['id', 'slug']),
            models.Index(fields=['name']),
            models.Index(fields=['-created']),
        ]

    def __str__(self):
        return self.name
```

Estos son los modelos `Category` y `Product`. El modelo `Category` consta de un campo `name` y un campo `slug` único (`unique` implica la creación de un índice). En la clase `Meta` del modelo `Category`, hemos definido un índice para el campo `name`.

Los campos del modelo `Product` son los siguientes:

- `category`: Un `ForeignKey` al modelo `Category`. Esta es una relación de uno a muchos: un producto pertenece a una categoría y una categoría contiene múltiples productos.
- `name`: El nombre del producto.
- `slug`: El slug para este producto para construir URLs amigables.
- `image`: Una imagen opcional del producto.
- `description`: Una descripción opcional del producto.
- `price`: Este campo utiliza el tipo `decimal.Decimal` de Python para almacenar un número decimal de precisión fija. El número máximo de dígitos (incluidos los decimales) se establece mediante el atributo `max_digits` y los lugares decimales con el atributo `decimal_places`.
- `available`: Un valor booleano que indica si el producto está disponible o no. Se utilizará para habilitar/deshabilitar el producto en el catálogo.
- `created`: Este campo almacena cuándo se creó el objeto.
- `updated`: Este campo almacena cuándo se actualizó por última vez el objeto.

> [!IMPORTANT]
> Para el campo `price`, usamos `DecimalField` en lugar de `FloatField` para evitar problemas de redondeo. Utiliza siempre `DecimalField` para almacenar importes monetarios. `FloatField` utiliza internamente el tipo `float` de Python, mientras que `DecimalField` utiliza el tipo `Decimal` de Python. Al utilizar el tipo `Decimal`, evitarás problemas de redondeo en operaciones con números de punto flotante.

En la clase `Meta` del modelo `Product`, hemos definido un índice de múltiples campos para los campos `id` y `slug`. Ambos campos se indexan juntos para mejorar el rendimiento de las consultas que utilizan ambos campos.

Planeamos consultar productos tanto por `id` como por `slug`. Hemos añadido un índice para el campo `name` y un índice para el campo `created`. Hemos utilizado un guión antes del nombre del campo para definir el índice en orden descendente.

La Figura 8.2 muestra los dos modelos de datos que has creado:

> *Figura 8.2: Modelos para el catálogo de productos*

En la Figura 8.2, puedes ver los diferentes campos de los modelos de datos y la relación de uno a muchos entre los modelos `Category` y `Product`.

Estos modelos darán como resultado las tablas de base de datos que se muestran en la Figura 8.3:

> *Figura 8.3: Tablas de base de datos para los modelos del catálogo de productos*

La relación de uno a muchos entre ambas tablas se define con el campo `category_id` en la tabla `shop_product`, que se utiliza para almacenar el ID del objeto `Category` relacionado para cada objeto `Product`.

Creemos las migraciones iniciales de la base de datos para la aplicación `shop`. Como vas a trabajar con imágenes en tus modelos, necesitarás instalar la biblioteca Pillow. Recuerda que en el Capítulo 4, *Creación de un sitio web social*, aprendiste a instalar la biblioteca Pillow para gestionar imágenes. Abre la consola e instala Pillow con el siguiente comando:

```bash
python -m pip install Pillow==11.2
```

Ahora ejecuta el siguiente comando para crear las migraciones iniciales de tu proyecto:

```bash
python manage.py makemigrations
```

Verás la siguiente salida:

```text
Migrations for 'shop':
  shop/migrations/0001_initial.py
    - Create model Category
    - Create model Product
```

Ejecuta el siguiente comando para sincronizar la base de datos:

```bash
python manage.py migrate
```

Verás una salida que incluye la siguiente línea:

```text
Applying shop.0001_initial... OK
```

La base de datos ahora está sincronizada con tus modelos.

#### Registro de los modelos del catálogo en el sitio de administración

Añadamos tus modelos al sitio de administración para que puedas gestionar fácilmente categorías y productos. Edita el archivo `admin.py` de la aplicación `shop` y añade el siguiente código:

```python
from django.contrib import admin
from .models import Category, Product


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ['name', 'slug']
    prepopulated_fields = {'slug': ('name',)}


@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    list_display = [
        'name', 'slug', 'price', 'available', 'created', 'updated'
    ]
    list_filter = ['available', 'created', 'updated']
    list_editable = ['price', 'available']
    prepopulated_fields = {'slug': ('name',)}
```

Recuerda que utilizas el atributo `prepopulated_fields` para especificar campos donde el valor se establece automáticamente utilizando el valor de otros campos. Como has visto anteriormente, esto es conveniente para generar slugs.

Utilizas el atributo `list_editable` en la clase `ProductAdmin` para establecer los campos que se pueden editar desde la página de visualización de lista del sitio de administración. Esto te permitirá editar múltiples filas a la vez. Cualquier campo en `list_editable` también debe aparecer en el atributo `list_display`, ya que solo se pueden editar los campos mostrados.

Ahora crea un superusuario para tu sitio usando el siguiente comando:

```bash
python manage.py createsuperuser
```

Introduce el nombre de usuario, correo electrónico y contraseña deseados. Ejecuta el servidor de desarrollo con el siguiente comando:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/admin/shop/product/add/` en tu navegador e inicia sesión con el usuario que acabas de crear. Añade una nueva categoría y un producto utilizando la interfaz de administración. El formulario para añadir productos se verá de la siguiente manera:

> *Figura 8.4: El formulario de creación de productos*

Haz clic en el botón **SAVE**. La página de lista de cambios de productos del sitio de administración se verá así:

> *Figura 8.5: La página de lista de cambios de productos*

#### Creación de las vistas del catálogo

Para mostrar el catálogo de productos, necesitas crear una vista para listar todos los productos o filtrar productos por una categoría determinada. Edita el archivo `views.py` de la aplicación `shop` y añade el siguiente código:

```python
from django.shortcuts import get_object_or_404, render
from .models import Category, Product


def product_list(request, category_slug=None):
    category = None
    categories = Category.objects.all()
    products = Product.objects.filter(available=True)
    if category_slug:
        category = get_object_or_404(Category, slug=category_slug)
        products = products.filter(category=category)
    return render(
        request,
        'shop/product/list.html',
        {
            'category': category,
            'categories': categories,
            'products': products
        }
    )
```

En el código anterior, filtras el QuerySet con `available=True` para recuperar solo los productos disponibles. Utilizas un parámetro opcional `category_slug` para filtrar opcionalmente los productos por una categoría dada.

También necesitas una vista para recuperar y mostrar un solo producto. Añade la siguiente vista al archivo `views.py`:

```python
def product_detail(request, id, slug):
    product = get_object_or_404(
        Product, id=id, slug=slug, available=True
    )
    return render(
        request,
        'shop/product/detail.html',
        {'product': product}
    )
```

La vista `product_detail` espera los parámetros `id` y `slug` para recuperar la instancia de `Product`. Podrías obtener esta instancia solo a través del ID, ya que es un atributo único. Sin embargo, incluimos el slug en la URL para crear URLs amigables para SEO en los productos.

Después de construir las vistas de lista y detalle de productos, debes definir patrones de URL para ellas. Crea un nuevo archivo dentro del directorio de la aplicación `shop` y nómbralo `urls.py`. Añádele el siguiente código:

```python
from django.urls import path
from . import views

app_name = 'shop'

urlpatterns = [
    path('', views.product_list, name='product_list'),
    path(
        '<slug:category_slug>/',
        views.product_list,
        name='product_list_by_category'
    ),
    path(
        '<int:id>/<slug:slug>/',
        views.product_detail,
        name='product_detail'
    ),
]
```

Estos son los patrones de URL para tu catálogo de productos. Has definido dos patrones de URL diferentes para la vista `product_list`: un patrón llamado `product_list`, que llama a la vista `product_list` sin ningún parámetro, y un patrón llamado `product_list_by_category`, que proporciona un parámetro `category_slug` a la vista para filtrar productos según una categoría determinada. Añadiste un patrón para la vista `product_detail`, que pasa los parámetros `id` y `slug` a la vista para recuperar un producto específico.

Edita el archivo `urls.py` del proyecto `myshop` para que se vea así:

```python
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('shop.urls', namespace='shop')),
]
```

En los patrones de URL principales del proyecto, incluyes URLs para la aplicación `shop` bajo un espacio de nombres personalizado llamado `shop`.

A continuación, edita el archivo `models.py` de la aplicación `shop`, importa la función `reverse()` y añade un método `get_absolute_url()` a los modelos `Category` y `Product`, de la siguiente manera:

```python
from django.db import models
from django.urls import reverse


class Category(models.Model):
    # ...
    def get_absolute_url(self):
        return reverse(
            'shop:product_list_by_category',
            args=[self.slug]
        )


class Product(models.Model):
    # ...
    def get_absolute_url(self):
        return reverse('shop:product_detail', args=[self.id, self.slug])
```

Como ya sabes, `get_absolute_url()` es la convención para recuperar la URL de un objeto determinado. Aquí, utilizas los patrones de URL que acabas de definir en el archivo `urls.py`.

#### Creación de las plantillas del catálogo

Ahora necesitas crear plantillas para las vistas de lista y detalle de productos. Crea la siguiente estructura de directorios y archivos dentro del directorio de la aplicación `shop`:

```text
templates/
    shop/
        base.html
        product/
            list.html
            detail.html
```

Debes definir una plantilla base y luego extenderla en las plantillas de lista y detalle de productos. Edita la plantilla `shop/base.html` y añade el siguiente código:

```html
{% load static %}
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>{% block title %}My shop{% endblock %}</title>
    <link href="{% static "css/base.css" %}" rel="stylesheet">
</head>
<body>
    <div id="header">
        <a href="/" class="logo">My shop</a>
    </div>
    <div id="subheader">
        <div class="cart">
            Your cart is empty.
        </div>
    </div>
    <div id="content">
        {% block content %}
        {% endblock %}
    </div>
</body>
</html>
```

Esta es la plantilla base que utilizarás para tu tienda. Para incluir los estilos CSS y las imágenes que utilizan las plantillas, debes copiar los archivos estáticos que acompañan a este capítulo, que se encuentran en el directorio `static/` de la aplicación `shop`. Cópialos en la misma ubicación en tu proyecto. Puedes encontrar el contenido del directorio en [https://github.com/PacktPublishing/Django-5-by-Example/tree/main/Chapter08/myshop/shop/static](https://github.com/PacktPublishing/Django-5-by-Example/tree/main/Chapter08/myshop/shop/static).

Edita la plantilla `shop/product/list.html` y añade el siguiente código:

```html
{% extends "shop/base.html" %}
{% load static %}

{% block title %}
    {% if category %}{{ category.name }}{% else %}Products{% endif %}
{% endblock %}

{% block content %}
    <div id="sidebar">
        <h3>Categories</h3>
        <ul>
            <li {% if not category %}class="selected"{% endif %}>
                <a href="{% url "shop:product_list" %}">All</a>
            </li>
            {% for c in categories %}
                <li {% if category.slug == c.slug %}class="selected"{% endif %}>
                    <a href="{{ c.get_absolute_url }}">{{ c.name }}</a>
                </li>
            {% endfor %}
        </ul>
    </div>
    <div id="main" class="product-list">
        <h1>{% if category %}{{ category.name }}{% else %}Products{% endif %}</h1>
        {% for product in products %}
            <div class="item">
                <a href="{{ product.get_absolute_url }}">
                    <img src="{% if product.image %}{{ product.image.url }}{% else %}{% static "img/no_image.png" %}{% endif %}">
                </a>
                <a href="{{ product.get_absolute_url }}">{{ product.name }}</a>
                <br>
                ${{ product.price }}
            </div>
        {% endfor %}
    </div>
{% endblock %}
```

Asegúrate de que ninguna etiqueta de plantilla se divida en varias líneas.

Esta es la plantilla de lista de productos. Extiende la plantilla `shop/base.html` y utiliza la variable de contexto `categories` para mostrar todas las categorías en una barra lateral, y `products` para mostrar los productos de la página actual. La misma plantilla se utiliza tanto para listar todos los productos disponibles como para listar productos filtrados por categoría. Dado que el campo `image` del modelo `Product` puede estar en blanco, debes proporcionar una imagen predeterminada para los productos que no tienen imagen. La imagen se encuentra en tu directorio de archivos estáticos con la ruta relativa `img/no_image.png`.

Dado que estás utilizando `ImageField` para almacenar imágenes de productos, necesitas que el servidor de desarrollo sirva los archivos de imagen subidos.

Edita el archivo `settings.py` de `myshop` y añade las siguientes configuraciones:

```python
MEDIA_URL = 'media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

`MEDIA_URL` es la URL base que sirve los archivos multimedia subidos por los usuarios. `MEDIA_ROOT` es la ruta local donde residen estos archivos, que construyes anteponiendo dinámicamente la variable `BASE_DIR`.

Para que Django sirva los archivos multimedia subidos usando el servidor de desarrollo, edita el archivo `urls.py` principal de `myshop` y añade el siguiente código:

```python
from django.conf import settings
from django.conf.urls.static import static
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('shop.urls', namespace='shop')),
]

if settings.DEBUG:
    urlpatterns += static(
        settings.MEDIA_URL,
        document_root=settings.MEDIA_ROOT
    )
```

Recuerda que solo sirves archivos estáticos de esta manera durante el desarrollo. En un entorno de producción, nunca debes servir archivos estáticos con Django; el servidor de desarrollo de Django no sirve archivos estáticos de manera eficiente.

Ejecuta el servidor de desarrollo con el siguiente comando:

```bash
python manage.py runserver
```

Añade un par de productos a tu tienda utilizando el sitio de administración y abre `http://127.0.0.1:8000/` en tu navegador. Verás la página de lista de productos, que se verá similar a esto:

> *Figura 8.6: La página de lista de productos (Créditos de imágenes: Té verde: Foto por Jia Ye en Unsplash; Té rojo: Foto por Manki Kim en Unsplash; Té en polvo: Foto por Phuong Nguyen en Unsplash)*

Si creas un producto utilizando el sitio de administración y no subes una imagen para él, se mostrará la imagen predeterminada `no_image.png` en su lugar:

> *Figura 8.7: La lista de productos mostrando una imagen predeterminada para los productos que no tienen imagen*

Edita la plantilla `shop/product/detail.html` y añade el siguiente código:

```html
{% extends "shop/base.html" %}
{% load static %}

{% block title %}
    {{ product.name }}
{% endblock %}

{% block content %}
    <div class="product-detail">
        <img src="{% if product.image %}{{ product.image.url }}{% else %}{% static "img/no_image.png" %}{% endif %}">
        <h1>{{ product.name }}</h1>
        <h2>
            <a href="{{ product.category.get_absolute_url }}">
                {{ product.category }}
            </a>
        </h2>
        <p class="price">${{ product.price }}</p>
        {{ product.description|linebreaks }}
    </div>
{% endblock %}
```

En el código anterior, llamas al método `get_absolute_url()` en el objeto de categoría relacionado para mostrar los productos disponibles que pertenecen a la misma categoría.

Ahora abre `http://127.0.0.1:8000/` en tu navegador y haz clic en cualquier producto para ver la página de detalle del producto:

> *Figura 8.8: La página de detalle de producto*

Ahora has creado un catálogo de productos básico. A continuación, implementarás un carrito de compras que permitirá a los usuarios añadir cualquier producto mientras navegan por la tienda online.

---

### Creación de un carrito de compras

Después de construir el catálogo de productos, el siguiente paso es crear un carrito de compras para que los usuarios puedan seleccionar los productos que desean comprar. Un carrito de compras permite a los usuarios seleccionar productos y establecer la cantidad que desean pedir, y luego almacenar esta información temporalmente mientras navegan por el sitio hasta que finalmente realizan un pedido. El carrito debe persistir en la sesión para que los artículos del carrito se mantengan durante la visita del usuario.

Utilizarás el framework de sesiones de Django para persistir el carrito. El carrito se mantendrá en la sesión hasta que finalice o el usuario tramite el pedido.

#### Uso de las sesiones de Django

Django proporciona un framework de sesiones que admite sesiones anónimas y de usuario. El framework de sesiones te permite almacenar datos arbitrarios para cada visitante. Los datos de la sesión se almacenan en el lado del servidor y las cookies contienen el ID de la sesión, a menos que utilices el motor de sesiones basado en cookies. El middleware de sesión gestiona el envío y la recepción de cookies. El motor de sesiones predeterminado almacena los datos de la sesión en la base de datos, pero puedes elegir otros motores de sesiones.

Para usar sesiones, debes asegurarte de que la configuración `MIDDLEWARE` de tu proyecto contenga `django.contrib.sessions.middleware.SessionMiddleware`. Este middleware gestiona las sesiones. Se añade de forma predeterminada a la configuración `MIDDLEWARE` cuando creas un nuevo proyecto utilizando el comando `startproject`.

El middleware de sesión hace que la sesión actual esté disponible en el objeto `request`. Puedes acceder a la sesión actual utilizando `request.session`, tratándola como un diccionario de Python para almacenar y recuperar datos de sesión. El diccionario de sesión acepta cualquier objeto de Python de forma predeterminada que se pueda serializar a JSON. Puedes establecer una variable en la sesión de esta manera:

```python
request.session['foo'] = 'bar'
```

Puedes recuperar una clave de sesión de la siguiente manera:

```python
request.session.get('foo')
```

Puedes eliminar una clave que almacenaste previamente en la sesión de la siguiente manera:

```python
del request.session['foo']
```

Cuando los usuarios inician sesión en el sitio, se pierde su sesión anónima y se crea una nueva sesión para usuarios autenticados. Si almacenas elementos en una sesión anónima que necesitas conservar después de que el usuario inicie sesión, tendrás que copiar los datos de la sesión anterior en la nueva sesión. Puedes hacer esto recuperando los datos de la sesión antes de iniciar sesión con la función `login()` del sistema de autenticación de Django y almacenándolos en la sesión después de eso.

##### Configuraciones de sesión

Hay varias configuraciones que puedes utilizar para configurar las sesiones de tu proyecto. La más importante es `SESSION_ENGINE`. Esta configuración te permite establecer el lugar donde se almacenan las sesiones. De forma predeterminada, Django almacena las sesiones en la base de datos utilizando el modelo `Session` de la aplicación `django.contrib.sessions`.

Django ofrece las siguientes opciones para almacenar datos de sesión:

- **Sesiones en base de datos (*Database sessions*):** Los datos de sesión se almacenan en la base de datos. Este es el motor de sesión predeterminado.
- **Sesiones basadas en archivos (*File-based sessions*):** Los datos de sesión se almacenan en el sistema de archivos.
- **Sesiones en caché (*Cached sessions*):** Los datos de sesión se almacenan en un backend de caché. Puedes especificar backends de caché utilizando la configuración `CACHES`. Almacenar datos de sesión en un sistema de caché proporciona el mejor rendimiento.
- **Sesiones en base de datos con caché (*Cached database sessions*):** Los datos de sesión se almacenan en una caché de escritura simultánea y en la base de datos. Las lecturas solo utilizan la base de datos si los datos aún no están en la caché.
- **Sesiones basadas en cookies (*Cookie-based sessions*):** Los datos de sesión se almacenan en las cookies que se envían al navegador.

> [!TIP]
> Para un mejor rendimiento, utiliza un motor de sesiones basado en caché. Django admite Memcached de fábrica y puedes encontrar backends de caché de terceros para Redis y otros sistemas de caché.

Puedes personalizar las sesiones con configuraciones específicas:

- `SESSION_COOKIE_AGE`: La duración de las cookies de sesión en segundos. El valor predeterminado es `1209600` (dos semanas).
- `SESSION_COOKIE_DOMAIN`: El dominio utilizado para las cookies de sesión.
- `SESSION_COOKIE_HTTPONLY`: Si se debe utilizar el indicador HttpOnly en la cookie de sesión. Si se establece en `True`, JavaScript en el lado del cliente no podrá acceder a la cookie de sesión. El valor predeterminado es `True` para mayor seguridad contra el secuestro de sesiones.
- `SESSION_COOKIE_SECURE`: Un booleano que indica que la cookie solo debe enviarse si la conexión es HTTPS. El valor predeterminado es `False`.
- `SESSION_EXPIRE_AT_BROWSER_CLOSE`: Un booleano que indica si la sesión debe expirar cuando se cierra el navegador. El valor predeterminado es `False`.
- `SESSION_SAVE_EVERY_REQUEST`: Un booleano que, si es `True`, guardará la sesión en la base de datos en cada petición. El valor predeterminado es `False`.

Puedes ver todas las configuraciones de sesión y sus valores predeterminados en [https://docs.djangoproject.com/en/5.2/ref/settings/#sessions](https://docs.djangoproject.com/en/5.2/ref/settings/#sessions).

##### Caducidad de la sesión

Puedes optar por utilizar sesiones de duración de navegador o sesiones persistentes utilizando la configuración `SESSION_EXPIRE_AT_BROWSER_CLOSE`. Esto se establece en `False` de forma predeterminada, forzando la duración de la sesión al valor almacenado en la configuración `SESSION_COOKIE_AGE`. Si estableces `SESSION_EXPIRE_AT_BROWSER_CLOSE` en `True`, la sesión caducará cuando el usuario cierre el navegador.

Puedes usar el método `set_expiry()` de `request.session` para sobrescribir la duración de la sesión actual.

#### Almacenamiento de carritos de compras en sesiones

Necesitas crear una estructura simple que se pueda serializar a JSON para almacenar los elementos del carrito en una sesión. El carrito debe incluir los siguientes datos para cada elemento contenido en él:

- El ID de una instancia de `Product`
- La cantidad seleccionada para el producto
- El precio unitario del producto

Dado que los precios de los productos pueden variar, adoptemos el enfoque de almacenar el precio del producto junto con el producto en sí cuando se añade al carrito. Al hacerlo, utilizas el precio actual del producto cuando los usuarios lo añaden a su carrito, sin importar si el precio del producto se modifica posteriormente.

A continuación, debes construir la funcionalidad para crear carritos de compras y asociarlos con sesiones:

- Cuando se necesita un carrito, compruebas si se ha establecido una clave de sesión personalizada. Si no hay ningún carrito establecido en la sesión, creas un nuevo carrito y lo guardas en la clave de sesión del carrito.
- Para peticiones sucesivas, realizas la misma comprobación y obtienes los elementos del carrito a partir de la clave de sesión del carrito. Recuperas los elementos del carrito de la sesión y sus objetos `Product` relacionados de la base de datos.

Edita el archivo `settings.py` de tu proyecto y añade la siguiente configuración:

```python
CART_SESSION_ID = 'cart'
```

Esta es la clave que vas a utilizar para almacenar el carrito en la sesión del usuario. Dado que las sesiones de Django se gestionan por visitante, puedes utilizar la misma clave de sesión del carrito para todas las sesiones.

Creemos una aplicación para gestionar los carritos de compras. Abre la consola y crea una nueva aplicación ejecutando el siguiente comando desde el directorio del proyecto:

```bash
python manage.py startapp cart
```

Luego, edita el archivo `settings.py` de tu proyecto y añade la nueva aplicación a la configuración `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    # ...
    'cart.apps.CartConfig',
    'shop.apps.ShopConfig',
]
```

Crea un nuevo archivo dentro del directorio de la aplicación `cart` y nómbralo `cart.py`. Añade el siguiente código:

```python
from decimal import Decimal
from django.conf import settings
from shop.models import Product


class Cart:
    def __init__(self, request):
        """
        Initialize the cart.
        """
        self.session = request.session
        cart = self.session.get(settings.CART_SESSION_ID)
        if not cart:
            # save an empty cart in the session
            cart = self.session[settings.CART_SESSION_ID] = {}
        self.cart = cart
```

Esta es la clase `Cart` que te permitirá gestionar el carrito de compras. Requieres que el carrito se inicialice con un objeto `request`. Almacenas la sesión actual usando `self.session = request.session` para que sea accesible para los otros métodos de la clase `Cart`.

Primero, intentas obtener el carrito de la sesión actual usando `self.session.get(settings.CART_SESSION_ID)`. Si no hay ningún carrito presente en la sesión, creas un carrito vacío estableciendo un diccionario vacío en la sesión.

Construirás tu diccionario del carrito con los IDs de productos como claves, y para cada clave de producto, un diccionario será un valor que incluye cantidad y precio. Al hacer esto, puedes garantizar que un producto no se añadirá más de una vez al carrito.

Creemos métodos para añadir productos al carrito, guardar, eliminar, iterar y calcular totales. Añade los siguientes métodos a la clase `Cart`:

```python
class Cart:
    # ...
    def add(self, product, quantity=1, override_quantity=False):
        """
        Add a product to the cart or update its quantity.
        """
        product_id = str(product.id)
        if product_id not in self.cart:
            self.cart[product_id] = {
                'quantity': 0,
                'price': str(product.price)
            }
        if override_quantity:
            self.cart[product_id]['quantity'] = quantity
        else:
            self.cart[product_id]['quantity'] += quantity
        self.save()

    def save(self):
        # mark the session as "modified" to make sure it gets saved
        self.session.modified = True

    def remove(self, product):
        """
        Remove a product from the cart.
        """
        product_id = str(product.id)
        if product_id in self.cart:
            del self.cart[product_id]
            self.save()

    def __iter__(self):
        """
        Iterate over the items in the cart and get the products from the database.
        """
        product_ids = self.cart.keys()
        # get the product objects and add them to the cart
        products = Product.objects.filter(id__in=product_ids)
        cart = self.cart.copy()
        for product in products:
            cart[str(product.id)]['product'] = product
        for item in cart.values():
            item['price'] = Decimal(item['price'])
            item['total_price'] = item['price'] * item['quantity']
            yield item

    def __len__(self):
        """
        Count all items in the cart.
        """
        return sum(item['quantity'] for item in self.cart.values())

    def get_total_price(self):
        return sum(
            Decimal(item['price']) * item['quantity']
            for item in self.cart.values()
        )

    def clear(self):
        # remove cart from session
        del self.session[settings.CART_SESSION_ID]
        self.save()
```

- `add()`: Recibe la instancia del producto, la cantidad y un booleano `override_quantity`. Convierte el ID del producto en una cadena porque Django utiliza JSON para serializar los datos de la sesión y JSON solo permite nombres de clave en formato de cadena.
- `save()`: Marca la sesión como modificada usando `self.session.modified = True`. Esto le dice a Django que la sesión ha cambiado y debe guardarse.
- `remove()`: Elimina un producto dado del diccionario del carrito y llama a `save()`.
- `__iter__()`: Recupera las instancias de `Product` que están presentes en el carrito para incluirlas en los elementos del carrito. Convierte el precio de cada elemento a `Decimal` y añade el atributo `total_price`.
- `__len__()`: Devuelve la suma de las cantidades de todos los elementos del carrito.
- `get_total_price()`: Calcula el costo total de los artículos en el carrito.
- `clear()`: Elimina el carrito de la sesión y la guarda.

Tu clase `Cart` ahora está lista para gestionar carritos de compras.

#### Creación de vistas para el carrito de compras

Ahora que tienes una clase `Cart` para gestionar el carrito, necesitas crear las vistas para añadir, actualizar o eliminar elementos de él.

##### Adición de elementos al carrito

Para añadir elementos al carrito, necesitas un formulario que permita al usuario seleccionar una cantidad. Crea un archivo `forms.py` dentro del directorio de la aplicación `cart` y añade el siguiente código:

```python
from django import forms

PRODUCT_QUANTITY_CHOICES = [(i, str(i)) for i in range(1, 21)]


class CartAddProductForm(forms.Form):
    quantity = forms.TypedChoiceField(
        choices=PRODUCT_QUANTITY_CHOICES,
        coerce=int
    )
    override = forms.BooleanField(
        required=False,
        initial=False,
        widget=forms.HiddenInput
    )
```

Tu clase `CartAddProductForm` contiene los siguientes dos campos:

- `quantity`: Permite al usuario seleccionar una cantidad entre 1 y 20. Utilizas un campo `TypedChoiceField` con `coerce=int` para convertir la entrada en un número entero.
- `override`: Te permite indicar si la cantidad debe sumarse a cualquier cantidad existente en el carrito para este producto (`False`) o si la cantidad existente debe sobrescribirse con la cantidad dada (`True`).

Creemos las vistas para el carrito. Edita el archivo `views.py` de la aplicación `cart` y añade el siguiente código:

```python
from django.shortcuts import get_object_or_404, redirect, render
from django.views.decorators.http import require_POST
from shop.models import Product
from .cart import Cart
from .forms import CartAddProductForm


@require_POST
def cart_add(request, product_id):
    cart = Cart(request)
    product = get_object_or_404(Product, id=product_id)
    form = CartAddProductForm(request.POST)
    if form.is_valid():
        cd = form.cleaned_data
        cart.add(
            product=product,
            quantity=cd['quantity'],
            override_quantity=cd['override']
        )
    return redirect('cart:cart_detail')


@require_POST
def cart_remove(request, product_id):
    cart = Cart(request)
    product = get_object_or_404(Product, id=product_id)
    cart.remove(product)
    return redirect('cart:cart_detail')


def cart_detail(request):
    cart = Cart(request)
    return render(request, 'cart/detail.html', {'cart': cart})
```

Añadamos patrones de URL para estas vistas. Crea un nuevo archivo dentro del directorio de la aplicación `cart` y nómbralo `urls.py`. Añade los siguientes patrones de URL:

```python
from django.urls import path
from . import views

app_name = 'cart'

urlpatterns = [
    path('', views.cart_detail, name='cart_detail'),
    path('add/<int:product_id>/', views.cart_add, name='cart_add'),
    path(
        'remove/<int:product_id>/',
        views.cart_remove,
        name='cart_remove'
    ),
]
```

Edita el archivo `urls.py` principal del proyecto `myshop` y añade el siguiente patrón de URL para incluir las URLs del carrito:

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('cart/', include('cart.urls', namespace='cart')),
    path('', include('shop.urls', namespace='shop')),
]
```

Asegúrate de incluir este patrón de URL antes del patrón `shop.urls`, ya que es más restrictivo que este último.

##### Creación de una plantilla para mostrar el carrito

Crea la siguiente estructura de archivos dentro del directorio de la aplicación `cart`:

```text
templates/
    cart/
        detail.html
```

Edita la plantilla `cart/detail.html` y añade el siguiente código:

```html
{% extends "shop/base.html" %}
{% load static %}

{% block title %}
    Your shopping cart
{% endblock %}

{% block content %}
    <h1>Your shopping cart</h1>
    <table class="cart">
        <thead>
            <tr>
                <th>Image</th>
                <th>Product</th>
                <th>Quantity</th>
                <th>Remove</th>
                <th>Unit price</th>
                <th>Price</th>
            </tr>
        </thead>
        <tbody>
            {% for item in cart %}
                {% with product=item.product %}
                    <tr>
                        <td>
                            <a href="{{ product.get_absolute_url }}">
                                <img src="{% if product.image %}{{ product.image.url }}{% else %}{% static "img/no_image.png" %}{% endif %}">
                            </a>
                        </td>
                        <td>{{ product.name }}</td>
                        <td>{{ item.quantity }}</td>
                        <td>
                            <form action="{% url "cart:cart_remove" product.id %}" method="post">
                                <input type="submit" value="Remove">
                                {% csrf_token %}
                            </form>
                        </td>
                        <td class="num">${{ item.price }}</td>
                        <td class="num">${{ item.total_price }}</td>
                    </tr>
                {% endwith %}
            {% endfor %}
            <tr class="total">
                <td>Total</td>
                <td colspan="4"></td>
                <td class="num">${{ cart.get_total_price }}</td>
            </tr>
        </tbody>
    </table>
    <p class="text-right">
        <a href="{% url "shop:product_list" %}" class="button light">Continue shopping</a>
        <a href="#" class="button">Checkout</a>
    </p>
{% endblock %}
```

##### Adición de productos al carrito

Ahora necesitas añadir un botón *Add to cart* a la página de detalle del producto. Edita el archivo `views.py` de la aplicación `shop` y añade `CartAddProductForm` a la vista `product_detail`, de la siguiente manera:

```python
from cart.forms import CartAddProductForm

# ...


def product_detail(request, id, slug):
    product = get_object_or_404(
        Product, id=id, slug=slug, available=True
    )
    cart_product_form = CartAddProductForm()
    return render(
        request,
        'shop/product/detail.html',
        {
            'product': product,
            'cart_product_form': cart_product_form
        }
    )
```

Edita la plantilla `shop/product/detail.html` de la aplicación `shop` y añade el siguiente formulario al precio del producto:

```html
...
<p class="price">${{ product.price }}</p>
<form action="{% url "cart:cart_add" product.id %}" method="post">
    {{ cart_product_form }}
    {% csrf_token %}
    <input type="submit" value="Add to cart">
</form>
{{ product.description|linebreaks }}
...
```

Ejecuta el servidor de desarrollo con `python manage.py runserver`, abre `http://127.0.0.1:8000/` en tu navegador y navega a la página de detalle de un producto:

> *Figura 8.9: La página de detalle de producto, incluyendo el botón Add to cart*

Elige una cantidad y haz clic en el botón **Add to cart**. El formulario se envía a la vista `cart_add` a través de POST y luego redirige a la página de detalle del carrito:

> *Figura 8.10: La página de detalle del carrito*

##### Actualización de cantidades de productos en el carrito

Vamos a permitir que los usuarios cambien las cantidades directamente desde la página de detalle del carrito.

Edita el archivo `views.py` de la aplicación `cart` y añade las siguientes líneas a la vista `cart_detail`:

```python
def cart_detail(request):
    cart = Cart(request)
    for item in cart:
        item['update_quantity_form'] = CartAddProductForm(
            initial={'quantity': item['quantity'], 'override': True}
        )
    return render(request, 'cart/detail.html', {'cart': cart})
```

Ahora edita la plantilla `cart/detail.html` de la aplicación `cart` y busca la siguiente línea:

```html
<td>{{ item.quantity }}</td>
```

Reemplaza la línea anterior con el siguiente código:

```html
<td>
    <form action="{% url "cart:cart_add" product.id %}" method="post">
        {{ item.update_quantity_form.quantity }}
        {{ item.update_quantity_form.override }}
        <input type="submit" value="Update">
        {% csrf_token %}
    </form>
</td>
```

Abre `http://127.0.0.1:8000/cart/` en tu navegador:

> *Figura 8.11: La página de detalle del carrito, incluyendo el formulario para actualizar cantidades de productos*

#### Creación de un procesador de contexto para el carrito actual

Debemos mostrar el número total de artículos en el carrito y el costo total en el encabezado de todas las páginas. Para lograr esto, crearemos un procesador de contexto (*context processor*).

##### Procesadores de contexto

Un procesador de contexto es una función de Python que toma el objeto `request` como argumento y devuelve un diccionario que se añade al contexto de la petición. Los procesadores de contexto resultan muy útiles cuando necesitas que algo esté disponible globalmente para todas las plantillas.

De forma predeterminada, cuando creas un nuevo proyecto utilizando el comando `startproject`, tu proyecto contiene procesadores de contexto de plantilla integrados como `debug`, `request`, `auth` y `messages`.

##### Configuración del carrito en el contexto de la petición

Crea un nuevo archivo dentro del directorio de la aplicación `cart` y nómbralo `context_processors.py`. Añade el siguiente código al archivo:

```python
from .cart import Cart


def cart(request):
    return {'cart': Cart(request)}
```

Edita el archivo `settings.py` de tu proyecto y añade `cart.context_processors.cart` a la opción `context_processors` dentro de la configuración `TEMPLATES`:

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
                'cart.context_processors.cart',
            ],
        },
    },
]
```

A continuación, edita la plantilla `shop/base.html` de la aplicación `shop` y busca las siguientes líneas:

```html
<div class="cart">
    Your cart is empty.
</div>
```

Reemplaza las líneas anteriores con el siguiente código:

```html
<div class="cart">
    {% with total_items=cart|length %}
        {% if total_items > 0 %}
            Your cart:
            <a href="{% url "cart:cart_detail" %}">
                {{ total_items }} item{{ total_items|pluralize }},
                ${{ cart.get_total_price }}
            </a>
        {% else %}
            Your cart is empty.
        {% endif %}
    {% endwith %}
</div>
```

Abre `http://127.0.0.1:8000/` en tu navegador y añade algunos productos al carrito. En el encabezado del sitio web, ahora puedes ver el número total de artículos en el carrito y el costo total:

> *Figura 8.12: El encabezado del sitio mostrando los artículos actuales en el carrito*

---

### Registro de pedidos de clientes

Cuando se tramita un carrito de compras, es necesario guardar un pedido en la base de datos. Los pedidos contendrán información sobre los clientes y los productos que están comprando.

Crea una nueva aplicación para gestionar los pedidos de los clientes utilizando el siguiente comando:

```bash
python manage.py startapp orders
```

Edita el archivo `settings.py` de tu proyecto y añade la nueva aplicación a la configuración `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    # ...
    'cart.apps.CartConfig',
    'orders.apps.OrdersConfig',
    'shop.apps.ShopConfig',
]
```

#### Creación de los modelos de pedidos

Necesitarás un modelo para almacenar los detalles del pedido y un segundo modelo para almacenar los artículos comprados, incluido su precio y cantidad. Edita el archivo `models.py` de la aplicación `orders` y añádele el siguiente código:

```python
from django.db import models


class Order(models.Model):
    first_name = models.CharField(max_length=50)
    last_name = models.CharField(max_length=50)
    email = models.EmailField()
    address = models.CharField(max_length=250)
    postal_code = models.CharField(max_length=20)
    city = models.CharField(max_length=100)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)
    paid = models.BooleanField(default=False)

    class Meta:
        ordering = ['-created']
        indexes = [
            models.Index(fields=['-created']),
        ]

    def __str__(self):
        return f'Order {self.id}'

    def get_total_cost(self):
        return sum(item.get_cost() for item in self.items.all())


class OrderItem(models.Model):
    order = models.ForeignKey(
        Order,
        related_name='items',
        on_delete=models.CASCADE
    )
    product = models.ForeignKey(
        'shop.Product',
        related_name='order_items',
        on_delete=models.CASCADE
    )
    price = models.DecimalField(
        max_digits=10,
        decimal_places=2
    )
    quantity = models.PositiveIntegerField(default=1)

    def __str__(self):
        return str(self.id)

    def get_cost(self):
        return self.price * self.quantity
```

Ejecuta el siguiente comando para crear las migraciones iniciales para la aplicación `orders`:

```bash
python manage.py makemigrations
```

Ejecuta el siguiente comando para aplicar la nueva migración:

```bash
python manage.py migrate
```

Tus modelos de pedidos ahora están sincronizados con la base de datos.

#### Inclusión de los modelos de pedidos en el sitio de administración

Edita el archivo `admin.py` de la aplicación `orders` y añade el siguiente código:

```python
from django.contrib import admin
from .models import Order, OrderItem


class OrderItemInline(admin.TabularInline):
    model = OrderItem
    raw_id_fields = ['product']


@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = [
        'id', 'first_name', 'last_name', 'email',
        'address', 'postal_code', 'city', 'paid',
        'created', 'updated'
    ]
    list_filter = ['paid', 'created', 'updated']
    inlines = [OrderItemInline]
```

Utilizas una clase `TabularInline` para el modelo `OrderItem` para incluirlo como un inline en la clase `OrderAdmin`. Un inline te permite incluir un modelo en la misma página de edición que su modelo relacionado.

Abre `http://127.0.0.1:8000/admin/orders/order/add/` en tu navegador:

> *Figura 8.13: El formulario Add order, incluyendo OrderItemInline*

#### Creación de pedidos de clientes

Un nuevo pedido se creará siguiendo estos pasos:

1. Presentar al usuario un formulario de pedido para que complete sus datos.
2. Crear una nueva instancia de `Order` con los datos introducidos y crear una instancia de `OrderItem` asociada para cada artículo del carrito.
3. Limpiar todos los contenidos del carrito y redirigir al usuario a una página de éxito.

Crea un nuevo archivo dentro del directorio de la aplicación `orders` y nómbralo `forms.py`. Añade el siguiente código:

```python
from django import forms
from .models import Order


class OrderCreateForm(forms.ModelForm):
    class Meta:
        model = Order
        fields = [
            'first_name', 'last_name', 'email',
            'address', 'postal_code', 'city'
        ]
```

Ahora necesitas una vista para gestionar el formulario y crear un nuevo pedido. Edita el archivo `views.py` de la aplicación `orders` y añade el siguiente código:

```python
from cart.cart import Cart
from django.shortcuts import render
from .forms import OrderCreateForm
from .models import OrderItem


def order_create(request):
    cart = Cart(request)
    if request.method == 'POST':
        form = OrderCreateForm(request.POST)
        if form.is_valid():
            order = form.save()
            for item in cart:
                OrderItem.objects.create(
                    order=order,
                    product=item['product'],
                    price=item['price'],
                    quantity=item['quantity']
                )
            # clear the cart
            cart.clear()
            return render(
                request,
                'orders/order/created.html',
                {'order': order}
            )
    else:
        form = OrderCreateForm()
    return render(
        request,
        'orders/order/create.html',
        {'cart': cart, 'form': form}
    )
```

Crea un nuevo archivo dentro del directorio de la aplicación `orders` y nómbralo `urls.py`. Añádele el siguiente código:

```python
from django.urls import path
from . import views

app_name = 'orders'

urlpatterns = [
    path('create/', views.order_create, name='order_create'),
]
```

Edita el archivo `urls.py` de `myshop` e incluye el siguiente patrón antes de `shop.urls`:

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('cart/', include('cart.urls', namespace='cart')),
    path('orders/', include('orders.urls', namespace='orders')),
    path('', include('shop.urls', namespace='shop')),
]
```

Edita la plantilla `cart/detail.html` de la aplicación `cart` y actualiza el enlace de Checkout:

```html
<a href="{% url "orders:order_create" %}" class="button">
    Checkout
</a>
```

Crea la siguiente estructura de archivos dentro del directorio de la aplicación `orders`:

```text
templates/
    orders/
        order/
            create.html
            created.html
```

Edita la plantilla `orders/order/create.html` y añade el siguiente código:

```html
{% extends "shop/base.html" %}

{% block title %}
    Checkout
{% endblock %}

{% block content %}
    <h1>Checkout</h1>
    <div class="order-info">
        <h3>Your order</h3>
        <ul>
            {% for item in cart %}
                <li>
                    {{ item.quantity }}x {{ item.product.name }}
                    <span>${{ item.total_price }}</span>
                </li>
            {% endfor %}
        </ul>
        <p>Total: ${{ cart.get_total_price }}</p>
    </div>
    <form method="post" class="order-form">
        {{ form.as_p }}
        <p><input type="submit" value="Place order"></p>
        {% csrf_token %}
    </form>
{% endblock %}
```

Edita la plantilla `orders/order/created.html` y añade el siguiente código:

```html
{% extends "shop/base.html" %}

{% block title %}
    Thank you
{% endblock %}

{% block content %}
    <h1>Thank you</h1>
    <p>Your order has been successfully completed. Your order number is <strong>{{ order.id }}</strong>.</p>
{% endblock %}
```

Abre `http://127.0.0.1:8000/` en tu navegador, añade un par de productos al carrito y continúa a la página de pago:

> *Figura 8.14: La página de creación de pedidos, incluido el formulario de tramitación del pedido y los detalles del pedido*

Completa el formulario con datos válidos y haz clic en el botón **Place order**. El pedido se creará y verás una página de éxito:

> *Figura 8.15: La plantilla de pedido creado mostrando el número de pedido*

Para evitar que se muestre el mensaje *Your cart is empty* cuando se crea un pedido, edita la plantilla `shop/base.html` de la aplicación `shop` y reemplaza la sección del carrito con:

```html
...
<div class="cart">
    {% with total_items=cart|length %}
        {% if total_items > 0 %}
            Your cart:
            <a href="{% url "cart:cart_detail" %}">
                {{ total_items }} item{{ total_items|pluralize }},
                ${{ cart.get_total_price }}
            </a>
        {% elif not order %}
            Your cart is empty.
        {% endif %}
    {% endwith %}
</div>
...
```

Abre el sitio de administración en `http://127.0.0.1:8000/admin/orders/order/` para verificar que el pedido se haya creado:

> *Figura 8.16: La sección de lista de pedidos del sitio de administración*

---

### Creación de tareas asíncronas

Al recibir una petición HTTP, debes devolver una respuesta al usuario lo más rápido posible. Las tareas de larga duración pueden ralentizar gravemente la respuesta del servidor. Podemos descargar trabajo del ciclo de petición/respuesta ejecutando ciertas tareas en segundo plano (*background*).

#### Trabajo con tareas asíncronas

La ejecución asíncrona es especialmente relevante para procesos intensivos en datos, intensivos en recursos y que consumen mucho tiempo, o procesos sujetos a fallos de conexión (como el envío de correos electrónicos), que podrían requerir una política de reintentos.

##### Workers, colas de mensajes y brokers de mensajes

Mientras tu servidor web procesa peticiones y devuelve respuestas, necesitas un segundo servidor basado en tareas, llamado **worker**, para procesar las tareas asíncronas. Para indicar a los workers qué tareas ejecutar, enviamos mensajes a una **cola de mensajes** (*message queue*, estructura FIFO).

Para gestionar la cola de mensajes, necesitamos un **broker de mensajes** (*message broker*). El broker de mensajes se utiliza para traducir mensajes a un protocolo formal de mensajería y gestionar las colas de mensajes para múltiples receptores.

La Figura 8.17 muestra cómo funciona una cola de mensajes:

> *Figura 8.17: Ejecución asíncrona mediante una cola de mensajes y workers*

#### Uso de Django con Celery y RabbitMQ

Celery es una cola de tareas distribuidas que puede procesar grandes cantidades de mensajes. Utilizaremos Celery para definir tareas asíncronas como funciones de Python dentro de nuestras aplicaciones de Django.

RabbitMQ es el broker de mensajes más ampliamente desplegado y el recomendado para Celery.

La Figura 8.18 muestra la arquitectura que utilizaremos:

> *Figura 8.18: Arquitectura para tareas asíncronas con Django, RabbitMQ y Celery*

##### Instalación de Celery

Instala Celery mediante pip usando el siguiente comando:

```bash
python -m pip install celery==5.4.0
```

##### Instalación de RabbitMQ

Descarga la imagen oficial de RabbitMQ en Docker ejecutando el siguiente comando en la consola:

```bash
docker pull rabbitmq:3.13.1-management
```

Ejecuta el siguiente comando en la consola para iniciar el servidor RabbitMQ con Docker:

```bash
docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.13.1-management
```

##### Acceso a la interfaz de administración de RabbitMQ

Abre `http://127.0.0.1:15672/` en tu navegador:

> *Figura 8.19: La pantalla de inicio de sesión de la interfaz de administración de RabbitMQ*

Introduce `guest` tanto como nombre de usuario como contraseña y haz clic en **Login**:

> *Figura 8.20: El panel de control de la interfaz de administración de RabbitMQ*

#### Adición de Celery a tu proyecto

Crea un nuevo archivo junto al archivo `settings.py` de `myshop` y nómbralo `celery.py`. Añade el siguiente código:

```python
import os
from celery import Celery

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myshop.settings')

app = Celery('myshop')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
```

Edita el archivo `myshop/__init__.py` y añade el siguiente código:

```python
# import celery
from .celery import app as celery_app

__all__ = ['celery_app']
```

##### Ejecución de un worker de Celery

Abre otra consola e inicia un worker de Celery desde el directorio de tu proyecto:

```bash
celery -A myshop worker -l info
```

Abre `http://127.0.0.1:15672/` en tu navegador para verificar las conexiones y colas activas en la interfaz de RabbitMQ:

> *Figura 8.21: El panel de control de RabbitMQ mostrando conexiones y colas*

> [!NOTE]
> La configuración `CELERY_ALWAYS_EAGER` te permite ejecutar tareas localmente de forma síncrona en lugar de enviarlas a la cola. Esto es útil para ejecutar pruebas unitarias o ejecutar la aplicación en tu entorno local sin ejecutar Celery.

#### Adición de tareas asíncronas a tu aplicación

Crea un nuevo archivo dentro de la aplicación `orders` y nómbralo `tasks.py`. Añade el siguiente código:

```python
from celery import shared_task
from django.core.mail import send_mail
from .models import Order


@shared_task
def order_created(order_id):
    """
    Task to send an e-mail notification when an order is successfully created.
    """
    order = Order.objects.get(id=order_id)
    subject = f'Order nr. {order.id}'
    message = (
        f'Dear {order.first_name},\n\n'
        f'You have successfully placed an order.'
        f'Your order ID is {order.id}.'
    )
    mail_sent = send_mail(
        subject,
        message,
        'admin@myshop.com',
        [order.email]
    )
    return mail_sent
```

> [!TIP]
> Siempre se recomienda pasar únicamente IDs a las funciones de tareas y recuperar los objetos de la base de datos cuando se ejecuta la tarea. Al hacerlo, evitamos acceder a información obsoleta.

Si no deseas configurar un servidor SMTP durante el desarrollo, puedes indicar a Django que escriba los correos electrónicos en la consola añadiendo la siguiente configuración al archivo `settings.py`:

```python
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

Ahora debes añadir la tarea a tu vista `order_create`. Edita el archivo `views.py` de la aplicación `orders`:

```python
# ...
from .tasks import order_created


def order_create(request):
    # ...
    if request.method == 'POST':
        # ...
        if form.is_valid():
            # ...
            cart.clear()
            # launch asynchronous task
            order_created.delay(order.id)
            # ...
```

Llamas al método `delay()` de la tarea para ejecutarla de forma asíncrona. La tarea se añadirá a la cola de mensajes y será ejecutada por el worker de Celery tan pronto como sea posible.

Reinicia el worker de Celery con:

```bash
celery -A myshop worker -l info
```

Realiza un nuevo pedido en tu navegador web. En la consola del worker de Celery verás una salida que confirma la ejecución exitosa de la tarea:

```text
[2024-01-02 20:25:19,569: INFO/MainProcess] Task orders.tasks.order_created[a94dc22e-372b-4339-bff7-52bc83161c5c] received
...
[2024-01-02 20:25:19,605: INFO/ForkPoolWorker-8] Task orders.tasks.order_created[a94dc22e-372b-4339-bff7-52bc83161c5c] succeeded in 0.015824042027816176s: 1
```

#### Monitorización de Celery con Flower

Flower es una herramienta basada en web para monitorizar Celery.

Instala Flower usando el siguiente comando:

```bash
python -m pip install flower==2.0.1
```

Inicia Flower ejecutando el siguiente comando desde el directorio de tu proyecto:

```bash
celery -A myshop flower
```

Abre `http://localhost:5555/` en tu navegador:

> *Figura 8.22: El panel de control de Flower*

Haz clic en el nombre del worker y luego en la pestaña **Queues**:

> *Figura 8.23: Flower – Colas de tareas del worker de Celery*

Haz clic en la pestaña **Tasks**:

> *Figura 8.24: Flower – Tareas del worker de Celery*

Realiza un pedido en `http://localhost:8000/` y revisa `http://localhost:5555/`:

> *Figura 8.25: Flower – Workers de Celery con tareas procesadas*

> *Figura 8.26: Flower – Detalles de tareas de Celery*

Para proteger Flower con autenticación básica, reinícialo con la opción `--basic-auth`:

```bash
celery -A myshop flower --basic-auth=user:pwd
```

Reemplaza `user` y `pwd` con el nombre de usuario y contraseña deseados. Al acceder a `http://localhost:5555/`, el navegador te solicitará credenciales:

> *Figura 8.27: Autenticación básica requerida para acceder a Flower*

---

### Resumen

En este capítulo, creaste una aplicación básica de comercio electrónico. Creaste un catálogo de productos y construiste un carrito de compras usando sesiones. Implementaste un procesador de contexto personalizado para hacer que el carrito estuviera disponible para todas las plantillas y creaste un formulario para realizar pedidos. También aprendiste a implementar tareas asíncronas utilizando Celery y RabbitMQ.

En el próximo capítulo, descubrirás cómo integrar una pasarela de pago en tu tienda, añadir acciones personalizadas al sitio de administración, exportar datos en formato CSV y generar archivos PDF dinámicamente.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter08](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter08)
- **Archivos estáticos para el proyecto:** [https://github.com/PacktPublishing/Django-5-by-Example/tree/main/Chapter08/myshop/shop/static](https://github.com/PacktPublishing/Django-5-by-Example/tree/main/Chapter08/myshop/shop/static)
- **Configuraciones de sesión de Django:** [https://docs.djangoproject.com/en/5.2/ref/settings/#sessions](https://docs.djangoproject.com/en/5.2/ref/settings/#sessions)
- **Procesadores de contexto integrados de Django:** [https://docs.djangoproject.com/en/5.2/ref/templates/api/#built-in-template-context-processors](https://docs.djangoproject.com/en/5.2/ref/templates/api/#built-in-template-context-processors)
- **Información sobre RequestContext:** [https://docs.djangoproject.com/en/5.2/ref/templates/api/#django.template.RequestContext](https://docs.djangoproject.com/en/5.2/ref/templates/api/#django.template.RequestContext)
- **Documentación de Celery:** [https://docs.celeryq.dev/en/stable/index.html](https://docs.celeryq.dev/en/stable/index.html)
- **Introducción a Celery:** [https://docs.celeryq.dev/en/stable/getting-started/introduction.html](https://docs.celeryq.dev/en/stable/getting-started/introduction.html)
- **Imagen oficial de RabbitMQ en Docker:** [https://hub.docker.com/_/rabbitmq](https://hub.docker.com/_/rabbitmq)
- **Instrucciones de instalación de RabbitMQ:** [https://www.rabbitmq.com/download.html](https://www.rabbitmq.com/download.html)
- **Documentación de Flower:** [https://flower.readthedocs.io/](https://flower.readthedocs.io/)
- **Métodos de autenticación de Flower:** [https://flower.readthedocs.io/en/latest/auth.html](https://flower.readthedocs.io/en/latest/auth.html)
