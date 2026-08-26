# Parte 3: Creación de una tienda online

## Capítulo 10: Extensión de tu tienda

### Introducción

En el capítulo anterior, aprendiste a integrar una pasarela de pago en tu tienda. También aprendiste a generar archivos CSV y PDF.

En este capítulo, añadirás un sistema de cupones a tu tienda y crearás un motor de recomendaciones de productos.

Este capítulo cubrirá los siguientes puntos:

- Creación de un sistema de cupones
- Aplicación de cupones al carrito de compras
- Aplicación de cupones a los pedidos
- Creación de cupones para Stripe Checkout
- Almacenamiento de productos que habitualmente se compran juntos
- Creación de un motor de recomendaciones de productos con Redis

---

### Visión general funcional

La Figura 10.1 muestra una representación de las vistas, plantillas y funcionalidades que se construirán en este capítulo:

> *Figura 10.1: Diagrama de funcionalidades construidas en el Capítulo 10*

En este capítulo, construirás una nueva aplicación `coupons` y crearás la vista `coupon_apply` para aplicar cupones de descuento a la sesión del carrito. Añadirás el descuento aplicado a la plantilla de la vista `cart_detail` de la aplicación `cart`. Cuando se cree un pedido con la vista `order_create` de la aplicación `orders`, guardarás el cupón en el pedido creado. Luego, cuando crees la sesión de Stripe en la vista `payment_process` de la aplicación `payment`, añadirás el cupón a la sesión de Stripe Checkout antes de redirigir al usuario a Stripe para completar el pago. Añadirás el descuento aplicado a las plantillas de las vistas de administración `admin_order_detail` y `admin_order_pdf` de la aplicación `orders`. Además del sistema de cupones, también implementarás un sistema de recomendaciones. Cuando la vista `stripe_webhook` reciba el evento `checkout.session.completed` de Stripe, guardarás los productos que se hayan comprado juntos en Redis. Añadirás recomendaciones de productos a las vistas `product_detail` y `cart_detail` recuperando de Redis los artículos que frecuentemente se compran juntos.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter10](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter10).

Todos los paquetes de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente del capítulo. Puedes seguir las instrucciones para instalar cada paquete de Python en las siguientes secciones, o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Creación de un sistema de cupones

Muchas tiendas online ofrecen cupones a sus clientes que pueden canjearse por descuentos en sus compras. Un cupón online suele consistir en un código que se entrega a los usuarios y que es válido durante un período de tiempo específico.

Vas a crear un sistema de cupones para tu tienda. Tus cupones serán válidos para los clientes durante un período determinado. Los cupones no tendrán ninguna limitación en cuanto al número de veces que se pueden canjear y se aplicarán al valor total del carrito de compras.

Para esta funcionalidad, necesitarás crear un modelo para almacenar el código del cupón, un período de validez y el descuento a aplicar.

Crea una nueva aplicación dentro del proyecto `myshop` usando el siguiente comando:

```bash
python manage.py startapp coupons
```

Edita el archivo `settings.py` de `myshop` y añade la aplicación a la configuración `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    # ...
    'coupons.apps.CouponsConfig',
]
```

La nueva aplicación ahora está activa en tu proyecto de Django.

#### Construcción del modelo de cupón

Comencemos creando el modelo `Coupon`. Edita el archivo `models.py` de la aplicación `coupons` y añádele el siguiente código:

```python
from django.core.validators import MaxValueValidator, MinValueValidator
from django.db import models


class Coupon(models.Model):
    code = models.CharField(max_length=50, unique=True)
    valid_from = models.DateTimeField()
    valid_to = models.DateTimeField()
    discount = models.IntegerField(
        validators=[MinValueValidator(0), MaxValueValidator(100)],
        help_text='Percentage value (0 to 100)'
    )
    active = models.BooleanField()

    def __str__(self):
        return self.code
```

Este es el modelo que vas a utilizar para almacenar cupones. El modelo `Coupon` contiene los siguientes campos:

- `code`: El código que los usuarios deben ingresar para aplicar el cupón a su compra.
- `valid_from`: El valor datetime que indica cuándo comienza a ser válido el cupón.
- `valid_to`: El valor datetime que indica cuándo deja de ser válido el cupón.
- `discount`: La tasa de descuento a aplicar (es un porcentaje, por lo que toma valores de 0 a 100). Utilizas validadores para este campo para limitar los valores mínimos y máximos aceptados.
- `active`: Un booleano que indica si el cupón está activo.

Ejecuta el siguiente comando para generar la migración inicial para la aplicación `coupons`:

```bash
python manage.py makemigrations
```

La salida debería incluir las siguientes líneas:

```text
Migrations for 'coupons':
  coupons/migrations/0001_initial.py
    - Create model Coupon
```

Luego, ejecuta el siguiente comando para aplicar las migraciones:

```bash
python manage.py migrate
```

Deberías ver una salida que incluye la siguiente línea:

```text
Applying coupons.0001_initial... OK
```

Las migraciones ahora se han aplicado a la base de datos. Añadamos el modelo `Coupon` al sitio de administración. Edita el archivo `admin.py` de la aplicación `coupons` y añádele el siguiente código:

```python
from django.contrib import admin
from .models import Coupon


@admin.register(Coupon)
class CouponAdmin(admin.ModelAdmin):
    list_display = [
        'code', 'valid_from', 'valid_to', 'discount', 'active'
    ]
    list_filter = ['active', 'valid_from', 'valid_to']
    search_fields = ['code']
```

El modelo `Coupon` ahora está registrado en el sitio de administración. Asegúrate de que tu servidor local esté ejecutándose con el siguiente comando:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/admin/coupons/coupon/add/` en tu navegador:

> *Figura 10.2: El formulario Add coupon en el sitio de administración de Django*

Completa el formulario para crear un nuevo cupón que sea válido para la fecha actual. Asegúrate de marcar la casilla **Active** y haz clic en el botón **SAVE**. La Figura 10.3 muestra un ejemplo de creación de un cupón:

> *Figura 10.3: El formulario Add coupon con datos de ejemplo*

Después de crear el cupón, la página de lista de cambios de cupones en el sitio de administración se verá similar a la Figura 10.4:

> *Figura 10.4: La página de lista de cambios de cupones en el sitio de administración de Django*

A continuación, implementaremos la funcionalidad para aplicar cupones al carrito de compras.

#### Aplicación de un cupón al carrito de compras

Puedes almacenar nuevos cupones y realizar consultas para recuperar cupones existentes. Ahora necesitas una forma para que los clientes apliquen cupones a sus compras. La funcionalidad para aplicar un cupón será la siguiente:

1. El usuario añade productos al carrito de compras.
2. El usuario puede ingresar un código de cupón en un formulario mostrado en la página de detalles del carrito de compras.
3. Cuando el usuario ingresa un código de cupón y envía el formulario, buscas un cupón existente con el código proporcionado que sea válido actualmente. Debes verificar que el código del cupón coincida con el ingresado por el usuario, que el atributo `active` sea `True`, y que la fecha y hora actual esté entre los valores `valid_from` y `valid_to`.
4. Si se encuentra un cupón, lo guardas en la sesión del usuario y muestras el carrito, incluido el descuento aplicado y el importe total actualizado.
5. Cuando el usuario realiza un pedido, guardas el cupón en el pedido correspondiente.

Crea un nuevo archivo dentro del directorio de la aplicación `coupons` y nómbralo `forms.py`. Añádele el siguiente código:

```python
from django import forms


class CouponApplyForm(forms.Form):
    code = forms.CharField()
```

Este es el formulario que vas a utilizar para que el usuario ingrese un código de cupón. Edita el archivo `views.py` dentro de la aplicación `coupons` y añade el siguiente código:

```python
from django.shortcuts import redirect
from django.utils import timezone
from django.views.decorators.http import require_POST
from .forms import CouponApplyForm
from .models import Coupon


@require_POST
def coupon_apply(request):
    now = timezone.now()
    form = CouponApplyForm(request.POST)
    if form.is_valid():
        code = form.cleaned_data['code']
        try:
            coupon = Coupon.objects.get(
                code__iexact=code,
                valid_from__lte=now,
                valid_to__gte=now,
                active=True
            )
            request.session['coupon_id'] = coupon.id
        except Coupon.DoesNotExist:
            request.session['coupon_id'] = None
    return redirect('cart:cart_detail')
```

La vista `coupon_apply` valida el cupón y lo almacena en la sesión del usuario. Aplicas el decorador `@require_POST` a esta vista para restringirla a peticiones POST. En la vista, realizas las siguientes tareas:

- Instancias el formulario `CouponApplyForm` utilizando los datos enviados por POST y verificas que sea válido.
- Si el formulario es válido, obtienes el código ingresado por el usuario del diccionario `cleaned_data` del formulario. Intentas recuperar el objeto `Coupon` con el código dado. Utilizas la búsqueda de campo `iexact` para realizar una coincidencia exacta insensible a mayúsculas y minúsculas. El cupón debe estar activo actualmente (`active=True`) y ser válido para la fecha y hora actual. Utilizas la función `timezone.now()` de Django para obtener la fecha y hora actual consciente de la zona horaria, y la comparas con los campos `valid_from` y `valid_to` mediante las búsquedas de campo `lte` (menor o igual que) y `gte` (mayor o igual que), respectivamente.
- Almacenas el ID del cupón en la sesión del usuario.
- Rediriges al usuario a la URL `cart_detail` para mostrar el carrito con el cupón aplicado.

Necesitas un patrón de URL para la vista `coupon_apply`. Crea un nuevo archivo dentro del directorio de la aplicación `coupons` y nómbralo `urls.py`. Añádele el siguiente código:

```python
from django.urls import path
from . import views

app_name = 'coupons'

urlpatterns = [
    path('apply/', views.coupon_apply, name='apply'),
]
```

Luego, edita el archivo `urls.py` principal del proyecto `myshop` e incluye los patrones de URL de cupones:

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('cart/', include('cart.urls', namespace='cart')),
    path('orders/', include('orders.urls', namespace='orders')),
    path('payment/', include('payment.urls', namespace='payment')),
    path('coupons/', include('coupons.urls', namespace='coupons')),
    path('', include('shop.urls', namespace='shop')),
]
```

Recuerda colocar este patrón antes del patrón `shop.urls`.

Ahora, edita el archivo `cart.py` de la aplicación `cart`. Incluye la siguiente importación:

```python
from coupons.models import Coupon
```

Añade el siguiente código al final del método `__init__()` de la clase `Cart` para inicializar el cupón a partir de la sesión actual:

```python
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
        # store current applied coupon
        self.coupon_id = self.session.get('coupon_id')
```

En este código, intentas obtener la clave de sesión `coupon_id` de la sesión actual y almacenar su valor en el objeto `Cart`. Añade los siguientes métodos al objeto `Cart`:

```python
class Cart:
    # ...
    @property
    def coupon(self):
        if self.coupon_id:
            try:
                return Coupon.objects.get(id=self.coupon_id)
            except Coupon.DoesNotExist:
                pass
        return None

    def get_discount(self):
        if self.coupon:
            return (
                self.coupon.discount / Decimal(100)
            ) * self.get_total_price()
        return Decimal(0)

    def get_total_price_after_discount(self):
        return self.get_total_price() - self.get_discount()
```

Estos métodos son los siguientes:

- `coupon`: Defines este método como una propiedad (`@property`). Si el carrito contiene un atributo `coupon_id`, se devuelve el objeto `Coupon` con el ID dado.
- `get_discount()`: Si el carrito contiene un cupón, recuperas su porcentaje de descuento y devuelves la cantidad a deducir del importe total del carrito.
- `get_total_price_after_discount()`: Devuelves el importe total del carrito tras deducir la cantidad devuelta por el método `get_discount()`.

La clase `Cart` ahora está preparada para gestionar un cupón aplicado a la sesión actual y aplicar el descuento correspondiente.

Incluyamos el sistema de cupones en la vista de detalle del carrito. Edita el archivo `views.py` de la aplicación `cart` y añade la siguiente importación en la parte superior del archivo:

```python
from coupons.forms import CouponApplyForm
```

Más abajo, edita la vista `cart_detail` y añádele el nuevo formulario:

```python
def cart_detail(request):
    cart = Cart(request)
    for item in cart:
        item['update_quantity_form'] = CartAddProductForm(
            initial={'quantity': item['quantity'], 'override': True}
        )
    coupon_apply_form = CouponApplyForm()
    return render(
        request,
        'cart/detail.html',
        {
            'cart': cart,
            'coupon_apply_form': coupon_apply_form
        }
    )
```

Edita la plantilla `cart/detail.html` de la aplicación `cart` y localiza las siguientes líneas:

```html
<tr class="total">
    <td>Total</td>
    <td colspan="4"></td>
    <td class="num">${{ cart.get_total_price }}</td>
</tr>
```

Reemplázalas con el siguiente código:

```html
{% if cart.coupon %}
    <tr class="subtotal">
        <td>Subtotal</td>
        <td colspan="4"></td>
        <td class="num">${{ cart.get_total_price|floatformat:2 }}</td>
    </tr>
    <tr>
        <td>
            "{{ cart.coupon.code }}" coupon ({{ cart.coupon.discount }}% off)
        </td>
        <td colspan="4"></td>
        <td class="num neg">
            - ${{ cart.get_discount|floatformat:2 }}
        </td>
    </tr>
{% endif %}
<tr class="total">
    <td>Total</td>
    <td colspan="4"></td>
    <td class="num">
        ${{ cart.get_total_price_after_discount|floatformat:2 }}
    </td>
</tr>
```

Este es el código para mostrar un cupón opcional y su tasa de descuento. Si el carrito contiene un cupón, muestras la primera fila, incluyendo el importe total del carrito como subtotal. Luego, utilizas una segunda fila para mostrar el cupón actual aplicado al carrito. Finalmente, muestras el precio total, incluyendo cualquier descuento, llamando al método `get_total_price_after_discount()` del objeto `cart`.

En el mismo archivo, incluye el siguiente código después de la etiqueta HTML `</table>`:

```html
<p>Apply a coupon:</p>
<form action="{% url "coupons:apply" %}" method="post">
    {{ coupon_apply_form }}
    <input type="submit" value="Apply">
    {% csrf_token %}
</form>
```

Esto mostrará el formulario para ingresar un código de cupón y aplicarlo al carrito actual.

Abre `http://127.0.0.1:8000/` en tu navegador y añade un producto al carrito. Verás que la página del carrito de compras ahora incluye un formulario para aplicar un cupón:

> *Figura 10.5: La página de detalle del carrito, incluyendo un formulario para aplicar un cupón (Imagen de Té en polvo: Foto por Phuong Nguyen en Unsplash)*

En el campo **Code**, ingresa el código del cupón que creaste usando el sitio de administración:

> *Figura 10.6: La página de detalle del carrito, incluyendo un código de cupón en el formulario*

Haz clic en el botón **Apply**. El cupón se aplicará y el carrito mostrará el descuento del cupón de la siguiente manera:

> *Figura 10.7: La página de detalle del carrito, incluyendo el cupón aplicado*

Añadamos el cupón al siguiente paso del proceso de compra. Edita la plantilla `orders/order/create.html` de la aplicación `orders` y localiza las siguientes líneas:

```html
<ul>
    {% for item in cart %}
        <li>
            {{ item.quantity }}x {{ item.product.name }}
            <span>${{ item.total_price }}</span>
        </li>
    {% endfor %}
</ul>
```

Reemplázalas con el siguiente código:

```html
<ul>
    {% for item in cart %}
        <li>
            {{ item.quantity }}x {{ item.product.name }}
            <span>${{ item.total_price|floatformat:2 }}</span>
        </li>
    {% endfor %}
    {% if cart.coupon %}
        <li>
            "{{ cart.coupon.code }}" ({{ cart.coupon.discount }}% off)
            <span class="neg">- ${{ cart.get_discount|floatformat:2 }}</span>
        </li>
    {% endif %}
</ul>
```

El resumen del pedido ahora incluirá el cupón aplicado, si lo hay. Ahora busca la siguiente línea:

```html
<p>Total: ${{ cart.get_total_price }}</p>
```

Reemplázala con lo siguiente:

```html
<p>Total: ${{ cart.get_total_price_after_discount|floatformat:2 }}</p>
```

Al hacer esto, el precio total también se calculará aplicando el descuento del cupón.

Abre `http://127.0.0.1:8000/orders/create/` en tu navegador. Deberías ver que el resumen del pedido incluye el cupón aplicado:

> *Figura 10.8: El resumen del pedido, incluyendo el cupón aplicado al carrito*

Los usuarios ahora pueden aplicar cupones a sus carritos de compras. Sin embargo, todavía necesitas almacenar la información del cupón en el pedido que se crea cuando los usuarios tramitan el carrito.

#### Aplicación de cupones a los pedidos

Vas a almacenar el cupón que se aplicó a cada pedido. Primero, necesitas modificar el modelo `Order` para almacenar el objeto `Coupon` relacionado, si lo hay.

Edita el archivo `models.py` de la aplicación `orders` y añádele las siguientes importaciones:

```python
from decimal import Decimal
from django.core.validators import MaxValueValidator, MinValueValidator
from coupons.models import Coupon
```

Luego, añade los siguientes campos al modelo `Order`:

```python
class Order(models.Model):
    # ...
    coupon = models.ForeignKey(
        Coupon,
        related_name='orders',
        null=True,
        blank=True,
        on_delete=models.SET_NULL
    )
    discount = models.IntegerField(
        default=0,
        validators=[MinValueValidator(0), MaxValueValidator(100)]
    )
```

Estos campos te permiten almacenar un cupón opcional para el pedido y el porcentaje de descuento aplicado con el cupón. El descuento se almacena en el objeto `Coupon` relacionado, pero lo incluimos en el modelo `Order` para preservarlo en caso de que el cupón sea modificado o eliminado. Estableces `on_delete` en `models.SET_NULL` para que si se elimina el cupón, el campo `coupon` se establezca en `Null`, pero el descuento se preserve.

Necesitas crear una migración para incluir los nuevos campos del modelo `Order`. Ejecuta el siguiente comando:

```bash
python manage.py makemigrations
```

Deberías ver una salida como la siguiente:

```text
Migrations for 'orders':
  orders/migrations/0003_order_coupon_order_discount.py
    - Add field coupon to order
    - Add field discount to order
```

Aplica la nueva migración con el siguiente comando:

```bash
python manage.py migrate orders
```

Deberías ver la siguiente confirmación indicando que la nueva migración se ha aplicado:

```text
Applying orders.0003_order_coupon_order_discount... OK
```

Los cambios en los campos del modelo `Order` ahora están sincronizados con la base de datos.

Edita el archivo `models.py` y añade dos nuevos métodos, `get_total_cost_before_discount()` y `get_discount()`, al modelo `Order`:

```python
class Order(models.Model):
    # ...
    def get_total_cost_before_discount(self):
        return sum(item.get_cost() for item in self.items.all())

    def get_discount(self):
        total_cost = self.get_total_cost_before_discount()
        if self.discount:
            return total_cost * (self.discount / Decimal(100))
        return Decimal(0)
```

Luego, edita el método `get_total_cost()` del modelo `Order` de la siguiente manera:

```python
    def get_total_cost(self):
        total_cost = self.get_total_cost_before_discount()
        return total_cost - self.get_discount()
```

El método `get_total_cost()` del modelo `Order` ahora tendrá en cuenta el descuento aplicado, si lo hay.

Edita el archivo `views.py` de la aplicación `orders` y modifica la vista `order_create` para guardar el cupón relacionado y su descuento al crear un nuevo pedido:

```python
def order_create(request):
    cart = Cart(request)
    if request.method == 'POST':
        form = OrderCreateForm(request.POST)
        if form.is_valid():
            order = form.save(commit=False)
            if cart.coupon:
                order.coupon = cart.coupon
                order.discount = cart.coupon.discount
            order.save()
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

En el nuevo código, creas un objeto `Order` utilizando el método `save()` del formulario `OrderCreateForm`. Evitas guardarlo en la base de datos todavía utilizando `commit=False`. Si el carrito contiene un cupón, almacenas el cupón relacionado y el descuento aplicado. Luego, guardas el objeto de pedido en la base de datos.

Edita la plantilla `payment/process.html` de la aplicación `payment` y localiza las siguientes líneas:

```html
<tr class="total">
    <td>Total</td>
    <td colspan="4"></td>
    <td class="num">${{ order.get_total_cost }}</td>
</tr>
```

Reemplázalas con el siguiente código:

```html
{% if order.coupon %}
    <tr class="subtotal">
        <td>Subtotal</td>
        <td colspan="3"></td>
        <td class="num">
            ${{ order.get_total_cost_before_discount|floatformat:2 }}
        </td>
    </tr>
    <tr>
        <td>
            "{{ order.coupon.code }}" coupon ({{ order.discount }}% off)
        </td>
        <td colspan="3"></td>
        <td class="num neg">
            - ${{ order.get_discount|floatformat:2 }}
        </td>
    </tr>
{% endif %}
<tr class="total">
    <td>Total</td>
    <td colspan="3"></td>
    <td class="num">
        ${{ order.get_total_cost|floatformat:2 }}
    </td>
</tr>
```

Hemos actualizado el resumen del pedido antes del pago.

Asegúrate de que el servidor de desarrollo esté ejecutándose con:

```bash
python manage.py runserver
```

Asegúrate de que Docker esté ejecutándose y ejecuta el siguiente comando en otra consola para iniciar el servidor RabbitMQ con Docker:

```bash
docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.13.1-management
```

Abre otra consola e inicia el worker de Celery desde el directorio de tu proyecto:

```bash
celery -A myshop worker -l info
```

Abre una consola adicional y ejecuta el siguiente comando para reenviar eventos de Stripe a tu URL local de webhook:

```bash
stripe listen --forward-to localhost:8000/payment/webhook/
```

Abre `http://127.0.0.1:8000/` en tu navegador y crea un pedido utilizando el cupón que creaste. Después de validar los artículos en el carrito de compras, en la página de resumen del pedido verás el cupón aplicado al pedido:

> *Figura 10.9: La página Order summary, incluyendo el cupón aplicado al pedido*

Si haces clic en **Pay now**, verás que Stripe no está al tanto del descuento aplicado:

> *Figura 10.10: Los detalles de los artículos de la página de Stripe Checkout, sin incluir cupón de descuento*

Stripe muestra el importe total a pagar sin ninguna deducción. Esto se debe a que no le estamos pasando el descuento a Stripe. Recuerda que en la vista `payment_process`, pasamos los artículos del pedido como `line_items` a Stripe, incluyendo el costo y la cantidad de cada artículo.

#### Creación de cupones para Stripe Checkout

Stripe te permite definir cupones de descuento y vincularlos a pagos únicos. Puedes encontrar más información sobre la creación de descuentos para Stripe Checkout en [https://stripe.com/docs/payments/checkout/discounts](https://stripe.com/docs/payments/checkout/discounts).

Editemos la vista `payment_process` para crear un cupón para Stripe Checkout. Edita el archivo `views.py` de la aplicación `payment` y añade el siguiente código a la vista `payment_process`:

```python
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
        # Stripe coupon
        if order.coupon:
            stripe_coupon = stripe.Coupon.create(
                name=order.coupon.code,
                percent_off=order.discount,
                duration='once'
            )
            session_data['discounts'] = [{'coupon': stripe_coupon.id}]
        # create Stripe checkout session
        session = stripe.checkout.Session.create(**session_data)
        # redirect to Stripe payment form
        return redirect(session.url, code=303)
    else:
        return render(request, 'payment/process.html', locals())
```

En el nuevo código, verificas si el pedido tiene un cupón relacionado. En ese caso, utilizas el SDK de Stripe para crear un cupón de Stripe con `stripe.Coupon.create()`. Utilizas los siguientes atributos para el cupón:

- `name`: Se utiliza el código del cupón relacionado con el objeto de pedido.
- `percent_off`: Se emite el descuento del objeto de pedido.
- `duration`: Se utiliza el valor `once`. Esto indica a Stripe que se trata de un cupón para un pago único.

Tras crear el cupón, su `id` se añade al diccionario `session_data` utilizado para crear la sesión de Stripe Checkout. Esto vincula el cupón a la sesión de checkout.

Abre `http://127.0.0.1:8000/` en tu navegador y completa una compra utilizando el cupón que creaste. Cuando seas redirigido a la página de Stripe Checkout, verás el cupón aplicado:

> *Figura 10.11: Los detalles de los artículos de la página de Stripe Checkout, incluyendo un cupón de descuento llamado SUMMER*

La página de Stripe Checkout ahora incluye el cupón del pedido y el importe total a pagar incluye la cantidad deducida mediante el cupón.

Completa la compra y luego abre `http://127.0.0.1:8000/admin/orders/order/` en tu navegador. Haz clic en el objeto de pedido para el cual se utilizó el cupón. El formulario de edición mostrará el descuento aplicado:

> *Figura 10.12: El formulario de edición de pedidos, incluyendo el cupón y el descuento aplicado*

Has almacenado con éxito cupones para pedidos y procesado pagos con descuentos. A continuación, añadirás cupones a la vista de detalle de pedidos del sitio de administración y a las facturas en PDF.

#### Adición de cupones a pedidos en el sitio de administración y a facturas en PDF

Añadamos el cupón a la página de detalle del pedido en el sitio de administración. Edita la plantilla `admin/orders/order/detail.html` de la aplicación `orders`:

```html
...
<table style="width:100%">
...
<tbody>
    {% for item in order.items.all %}
        <tr class="row{% cycle "1" "2" %}">
            <td>{{ item.product.name }}</td>
            <td class="num">${{ item.price }}</td>
            <td class="num">{{ item.quantity }}</td>
            <td class="num">${{ item.get_cost }}</td>
        </tr>
    {% endfor %}
    {% if order.coupon %}
        <tr class="subtotal">
            <td colspan="3">Subtotal</td>
            <td class="num">
                ${{ order.get_total_cost_before_discount|floatformat:2 }}
            </td>
        </tr>
        <tr>
            <td colspan="3">
                "{{ order.coupon.code }}" coupon ({{ order.discount }}% off)
            </td>
            <td class="num neg">
                - ${{ order.get_discount|floatformat:2 }}
            </td>
        </tr>
    {% endif %}
    <tr class="total">
        <td colspan="3">Total</td>
        <td class="num">
            ${{ order.get_total_cost|floatformat:2 }}
        </td>
    </tr>
</tbody>
</table>
...
```

Accede a `http://127.0.0.1:8000/admin/orders/order/` con tu navegador y haz clic en el enlace **View** del último pedido. La tabla *Items bought* ahora incluirá el cupón utilizado:

> *Figura 10.13: La página de detalle de producto en el sitio de administración, incluyendo el cupón utilizado*

Ahora, modifiquemos la plantilla de factura del pedido para incluir el cupón utilizado. Edita la plantilla `orders/order/pdf.html` de la aplicación `orders`:

```html
...
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
        {% if order.coupon %}
            <tr class="subtotal">
                <td colspan="3">Subtotal</td>
                <td class="num">
                    ${{ order.get_total_cost_before_discount|floatformat:2 }}
                </td>
            </tr>
            <tr>
                <td colspan="3">
                    "{{ order.coupon.code }}" coupon ({{ order.discount }}% off)
                </td>
                <td class="num neg">
                    - ${{ order.get_discount|floatformat:2 }}
                </td>
            </tr>
        {% endif %}
        <tr class="total">
            <td colspan="3">Total</td>
            <td class="num">${{ order.get_total_cost|floatformat:2 }}</td>
        </tr>
    </tbody>
</table>
...
```

Accede a `http://127.0.0.1:8000/admin/orders/order/` con tu navegador y haz clic en el enlace **PDF** del último pedido. La tabla *Items bought* ahora incluirá el cupón utilizado:

> *Figura 10.14: La factura de pedido en PDF, incluyendo el cupón utilizado*

Has añadido exitosamente un sistema de cupones a tu tienda. A continuación, vas a construir un motor de recomendaciones de productos.

---

### Creación de un motor de recomendaciones

Un motor de recomendaciones (*recommendation engine*) es un sistema que predice la preferencia o calificación que un usuario le daría a un elemento. El sistema selecciona elementos relevantes para un usuario basándose en su comportamiento y en el conocimiento que tiene sobre él. Hoy en día, los sistemas de recomendación se utilizan en muchos servicios en línea. Ayudan a los usuarios seleccionando el contenido que podría interesarles entre la gran cantidad de datos disponibles que les resultan irrelevantes. Ofrecer buenas recomendaciones mejora la interacción (*engagement*) del usuario. Los sitios de comercio electrónico también se benefician de ofrecer recomendaciones de productos relevantes al aumentar sus ingresos medios por usuario (*ARPU*).

Vas a crear un motor de recomendaciones simple pero potente que sugiere productos que habitualmente se compran juntos. Sugerirás productos basándote en el histórico de ventas, identificando así qué productos se compran conjuntamente. Vas a sugerir productos complementarios en dos escenarios diferentes:

- **Página de detalle del producto:** Mostrarás una lista de productos que habitualmente se compran con el producto dado. Esto se mostrará como *"Los usuarios que compraron esto también compraron X, Y y Z"*. Necesitas una estructura de datos que te permita almacenar el número de veces que cada producto se ha comprado junto con el producto que se está mostrando.
- **Página de detalle del carrito:** Basándote en los productos que los usuarios añaden al carrito, vas a sugerir productos que habitualmente se compran junto con estos. En este caso, la puntuación que calculas para obtener productos relacionados debe agregarse.

Vas a utilizar Redis para almacenar los productos que habitualmente se compran juntos. Recuerda que ya utilizaste Redis en el Capítulo 7, *Seguimiento de acciones de usuario*.

#### Recomendación de productos basada en compras anteriores

Recomendaremos productos a los usuarios basándonos en los artículos que frecuentemente se compran juntos. Para ello, utilizaremos conjuntos ordenados (*sorted sets*) de Redis. Recuerda que utilizaste conjuntos ordenados en el Capítulo 7 para crear una clasificación de las imágenes más vistas en tu sitio.

La Figura 10.15 muestra una representación de un conjunto ordenado, donde los miembros del conjunto son cadenas asociadas a una puntuación:

> *Figura 10.15: Representación de un sorted set de Redis*

Almacenaremos una clave en Redis para cada producto comprado en el sitio. La clave del producto contendrá un conjunto ordenado de Redis con puntuaciones. Cada vez que se complete una nueva compra, incrementaremos la puntuación en 1 para cada producto comprado conjuntamente. El conjunto ordenado te permitirá otorgar puntuaciones a los productos que se compran juntos. Utilizaremos el número de veces que el producto se compra con otro producto como la puntuación para ese artículo.

Recuerda instalar `redis-py` en tu entorno utilizando el siguiente comando:

```bash
python -m pip install redis==5.2.1
```

Edita el archivo `settings.py` de tu proyecto y añádele las siguientes configuraciones:

```python
# Redis settings
REDIS_HOST = 'localhost'
REDIS_PORT = 6379
REDIS_DB = 1
```

Estas son las configuraciones necesarias para establecer una conexión con el servidor Redis. Crea un nuevo archivo dentro del directorio de la aplicación `shop` y nómbralo `recommender.py`. Añádele el siguiente código:

```python
import redis
from django.conf import settings
from .models import Product

# connect to redis
r = redis.Redis(
    host=settings.REDIS_HOST,
    port=settings.REDIS_PORT,
    db=settings.REDIS_DB
)


class Recommender:
    def get_product_key(self, id):
        return f'product:{id}:purchased_with'

    def products_bought(self, products):
        product_ids = [p.id for p in products]
        for product_id in product_ids:
            for with_id in product_ids:
                # get the other products bought with each product
                if product_id != with_id:
                    # increment score for product purchased together
                    r.zincrby(
                        self.get_product_key(product_id),
                        1,
                        with_id
                    )
```

Esta es la clase `Recommender`, que te permitirá almacenar compras de productos y recuperar sugerencias de productos para un producto o productos dados.

El método `get_product_key()` recibe el ID de un objeto `Product` y construye la clave de Redis para el conjunto ordenado donde se almacenan los productos relacionados, que tiene el formato `product:[id]:purchased_with`.

El método `products_bought()` recibe una lista de objetos `Product` que se han comprado juntos (es decir, pertenecen al mismo pedido).

En este método, realizas las siguientes tareas:

- Obtienes los IDs de los productos para los objetos `Product` dados.
- Iteras sobre los IDs de los productos. Para cada ID, iteras nuevamente sobre los IDs de los productos y omites el mismo producto para obtener los productos que se compran junto con cada producto.
- Obtienes la clave de producto de Redis para cada producto comprado usando el método `get_product_key()`. Para un producto con un ID de 33, este método devuelve la clave `product:33:purchased_with`. Esta es la clave para el conjunto ordenado que contiene los IDs de los productos que se compraron junto con este.
- Incrementas la puntuación de cada ID de producto contenido en el conjunto ordenado en 1 utilizando la operación `ZINCRBY` de Redis. La puntuación representa el número de veces que otro producto se ha comprado junto con el producto dado.

La Figura 10.16 muestra un ejemplo de cinco productos diferentes con IDs del 1 al 5 y cinco pedidos de compra de diferentes combinaciones de productos:

> *Figura 10.16: Cinco productos con sus respectivos IDs y combinaciones de pedidos de compra*

En la figura, puedes ver un conjunto ordenado creado en Redis para cada producto, con la clave `product:<id>:purchased_with`, donde `<id>` es el identificador único del producto. Los miembros del conjunto ordenado son los IDs de los productos que se han comprado junto con el producto principal. La puntuación para cada miembro refleja el recuento acumulativo de compras conjuntas. La figura muestra la operación `ZINCRBY` de Redis para incrementar en 1 la puntuación de los productos comprados juntos en un pedido.

Ahora tienes un método para almacenar y puntuar los productos que se compraron juntos. A continuación, necesitas un método para recuperar los productos que se compraron juntos para una lista de productos dados. Añade el siguiente método `suggest_products_for()` a la clase `Recommender`:

```python
    def suggest_products_for(self, products, max_results=6):
        product_ids = [p.id for p in products]
        if len(products) == 1:
            # only 1 product
            suggestions = r.zrange(
                self.get_product_key(product_ids[0]),
                0,
                -1,
                desc=True
            )[:max_results]
        else:
            # generate a temporary key
            flat_ids = ''.join([str(id) for id in product_ids])
            tmp_key = f'tmp_{flat_ids}'
            # multiple products, combine scores of all products
            # store the resulting sorted set in a temporary key
            keys = [self.get_product_key(id) for id in product_ids]
            r.zunionstore(tmp_key, keys)
            # remove ids for the products the recommendation is for
            r.zrem(tmp_key, *product_ids)
            # get the product ids by their score, descendant sort
            suggestions = r.zrange(
                tmp_key,
                0,
                -1,
                desc=True
            )[:max_results]
            # remove the temporary key
            r.delete(tmp_key)
        suggested_products_ids = [int(id) for id in suggestions]
        # get suggested products and sort by order of appearance
        suggested_products = list(
            Product.objects.filter(id__in=suggested_products_ids)
        )
        suggested_products.sort(
            key=lambda x: suggested_products_ids.index(x.id)
        )
        return suggested_products
```

El método `suggest_products_for()` recibe los siguientes parámetros:

- `products`: Esta es una lista de objetos `Product` para obtener recomendaciones. Puede contener uno o más productos.
- `max_results`: Este es un entero que representa el número máximo de recomendaciones a devolver.

En este método, realizas las siguientes acciones:

- Obtienes los IDs de producto para los objetos `Product` dados.
- Si solo se proporciona un producto, recuperas el ID de los productos que se compraron junto con el producto dado, ordenados por el número total de veces que se compraron juntos. Para hacerlo, utilizas el comando `ZRANGE` de Redis. Limitas el número de resultados al número especificado en el atributo `max_results` (6 de forma predeterminada). Puedes leer más sobre el comando `ZRANGE` en [https://redis.io/commands/zrange/](https://redis.io/commands/zrange/).
- Si se proporciona más de un producto, generas una clave temporal de Redis construida con los IDs de los productos.
- Combinas y sumas todas las puntuaciones de los elementos contenidos en el conjunto ordenado de cada uno de los productos dados. Esto se realiza utilizando el comando `ZUNIONSTORE` de Redis. El comando `ZUNIONSTORE` realiza una unión de los conjuntos ordenados con las claves dadas y almacena la suma agregada de las puntuaciones de los elementos en una nueva clave de Redis. Puedes leer más sobre este comando en [https://redis.io/commands/zunionstore/](https://redis.io/commands/zunionstore/). Guardas las puntuaciones agregadas en la clave temporal.
- Dado que estás agregando puntuaciones, podrías obtener los mismos productos para los que estás solicitando recomendaciones. Los eliminas del conjunto ordenado generado utilizando el comando `ZREM`. Puedes leer más sobre el comando `ZREM` en [https://redis.io/commands/zrem/](https://redis.io/commands/zrem/).
- Recuperas los IDs de los productos de la clave temporal, ordenados por sus puntuaciones utilizando el comando `ZRANGE`. Limitas el número de resultados al número especificado en el atributo `max_results`.
- Luego, eliminas la clave temporal utilizando el método `delete()` de `redis-py`, que ejecuta el comando `DEL` de Redis. Puedes leer más sobre el comando `DEL` en [https://redis.io/commands/del/](https://redis.io/commands/del/).
- Finalmente, obtienes los objetos `Product` con los IDs dados y los ordenas en el mismo orden que los miembros del conjunto ordenado.

La Figura 10.17 muestra un ejemplo de una sesión en la que se han añadido dos productos al carrito de compras y las operaciones de Redis realizadas para obtener recomendaciones de productos relacionados:

> *Figura 10.17: Sistema de recomendación de productos*

En la figura, puedes ver los cuatro pasos para generar recomendaciones de productos para los artículos en el carrito:

1. El comando `ZUNIONSTORE` de Redis se utiliza para agregar las puntuaciones de los productos comprados frecuentemente con los productos del carrito de compras. El conjunto ordenado resultante de esta operación se almacena en una nueva clave de Redis nombrada con los IDs de los productos en el carrito de compras, `tmp_34` para los IDs 3 y 4.
2. El comando `ZREM` se utiliza para eliminar los productos que se están comprando del conjunto ordenado, para evitar recomendar productos que ya están en el carrito de compras.
3. El comando `ZRANGE` se utiliza para devolver los miembros del conjunto ordenado `tmp_34` ordenados por puntuación.
4. Finalmente, el comando `DEL` se utiliza para eliminar la clave de Redis `tmp_34`.

Para propósitos prácticos, añadamos también un método para borrar las recomendaciones. Añade el siguiente método a la clase `Recommender`:

```python
    def clear_purchases(self):
        for id in Product.objects.values_list('id', flat=True):
            r.delete(self.get_product_key(id))
```

Probemos el motor de recomendaciones. Asegúrate de incluir varios objetos `Product` en la base de datos e inicializa el contenedor Docker de Redis utilizando el siguiente comando:

```bash
docker run -it --rm --name redis -p 6379:6379 redis:7.2.4
```

Abre otra consola y ejecuta el siguiente comando para abrir la shell de Python:

```bash
python manage.py shell
```

Asegúrate de tener al menos cuatro productos diferentes en tu base de datos. Recupera cuatro productos diferentes por sus nombres:

```python
from shop.models import Product
black_tea = Product.objects.get(name='Black tea')
red_tea = Product.objects.get(name='Red tea')
green_tea = Product.objects.get(name='Green tea')
tea_powder = Product.objects.get(name='Tea powder')
```

Luego, añade algunas compras de prueba al motor de recomendaciones:

```python
from shop.recommender import Recommender
r = Recommender()
r.products_bought([black_tea, red_tea])
r.products_bought([black_tea, green_tea])
r.products_bought([red_tea, black_tea, tea_powder])
r.products_bought([green_tea, tea_powder])
r.products_bought([black_tea, tea_powder])
r.products_bought([red_tea, green_tea])
```

Has almacenado las siguientes puntuaciones:

```text
black_tea: red_tea (2), tea_powder (2), green_tea (1)
red_tea: black_tea (2), tea_powder (1), green_tea (1)
green_tea: black_tea (1), tea_powder (1), red_tea(1)
tea_powder: black_tea (2), red_tea (1), green_tea (1)
```

Esta es una representación de los productos que se han comprado junto con cada uno de los productos, incluyendo cuántas veces se han comprado juntos.

Recuperemos recomendaciones de productos para un solo producto:

```python
>>> r.suggest_products_for([black_tea])
[<Product: Tea powder>, <Product: Red tea>, <Product: Green tea>]
>>> r.suggest_products_for([red_tea])
[<Product: Black tea>, <Product: Tea powder>, <Product: Green tea>]
>>> r.suggest_products_for([green_tea])
[<Product: Black tea>, <Product: Tea powder>, <Product: Red tea>]
>>> r.suggest_products_for([tea_powder])
[<Product: Black tea>, <Product: Red tea>, <Product: Green tea>]
```

Puedes ver que el orden de los productos recomendados se basa en su puntuación. Obtengamos recomendaciones para múltiples productos con puntuaciones agregadas:

```python
>>> r.suggest_products_for([black_tea, red_tea])
[<Product: Tea powder>, <Product: Green tea>]
>>> r.suggest_products_for([green_tea, red_tea])
[<Product: Black tea>, <Product: Tea powder>]
>>> r.suggest_products_for([tea_powder, black_tea])
[<Product: Red tea>, <Product: Green tea>]
```

Puedes ver que el orden de los productos sugeridos coincide con las puntuaciones agregadas. Por ejemplo, los productos sugeridos para `black_tea` y `red_tea` son `tea_powder` (2+1) y `green_tea` (1+1).

Has verificado que tu algoritmo de recomendación funciona según lo previsto.

Almacenemos los productos que se compran juntos cada vez que se confirma un pago. Edita el archivo `webhooks.py` de la aplicación `payment` y añade el siguiente código:

```python
# ...
from shop.models import Product
from shop.recommender import Recommender


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
            # save items bought for product recommendations
            product_ids = order.items.values_list('product_id')
            products = Product.objects.filter(id__in=product_ids)
            r = Recommender()
            r.products_bought(products)
            # launch asynchronous task
            payment_completed.delay(order.id)
    return HttpResponse(status=200)
```

En el nuevo código, cuando se confirma el pago de un nuevo pedido, recuperas los objetos `Product` asociados con los artículos del pedido. Luego, creas una instancia de la clase `Recommender` y llamas al método `products_bought()` para almacenar en Redis los productos comprados juntos.

Ahora estás almacenando los productos relacionados que se compran juntos cuando se pagan los pedidos. Mostremos ahora recomendaciones para productos en tu sitio.

Edita el archivo `views.py` de la aplicación `shop`. Añade la funcionalidad para recuperar un máximo de cuatro productos recomendados en la vista `product_detail`:

```python
from .recommender import Recommender


def product_detail(request, id, slug):
    product = get_object_or_404(
        Product, id=id, slug=slug, available=True
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

Edita la plantilla `shop/product/detail.html` de la aplicación `shop` y añade el siguiente código después de `{{ product.description|linebreaks }}`:

```html
{% if recommended_products %}
    <div class="recommendations">
        <h3>People who bought this also bought</h3>
        {% for p in recommended_products %}
            <div class="item">
                <a href="{{ p.get_absolute_url }}">
                    <img src="{% if p.image %}{{ p.image.url }}{% else %} {% static "img/no_image.png" %}{% endif %}">
                </a>
                <p><a href="{{ p.get_absolute_url }}">{{ p.name }}</a></p>
            </div>
        {% endfor %}
    </div>
{% endif %}
```

Ejecuta el servidor de desarrollo y abre `http://127.0.0.1:8000/` en tu navegador. Haz clic en cualquier producto para ver sus detalles. Deberías ver que los productos recomendados se muestran debajo del producto:

> *Figura 10.18: La página de detalle de producto, incluyendo productos recomendados (Créditos de imágenes: Té verde: Foto por Jia Ye en Unsplash; Té rojo: Foto por Manki Kim en Unsplash; Té en polvo: Foto por Phuong Nguyen en Unsplash)*

También vas a incluir recomendaciones de productos en el carrito. Las recomendaciones se basarán en los productos que el usuario haya añadido al carrito.

Edita `views.py` dentro de la aplicación `cart`, importa la clase `Recommender` y edita la vista `cart_detail` para que se vea así:

```python
from shop.recommender import Recommender


def cart_detail(request):
    cart = Cart(request)
    for item in cart:
        item['update_quantity_form'] = CartAddProductForm(
            initial={'quantity': item['quantity'], 'override': True}
        )
    coupon_apply_form = CouponApplyForm()
    r = Recommender()
    cart_products = [item['product'] for item in cart]
    if(cart_products):
        recommended_products = r.suggest_products_for(
            cart_products,
            max_results=4
        )
    else:
        recommended_products = []
    return render(
        request,
        'cart/detail.html',
        {
            'cart': cart,
            'coupon_apply_form': coupon_apply_form,
            'recommended_products': recommended_products
        }
    )
```

Edita la plantilla `cart/detail.html` de la aplicación `cart` y añade el siguiente código justo después de la etiqueta HTML `</table>`:

```html
{% if recommended_products %}
    <div class="recommendations cart">
        <h3>People who bought this also bought</h3>
        {% for p in recommended_products %}
            <div class="item">
                <a href="{{ p.get_absolute_url }}">
                    <img src="{% if p.image %}{{ p.image.url }}{% else %} {% static "img/no_image.png" %}{% endif %}">
                </a>
                <p><a href="{{ p.get_absolute_url }}">{{ p.name }}</a></p>
            </div>
        {% endfor %}
    </div>
{% endif %}
```

Abre `http://127.0.0.1:8000/` en tu navegador y añade un par de productos a tu carrito. Cuando navegues a `http://127.0.0.1:8000/cart/`, deberías ver las recomendaciones de productos agregadas para los artículos en el carrito:

> *Figura 10.19: La página de detalles del carrito de compras, incluyendo productos recomendados*

¡Felicidades! Has construido un motor de recomendaciones completo usando Django y Redis.

---

### Resumen

En este capítulo, creaste un sistema de cupones utilizando sesiones de Django y lo integraste con Stripe. También construiste un motor de recomendaciones utilizando Redis para recomendar productos que habitualmente se compran juntos.

El próximo capítulo te proporcionará una visión profunda sobre la internacionalización y localización de proyectos Django. Aprenderás a traducir código y gestionar traducciones con Rosetta. Implementarás URLs para traducciones y construirás un selector de idioma. También implementarás traducciones de modelos usando `django-parler` y validarás campos de formulario localizados usando `django-localflavor`.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter10](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter10)
- **Descuentos para Stripe Checkout:** [https://stripe.com/docs/payments/checkout/discounts](https://stripe.com/docs/payments/checkout/discounts)
- **El comando ZRANGE de Redis:** [https://redis.io/commands/zrange/](https://redis.io/commands/zrange/)
- **El comando ZUNIONSTORE de Redis:** [https://redis.io/commands/zunionstore/](https://redis.io/commands/zunionstore/)
- **El comando ZREM de Redis:** [https://redis.io/commands/zrem/](https://redis.io/commands/zrem/)
- **El comando DEL de Redis:** [https://redis.io/commands/del/](https://redis.io/commands/del/)
