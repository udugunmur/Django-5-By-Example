# Parte 3: Creación de una tienda online

## Capítulo 9: Gestión de pagos y pedidos

### Introducción

En el capítulo anterior, creaste una tienda online básica con un catálogo de productos y un carrito de compras. Aprendiste a usar sesiones de Django y construiste un procesador de contexto personalizado. También aprendiste a lanzar tareas asíncronas utilizando Celery y RabbitMQ.

En este capítulo, aprenderás a integrar una pasarela de pago en tu sitio para permitir que los usuarios paguen con tarjeta de crédito y gestionar los pagos de los pedidos. También ampliarás el sitio de administración con diferentes funcionalidades.

En este capítulo, aprenderás a:

- Integrar la pasarela de pago Stripe en tu proyecto
- Procesar pagos con tarjeta de crédito con Stripe
- Gestionar notificaciones de pago y marcar pedidos como pagados
- Exportar pedidos a archivos CSV
- Crear vistas personalizadas para el sitio de administración
- Generar facturas en PDF dinámicamente

---

### Visión general funcional

La Figura 9.1 muestra una representación de las vistas, plantillas y funcionalidades que se construirán en este capítulo:

> *Figura 9.1: Diagrama de las funcionalidades construidas en el Capítulo 9*

En este capítulo, crearás una nueva aplicación `payment`, donde implementarás la vista `payment_process` para iniciar una sesión de pago (*checkout session*) para pagar pedidos con Stripe. Construirás la vista `payment_completed` para redirigir a los usuarios tras pagos exitosos y la vista `payment_canceled` para redirigir a los usuarios si el pago se cancela. Implementarás la acción de administración `export_to_csv` para exportar pedidos en formato CSV en el sitio de administración. También construirás la vista de administración `admin_order_detail` para mostrar detalles del pedido y la vista `admin_order_pdf` para generar facturas en PDF dinámicamente. Implementarás el webhook `stripe_webhook` para recibir notificaciones de pago asíncronas de Stripe, y crearás la tarea asíncrona `payment_completed` para enviar facturas a los clientes cuando los pedidos estén pagados.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter09](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter09).

Todos los paquetes de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente del capítulo. Puedes seguir las instrucciones para instalar cada paquete de Python en las siguientes secciones, o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Integración de una pasarela de pago

Una pasarela de pago (*payment gateway*) es una tecnología utilizada por los comerciantes para procesar pagos de clientes en línea. Mediante una pasarela de pago, puedes gestionar los pedidos de los clientes y delegar el procesamiento del pago a un tercero confiable y seguro. Al utilizar una pasarela de pago de confianza, no tendrás que preocuparte por la complejidad técnica, de seguridad y regulatoria de procesar tarjetas de crédito en tu propio sistema.

Hay varios proveedores de pasarelas de pago para elegir. Vamos a integrar Stripe, que es una pasarela de pago muy popular utilizada por servicios en línea como Shopify, Uber, Twitch y GitHub, entre otros.

Stripe proporciona una Interfaz de Programación de Aplicaciones (API) que te permite procesar pagos en línea con múltiples métodos de pago, como tarjeta de crédito, Google Pay y Apple Pay. Puedes obtener más información sobre Stripe en [https://www.stripe.com/](https://www.stripe.com/).

Stripe ofrece diferentes productos relacionados con el procesamiento de pagos. Puede gestionar pagos únicos, pagos recurrentes para servicios de suscripción, pagos multipartitos para plataformas y mercados (*marketplaces*), y más.

Stripe ofrece diferentes métodos de integración, desde formularios de pago alojados por Stripe hasta flujos de pago totalmente personalizables. Integraremos el producto Stripe Checkout, que consiste en una página de pago optimizada para la conversión. Los usuarios podrán pagar fácilmente con tarjeta de crédito u otros métodos de pago por los artículos que pidan. Recibiremos notificaciones de pago de Stripe. Puedes ver la documentación de Stripe Checkout en [https://stripe.com/docs/payments/checkout](https://stripe.com/docs/payments/checkout).

Al aprovechar Stripe Checkout para procesar pagos, confías en una solución que es segura y cumple con los requisitos del Estándar de Seguridad de Datos para la Industria de Tarjeta de Pago (PCI DSS). Podrás cobrar pagos de Google Pay, Apple Pay, Afterpay, Alipay, débitos directos SEPA, débitos directos Bacs, débitos directos BECS, iDEAL, Sofort, GrabPay, FPX y otros métodos de pago.

#### Creación de una cuenta de Stripe

Necesitas una cuenta de Stripe para integrar la pasarela de pago en tu sitio. Creemos una cuenta para probar la API de Stripe. Abre [https://dashboard.stripe.com/register](https://dashboard.stripe.com/register) en tu navegador.

Verás un formulario como el siguiente:

> *Figura 9.2: El formulario de registro de Stripe*

Completa el formulario con tus propios datos y haz clic en **Create account**. Recibirás un correo electrónico de Stripe con un enlace para verificar tu dirección de correo electrónico:

> *Figura 9.3: El correo de verificación para confirmar tu dirección de correo electrónico*

Abre el correo en tu bandeja de entrada y haz clic en **Verify email**.

Serás redirigido a la pantalla del panel de control de Stripe, que se verá así:

> *Figura 9.4: El panel de control de Stripe después de verificar la dirección de correo electrónico*

En la parte superior derecha de la pantalla, puedes ver que el modo de prueba (*Test mode*) está activado. Stripe te proporciona un entorno de prueba y un entorno de producción. Si posees un negocio o eres trabajador independiente, puedes añadir los datos de tu empresa para activar la cuenta y obtener acceso para procesar pagos reales. Sin embargo, esto no es necesario para implementar y probar pagos a través de Stripe, ya que trabajaremos en el entorno de prueba.

Debes añadir un nombre de cuenta para procesar pagos. Abre [https://dashboard.stripe.com/settings/account](https://dashboard.stripe.com/settings/account) en tu navegador:

> *Figura 9.5: La configuración de la cuenta de Stripe*

En **Account name**, introduce el nombre que elijas y luego haz clic en **Save**. Vuelve al panel de control de Stripe. Verás el nombre de tu cuenta mostrado en el encabezado:

> *Figura 9.6: El encabezado del panel de control de Stripe incluyendo el nombre de la cuenta*

Continuaremos instalando el SDK de Python de Stripe y añadiendo Stripe a nuestro proyecto de Django.

#### Instalación de la biblioteca de Python de Stripe

Stripe proporciona una biblioteca de Python que simplifica el trato con su API. Vamos a integrar la pasarela de pago en el proyecto utilizando la biblioteca `stripe`.

Puedes encontrar el código fuente de la biblioteca de Python de Stripe en [https://github.com/stripe/stripe-python](https://github.com/stripe/stripe-python).

Instala la biblioteca `stripe` desde la consola utilizando el siguiente comando:

```bash
python -m pip install stripe==12.2.0
```

#### Adición de Stripe a tu proyecto

Abre [https://dashboard.stripe.com/test/apikeys](https://dashboard.stripe.com/test/apikeys) en tu navegador. También puedes acceder a esta página desde el panel de control de Stripe haciendo clic en **Developers** y luego en **API keys**. Verás la siguiente pantalla:

> *Figura 9.7: La pantalla de claves API de prueba de Stripe*

Stripe proporciona un par de claves para los dos entornos diferentes: prueba (*test*) y producción (*live*). Hay una clave publicable (*Publishable key*) y una clave secreta (*Secret key*) para cada entorno. Las claves publicables del modo de prueba tienen el prefijo `pk_test_` y las claves publicables del modo de producción tienen el prefijo `pk_live_`. Las claves secretas del modo de prueba tienen el prefijo `sk_test_` y las claves secretas del modo de producción tienen el prefijo `sk_live_`.

Necesitarás esta información para autenticar las peticiones a la API de Stripe. Siempre debes mantener tu clave privada en secreto y almacenarla de forma segura. La clave publicable se puede utilizar en código del lado del cliente, como scripts de JavaScript. Puedes leer más sobre las claves API de Stripe en [https://stripe.com/docs/keys](https://stripe.com/docs/keys).

Para facilitar la separación de la configuración del código, vamos a utilizar `python-decouple`. Ya utilizaste esta biblioteca en el Capítulo 2, *Mejora de tu blog y adición de funciones sociales*.

Crea un nuevo archivo dentro del directorio raíz de tu proyecto y nómbralo `.env`. El archivo `.env` contendrá pares clave-valor de variables de entorno. Añade las credenciales de Stripe al nuevo archivo, de la siguiente manera:

```text
STRIPE_PUBLISHABLE_KEY=pk_test_XXXX
STRIPE_SECRET_KEY=sk_test_XXXX
```

Reemplaza los valores de `STRIPE_PUBLISHABLE_KEY` y `STRIPE_SECRET_KEY` con los valores de la clave publicable y secreta de prueba proporcionados por Stripe.

> [!IMPORTANT]
> Si estás utilizando un repositorio git para tu código, asegúrate de incluir `.env` en el archivo `.gitignore` de tu repositorio. Al hacerlo, garantizas que las credenciales queden excluidas del repositorio.

Instala `python-decouple` mediante pip ejecutando el siguiente comando:

```bash
python -m pip install python-decouple==3.8
```

Edita el archivo `settings.py` de tu proyecto y añádele el siguiente código:

```python
from decouple import config

# ...

STRIPE_PUBLISHABLE_KEY = config('STRIPE_PUBLISHABLE_KEY')
STRIPE_SECRET_KEY = config('STRIPE_SECRET_KEY')
STRIPE_API_VERSION = '2024-04-10'
```

Utilizarás la versión `2024-04-10` de la API de Stripe. Puedes ver las notas de la versión para esta versión de la API en [https://stripe.com/docs/upgrades#2024-04-10](https://stripe.com/docs/upgrades#2024-04-10).

Estás utilizando las claves del entorno de prueba para el proyecto. Una vez que pases a producción y valides tu cuenta de Stripe, obtendrás las claves del entorno de producción.

Integremos la pasarela de pago en el proceso de compra. Puedes encontrar la documentación de Python para Stripe en [https://stripe.com/docs/api?lang=python](https://stripe.com/docs/api?lang=python).

#### Construcción del proceso de pago

El proceso de tramitación del pedido funcionará de la siguiente manera:

1. Añadir artículos al carrito de compras.
2. Tramitar el carrito de compras (*checkout*).
3. Introducir los datos de la tarjeta de crédito y pagar.

Vamos a crear una nueva aplicación para gestionar los pagos. Crea una nueva aplicación en tu proyecto utilizando el siguiente comando:

```bash
python manage.py startapp payment
```

Edita el archivo `settings.py` del proyecto y añade la nueva aplicación a la configuración `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    # ...
    'cart.apps.CartConfig',
    'orders.apps.OrdersConfig',
    'payment.apps.PaymentConfig',
    'shop.apps.ShopConfig',
]
```

La aplicación `payment` ahora está activa en el proyecto.

Actualmente, los usuarios pueden realizar pedidos pero no pueden pagarlos. Después de que los clientes realicen un pedido, debemos redirigirlos al proceso de pago.

Edita el archivo `views.py` de la aplicación `orders` e incluye la siguiente importación:

```python
from django.shortcuts import redirect, render
```

En el mismo archivo, busca las siguientes líneas de la vista `order_create`:

```python
# launch asynchronous task
order_created.delay(order.id)
return render(
    request,
    'orders/order/created.html',
    {'order': order}
)
```

Reemplázalas con el siguiente código:

```python
# launch asynchronous task
order_created.delay(order.id)
# set the order in the session
request.session['order_id'] = order.id
# redirect for payment
return redirect('payment:process')
```

La vista editada debería verse de la siguiente manera:

```python
from django.shortcuts import redirect, render
# ...


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
            # launch asynchronous task
            order_created.delay(order.id)
            # set the order in the session
            request.session['order_id'] = order.id
            # redirect for payment
            return redirect('payment:process')
    else:
        form = OrderCreateForm()
    return render(
        request,
        'orders/order/create.html',
        {'cart': cart, 'form': form}
    )
```

En lugar de renderizar la plantilla `orders/order/created.html` al realizar un nuevo pedido, el ID del pedido se almacena en la sesión del usuario y el usuario es redirigido a la URL `payment:process`. Implementaremos esta URL más adelante. Recuerda que Celery debe estar ejecutándose para que la tarea `order_created` se ponga en cola y se ejecute.

##### Integración de Stripe Checkout

La integración de Stripe Checkout consta de una página de pago alojada por Stripe que permite al usuario introducir los detalles del pago, normalmente una tarjeta de crédito, y luego cobra el pago. Si el pago es exitoso, Stripe redirige al cliente a una página de éxito. Si el cliente cancela el pago, lo redirige a una página de cancelación.

Implementaremos tres vistas:

- `payment_process`: Crea una sesión de Stripe Checkout y redirige al cliente al formulario de pago alojado por Stripe. Una sesión de checkout es una representación programática de lo que el cliente ve cuando es redirigido al formulario de pago, incluidos los productos, las cantidades, la moneda y el importe a cobrar.
- `payment_completed`: Muestra un mensaje para pagos exitosos. El usuario es redirigido a esta vista si el pago tiene éxito.
- `payment_canceled`: Muestra un mensaje para pagos cancelados. El usuario es redirigido a esta vista si se cancela el pago.

La Figura 9.8 muestra el flujo de pago del proceso de compra:

> *Figura 9.8: El flujo de pago del proceso de compra*

El proceso completo funcionará de la siguiente manera:

1. Después de crear un pedido, el usuario es redirigido a la vista `payment_process`. Al usuario se le presenta un resumen del pedido y un botón para proceder con el pago.
2. Cuando el usuario procede a pagar, se crea una sesión de Stripe Checkout. La sesión de checkout incluye la lista de artículos que el usuario comprará, una URL para redirigir al usuario tras un pago exitoso y una URL para redirigir al usuario si el pago se cancela.
3. La vista redirige al usuario a la página de pago alojada por Stripe. Esta página incluye el formulario de pago. El cliente introduce los datos de su tarjeta de crédito y envía el formulario.
4. Stripe procesa el pago y redirige al cliente a la vista `payment_completed`. Si el cliente no completa el pago, Stripe redirige al cliente a la vista `payment_canceled` en su lugar.

Comencemos a construir las vistas de pago. Edita el archivo `views.py` de la aplicación `payment` y añádele el siguiente código:

```python
from decimal import Decimal
import stripe
from django.conf import settings
from django.shortcuts import get_object_or_404, redirect, render
from django.urls import reverse
from orders.models import Order

# create the Stripe instance
stripe.api_key = settings.STRIPE_SECRET_KEY
stripe.api_version = settings.STRIPE_API_VERSION


def payment_process(request):
    order_id = request.session.get('order_id')
    order = get_object_or_404(Order, id=order_id)
    if request.method == 'POST':
        success_url = request.build_absolute_uri(
            reverse('payment:completed')
        )
        cancel_url = request.build_absolute_uri(
            reverse('payment:canceled')
        )
        # Stripe checkout session data
        session_data = {
            'mode': 'payment',
            'client_reference_id': order.id,
            'success_url': success_url,
            'cancel_url': cancel_url,
            'line_items': []
        }
        # add order items to the Stripe checkout session
        for item in order.items.all():
            session_data['line_items'].append(
                {
                    'price_data': {
                        'unit_amount': int(item.price * Decimal('100')),
                        'currency': 'usd',
                        'product_data': {
                            'name': item.product.name,
                        },
                    },
                    'quantity': item.quantity,
                }
            )
        # create Stripe checkout session
        session = stripe.checkout.Session.create(**session_data)
        # redirect to Stripe payment form
        return redirect(session.url, code=303)
    else:
        return render(request, 'payment/process.html', locals())
```

En el código anterior, se importa el módulo `stripe` y se establece la clave API de Stripe utilizando el valor de la configuración `STRIPE_SECRET_KEY`. La versión de la API que se utilizará también se establece utilizando el valor de la configuración `STRIPE_API_VERSION`.

La vista `payment_process` realiza las siguientes tareas:

- El objeto `Order` actual se recupera de la base de datos utilizando la clave de sesión `order_id`.
- Se recupera el objeto `Order` para el ID dado mediante `get_object_or_404()`.
- Si la vista se carga con una petición GET, se renderiza la plantilla `payment/process.html`. Esta plantilla incluirá el resumen del pedido y un botón para proceder con el pago mediante POST.
- Si la vista se carga con una petición POST, se crea una sesión de Stripe Checkout con `stripe.checkout.Session.create()` utilizando los siguientes parámetros:
  - `mode`: El modo de la sesión de pago. Usamos `payment` para un pago único.
  - `client_reference_id`: La referencia única para este pago. Usaremos esto para conciliar la sesión de Stripe Checkout con nuestro pedido. Al pasar el ID del pedido, vinculamos los pagos de Stripe a los pedidos en nuestro sistema.
  - `success_url`: La URL para que Stripe redirija al usuario si el pago es exitoso. Usamos `request.build_absolute_uri()` para generar un URI absoluto.
  - `cancel_url`: La URL para que Stripe redirija al usuario si el pago se cancela.
  - `line_items`: La lista de artículos del pedido que se van a comprar. Para cada artículo, definimos:
    - `unit_amount`: El importe en centavos a cobrar. Es un entero positivo sin decimales. Multiplicamos `item.price` por `Decimal('100')` y lo convertimos a entero.
    - `currency`: La moneda en formato ISO de tres letras (usamos `usd`).
    - `product_data`: Información del producto (su nombre).
    - `quantity`: El número de unidades compradas.
- Tras crear la sesión de checkout, se devuelve una redirección HTTP con código de estado `303` para redirigir al usuario a Stripe.

Añadamos las vistas para las páginas de éxito y cancelación del pago. Añade el siguiente código al archivo `views.py` de la aplicación `payment`:

```python
def payment_completed(request):
    return render(request, 'payment/completed.html')


def payment_canceled(request):
    return render(request, 'payment/canceled.html')
```

Crea un nuevo archivo dentro del directorio de la aplicación `payment` y nómbralo `urls.py`. Añade el siguiente código:

```python
from django.urls import path
from . import views

app_name = 'payment'

urlpatterns = [
    path('process/', views.payment_process, name='process'),
    path('completed/', views.payment_completed, name='completed'),
    path('canceled/', views.payment_canceled, name='canceled'),
]
```

Edita el archivo `urls.py` principal del proyecto `myshop` e incluye los patrones de URL para la aplicación `payment`:

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('cart/', include('cart.urls', namespace='cart')),
    path('orders/', include('orders.urls', namespace='orders')),
    path('payment/', include('payment.urls', namespace='payment')),
    path('', include('shop.urls', namespace='shop')),
]
```

Crea la siguiente estructura de archivos dentro del directorio de la aplicación `payment`:

```text
templates/
    payment/
        process.html
        completed.html
        canceled.html
```

Edita la plantilla `payment/process.html` y añade el siguiente código:

```html
{% extends "shop/base.html" %}
{% load static %}

{% block title %}Pay your order{% endblock %}

{% block content %}
    <h1>Order summary</h1>
    <table class="cart">
        <thead>
            <tr>
                <th>Image</th>
                <th>Product</th>
                <th>Price</th>
                <th>Quantity</th>
                <th>Total</th>
            </tr>
        </thead>
        <tbody>
            {% for item in order.items.all %}
                <tr class="row{% cycle "1" "2" %}">
                    <td>
                        <img src="{% if item.product.image %}{{ item.product.image.url }} {% else %}{% static "img/no_image.png" %}{% endif %}">
                    </td>
                    <td>{{ item.product.name }}</td>
                    <td class="num">${{ item.price }}</td>
                    <td class="num">{{ item.quantity }}</td>
                    <td class="num">${{ item.get_cost }}</td>
                </tr>
            {% endfor %}
            <tr class="total">
                <td colspan="4">Total</td>
                <td class="num">${{ order.get_total_cost }}</td>
            </tr>
        </tbody>
    </table>
    <form action="{% url "payment:process" %}" method="post">
        <input type="submit" value="Pay now">
        {% csrf_token %}
    </form>
{% endblock %}
```

Edita la plantilla `payment/completed.html` y añade el siguiente código:

```html
{% extends "shop/base.html" %}

{% block title %}Payment successful{% endblock %}

{% block content %}
    <h1>Your payment was successful</h1>
    <p>Your payment has been processed successfully.</p>
{% endblock %}
```

Edita la plantilla `payment/canceled.html` y añade el siguiente código:

```html
{% extends "shop/base.html" %}

{% block title %}Payment canceled{% endblock %}

{% block content %}
    <h1>Your payment has not been processed</h1>
    <p>There was a problem processing your payment.</p>
{% endblock %}
```

##### Prueba del proceso de pago

Ejecuta RabbitMQ con Docker:

```bash
docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.13.1-management
```

Abre otra consola e inicia el worker de Celery:

```bash
celery -A myshop worker -l info
```

Abre otra consola e inicia el servidor de desarrollo:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/` en tu navegador, añade algunos productos al carrito y completa el formulario de pedido. Haz clic en **Place order**. El pedido persistirá en la base de datos y serás redirigido a la página del proceso de pago:

> *Figura 9.9: La página del proceso de pago incluyendo el resumen del pedido*

Haz clic en **Pay now**. La vista `payment_process` creará una sesión de Stripe Checkout y serás redirigido al formulario de pago de Stripe:

> *Figura 9.10: El formulario de pago de Stripe Checkout*

##### Uso de tarjetas de crédito de prueba

Stripe proporciona diferentes tarjetas de crédito de prueba para simular diferentes escenarios:

| Resultado | Tarjeta de crédito de prueba | CVC | Fecha de caducidad |
| --- | --- | --- | --- |
| Pago exitoso (*Successful payment*) | `4242 4242 4242 4242` | Cualquier 3 dígitos | Cualquier fecha futura |
| Pago fallido (*Failed payment*) | `4000 0000 0000 0002` | Cualquier 3 dígitos | Cualquier fecha futura |
| Requiere autenticación 3D Secure | `4000 0025 0000 3155` | Cualquier 3 dígitos | Cualquier fecha futura |

Puedes encontrar la lista completa de tarjetas de prueba en [https://stripe.com/docs/testing](https://stripe.com/docs/testing).

Introduce los datos de la tarjeta de crédito válida `4242 4242 4242 4242`, CVC `123` y fecha `12/29`:

> *Figura 9.11: El formulario de pago con los datos de la tarjeta de crédito de prueba válida*

Haz clic en el botón **Pay**:

> *Figura 9.12: El formulario de pago procesándose*

> *Figura 9.13: El formulario de pago tras el pago exitoso*

Stripe redirige a la URL de pago completado:

> *Figura 9.14: La página de pago exitoso*

##### Comprobación de la información de pago en el panel de control de Stripe

Accede al panel de control de Stripe en [https://dashboard.stripe.com/test/payments](https://dashboard.stripe.com/test/payments). En **Payments**, podrás ver el pago con estado **Succeeded**:

> *Figura 9.15: El objeto de pago con el estado Succeeded en el panel de control de Stripe*

Haz clic en la transacción para ver los detalles:

> *Figura 9.16: Detalles de pago para una transacción de Stripe*

> *Figura 9.17: Método de pago utilizado en la transacción de Stripe*

> *Figura 9.18: Eventos y registros para una transacción de Stripe*

#### Uso de webhooks para recibir notificaciones de pago

Stripe puede enviar eventos en tiempo real a nuestra aplicación mediante **webhooks** (llamadas de retorno o *callbacks*). Un webhook es una API orientada a eventos (*event-driven API*). En lugar de sondear la API de Stripe con frecuencia, Stripe envía una petición HTTP a una URL de nuestra aplicación para notificarnos pagos exitosos en tiempo real.

##### Creación de un endpoint de webhook

Abre [https://dashboard.stripe.com/test/webhooks](https://dashboard.stripe.com/test/webhooks) en tu navegador:

> *Figura 9.19: La pantalla predeterminada de webhooks de Stripe*

Haz clic en **Test in a local environment**:

> *Figura 9.20: La pantalla de configuración del webhook de Stripe*

Copia el valor de `endpoint_secret`.

Edita el archivo `.env` de tu proyecto y añade la variable de entorno:

```text
STRIPE_PUBLISHABLE_KEY=pk_test_XXXX
STRIPE_SECRET_KEY=sk_test_XXXX
STRIPE_WEBHOOK_SECRET=whsec_XXXX
```

Edita el archivo `settings.py` de `myshop` y añade la configuración:

```python
STRIPE_PUBLISHABLE_KEY = config('STRIPE_PUBLISHABLE_KEY')
STRIPE_SECRET_KEY = config('STRIPE_SECRET_KEY')
STRIPE_API_VERSION = '2024-04-10'
STRIPE_WEBHOOK_SECRET = config('STRIPE_WEBHOOK_SECRET')
```

Stripe firma los eventos de webhook que envía incluyendo un encabezado `Stripe-Signature`. Al verificar la firma, nos aseguramos de que los eventos fueron enviados por Stripe y no por un atacante.

Crea un nuevo archivo en el directorio `payment/` y nómbralo `webhooks.py`. Añade el siguiente código:

```python
import stripe
from django.conf import settings
from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt
from orders.models import Order


@csrf_exempt
def stripe_webhook(request):
    payload = request.body
    sig_header = request.META['HTTP_STRIPE_SIGNATURE']
    event = None
    try:
        event = stripe.Webhook.construct_event(
            payload,
            sig_header,
            settings.STRIPE_WEBHOOK_SECRET
        )
    except ValueError as e:
        # Invalid payload
        return HttpResponse(status=400)
    except stripe.error.SignatureVerificationError as e:
        # Invalid signature
        return HttpResponse(status=400)
    if event.type == 'checkout.session.completed':
        session = event.data.object
        if (
            session.mode == 'payment'
            and session.payment_status == 'paid'
        ):
            try:
                order = Order.objects.get(
                    id=session.client_reference_id
                )
            except Order.DoesNotExist:
                return HttpResponse(status=404)
            # mark order as paid
            order.paid = True
            order.save()
    return HttpResponse(status=200)
```

Edita el archivo `urls.py` de la aplicación `payment` y añade el patrón de URL del webhook:

```python
from django.urls import path
from . import views, webhooks

app_name = 'payment'

urlpatterns = [
    path('process/', views.payment_process, name='process'),
    path('completed/', views.payment_completed, name='completed'),
    path('canceled/', views.payment_canceled, name='canceled'),
    path('webhook/', webhooks.stripe_webhook, name='stripe-webhook'),
]
```

##### Prueba de notificaciones de webhook

Para probar webhooks localmente, instala la CLI de Stripe. En macOS o Linux con Homebrew:

```bash
brew install stripe/stripe-cli/stripe
```

Inicia sesión con:

```bash
stripe login
```

> *Figura 9.21: La pantalla de emparejamiento de la CLI de Stripe*

> *Figura 9.22: La confirmación de emparejamiento de la CLI de Stripe*

Ahora, ejecuta el reenvío de eventos a tu entorno local:

```bash
stripe listen --forward-to 127.0.0.1:8000/payment/webhook/
```

> *Figura 9.23: La página de Webhooks de Stripe con el oyente local registrado*

Realiza un pedido y completa el pago. En la consola donde ejecutas la CLI de Stripe, verás los eventos recibidos:

```text
2024-01-03 18:06:13 --> payment_intent.created [evt_...]
2024-01-03 18:06:13 <-- [200] POST http://127.0.0.1:8000/payment/webhook/ [evt_...]
2024-01-03 18:06:13 --> payment_intent.succeeded [evt_...]
2024-01-03 18:06:13 <-- [200] POST http://127.0.0.1:8000/payment/webhook/ [evt_...]
2024-01-03 18:06:13 --> charge.succeeded [evt_...]
2024-01-03 18:06:13 <-- [200] POST http://127.0.0.1:8000/payment/webhook/ [evt_...]
2024-01-03 18:06:14 --> checkout.session.completed [evt_...]
2024-01-03 18:06:14 <-- [200] POST http://127.0.0.1:8000/payment/webhook/ [evt_...]
```

Abre `http://127.0.0.1:8000/admin/orders/order/` en tu navegador. El pedido se marcará automáticamente como pagado:

> *Figura 9.24: Un pedido marcado como pagado en la lista de pedidos del sitio de administración*

##### Referencia de pagos de Stripe en los pedidos

Añadamos un campo `stripe_id` al modelo `Order` para almacenar el identificador del pago de Stripe. Edita `orders/models.py`:

```python
class Order(models.Model):
    # ...
    stripe_id = models.CharField(max_length=250, blank=True)
```

Genera y aplica las migraciones:

```bash
python manage.py makemigrations
python manage.py migrate
```

Actualiza la función `stripe_webhook` en `payment/webhooks.py`:

```python
# ...
@csrf_exempt
def stripe_webhook(request):
    # ...
    if event.type == 'checkout.session.completed':
        session = event.data.object
        if (
            session.mode == 'payment'
            and session.payment_status == 'paid'
        ):
            try:
                order = Order.objects.get(
                    id=session.client_reference_id
                )
            except Order.DoesNotExist:
                return HttpResponse(status=404)
            # mark order as paid
            order.paid = True
            # store Stripe payment ID
            order.stripe_id = session.payment_intent
            order.save()
    return HttpResponse(status=200)
```

Almacenamos el `payment_intent` en `order.stripe_id`:

> *Figura 9.25: El campo Stripe id con el ID del payment intent*

Añade un método `get_stripe_url()` al modelo `Order` en `orders/models.py`:

```python
from django.conf import settings
from django.db import models


class Order(models.Model):
    # ...
    class Meta:
        ordering = ['-created']
        indexes = [
            models.Index(fields=['-created']),
        ]

    def __str__(self):
        return f'Order {self.id}'

    def get_total_cost(self):
        return sum(item.get_cost() for item in self.items.all())

    def get_stripe_url(self):
        if not self.stripe_id:
            # no payment associated
            return ''
        if '_test_' in settings.STRIPE_SECRET_KEY:
            # Stripe path for test payments
            path = '/test/'
        else:
            # Stripe path for real payments
            path = '/'
        return f'https://dashboard.stripe.com{path}payments/{self.stripe_id}'
```

Edita `orders/admin.py` para mostrar el enlace del pago de Stripe en el sitio de administración:

```python
# ...
from django.utils.safestring import mark_safe


def order_payment(obj):
    url = obj.get_stripe_url()
    if obj.stripe_id:
        html = f'<a href="{url}" target="_blank">{obj.stripe_id}</a>'
        return mark_safe(html)
    return ''


order_payment.short_description = 'Stripe payment'


@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = [
        'id', 'first_name', 'last_name', 'email',
        'address', 'postal_code', 'city', 'paid',
        order_payment, 'created', 'updated'
    ]
    # ...
```

> *Figura 9.26: El ID de pago de Stripe para un objeto Order en el sitio de administración*

##### Puesta en producción (Going live)

Cuando estés listo para pasar a producción, reemplaza tus credenciales de prueba con las de producción en el archivo `.env` o en las variables de entorno de tu servidor y añade el endpoint de webhook en [https://dashboard.stripe.com/webhooks](https://dashboard.stripe.com/webhooks).

---

### Exportación de pedidos a archivos CSV

El formato CSV (valores separados por comas) es uno de los formatos más utilizados para exportar e importar datos entre sistemas. Personalizaremos el sitio de administración para exportar pedidos a archivos CSV.

#### Adición de acciones personalizadas al sitio de administración

Las acciones de administración (*admin actions*) permiten a los usuarios del personal aplicar operaciones en lote a múltiples elementos seleccionados en la vista de lista de cambios.

> *Figura 9.27: El menú desplegable para las acciones de administración de Django*

Edita `orders/admin.py` y añade la siguiente función `export_to_csv` antes de `OrderAdmin`:

```python
import csv
import datetime
from django.http import HttpResponse


def export_to_csv(modeladmin, request, queryset):
    opts = modeladmin.model._meta
    content_disposition = (
        f'attachment; filename={opts.verbose_name}.csv'
    )
    response = HttpResponse(content_type='text/csv')
    response['Content-Disposition'] = content_disposition
    writer = csv.writer(response)
    fields = [
        field for field in opts.get_fields()
        if not field.many_to_many and not field.one_to_many
    ]
    # Write a first row with header information
    writer.writerow([field.verbose_name for field in fields])
    # Write data rows
    for obj in queryset:
        data_row = []
        for field in fields:
            value = getattr(obj, field.name)
            if isinstance(value, datetime.datetime):
                value = value.strftime('%d/%m/%Y')
            data_row.append(value)
        writer.writerow(data_row)
    return response


export_to_csv.short_description = 'Export to CSV'
```

Añade `export_to_csv` a la lista `actions` de `OrderAdmin`:

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    # ...
    actions = [export_to_csv]
```

Abre `http://127.0.0.1:8000/admin/orders/order/` en tu navegador:

> *Figura 9.28: Uso de la acción personalizada Export to CSV en el sitio de administración*

Selecciona varios pedidos, elige **Export to CSV** y haz clic en **Go**. Se descargará un archivo CSV con el siguiente formato:

```text
ID,first name,last name,email,address,postal code,city,created,updated,paid,stripe id
4,Antonio,Melé,email@domain.com,20 W 34th St,10001,New York,03/01/2024,03/01/2024,True,pi_3ORvzkGNwIe5nm8S1wVd7l7i
...
```

---

### Extensión del sitio de administración con vistas personalizadas

Creemos una vista de administración personalizada para mostrar información detallada sobre un pedido.

Edita el archivo `views.py` de la aplicación `orders`:

```python
from django.contrib.admin.views.decorators import staff_member_required
from django.shortcuts import get_object_or_404, redirect, render
from cart.cart import Cart
from .forms import OrderCreateForm
from .models import Order, OrderItem
from .tasks import order_created


def order_create(request):
    # ...
    pass


@staff_member_required
def admin_order_detail(request, order_id):
    order = get_object_or_404(Order, id=order_id)
    return render(
        request,
        'admin/orders/order/detail.html',
        {'order': order}
    )
```

Edita `orders/urls.py` y añade el patrón de URL:

```python
urlpatterns = [
    path('create/', views.order_create, name='order_create'),
    path(
        'admin/order/<int:order_id>/',
        views.admin_order_detail,
        name='admin_order_detail'
    ),
]
```

Crea la plantilla `templates/admin/orders/order/detail.html`:

```html
{% extends "admin/base_site.html" %}

{% block title %}
    Order {{ order.id }} {{ block.super }}
{% endblock %}

{% block breadcrumbs %}
    <div class="breadcrumbs">
        <a href="{% url "admin:index" %}">Home</a> &rsaquo;
        <a href="{% url "admin:orders_order_changelist" %}">Orders</a> &rsaquo;
        <a href="{% url "admin:orders_order_change" order.id %}">Order {{ order.id }}</a> &rsaquo;
        Detail
    </div>
{% endblock %}

{% block content %}
    <div class="module">
        <h1>Order {{ order.id }}</h1>
        <ul class="object-tools">
            <li>
                <a href="#" onclick="window.print();">
                    Print order
                </a>
            </li>
        </ul>
        <table>
            <tr>
                <th>Created</th>
                <td>{{ order.created }}</td>
            </tr>
            <tr>
                <th>Customer</th>
                <td>{{ order.first_name }} {{ order.last_name }}</td>
            </tr>
            <tr>
                <th>E-mail</th>
                <td><a href="mailto:{{ order.email }}">{{ order.email }}</a></td>
            </tr>
            <tr>
                <th>Address</th>
                <td>
                    {{ order.address }}, {{ order.postal_code }} {{ order.city }}
                </td>
            </tr>
            <tr>
                <th>Total amount</th>
                <td>${{ order.get_total_cost }}</td>
            </tr>
            <tr>
                <th>Status</th>
                <td>{% if order.paid %}Paid{% else %}Pending payment{% endif %}</td>
            </tr>
            <tr>
                <th>Stripe payment</th>
                <td>
                    {% if order.stripe_id %}
                        <a href="{{ order.get_stripe_url }}" target="_blank">
                            {{ order.stripe_id }}
                        </a>
                    {% endif %}
                </td>
            </tr>
        </table>
    </div>
    <div class="module">
        <h2>Items bought</h2>
        <table style="width:100%">
            <thead>
                <tr>
                    <th>Product</th>
                    <th>Price</th>
                    <th>Quantity</th>
                    <th>Total</th>
                </tr>
            </thead>
            <tbody>
                {% for item in order.items.all %}
                    <tr class="row{% cycle "1" "2" %}">
                        <td>{{ item.product.name }}</td>
                        <td class="num">${{ item.price }}</td>
                        <td class="num">{{ item.quantity }}</td>
                        <td class="num">${{ item.get_cost }}</td>
                    </tr>
                {% endfor %}
                <tr class="total">
                    <td colspan="3">Total</td>
                    <td class="num">${{ order.get_total_cost }}</td>
                </tr>
            </tbody>
        </table>
    </div>
{% endblock %}
```

Añade un enlace `View` a cada fila en `orders/admin.py`:

```python
from django.urls import reverse


def order_detail(obj):
    url = reverse('orders:admin_order_detail', args=[obj.id])
    return mark_safe(f'<a href="{url}">View</a>')


class OrderAdmin(admin.ModelAdmin):
    list_display = [
        'id', 'first_name', 'last_name', 'email',
        'address', 'postal_code', 'city', 'paid',
        order_payment, 'created', 'updated',
        order_detail,
    ]
    # ...
```

> *Figura 9.29: El enlace View incluido en cada fila de pedido*

> *Figura 9.30: La página personalizada de detalle de pedido en el sitio de administración*

---

### Generación dinámica de facturas en PDF

Generaremos facturas en formato PDF dinámicamente convirtiendo una plantilla HTML en un archivo PDF utilizando WeasyPrint.

#### Instalación de WeasyPrint

Instala las dependencias del sistema operativo para WeasyPrint desde [https://doc.courtbouillon.org/weasyprint/stable/first_steps.html](https://doc.courtbouillon.org/weasyprint/stable/first_steps.html). Luego, instala WeasyPrint vía pip:

```bash
python -m pip install WeasyPrint==61.2
```

#### Creación de una plantilla PDF

Crea la plantilla `templates/orders/order/pdf.html` en la aplicación `orders`:

```html
<html>
<body>
    <h1>My Shop</h1>
    <p>
        Invoice no. {{ order.id }}<br>
        <span class="secondary">
            {{ order.created|date:"M d, Y" }}
        </span>
    </p>
    <h3>Bill to</h3>
    <p>
        {{ order.first_name }} {{ order.last_name }}<br>
        {{ order.email }}<br>
        {{ order.address }}<br>
        {{ order.postal_code }}, {{ order.city }}
    </p>
    <h3>Items bought</h3>
    <table>
        <thead>
            <tr>
                <th>Product</th>
                <th>Price</th>
                <th>Quantity</th>
                <th>Cost</th>
            </tr>
        </thead>
        <tbody>
            {% for item in order.items.all %}
                <tr class="row{% cycle "1" "2" %}">
                    <td>{{ item.product.name }}</td>
                    <td class="num">${{ item.price }}</td>
                    <td class="num">{{ item.quantity }}</td>
                    <td class="num">${{ item.get_cost }}</td>
                </tr>
            {% endfor %}
            <tr class="total">
                <td colspan="3">Total</td>
                <td class="num">${{ order.get_total_cost }}</td>
            </tr>
        </tbody>
    </table>
    <span class="{% if order.paid %}paid{% else %}pending{% endif %}">
        {% if order.paid %}Paid{% else %}Pending payment{% endif %}
    </span>
</body>
</html>
```

#### Renderizado de archivos PDF

Edita el archivo `views.py` de la aplicación `orders` y añade la vista `admin_order_pdf`:

```python
import weasyprint
from django.contrib.staticfiles import finders
from django.http import HttpResponse
from django.template.loader import render_to_string


@staff_member_required
def admin_order_pdf(request, order_id):
    order = get_object_or_404(Order, id=order_id)
    html = render_to_string('orders/order/pdf.html', {'order': order})
    response = HttpResponse(content_type='application/pdf')
    response['Content-Disposition'] = f'filename=order_{order.id}.pdf'
    weasyprint.HTML(string=html).write_pdf(
        response,
        stylesheets=[weasyprint.CSS(finders.find('css/pdf.css'))]
    )
    return response
```

Añade `STATIC_ROOT` a `settings.py` y recopila los archivos estáticos:

```python
STATIC_ROOT = BASE_DIR / 'static'
```

```bash
python manage.py collectstatic
```

Añade la URL en `orders/urls.py`:

```python
urlpatterns = [
    # ...
    path(
        'admin/order/<int:order_id>/pdf/',
        views.admin_order_pdf,
        name='admin_order_pdf'
    ),
]
```

Añade el enlace PDF en `orders/admin.py`:

```python
def order_pdf(obj):
    url = reverse('orders:admin_order_pdf', args=[obj.id])
    return mark_safe(f'<a href="{url}">PDF</a>')


order_pdf.short_description = 'Invoice'


class OrderAdmin(admin.ModelAdmin):
    list_display = [
        'id', 'first_name', 'last_name', 'email',
        'address', 'postal_code', 'city', 'paid',
        order_payment, 'created', 'updated',
        order_detail, order_pdf,
    ]
```

> *Figura 9.31: El enlace PDF incluido en cada fila de pedido*

> *Figura 9.32: La factura PDF para un pedido no pagado*

> *Figura 9.33: La factura PDF para un pedido pagado*

#### Envío de archivos PDF por correo electrónico

Cuando un pago se complete exitosamente, enviaremos un correo electrónico automático al cliente con la factura en PDF adjunta mediante una tarea asíncrona de Celery.

Crea `payment/tasks.py`:

```python
from io import BytesIO
import weasyprint
from celery import shared_task
from django.contrib.staticfiles import finders
from django.core.mail import EmailMessage
from django.template.loader import render_to_string
from orders.models import Order


@shared_task
def payment_completed(order_id):
    """
    Task to send an e-mail notification when an order is successfully paid.
    """
    order = Order.objects.get(id=order_id)
    # create invoice e-mail
    subject = f'My Shop - Invoice no. {order.id}'
    message = (
        'Please, find attached the invoice for your recent purchase.'
    )
    email = EmailMessage(
        subject,
        message,
        'admin@myshop.com',
        [order.email]
    )
    # generate PDF
    html = render_to_string('orders/order/pdf.html', {'order': order})
    out = BytesIO()
    stylesheets = [weasyprint.CSS(finders.find('css/pdf.css'))]
    weasyprint.HTML(string=html).write_pdf(out, stylesheets=stylesheets)
    # attach PDF file
    email.attach(
        f'order_{order.id}.pdf',
        out.getvalue(),
        'application/pdf'
    )
    # send e-mail
    email.send()
```

Llama a la tarea en `payment/webhooks.py`:

```python
import stripe
from django.conf import settings
from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt
from orders.models import Order
from .tasks import payment_completed


@csrf_exempt
def stripe_webhook(request):
    payload = request.body
    sig_header = request.META['HTTP_STRIPE_SIGNATURE']
    event = None
    try:
        event = stripe.Webhook.construct_event(
            payload,
            sig_header,
            settings.STRIPE_WEBHOOK_SECRET
        )
    except ValueError as e:
        # Invalid payload
        return HttpResponse(status=400)
    except stripe.error.SignatureVerificationError as e:
        # Invalid signature
        return HttpResponse(status=400)
    if event.type == 'checkout.session.completed':
        session = event.data.object
        if (
            session.mode == 'payment'
            and session.payment_status == 'paid'
        ):
            try:
                order = Order.objects.get(
                    id=session.client_reference_id
                )
            except Order.DoesNotExist:
                return HttpResponse(status=404)
            # mark order as paid
            order.paid = True
            # store Stripe payment ID
            order.stripe_id = session.payment_intent
            order.save()
            # launch asynchronous task
            payment_completed.delay(order.id)
    return HttpResponse(status=200)
```

Al completarse el pago, la tarea `payment_completed` generará el PDF y lo enviará adjunto por correo electrónico al cliente.

---

### Resumen

En este capítulo, integraste la pasarela de pago Stripe en tu proyecto y creaste un endpoint de webhook para recibir notificaciones de pago. Construiste una acción de administración personalizada para exportar pedidos a CSV. También personalizaste el sitio de administración de Django usando vistas y plantillas personalizadas. Finalmente, aprendiste a generar archivos PDF con WeasyPrint y a adjuntarlos a correos electrónicos.

En el próximo capítulo, aprenderás a crear un sistema de cupones usando sesiones de Django y construirás un motor de recomendaciones de productos con Redis.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter09](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter09)
- **Sitio web de Stripe:** [https://www.stripe.com/](https://www.stripe.com/)
- **Documentación de Stripe Checkout:** [https://stripe.com/docs/payments/checkout](https://stripe.com/docs/payments/checkout)
- **Creación de cuenta en Stripe:** [https://dashboard.stripe.com/register](https://dashboard.stripe.com/register)
- **Configuración de cuenta de Stripe:** [https://dashboard.stripe.com/settings/account](https://dashboard.stripe.com/settings/account)
- **Biblioteca de Python de Stripe:** [https://github.com/stripe/stripe-python](https://github.com/stripe/stripe-python)
- **Claves API de prueba de Stripe:** [https://dashboard.stripe.com/test/apikeys](https://dashboard.stripe.com/test/apikeys)
- **Documentación de claves API de Stripe:** [https://stripe.com/docs/keys](https://stripe.com/docs/keys)
- **Versión de la API de Stripe 2024-04-10:** [https://stripe.com/docs/upgrades#2024-04-10](https://stripe.com/docs/upgrades#2024-04-10)
- **Modos de sesión de Stripe Checkout:** [https://stripe.com/docs/api/checkout/sessions/object#checkout_session_object-mode](https://stripe.com/docs/api/checkout/sessions/object#checkout_session_object-mode)
- **Construcción de URIs absolutas con Django:** [https://docs.djangoproject.com/en/5.2/ref/request-response/#django.http.HttpRequest.build_absolute_uri](https://docs.djangoproject.com/en/5.2/ref/request-response/#django.http.HttpRequest.build_absolute_uri)
- **Creación de sesiones de Stripe:** [https://stripe.com/docs/api/checkout/sessions/create](https://stripe.com/docs/api/checkout/sessions/create)
- **Monedas admitidas por Stripe:** [https://stripe.com/docs/currencies](https://stripe.com/docs/currencies)
- **Panel de pagos de Stripe:** [https://dashboard.stripe.com/test/payments](https://dashboard.stripe.com/test/payments)
- **Tarjetas de crédito para pruebas con Stripe:** [https://stripe.com/docs/testing](https://stripe.com/docs/testing)
- **Webhooks de Stripe:** [https://dashboard.stripe.com/test/webhooks](https://dashboard.stripe.com/test/webhooks)
- **Tipos de eventos enviados por Stripe:** [https://stripe.com/docs/api/events/types](https://stripe.com/docs/api/events/types)
- **Instalación de la CLI de Stripe:** [https://stripe.com/docs/stripe-cli#install](https://stripe.com/docs/stripe-cli#install)
- **Última versión de la CLI de Stripe:** [https://github.com/stripe/stripe-cli/releases/latest](https://github.com/stripe/stripe-cli/releases/latest)
- **Generación de archivos CSV con Django:** [https://docs.djangoproject.com/en/5.2/howto/outputting-csv/](https://docs.djangoproject.com/en/5.2/howto/outputting-csv/)
- **Aplicación django-import-export:** [https://django-import-export.readthedocs.io/en/latest/](https://django-import-export.readthedocs.io/en/latest/)
- **Aplicación django-import-export-celery:** [https://github.com/auto-mat/django-import-export-celery](https://github.com/auto-mat/django-import-export-celery)
- **Plantillas de administración de Django:** [https://github.com/django/django/tree/5.2/django/contrib/admin/templates/admin](https://github.com/django/django/tree/5.2/django/contrib/admin/templates/admin)
- **Generación de archivos PDF con ReportLab:** [https://docs.djangoproject.com/en/5.2/howto/outputting-pdf/](https://docs.djangoproject.com/en/5.2/howto/outputting-pdf/)
- **Instalación de WeasyPrint:** [https://doc.courtbouillon.org/weasyprint/stable/first_steps.html](https://doc.courtbouillon.org/weasyprint/stable/first_steps.html)
- **Archivos estáticos para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-Example/tree/main/Chapter09/myshop/shop/static](https://github.com/PacktPublishing/Django-5-by-Example/tree/main/Chapter09/myshop/shop/static)
