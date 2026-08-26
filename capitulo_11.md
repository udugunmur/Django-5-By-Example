# Parte 3: Creación de una tienda online

## Capítulo 11: Adición de internacionalización a tu tienda

### Introducción

En el capítulo anterior, añadiste un sistema de cupones a tu tienda y construiste un motor de recomendaciones de productos.

En este capítulo, aprenderás cómo funcionan la internacionalización y la localización. Al hacer que tu aplicación sea accesible en múltiples idiomas, puedes atender a una gama más amplia de usuarios. Además, al adaptar tu aplicación a las convenciones de formato locales, como el formato de fechas o números, mejoras su usabilidad. Al traducir y localizar tu aplicación, la harás más intuitiva para usuarios de diferentes orígenes culturales y aumentarás la interacción (*user engagement*).

Este capítulo cubrirá los siguientes temas:

- Preparación de tu proyecto para la internacionalización
- Gestión de archivos de traducción
- Traducción de código Python
- Traducción de plantillas
- Uso de Rosetta para gestionar traducciones
- Traducción de patrones de URL y uso de un prefijo de idioma en las URLs
- Permitir a los usuarios cambiar de idioma
- Traducción de modelos usando `django-parler`
- Uso de traducciones de modelos con el ORM
- Adaptación de vistas para usar traducciones
- Uso de los campos de formulario localizados de `django-localflavor`

---

### Visión general funcional

La Figura 11.1 muestra una representación de las vistas, plantillas y funcionalidades que se construirán en este capítulo:

> *Figura 11.1: Diagrama de las funcionalidades construidas en el Capítulo 11*

En este capítulo, implementarás la internacionalización en tu proyecto y traducirás plantillas, URLs y modelos. Añadirás enlaces de selección de idioma en el encabezado de tu sitio y crearás URLs específicas para cada idioma. Modificarás las vistas `product_list` y `product_detail` de la aplicación `shop` para recuperar objetos `Category` y `Product` mediante sus slugs traducidos. También añadirás un campo de código postal localizado al formulario utilizado en la vista `order_create`.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter11](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter11).

Todos los módulos de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente que acompaña a este capítulo. Puedes seguir las instrucciones para instalar cada módulo de Python a continuación, o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Internacionalización con Django

Django ofrece soporte completo para la internacionalización y la localización. Te permite traducir tu aplicación a múltiples idiomas y gestiona el formato específico de la configuración regional (*locale*) para fechas, horas, números y zonas horarias. Aclaremos la diferencia entre internacionalización y localización:

- **Internacionalización** (frecuentemente abreviada como **i18n**): Es el proceso de adaptar el software para el uso potencial de diferentes idiomas y configuraciones regionales, de modo que no esté limitado a un idioma o región específicos.
- **Localización** (abreviada como **l10n**): Es el proceso de traducir efectivamente el software y adaptarlo a una configuración regional particular. El propio Django está traducido a más de 50 idiomas utilizando su framework de internacionalización.

El framework de internacionalización te permite marcar fácilmente cadenas para su traducción, tanto en código Python como en tus plantillas. Se apoya en el conjunto de herramientas GNU gettext para generar y gestionar archivos de mensajes. Un archivo de mensajes es un archivo de texto plano que representa un idioma. Contiene una parte, o la totalidad, de las cadenas de traducción encontradas en tu aplicación y sus respectivas traducciones para un único idioma. Los archivos de mensajes tienen la extensión `.po`. Una vez realizada la traducción, los archivos de mensajes se compilan para ofrecer un acceso rápido a las cadenas traducidas. Los archivos de traducción compilados tienen la extensión `.mo`.

Revisemos las configuraciones que proporciona Django para la internacionalización y localización.

#### Configuraciones de internacionalización y localización

Django proporciona varias configuraciones para la internacionalización. Las siguientes son las más relevantes:

- `USE_I18N`: Un booleano que especifica si el sistema de traducción de Django está habilitado. Es `True` de forma predeterminada.
- `USE_TZ`: Un booleano que especifica si las fechas y horas son conscientes de la zona horaria (*time-zone-aware*). Cuando creas un proyecto con el comando `startproject`, se establece en `True`.
- `LANGUAGE_CODE`: El código de idioma predeterminado para el proyecto. Está en el formato estándar de identificación de idioma, por ejemplo, `en-us` para inglés estadounidense o `en-gb` para inglés británico. Esta configuración requiere que `USE_I18N` esté establecido en `True` para que surta efecto. Puedes encontrar una lista de identificadores de idioma válidos en [http://www.i18nguy.com/unicode/language-identifiers.html](http://www.i18nguy.com/unicode/language-identifiers.html).
- `LANGUAGES`: Una tupla que contiene los idiomas disponibles para el proyecto. Vienen en tuplas de dos elementos con un código de idioma y un nombre de idioma. Puedes ver la lista de idiomas disponibles en `django.conf.global_settings`. Cuando eliges en qué idiomas estará disponible tu sitio, estableces `LANGUAGES` como un subconjunto de esa lista.
- `LOCALE_PATHS`: Una lista de directorios donde Django busca archivos de mensajes que contienen traducciones para el proyecto.
- `TIME_ZONE`: Una cadena que representa la zona horaria del proyecto. Se establece en `'UTC'` cuando creas un nuevo proyecto utilizando el comando `startproject`. Puedes configurarlo en cualquier otra zona horaria, como `'Europe/Madrid'`.

Puedes encontrar la lista completa de configuraciones de internacionalización y localización en [https://docs.djangoproject.com/en/5.2/ref/settings/#globalization-i18n-l10n](https://docs.djangoproject.com/en/5.2/ref/settings/#globalization-i18n-l10n).

#### Comandos de gestión de internacionalización

Django incluye los siguientes comandos de gestión para administrar las traducciones:

- `makemessages`: Recorre el árbol de código fuente para encontrar todas las cadenas marcadas para traducción y crea o actualiza los archivos de mensajes `.po` en el directorio `locale`. Se crea un único archivo `.po` para cada idioma.
- `compilemessages`: Compila los archivos de mensajes `.po` existentes en archivos `.mo`, que se utilizan para recuperar traducciones rápidamente.

Django depende del kit de herramientas gettext para generar y compilar archivos de traducción.

#### Instalación del conjunto de herramientas gettext

Necesitarás el conjunto de herramientas gettext para poder crear, actualizar y compilar archivos de mensajes. La mayoría de las distribuciones de Linux incluyen el kit de herramientas gettext. Si estás utilizando macOS, la forma más sencilla de instalarlo es a través de Homebrew ([https://brew.sh/](https://brew.sh/)), con el siguiente comando:

```bash
brew install gettext
```

Es posible que también necesites forzar su enlace con el siguiente comando:

```bash
brew link --force gettext
```

Si estás utilizando Windows, sigue los pasos descritos en [https://docs.djangoproject.com/en/5.2/topics/i18n/translation/#gettext-on-windows](https://docs.djangoproject.com/en/5.2/topics/i18n/translation/#gettext-on-windows). Puedes descargar un instalador binario precompilado de gettext para Windows desde [https://mlocati.github.io/articles/gettext-iconv-windows.html](https://mlocati.github.io/articles/gettext-iconv-windows.html).

#### Cómo añadir traducciones a un proyecto de Django

Este es el proceso necesario para traducir un proyecto de Django:

1. Marcar las cadenas para traducción en tu código Python y en tus plantillas.
2. Ejecutar el comando `makemessages` para crear o actualizar los archivos de mensajes que incluyen todas las cadenas de traducción de tu código.
3. Traducir las cadenas contenidas en los archivos de mensajes.
4. Compilar los archivos de mensajes utilizando el comando de gestión `compilemessages`.

#### Cómo determina Django el idioma actual

Django incluye un middleware que determina el idioma actual basándose en los datos de la petición: `django.middleware.locale.LocaleMiddleware`. Este middleware realiza las siguientes tareas:

1. Si estás utilizando `i18n_patterns` (patrones de URL traducidos), busca un prefijo de idioma en la URL solicitada para determinar el idioma actual.
2. Si no se encuentra ningún prefijo de idioma, busca una clave `LANGUAGE_SESSION_KEY` existente en la sesión del usuario actual.
3. Si el idioma no está configurado en la sesión, busca una cookie existente con el idioma actual (`django_language` de forma predeterminada, configurable mediante `LANGUAGE_COOKIE_NAME`).
4. Si no se encuentra ninguna cookie, busca el encabezado HTTP `Accept-Language` de la petición.
5. Si el encabezado `Accept-Language` no especifica un idioma, Django utiliza el idioma definido en la configuración `LANGUAGE_CODE`.

---

### Preparación de tu proyecto para la internacionalización

Prepararemos nuestro proyecto para utilizar diferentes idiomas. Vamos a crear una versión en inglés y otra en español para la tienda online.

Edita el archivo `settings.py` de tu proyecto y añade la configuración `LANGUAGES`. Colócala junto a la configuración `LANGUAGE_CODE`:

```python
LANGUAGES = [
    ('en', 'English'),
    ('es', 'Spanish'),
]
```

Asegúrate de que la configuración `LANGUAGE_CODE` sea la siguiente:

```python
LANGUAGE_CODE = 'en'
```

Añade `'django.middleware.locale.LocaleMiddleware'` a la configuración `MIDDLEWARE`. Asegúrate de que este middleware se ubique después de `SessionMiddleware` y antes de `CommonMiddleware`. La configuración `MIDDLEWARE` debería verse así:

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.locale.LocaleMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

Crea la siguiente estructura de directorios dentro del directorio principal del proyecto, junto al archivo `manage.py`:

```text
locale/
    en/
    es/
```

Edita nuevamente el archivo `settings.py` y añade la siguiente configuración:

```python
LOCALE_PATHS = [
    BASE_DIR / 'locale',
]
```

---

### Traducción de código Python

Para traducir literales en tu código Python, puedes marcar cadenas para traducción utilizando la función `gettext()` incluida en `django.utils.translation`. La convención es importar esta función con el alias `_` (el carácter de subrayado).

#### Traducciones estándar

El siguiente código muestra cómo marcar una cadena para traducción:

```python
from django.utils.translation import gettext as _
output = _('Text to be translated.')
```

#### Traducciones perezosas (Lazy translations)

Django incluye versiones perezosas (*lazy*) para todas sus funciones de traducción, las cuales tienen el sufijo `_lazy()`. Al usar funciones perezosas, las cadenas se traducen cuando se accede al valor, en lugar de cuando se llama a la función:

```python
from django.utils.translation import gettext_lazy as _
```

Las funciones de traducción perezosa son especialmente útiles cuando las cadenas marcadas para traducción se encuentran en rutas que se ejecutan al cargar los módulos (por ejemplo, en `models.py` o `settings.py`).

#### Traducciones que incluyen variables

Las cadenas marcadas para traducción pueden incluir marcadores de posición (*placeholders*) para incorporar variables:

```python
from django.utils.translation import gettext as _
month = _('April')
day = '14'
output = _('Today is %(month)s %(day)s') % {'month': month, 'day': day}
```

Al utilizar marcadores de posición, puedes reordenar las variables según el idioma.

#### Formas plurales en traducciones

Para formas plurales, Django proporciona `ngettext()` y `ngettext_lazy()`:

```python
output = ngettext(
    'there is %(count)d product',  # Singular form
    'there are %(count)d products',  # Plural form
    count  # Numeric value to determine form
) % {'count': count}
```

#### Traducción de tu propio código

Primero, traduciremos los nombres de los idiomas. Edita `settings.py`, importa `gettext_lazy` y modifica `LANGUAGES`:

```python
from django.utils.translation import gettext_lazy as _

# ...

LANGUAGES = [
    ('en', _('English')),
    ('es', _('Spanish')),
]
```

Ejecuta el siguiente comando desde el directorio de tu proyecto:

```bash
django-admin makemessages --all
```

Verás la siguiente salida:

```text
processing locale es
processing locale en
```

Abre `locale/es/LC_MESSAGES/django.po` con un editor de texto y completa las traducciones:

```po
#: myshop/settings.py:118
msgid "English"
msgstr "Inglés"

#: myshop/settings.py:119
msgid "Spanish"
msgstr "Español"
```

Guarda el archivo y ejecuta:

```bash
django-admin compilemessages
```

Ahora traduciremos los nombres de los campos de los modelos. Edita `orders/models.py`:

```python
from django.utils.translation import gettext_lazy as _


class Order(models.Model):
    first_name = models.CharField(_('first name'), max_length=50)
    last_name = models.CharField(_('last name'), max_length=50)
    email = models.EmailField(_('e-mail'))
    address = models.CharField(_('address'), max_length=250)
    postal_code = models.CharField(_('postal code'), max_length=20)
    city = models.CharField(_('city'), max_length=100)
    # ...
```

Crea la estructura `locale/` dentro del directorio de la aplicación `orders`:

```text
locale/
    en/
    es/
```

Ejecuta:

```bash
django-admin makemessages --all
```

Abre `orders/locale/es/LC_MESSAGES/django.po` y completa las traducciones:

```po
#: orders/models.py:12
msgid "first name"
msgstr "nombre"

#: orders/models.py:14
msgid "last name"
msgstr "apellidos"

#: orders/models.py:16
msgid "e-mail"
msgstr "e-mail"

#: orders/models.py:17
msgid "address"
msgstr "dirección"

#: orders/models.py:19
msgid "postal code"
msgstr "código postal"

#: orders/models.py:21
msgid "city"
msgstr "ciudad"
```

Edita `cart/forms.py` para traducir el formulario:

```python
from django import forms
from django.utils.translation import gettext_lazy as _

PRODUCT_QUANTITY_CHOICES = [(i, str(i)) for i in range(1, 21)]


class CartAddProductForm(forms.Form):
    quantity = forms.TypedChoiceField(
        choices=PRODUCT_QUANTITY_CHOICES,
        coerce=int,
        label=_('Quantity')
    )
    override = forms.BooleanField(
        required=False,
        initial=False,
        widget=forms.HiddenInput
    )
```

Edita `coupons/forms.py`:

```python
from django import forms
from django.utils.translation import gettext_lazy as _


class CouponApplyForm(forms.Form):
    code = forms.CharField(label=_('Coupon'))
```

---

### Traducción de plantillas

Django ofrece las etiquetas de plantilla `{% translate %}` y `{% blocktranslate %}` para traducir cadenas en plantillas. Para utilizarlas, debes añadir `{% load i18n %}` en la parte superior de tu plantilla.

#### La etiqueta de plantilla {% translate %}

Permite marcar un texto literal para traducción:

```html
{% translate "Text to be translated" %}
```

Puedes utilizar `as` para almacenar el contenido traducido en una variable:

```html
{% translate "Hello!" as greeting %}
<h1>{{ greeting }}</h1>
```

#### La etiqueta de plantilla {% blocktranslate %}

Permite marcar contenido que incluye texto literal y variables utilizando marcadores de posición:

```html
{% blocktranslate %}Hello {{ name }}!{% endblocktranslate %}
```

Puedes usar `with` para incluir expresiones de plantilla:

```html
{% blocktranslate with name=user.name|capfirst %}
    Hello {{ name }}!
{% endblocktranslate %}
```

#### Traducción de las plantillas de la tienda

Edita la plantilla `shop/base.html`:

```html
{% load i18n static %}
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>
        {% block title %}{% translate "My shop" %}{% endblock %}
    </title>
    <link href="{% static "css/base.css" %}" rel="stylesheet">
</head>
<body>
    <div id="header">
        <a href="/" class="logo">{% translate "My shop" %}</a>
    </div>
    <div id="subheader">
        <div class="cart">
            {% with total_items=cart|length %}
                {% if total_items > 0 %}
                    {% translate "Your cart" %}:
                    <a href="{% url "cart:cart_detail" %}">
                        {% blocktranslate with total=cart.get_total_price count items=total_items %}
                            {{ items }} item, ${{ total }}
                        {% plural %}
                            {{ items }} items, ${{ total }}
                        {% endblocktranslate %}
                    </a>
                {% elif not order %}
                    {% translate "Your cart is empty." %}
                {% endif %}
            {% endwith %}
        </div>
    </div>
    <div id="content">
        {% block content %}
        {% endblock %}
    </div>
</body>
</html>
```

Edita `shop/product/detail.html`:

```html
{% extends "shop/base.html" %}
{% load i18n static %}
...
```

Reemplaza los textos correspondientes con:

```html
<input type="submit" value="{% translate "Add to cart" %}">
```

```html
<h3>{% translate "People who bought this also bought" %}</h3>
```

Edita `orders/order/create.html`:

```html
{% extends "shop/base.html" %}
{% load i18n %}

{% block title %}
    {% translate "Checkout" %}
{% endblock %}

{% block content %}
    <h1>{% translate "Checkout" %}</h1>
    <div class="order-info">
        <h3>{% translate "Your order" %}</h3>
        <ul>
            {% for item in cart %}
                <li>
                    {{ item.quantity }}x {{ item.product.name }}
                    <span>${{ item.total_price }}</span>
                </li>
            {% endfor %}
            {% if cart.coupon %}
                <li>
                    {% blocktranslate with code=cart.coupon.code discount=cart.coupon.discount %}
                        "{{ code }}" ({{ discount }}% off)
                    {% endblocktranslate %}
                    <span class="neg">- ${{ cart.get_discount|floatformat:2 }}</span>
                </li>
            {% endif %}
        </ul>
        <p>{% translate "Total" %}: ${{ cart.get_total_price_after_discount|floatformat:2 }}</p>
    </div>
    <form method="post" class="order-form">
        {{ form.as_p }}
        <p><input type="submit" value="{% translate "Place order" %}"></p>
        {% csrf_token %}
    </form>
{% endblock %}
```

Actualiza los archivos de mensajes con:

```bash
django-admin makemessages --all
```

Traduce los archivos `.po` y compílalos con:

```bash
django-admin compilemessages
```

---

### Uso de la interfaz de traducción Rosetta

Rosetta es una aplicación de terceros que te permite editar traducciones directamente en el navegador utilizando la misma interfaz que el sitio de administración de Django.

Instala Rosetta vía pip:

```bash
python -m pip install django-rosetta==0.10.0
```

Añade `'rosetta'` a `INSTALLED_APPS` en `settings.py`:

```python
INSTALLED_APPS = [
    # ...
    'rosetta',
]
```

Edita `urls.py` principal del proyecto `myshop`:

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('cart/', include('cart.urls', namespace='cart')),
    path('orders/', include('orders.urls', namespace='orders')),
    path('payment/', include('payment.urls', namespace='payment')),
    path('coupons/', include('coupons.urls', namespace='coupons')),
    path('rosetta/', include('rosetta.urls')),
    path('', include('shop.urls', namespace='shop')),
]
```

Abre `http://127.0.0.1:8000/rosetta/` en tu navegador:

> *Figura 11.2: La interfaz de administración de Rosetta*

Haz clic en el enlace **Myshop** en la sección de español:

> *Figura 11.3: Edición de traducciones al español mediante Rosetta*

> *Figura 11.4: Traducciones que incluyen marcadores de posición*

Por ejemplo, la cadena:

```text
%(items)s items, $%(total)s
```

Se traduce al español como:

```text
%(items)s productos, $%(total)s
```

Haz clic en **Save and translate next block** para guardar.

---

### Traducciones difusas (Fuzzy translations)

Al editar traducciones en Rosetta, verás una columna **FUZZY**. Si la bandera `FUZZY` está activa para una traducción, no se incluirá en los archivos de mensajes compilados. Marca cadenas de traducción que necesitan ser revisadas por un traductor cuando gettext detecta que un `msgid` ha cambiado ligeramente. El traductor debe revisar el texto, desmarcar la casilla `FUZZY` y guardar los cambios.

---

### Patrones de URL para internacionalización

Django incluye dos características principales para URLs internacionalizadas:

1. **Prefijo de idioma en los patrones de URL**: Añade un prefijo como `/en/` o `/es/` a las URLs.
2. **Patrones de URL traducidos**: Permite traducir las propias rutas URL para cada idioma.

#### Adición de un prefijo de idioma a los patrones de URL

Edita el archivo `urls.py` principal de `myshop` y utiliza `i18n_patterns()`:

```python
from django.conf.urls.i18n import i18n_patterns

urlpatterns = i18n_patterns(
    path('admin/', admin.site.urls),
    path('cart/', include('cart.urls', namespace='cart')),
    path('orders/', include('orders.urls', namespace='orders')),
    path('payment/', include('payment.urls', namespace='payment')),
    path('coupons/', include('coupons.urls', namespace='coupons')),
    path('rosetta/', include('rosetta.urls')),
    path('', include('shop.urls', namespace='shop')),
)
```

Al acceder a `http://127.0.0.1:8000/`, serás redirigido automáticamente a `http://127.0.0.1:8000/en/` o `http://127.0.0.1:8000/es/`.

#### Traducción de patrones de URL

Marca las cadenas de las rutas URL para traducción utilizando `gettext_lazy()`. Edita el archivo `urls.py` principal de `myshop`:

```python
from django.utils.translation import gettext_lazy as _

urlpatterns = i18n_patterns(
    path('admin/', admin.site.urls),
    path(_('cart/'), include('cart.urls', namespace='cart')),
    path(_('orders/'), include('orders.urls', namespace='orders')),
    path(_('payment/'), include('payment.urls', namespace='payment')),
    path(_('coupons/'), include('coupons.urls', namespace='coupons')),
    path('rosetta/', include('rosetta.urls')),
    path('', include('shop.urls', namespace='shop')),
)
```

Edita `orders/urls.py`:

```python
from django.utils.translation import gettext_lazy as _

urlpatterns = [
    path(_('create/'), views.order_create, name='order_create'),
    # ...
]
```

Edita `payment/urls.py`:

```python
from django.utils.translation import gettext_lazy as _

urlpatterns = [
    path(_('process/'), views.payment_process, name='process'),
    path(_('completed/'), views.payment_completed, name='completed'),
    path(_('canceled/'), views.payment_canceled, name='canceled'),
]
```

Para asegurar una URL única sin prefijo para el webhook de Stripe, mantén su ruta fuera de `i18n_patterns()` en `myshop/urls.py`:

```python
from django.utils.translation import gettext_lazy as _
from payment import webhooks

urlpatterns = i18n_patterns(
    path('admin/', admin.site.urls),
    path(_('cart/'), include('cart.urls', namespace='cart')),
    path(_('orders/'), include('orders.urls', namespace='orders')),
    path(_('payment/'), include('payment.urls', namespace='payment')),
    path(_('coupons/'), include('coupons.urls', namespace='coupons')),
    path('rosetta/', include('rosetta.urls')),
    path('', include('shop.urls', namespace='shop')),
)

urlpatterns += [
    path(
        'payment/webhook/',
        webhooks.stripe_webhook,
        name='stripe-webhook'
    ),
]

if settings.DEBUG:
    urlpatterns += static(
        settings.MEDIA_URL,
        document_root=settings.MEDIA_ROOT
    )
```

Ejecuta `django-admin makemessages --all` y abre `http://127.0.0.1:8000/en/rosetta/` para traducir los patrones de URL:

> *Figura 11.5: Patrones de URL para traducción en la interfaz de Rosetta*

> *Figura 11.6: Traducciones al español para patrones de URL en la interfaz de Rosetta*

> *Figura 11.7: Traducciones difusas (fuzzy) en la interfaz de Rosetta*

> *Figura 11.8: Corrección de traducciones difusas en la interfaz de Rosetta*

---

### Permitir a los usuarios cambiar de idioma

Añadiremos un selector de idioma en el encabezado del sitio. Edita `shop/templates/shop/base.html`:

```html
<div id="header">
    <a href="/" class="logo">{% translate "My shop" %}</a>
    {% get_current_language as LANGUAGE_CODE %}
    {% get_available_languages as LANGUAGES %}
    {% get_language_info_list for LANGUAGES as languages %}
    <div class="languages">
        <p>{% translate "Language" %}:</p>
        <ul class="languages">
            {% for language in languages %}
                <li>
                    <a href="/{{ language.code }}/" {% if language.code == LANGUAGE_CODE %} class="selected"{% endif %}>
                        {{ language.name_local }}
                    </a>
                </li>
            {% endfor %}
        </ul>
    </div>
</div>
```

Abre `http://127.0.0.1:8000/` en tu navegador:

> *Figura 11.9: La página de lista de productos, incluyendo el selector de idioma en el encabezado (Créditos de imágenes: Té verde: Foto por Jia Ye en Unsplash; Té rojo: Foto por Manki Kim en Unsplash; Té en polvo: Foto por Phuong Nguyen en Unsplash)*

---

### Traducción de modelos con django-parler

`django-parler` genera una tabla de base de datos separada para cada modelo que contiene traducciones. Esta tabla incluye todos los campos traducidos, una clave foránea hacia el objeto original y un campo `language_code`.

#### Instalación de django-parler

Instala `django-parler` vía pip:

```bash
python -m pip install django-parler==2.3
```

Edita `settings.py` de `myshop` y añade `'parler'` a `INSTALLED_APPS` junto con su configuración:

```python
INSTALLED_APPS = [
    # ...
    'parler',
]

# django-parler settings
PARLER_LANGUAGES = {
    None: (
        {'code': 'en'},
        {'code': 'es'},
    ),
    'default': {
        'fallback': 'en',
        'hide_untranslated': False,
    }
}
```

#### Traducción de campos de modelos

Edita `shop/models.py`:

```python
from parler.models import TranslatableModel, TranslatedFields


class Category(TranslatableModel):
    translations = TranslatedFields(
        name = models.CharField(max_length=200),
        slug = models.SlugField(max_length=200, unique=True),
    )

    class Meta:
        # ordering = ['name']
        # indexes = [
        #     models.Index(fields=['name']),
        # ]
        verbose_name = 'category'
        verbose_name_plural = 'categories'

    def __str__(self):
        return self.name

    def get_absolute_url(self):
        return reverse(
            'shop:product_list_by_category',
            args=[self.slug]
        )


class Product(TranslatableModel):
    translations = TranslatedFields(
        name = models.CharField(max_length=200),
        slug = models.SlugField(max_length=200),
        description = models.TextField(blank=True)
    )
    category = models.ForeignKey(
        Category,
        related_name='products',
        on_delete=models.CASCADE
    )
    image = models.ImageField(
        upload_to='products/%Y/%m/%d',
        blank=True
    )
    price = models.DecimalField(max_digits=10, decimal_places=2)
    available = models.BooleanField(default=True)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

    class Meta:
        # ordering = ['name']
        indexes = [
            # models.Index(fields=['id', 'slug']),
            # models.Index(fields=['name']),
            models.Index(fields=['-created']),
        ]

    def __str__(self):
        return self.name

    def get_absolute_url(self):
        return reverse('shop:product_detail', args=[self.id, self.slug])
```

La Figura 11.10 muestra el modelo `Product` y el modelo `ProductTranslation` generado por `django-parler`:

> *Figura 11.10: El modelo Product y el modelo relacionado ProductTranslation generado por django-parler*

#### Integración de traducciones en el sitio de administración

Edita `shop/admin.py`:

```python
from django.contrib import admin
from parler.admin import TranslatableAdmin
from .models import Category, Product


@admin.register(Category)
class CategoryAdmin(TranslatableAdmin):
    list_display = ['name', 'slug']

    def get_prepopulated_fields(self, request, obj=None):
        return {'slug': ('name',)}


@admin.register(Product)
class ProductAdmin(TranslatableAdmin):
    list_display = [
        'name', 'slug', 'price', 'available', 'created', 'updated'
    ]
    list_filter = ['available', 'created', 'updated']
    list_editable = ['price', 'available']

    def get_prepopulated_fields(self, request, obj=None):
        return {'slug': ('name',)}
```

#### Creación de migraciones para traducciones de modelos

Genera la migración con:

```bash
python manage.py makemigrations shop --name "translations"
```

Edita el archivo `shop/migrations/0002_translations.py` y reemplaza las dos apariciones de:

```python
bases=(parler.models.TranslatedFieldsModelMixin, models.Model),
```

Por:

```python
bases=(parler.models.TranslatableModel, models.Model),
```

Aplica la migración:

```bash
python manage.py migrate shop
```

Abre `http://127.0.0.1:8000/en/admin/shop/category/` en tu navegador:

> *Figura 11.11: La lista de categorías en el sitio de administración tras crear los modelos de traducción*

Edita las categorías y productos introduciendo los valores en inglés y español:

> *Figura 11.12: El formulario de edición de categoría con pestañas de idioma*

> *Figura 11.13: La traducción al español del formulario de edición de categoría*

#### Uso de traducciones en QuerySets

En la shell de Python (`python manage.py shell`):

```python
>>> from shop.models import Product
>>> from django.utils.translation import activate
>>> activate('es')
>>> product = Product.objects.first()
>>> product.name
'Té verde'
```

Usando el manager `language()`:

```python
>>> product = Product.objects.language('en').first()
>>> product.name
'Green tea'
```

Cambiando el idioma actual de una instancia:

```python
>>> product.set_current_language('es')
>>> product.name
'Té verde'
>>> product.get_current_language()
'es'
```

Filtrando por campos traducidos:

```python
>>> Product.objects.filter(translations__name='Green tea')
<TranslatableQuerySet [<Product: Té verde>]>
```

#### Adaptación de vistas para traducciones

Edita `shop/views.py`:

```python
def product_list(request, category_slug=None):
    category = None
    categories = Category.objects.all()
    products = Product.objects.filter(available=True)
    if category_slug:
        language = request.LANGUAGE_CODE
        category = get_object_or_404(
            Category,
            translations__language_code=language,
            translations__slug=category_slug
        )
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

```python
def product_detail(request, id, slug):
    language = request.LANGUAGE_CODE
    product = get_object_or_404(
        Product,
        id=id,
        translations__language_code=language,
        translations__slug=slug,
        available=True
    )
    cart_product_form = CartAddProductForm()
    r = Recommender()
    recommended_products = r.suggest_products_for([product], 4)
    return render(
        request,
        'shop/product/detail.html',
        {
            'product': product,
            'cart_product_form': cart_product_form,
            'recommended_products': recommended_products
        }
    )
```

Abre `http://127.0.0.1:8000/es/` en tu navegador:

> *Figura 11.14: La versión en español de la página de lista de productos*

> *Figura 11.15: La versión en español de la página de detalle de producto*

---

### Localización de formatos

Desde Django 5.0, el formateo localizado de datos está siempre habilitado. Django muestra números y fechas utilizando el formato de la configuración regional activa.

> *Figura 11.16: Localización de formato en inglés y español*

Puedes activar o desactivar la localización en fragmentos de plantilla mediante la etiqueta `{% localize %}`:

```html
{% load l10n %}
{% localize on %}
    {{ value }}
{% endlocalize %}
{% localize off %}
    {{ value }}
{% endlocalize %}
```

O usando los filtros `localize` y `unlocalize`:

```html
{{ value|localize }}
{{ value|unlocalize }}
```

---

### Uso de django-localflavor para validar campos de formularios

`django-localflavor` proporciona componentes específicos por país (campos de formulario, campos de modelo y validadores).

Instala `django-localflavor` vía pip:

```bash
python -m pip install django-localflavor==4.0
```

Añade `'localflavor'` a `INSTALLED_APPS` en `settings.py`:

```python
INSTALLED_APPS = [
    # ...
    'localflavor',
]
```

Edita `orders/forms.py` para usar `USZipCodeField`:

```python
from django import forms
from localflavor.us.forms import USZipCodeField
from .models import Order


class OrderCreateForm(forms.ModelForm):
    postal_code = USZipCodeField()

    class Meta:
        model = Order
        fields = [
            'first_name', 'last_name', 'email',
            'address', 'postal_code', 'city'
        ]
```

Abre `http://127.0.0.1:8000/en/orders/create/` e ingresa un código postal inválido para ver el error de validación:

```text
Enter a zip code in the format XXXXX or XXXXX-XXXX.
```

> *Figura 11.17: El error de validación para un código postal de EE. UU. inválido*

---

### Ampliación de tu proyecto mediante IA

En esta sección, se te presenta una tarea para ampliar tu proyecto junto con un prompt de ejemplo para interactuar con ChatGPT en [https://chatgpt.com/](https://chatgpt.com/).

En este proyecto de tienda online hemos añadido pedidos, pagos, cupones y recomendaciones. Otra característica típica del comercio electrónico es la gestión de gastos de envío. Considera añadir un atributo `weight` a los productos e implementar gastos de envío basados en el peso total de los artículos enviados. Utiliza ChatGPT para ayudarte a implementar esta funcionalidad y garantizar que Stripe cobre el importe correcto con los gastos de envío calculados. Puedes consultar el prompt disponible en [https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter11/prompts/task.md](https://github.com/PacktPublishing/Django-5-by-example/blob/main/Chapter11/prompts/task.md).

---

### Resumen

En este capítulo, aprendiste los aspectos fundamentales de la internacionalización y localización en proyectos Django. Marcaste cadenas de código y plantillas para traducción, y aprendiste a generar y compilar archivos de mensajes. Instalaste Rosetta para gestionar traducciones desde la web, tradujiste patrones de URL y creaste un selector de idioma. Utilizaste `django-parler` para traducir modelos y `django-localflavor` para validar campos de formularios específicos por país.

En el próximo capítulo, comenzarás un nuevo proyecto que consistirá en una plataforma de e-learning. Aprenderás sobre herencia de modelos para implementar polimorfismo, sentarás las bases de un sistema de gestión de contenidos (*CMS*) flexible, crearás fixtures de datos iniciales y construirás campos de modelo personalizados.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter11](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter11)
- **Lista de identificadores de idioma válidos:** [http://www.i18nguy.com/unicode/language-identifiers.html](http://www.i18nguy.com/unicode/language-identifiers.html)
- **Configuraciones de internacionalización y localización de Django:** [https://docs.djangoproject.com/en/5.2/ref/settings/#globalization-i18n-l10n](https://docs.djangoproject.com/en/5.2/ref/settings/#globalization-i18n-l10n)
- **Gestor de paquetes Homebrew:** [https://brew.sh/](https://brew.sh/)
- **Instalación de gettext en Windows:** [https://docs.djangoproject.com/en/5.2/topics/i18n/translation/#gettext-on-windows](https://docs.djangoproject.com/en/5.2/topics/i18n/translation/#gettext-on-windows)
- **Instalador binario precompilado de gettext para Windows:** [https://mlocati.github.io/articles/gettext-iconv-windows.html](https://mlocati.github.io/articles/gettext-iconv-windows.html)
- **Documentación sobre traducciones en Django:** [https://docs.djangoproject.com/en/5.2/topics/i18n/translation/](https://docs.djangoproject.com/en/5.2/topics/i18n/translation/)
- **Editor de archivos de traducción Poedit:** [https://poedit.net/](https://poedit.net/)
- **Documentación de Django Rosetta:** [https://django-rosetta.readthedocs.io/](https://django-rosetta.readthedocs.io/)
- **Compatibilidad de django-parler con Django:** [https://django-parler.readthedocs.io/en/latest/compatibility.html](https://django-parler.readthedocs.io/en/latest/compatibility.html)
- **Documentación de django-parler:** [https://django-parler.readthedocs.io/en/latest/](https://django-parler.readthedocs.io/en/latest/)
- **Configuración de formatos de Django para inglés:** [https://github.com/django/django/blob/stable/5.2.x/django/conf/locale/en/formats.py](https://github.com/django/django/blob/stable/5.2.x/django/conf/locale/en/formats.py)
- **Configuración de formatos de Django para español:** [https://github.com/django/django/blob/stable/5.2.x/django/conf/locale/es/formats.py](https://github.com/django/django/blob/stable/5.2.x/django/conf/locale/es/formats.py)
- **Localización de formatos en Django:** [https://docs.djangoproject.com/en/5.2/topics/i18n/formatting/](https://docs.djangoproject.com/en/5.2/topics/i18n/formatting/)
- **Documentación de django-localflavor:** [https://django-localflavor.readthedocs.io/en/latest/](https://django-localflavor.readthedocs.io/en/latest/)
