# Parte 4: Creación de una plataforma de E-Learning

## Capítulo 13: Creación de un sistema de gestión de contenidos (CMS)

### Introducción

En el capítulo anterior, creaste los modelos de la aplicación para la plataforma de e-learning y aprendiste a crear y aplicar fixtures de datos para los modelos. Creaste un campo de modelo personalizado para ordenar objetos e implementaste la autenticación de usuarios.

En este capítulo, aprenderás a construir la funcionalidad para que los instructores creen cursos y gestionen el contenido de dichos cursos de manera versátil y eficiente.

Se te presentarán las vistas basadas en clases (*class-based views*), que ofrecen una nueva perspectiva para construir tu aplicación en comparación con las vistas basadas en funciones que has construido en ejemplos anteriores. También explorarás la reutilización del código y la modularidad a través del uso de mixins, técnicas que podrás aplicar en futuros proyectos.

En este capítulo, aprenderás a:

- Crear un sistema de gestión de contenidos (*CMS*) utilizando vistas basadas en clases y mixins
- Construir formsets y model formsets para editar módulos de cursos y contenidos de módulos
- Gestionar grupos y permisos
- Implementar una funcionalidad de arrastrar y soltar (*drag-and-drop*) para reordenar módulos y contenidos

---

### Visión general funcional

La Figura 13.1 muestra una representación de las vistas, plantillas y funcionalidades que se construirán en este capítulo:

> *Figura 13.1: Diagrama de las funcionalidades construidas en el Capítulo 13*

En este capítulo, implementarás diferentes vistas basadas en clases. Crearás las clases mixin `OwnerMixin`, `OwnerEditMixin` y `OwnerCourseMixin`, que contendrán la funcionalidad común que reutilizarás en otras clases. Crearás vistas CRUD (*Create, Read, Update, Delete*) para el modelo `Course` implementando `ManageCourseListView` para listar cursos, `CourseCreateView` para crear cursos, `CourseUpdateView` para actualizar cursos y `CourseDeleteView` para eliminar cursos. Construirás la vista `CourseModuleUpdateView` para añadir/editar/eliminar módulos de cursos y `ModuleContentListView` para listar los contenidos de los módulos. También implementarás `ContentCreateUpdateView` para crear y actualizar contenidos de cursos y `ContentDeleteView` para eliminar contenidos. Finalmente, implementarás una funcionalidad de arrastrar y soltar para reordenar módulos y contenidos de cursos utilizando las vistas `ModuleOrderView` y `ContentOrderView`, respectivamente.

Ten en cuenta que todas las vistas que heredan el mixin `OwnerCourseMixin` redirigen al usuario de regreso a la vista `ManageCourseListView` tras una acción exitosa. Estas redirecciones no se han añadido al diagrama para simplificarlo.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter13](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter13).

Todos los módulos de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente que acompaña a este capítulo. Puedes seguir las instrucciones para instalar cada módulo de Python a continuación o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Creación de un CMS

Ahora que has creado un modelo de datos versátil, vas a construir el CMS. El CMS permitirá a los instructores crear cursos y gestionar su contenido. Necesitas proporcionar la siguiente funcionalidad:

- Listar los cursos creados por el instructor
- Crear, editar y eliminar cursos
- Añadir módulos a un curso y reordenarlos
- Añadir diferentes tipos de contenido a cada módulo
- Reordenar módulos y contenidos de cursos

Comencemos con las vistas CRUD básicas.

#### Creación de vistas basadas en clases

Vas a construir vistas para crear, editar y eliminar cursos. Utilizarás vistas basadas en clases para esto. Edita el archivo `views.py` de la aplicación `courses` y añade el siguiente código:

```python
from django.views.generic.list import ListView
from .models import Course


class ManageCourseListView(ListView):
    model = Course
    template_name = 'courses/manage/course/list.html'

    def get_queryset(self):
        qs = super().get_queryset()
        return qs.filter(owner=self.request.user)
```

Esta es la vista `ManageCourseListView`. Hereda de la vista genérica `ListView` de Django. Sobrescribes el método `get_queryset()` de la vista para recuperar únicamente los cursos creados por el usuario actual. Para evitar que los usuarios editen, actualicen o eliminen cursos que no han creado, también necesitarás sobrescribir el método `get_queryset()` en las vistas de creación, actualización y eliminación. Cuando necesitas proporcionar un comportamiento específico para varias vistas basadas en clases, se recomienda utilizar mixins.

#### Uso de mixins para vistas basadas en clases

Los mixins son un tipo especial de herencia múltiple para una clase. Si no estás familiarizado con los mixins en Python, todo lo que necesitas entender es que son un tipo de clase diseñada para suministrar métodos a otras clases pero que no están destinadas a ser utilizadas de forma independiente. Esto te permite desarrollar funcionalidades compartidas que pueden incorporarse en varias clases de manera modular, simplemente haciendo que esas clases hereden de los mixins. El concepto es similar a una clase base, pero puedes utilizar múltiples mixins para extender la funcionalidad de una clase determinada.

Hay dos situaciones principales para utilizar mixins:

1. Quieres proporcionar múltiples características opcionales para una clase.
2. Quieres utilizar una característica particular en varias clases.

Django incluye varios mixins que proporcionan funcionalidad adicional a tus vistas basadas en clases. Puedes obtener más información sobre los mixins en [https://docs.djangoproject.com/en/5.2/topics/class-based-views/mixins/](https://docs.djangoproject.com/en/5.2/topics/class-based-views/mixins/).

Vas a implementar un comportamiento común para múltiples vistas en clases mixin y lo utilizarás para las vistas de los cursos. Edita el archivo `views.py` de la aplicación `courses` y modifícalo de la siguiente manera:

```python
from django.urls import reverse_lazy
from django.views.generic.edit import CreateView, DeleteView, UpdateView
from django.views.generic.list import ListView
from .models import Course


class OwnerMixin:
    def get_queryset(self):
        qs = super().get_queryset()
        return qs.filter(owner=self.request.user)


class OwnerEditMixin:
    def form_valid(self, form):
        form.instance.owner = self.request.user
        return super().form_valid(form)


class OwnerCourseMixin(OwnerMixin):
    model = Course
    fields = ['subject', 'title', 'slug', 'overview']
    success_url = reverse_lazy('manage_course_list')


class OwnerCourseEditMixin(OwnerCourseMixin, OwnerEditMixin):
    template_name = 'courses/manage/course/form.html'


class ManageCourseListView(OwnerCourseMixin, ListView):
    template_name = 'courses/manage/course/list.html'


class CourseCreateView(OwnerCourseEditMixin, CreateView):
    pass


class CourseUpdateView(OwnerCourseEditMixin, UpdateView):
    pass


class CourseDeleteView(OwnerCourseMixin, DeleteView):
    template_name = 'courses/manage/course/delete.html'
```

En este código, creas los mixins `OwnerMixin` y `OwnerEditMixin`. Utilizarás estos mixins junto con las vistas `ListView`, `CreateView`, `UpdateView` y `DeleteView` proporcionadas por Django. `OwnerMixin` implementa el método `get_queryset()`, que es utilizado por las vistas para obtener el QuerySet base. Tu mixin sobrescribirá este método para filtrar objetos por el atributo `owner` para recuperar objetos que pertenecen al usuario actual (`request.user`).

`OwnerEditMixin` implementa el método `form_valid()`, que es utilizado por las vistas que usan el mixin `ModelFormMixin` de Django, es decir, vistas con formularios o formularios de modelos como `CreateView` y `UpdateView`. `form_valid()` se ejecuta cuando el formulario enviado es válido.

El comportamiento predeterminado de este método es guardar la instancia (para formularios de modelos) y redirigir al usuario a `success_url`. Sobrescribes este método para establecer automáticamente el usuario actual en el atributo `owner` del objeto que se está guardando. Al hacerlo, estableces el propietario de un objeto automáticamente cuando se guarda.

Tu clase `OwnerMixin` se puede utilizar para vistas que interactúan con cualquier modelo que contenga un atributo `owner`.

También defines una clase `OwnerCourseMixin` que hereda de `OwnerMixin` y proporciona los siguientes atributos para las vistas hijas:

- `model`: El modelo utilizado para los QuerySets; es utilizado por todas las vistas.
- `fields`: Los campos del modelo para construir el formulario de modelo de las vistas `CreateView` y `UpdateView`.
- `success_url`: Utilizado por `CreateView`, `UpdateView` y `DeleteView` para redirigir al usuario después de que el formulario se envíe con éxito o el objeto se elimine. Utilizas una URL con el nombre `manage_course_list`, que crearás más adelante.

Defines un mixin `OwnerCourseEditMixin` con el siguiente atributo:

- `template_name`: La plantilla que utilizarás para las vistas `CreateView` y `UpdateView`.

Finalmente, creas las siguientes vistas que heredan de `OwnerCourseMixin`:

- `ManageCourseListView`: Lista los cursos creados por el usuario. Hereda de `OwnerCourseMixin` y `ListView`. Define un atributo `template_name` específico para una plantilla para listar cursos.
- `CourseCreateView`: Utiliza un formulario de modelo para crear un nuevo objeto `Course`. Utiliza los campos definidos en `OwnerCourseMixin` para construir un formulario de modelo y también hereda de `CreateView`. Utiliza la plantilla definida en `OwnerCourseEditMixin`.
- `CourseUpdateView`: Permite editar un objeto `Course` existente. Utiliza los campos definidos en `OwnerCourseMixin` para construir un formulario de modelo y también hereda de `UpdateView`. Utiliza la plantilla definida en `OwnerCourseEditMixin`.
- `CourseDeleteView`: Hereda de `OwnerCourseMixin` y de la vista genérica `DeleteView`. Define un atributo `template_name` específico para una plantilla para confirmar la eliminación del curso.

Has creado las vistas básicas para gestionar cursos. Si bien has implementado vistas CRUD por tu cuenta, la aplicación de terceros Neapolitan te permite implementar las vistas estándar de lista, detalle, creación y eliminación dentro de una sola vista. Puedes aprender más sobre Neapolitan en [https://github.com/carltongibson/neapolitan](https://github.com/carltongibson/neapolitan).

A continuación, vas a utilizar grupos y permisos de autenticación de Django para limitar el acceso a estas vistas.

#### Trabajo con grupos y permisos

Actualmente, cualquier usuario puede acceder a las vistas para gestionar cursos. Quieres restringir estas vistas para que solo los instructores tengan permiso para crear y gestionar cursos.

El framework de autenticación de Django incluye un sistema de permisos. De forma predeterminada, Django genera cuatro permisos para cada modelo en las aplicaciones instaladas: *add*, *view*, *change* y *delete*. Estos permisos corresponden a las acciones de crear nuevas instancias, ver las existentes, editar y eliminar instancias de un modelo.

Los permisos se pueden asignar directamente a usuarios individuales o a grupos de usuarios. Este enfoque simplifica la gestión de usuarios agrupando permisos y mejora la seguridad de tu aplicación.

Vas a crear un grupo para usuarios instructores y asignarás permisos para crear, actualizar y eliminar cursos.

Ejecuta el servidor de desarrollo usando el siguiente comando:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/admin/auth/group/add/` en tu navegador para crear un nuevo objeto `Group`. Añade el nombre `Instructors` y elige todos los permisos de la aplicación `courses`, excepto los del modelo `Subject`:

> *Figura 13.2: Permisos del grupo Instructors*

Como puedes ver, hay cuatro permisos diferentes para cada modelo: *can view*, *can add*, *can change* y *can delete*. Después de elegir los permisos para este grupo, haz clic en el botón **SAVE**.

Abre `http://127.0.0.1:8000/admin/auth/user/add/` y crea un nuevo usuario. Edita el usuario y añádelo al grupo `Instructors`:

> *Figura 13.3: Selección de grupo de usuarios*

Los usuarios heredan los permisos de los grupos a los que pertenecen, pero también puedes añadir permisos individuales a un solo usuario utilizando el sitio de administración. Los usuarios que tienen `is_superuser` establecido en `True` tienen todos los permisos automáticamente.

A continuación, aplicarás permisos en la práctica incorporándolos a nuestras vistas.

#### Restricción de acceso a vistas basadas en clases

Vas a restringir el acceso a las vistas para que solo los usuarios con los permisos adecuados puedan añadir, modificar o eliminar objetos `Course`. Utilizarás los siguientes dos mixins proporcionados por `django.contrib.auth` para limitar el acceso a las vistas:

- `LoginRequiredMixin`: Replica la funcionalidad del decorador `login_required`.
- `PermissionRequiredMixin`: Concede acceso a la vista a los usuarios con un permiso específico. Recuerda que los superusuarios tienen automáticamente todos los permisos.

Edita el archivo `views.py` de la aplicación `courses` y añade la siguiente importación:

```python
from django.contrib.auth.mixins import (
    LoginRequiredMixin,
    PermissionRequiredMixin
)
```

Haz que `OwnerCourseMixin` herede de `LoginRequiredMixin` y `PermissionRequiredMixin`, de la siguiente manera:

```python
class OwnerCourseMixin(
    OwnerMixin,
    LoginRequiredMixin,
    PermissionRequiredMixin
):
    model = Course
    fields = ['subject', 'title', 'slug', 'overview']
    success_url = reverse_lazy('manage_course_list')
```

Luego, añade un atributo `permission_required` a las vistas de cursos, de la siguiente manera:

```python
class ManageCourseListView(OwnerCourseMixin, ListView):
    template_name = 'courses/manage/course/list.html'
    permission_required = 'courses.view_course'


class CourseCreateView(OwnerCourseEditMixin, CreateView):
    permission_required = 'courses.add_course'


class CourseUpdateView(OwnerCourseEditMixin, UpdateView):
    permission_required = 'courses.change_course'


class CourseDeleteView(OwnerCourseMixin, DeleteView):
    template_name = 'courses/manage/course/delete.html'
    permission_required = 'courses.delete_course'
```

`PermissionRequiredMixin` comprueba que el usuario que accede a la vista tenga el permiso especificado en el atributo `permission_required`. Tus vistas ahora solo son accesibles para usuarios con los permisos correspondientes.

Creemos las URLs para estas vistas. Crea un nuevo archivo dentro del directorio de la aplicación `courses` y nómbralo `urls.py`. Añade el siguiente código:

```python
from django.urls import path
from . import views

urlpatterns = [
    path(
        'mine/',
        views.ManageCourseListView.as_view(),
        name='manage_course_list'
    ),
    path(
        'create/',
        views.CourseCreateView.as_view(),
        name='course_create'
    ),
    path(
        '<pk>/edit/',
        views.CourseUpdateView.as_view(),
        name='course_edit'
    ),
    path(
        '<pk>/delete/',
        views.CourseDeleteView.as_view(),
        name='course_delete'
    ),
]
```

Estos son los patrones de URL para las vistas de listar, crear, editar y eliminar cursos. El parámetro `pk` hace referencia al campo de clave primaria (*primary key*). Recuerda que `pk` es la abreviatura de clave primaria. Cada modelo de Django tiene un campo que sirve como su clave primaria. De forma predeterminada, la clave primaria es el campo `id` generado automáticamente. Las vistas genéricas de Django para objetos individuales recuperan un objeto mediante su campo `pk`. Edita el archivo `urls.py` principal del proyecto `educa` e incluye los patrones de URL de la aplicación `courses`, como se muestra a continuación:

```python
from django.conf import settings
from django.conf.urls.static import static
from django.contrib import admin
from django.contrib.auth import views as auth_views
from django.urls import include, path

urlpatterns = [
    path(
        'accounts/login/',
        auth_views.LoginView.as_view(),
        name='login'
    ),
    path(
        'accounts/logout/',
        auth_views.LogoutView.as_view(),
        name='logout'
    ),
    path('admin/', admin.site.urls),
    path('course/', include('courses.urls')),
]

if settings.DEBUG:
    urlpatterns += static(
        settings.MEDIA_URL,
        document_root=settings.MEDIA_ROOT
    )
```

Necesitas crear las plantillas para estas vistas. Crea los siguientes directorios y archivos dentro del directorio `templates/` de la aplicación `courses`:

```text
courses/
    manage/
        course/
            list.html
            form.html
            delete.html
```

Edita la plantilla `courses/manage/course/list.html` y añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}My courses{% endblock %}

{% block content %}
    <h1>My courses</h1>
    <div class="module">
        {% for course in object_list %}
            <div class="course-info">
                <h3>{{ course.title }}</h3>
                <p>
                    <a href="{% url "course_edit" course.id %}">Edit</a>
                    <a href="{% url "course_delete" course.id %}">Delete</a>
                </p>
            </div>
        {% empty %}
            <p>You haven't created any courses yet.</p>
        {% endfor %}
        <p>
            <a href="{% url "course_create" %}" class="button">Create new course</a>
        </p>
    </div>
{% endblock %}
```

Esta es la plantilla para la vista `ManageCourseListView`. En esta plantilla, listas los cursos creados por el usuario actual. Incluyes enlaces para editar o eliminar cada curso y un enlace para crear nuevos cursos.

Ejecuta el servidor de desarrollo usando el siguiente comando:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/accounts/login/?next=/course/mine/` en tu navegador e inicia sesión con un usuario perteneciente al grupo `Instructors`. Tras iniciar sesión, serás redirigido a la URL `http://127.0.0.1:8000/course/mine/` y deberías ver la siguiente página:

> *Figura 13.4: La página de cursos del instructor sin ningún curso*

Esta página mostrará todos los cursos creados por el usuario actual.

Creemos la plantilla que muestra el formulario para las vistas de creación y actualización de cursos. Edita la plantilla `courses/manage/course/form.html` y escribe el siguiente código:

```html
{% extends "base.html" %}

{% block title %}
    {% if object %}
        Edit course "{{ object.title }}"
    {% else %}
        Create a new course
    {% endif %}
{% endblock %}

{% block content %}
    <h1>
        {% if object %}
            Edit course "{{ object.title }}"
        {% else %}
            Create a new course
        {% endif %}
    </h1>
    <div class="module">
        <h2>Course info</h2>
        <form method="post">
            {{ form.as_p }}
            {% csrf_token %}
            <p><input type="submit" value="Save course"></p>
        </form>
    </div>
{% endblock %}
```

La plantilla `form.html` se utiliza tanto para la vista `CourseCreateView` como para `CourseUpdateView`. En esta plantilla, compruebas si hay una variable `object` en el contexto. Si `object` existe en el contexto, sabes que estás actualizando un curso existente y lo utilizas en el título de la página. De lo contrario, estás creando un nuevo objeto `Course`.

Abre `http://127.0.0.1:8000/course/mine/` en tu navegador y haz clic en el botón **CREATE NEW COURSE**. Verás la siguiente página:

> *Figura 13.5: El formulario para crear un nuevo curso*

Rellena el formulario y haz clic en el botón **SAVE COURSE**. El curso se guardará y serás redirigido a la página de lista de cursos:

> *Figura 13.6: La página de cursos del instructor con un curso*

Luego, haz clic en el enlace **Edit** para el curso que acabas de crear. Verás el formulario de nuevo pero, esta vez, estás editando un objeto `Course` existente en lugar de crear uno.

Finalmente, edita la plantilla `courses/manage/course/delete.html` y añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}Delete course{% endblock %}

{% block content %}
    <h1>Delete course "{{ object.title }}"</h1>
    <div class="module">
        <form action="" method="post">
            {% csrf_token %}
            <p>Are you sure you want to delete "{{ object }}"?</p>
            <input type="submit" value="Confirm">
        </form>
    </div>
{% endblock %}
```

Esta es la plantilla para la vista `CourseDeleteView`. Esta vista hereda de `DeleteView`, proporcionada por Django, que espera la confirmación del usuario para eliminar un objeto.

Abre la lista de cursos en el navegador y haz clic en el enlace **Delete** de tu curso. Deberías ver la siguiente página de confirmación:

> *Figura 13.7: La página de confirmación de eliminación de curso*

Haz clic en el botón **CONFIRM**. El curso se eliminará y serás redirigido a la página de lista de cursos de nuevo.

Los instructores ahora pueden crear, editar y eliminar cursos. A continuación, necesitas proporcionarles un CMS para añadir módulos de cursos y sus contenidos. Comenzarás gestionando los módulos de los cursos.

---

### Gestión de módulos de cursos y sus contenidos

Vas a construir un sistema para gestionar los módulos de los cursos y sus contenidos. Necesitarás construir formularios que puedan ser utilizados para gestionar múltiples módulos por curso y diferentes tipos de contenido para cada módulo. Tanto los módulos como sus contenidos tendrán que seguir un orden específico y deberías poder reordenarlos utilizando el CMS.

#### Uso de formsets para módulos de cursos

Django incluye una capa de abstracción para trabajar con múltiples formularios en la misma página. Estos grupos de formularios se conocen como **formsets**. Los formsets gestionan múltiples instancias de un determinado `Form` o `ModelForm`. Todos los formularios se envían a la vez y el formset se encarga del número inicial de formularios a mostrar, limitando el número máximo de formularios que se pueden enviar y validando todos los formularios.

Los formsets incluyen un método `is_valid()` para validar todos los formularios a la vez. También puedes proporcionar datos iniciales para los formularios y especificar cuántos formularios vacíos adicionales mostrar. Puedes obtener más información sobre los formsets en [https://docs.djangoproject.com/en/5.2/topics/forms/formsets/](https://docs.djangoproject.com/en/5.2/topics/forms/formsets/) y sobre los model formsets en [https://docs.djangoproject.com/en/5.2/topics/forms/modelforms/#model-formsets](https://docs.djangoproject.com/en/5.2/topics/forms/modelforms/#model-formsets).

Dado que un curso se divide en un número variable de módulos, tiene sentido utilizar formsets para gestionarlos. Crea un archivo `forms.py` en el directorio de la aplicación `courses` y añade el siguiente código:

```python
from django.forms.models import inlineformset_factory
from .models import Course, Module

ModuleFormSet = inlineformset_factory(
    Course,
    Module,
    fields=['title', 'description'],
    extra=2,
    can_delete=True
)
```

Este es el formset `ModuleFormSet`. Lo construyes utilizando la función `inlineformset_factory()` proporcionada por Django. Los *inline formsets* son una pequeña abstracción sobre los formsets que simplifica el trabajo con objetos relacionados. Esta función te permite construir un formset de modelo dinámicamente para los objetos `Module` relacionados con un objeto `Course`.

Utilizas los siguientes parámetros para construir el formset:

- `fields`: Los campos que se incluirán en cada formulario del formset.
- `extra`: Te permite establecer el número de formularios adicionales vacíos a mostrar en el formset.
- `can_delete`: Si lo estableces en `True`, Django incluirá un campo booleano para cada formulario que se representará como una casilla de verificación (*checkbox*). Te permite marcar los objetos que deseas eliminar.

Edita el archivo `views.py` de la aplicación `courses` y añade el siguiente código:

```python
from django.shortcuts import get_object_or_404, redirect
from django.views.generic.base import TemplateResponseMixin, View
from .forms import ModuleFormSet


class CourseModuleUpdateView(TemplateResponseMixin, View):
    template_name = 'courses/manage/module/formset.html'
    course = None

    def get_formset(self, data=None):
        return ModuleFormSet(instance=self.course, data=data)

    def dispatch(self, request, pk):
        self.course = get_object_or_404(
            Course, id=pk, owner=request.user
        )
        return super().dispatch(request, pk)

    def get(self, request, *args, **kwargs):
        formset = self.get_formset()
        return self.render_to_response(
            {'course': self.course, 'formset': formset}
        )

    def post(self, request, *args, **kwargs):
        formset = self.get_formset(data=request.POST)
        if formset.is_valid():
            formset.save()
            return redirect('manage_course_list')
        return self.render_to_response(
            {'course': self.course, 'formset': formset}
        )
```

La vista `CourseModuleUpdateView` maneja el formset para añadir, actualizar y eliminar módulos para un curso específico. Esta vista hereda de los siguientes mixins y vistas:

- `TemplateResponseMixin`: Este mixin se encarga de renderizar plantillas y devolver una respuesta HTTP. Requiere un atributo `template_name` que indica la plantilla a renderizar y proporciona el método `render_to_response()` para pasarle un contexto y renderizar la plantilla.
- `View`: La vista básica basada en clases proporcionada por Django.

En esta vista, implementas los siguientes métodos:

- `get_formset()`: Defines este método para evitar repetir el código para construir el formset. Creas un objeto `ModuleFormSet` para el objeto `Course` dado con datos opcionales.
- `dispatch()`: Este método es proporcionado por la clase `View`. Toma una petición HTTP y sus parámetros e intenta delegar en un método en minúsculas que coincida con el método HTTP utilizado. Una petición GET se delega al método `get()` y una petición POST a `post()`, respectivamente. En este método, utilizas la función auxiliar `get_object_or_404()` para obtener el objeto `Course` para el parámetro `id` dado que pertenece al usuario actual. Incluyes este código en el método `dispatch()` porque necesitas recuperar el curso tanto para peticiones GET como POST. Lo guardas en el atributo `course` de la vista para que sea accesible a otros métodos.
- `get()`: Se ejecuta para peticiones GET. Construyes un formset `ModuleFormSet` vacío y lo renderizas en la plantilla junto con el objeto `Course` actual, utilizando el método `render_to_response()` proporcionado por `TemplateResponseMixin`.
- `post()`: Se ejecuta para peticiones POST. En este método, realizas las siguientes acciones: construyes una instancia de `ModuleFormSet` utilizando los datos enviados, ejecutas el método `is_valid()` del formset para validar todos sus formularios y, si el formset es válido, lo guardas llamando al método `save()`. En este punto, cualquier cambio realizado (como añadir, actualizar o marcar módulos para su eliminación) se aplica a la base de datos. Luego, rediriges a los usuarios a la URL `manage_course_list`. Si el formset no es válido, renderizas la plantilla para mostrar los errores.

Edita el archivo `urls.py` de la aplicación `courses` y añade el siguiente patrón de URL:

```python
path(
    '<pk>/module/',
    views.CourseModuleUpdateView.as_view(),
    name='course_module_update'
),
```

Crea un nuevo directorio dentro del directorio de plantillas `courses/manage/` y nómbralo `module`. Crea una plantilla `courses/manage/module/formset.html` y añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}
    Edit "{{ course.title }}"
{% endblock %}

{% block content %}
    <h1>Edit "{{ course.title }}"</h1>
    <div class="module">
        <h2>Course modules</h2>
        <form method="post">
            {{ formset }}
            {{ formset.management_form }}
            {% csrf_token %}
            <input type="submit" value="Save modules">
        </form>
    </div>
{% endblock %}
```

En esta plantilla, creas un elemento HTML `<form>` en el que incluyes `formset`. También incluyes el formulario de gestión (*management form*) para el formset con la variable `{{ formset.management_form }}`. El formulario de gestión incluye campos ocultos para controlar el número inicial, total, mínimo y máximo de formularios.

Edita la plantilla `courses/manage/course/list.html` y añade el siguiente enlace para la URL `course_module_update` debajo de los enlaces Edit y Delete del curso:

```html
<a href="{% url "course_edit" course.id %}">Edit</a>
<a href="{% url "course_delete" course.id %}">Delete</a>
<a href="{% url "course_module_update" course.id %}">Edit modules</a>
```

Abre `http://127.0.0.1:8000/course/mine/` en tu navegador. Crea un curso y haz clic en el enlace **Edit modules** para el mismo. Deberías ver un formset:

> *Figura 13.8: La página de edición del curso, incluyendo el formset para módulos del curso*

El formset incluye un formulario para cada objeto `Module` contenido en el curso. Después de estos, se muestran dos formularios vacíos adicionales porque estableciste `extra=2` para `ModuleFormSet`. Cuando guardes el formset, Django incluirá otros dos campos adicionales para añadir nuevos módulos.

#### Adición de contenido a los módulos del curso

Ahora necesitas una forma de añadir contenido a los módulos de los cursos. Tienes cuatro tipos diferentes de contenido: texto, vídeo, imagen y archivo. Podrías considerar crear cuatro vistas diferentes para crear contenido, con un formulario para cada modelo. Sin embargo, vas a adoptar un enfoque más versátil y crearás una vista que maneje la creación o actualización de los objetos de cualquier modelo de contenido. Construirás el formulario para esta vista dinámicamente, según el tipo de contenido que el instructor desee añadir al curso: `Text`, `Video`, `Image` o `File`.

Edita el archivo `views.py` de la aplicación `courses` y añade el siguiente código:

```python
from django.apps import apps
from django.forms.models import modelform_factory
from .models import Module, Content


class ContentCreateUpdateView(TemplateResponseMixin, View):
    module = None
    model = None
    obj = None
    template_name = 'courses/manage/content/form.html'

    def get_model(self, model_name):
        if model_name in ['text', 'video', 'image', 'file']:
            return apps.get_model(
                app_label='courses', model_name=model_name
            )
        return None

    def get_form(self, model, *args, **kwargs):
        Form = modelform_factory(
            model, exclude=['owner', 'order', 'created', 'updated']
        )
        return Form(*args, **kwargs)

    def dispatch(self, request, module_id, model_name, id=None):
        self.module = get_object_or_404(
            Module, id=module_id, course__owner=request.user
        )
        self.model = self.get_model(model_name)
        if id:
            self.obj = get_object_or_404(
                self.model, id=id, owner=request.user
            )
        return super().dispatch(request, module_id, model_name, id)
```

Esta es la primera parte de `ContentCreateUpdateView`. Te permitirá crear y actualizar contenidos de diferentes modelos. Esta vista define los siguientes métodos:

- `get_model()`: Aquí compruebas que el nombre del modelo dado sea uno de los cuatro modelos de contenido: `Text`, `Video`, `Image` o `File`. Luego, utilizas el módulo `apps` de Django para obtener la clase real para el nombre de modelo dado. Si el nombre de modelo dado no es uno de los válidos, devuelves `None`.
- `get_form()`: Construyes un formulario dinámico utilizando la función `modelform_factory()` del framework de formularios. Dado que vas a construir un formulario para los modelos `Text`, `Video`, `Image` y `File`, utilizas el parámetro `exclude` para especificar los campos comunes que se excluirán del formulario y dejas que todos los demás atributos se incluyan automáticamente. Al hacerlo, no tienes que saber qué campos incluir según el modelo.
- `dispatch()`: Recibe los parámetros de URL y almacena el módulo, modelo y objeto de contenido correspondientes como atributos de clase:
  - `module_id`: El ID del módulo con el que el contenido está/estará asociado.
  - `model_name`: El nombre del modelo del contenido a crear/actualizar.
  - `id`: El ID del objeto que se está actualizando. Es `None` para crear nuevos objetos.

Añade los siguientes métodos `get()` y `post()` a `ContentCreateUpdateView`:

```python
    def get(self, request, module_id, model_name, id=None):
        form = self.get_form(self.model, instance=self.obj)
        return self.render_to_response(
            {'form': form, 'object': self.obj}
        )

    def post(self, request, module_id, model_name, id=None):
        form = self.get_form(
            self.model,
            instance=self.obj,
            data=request.POST,
            files=request.FILES
        )
        if form.is_valid():
            obj = form.save(commit=False)
            obj.owner = request.user
            obj.save()
            if not id:
                # new content
                Content.objects.create(module=self.module, item=obj)
            return redirect('module_content_list', self.module.id)
        return self.render_to_response(
            {'form': form, 'object': self.obj}
        )
```

Estos métodos son los siguientes:

- `get()`: Se ejecuta cuando se recibe una petición GET. Construyes el formulario de modelo para la instancia de `Text`, `Video`, `Image` o `File` que se está actualizando. De lo contrario, no pasas ninguna instancia para crear un nuevo objeto, ya que `self.obj` es `None` si no se proporciona ningún ID.
- `post()`: Se ejecuta cuando se recibe una petición POST. Construyes el formulario de modelo, pasándole los datos y archivos enviados. Luego, lo validas. Si el formulario es válido, creas un nuevo objeto y asignas `request.user` como su propietario antes de guardarlo en la base de datos. Compruebas el parámetro `id`. Si no se proporciona ningún ID, sabes que el usuario está creando un nuevo objeto en lugar de actualizar uno existente. Si este es un nuevo objeto, creas un objeto `Content` para el módulo dado y asocias el nuevo contenido con él.

Edita el archivo `urls.py` de la aplicación `courses` y añade los siguientes patrones de URL:

```python
path(
    'module/<int:module_id>/content/<model_name>/create/',
    views.ContentCreateUpdateView.as_view(),
    name='module_content_create'
),
path(
    'module/<int:module_id>/content/<model_name>/<id>/',
    views.ContentCreateUpdateView.as_view(),
    name='module_content_update'
),
```

Los nuevos patrones de URL son los siguientes:

- `module_content_create`: Para crear nuevos objetos de texto, vídeo, imagen o archivo y añadirlos a un módulo. Incluye los parámetros `module_id` y `model_name`. El primero te permite vincular el nuevo objeto de contenido al módulo dado. El segundo especifica el modelo de contenido para el cual construir el formulario.
- `module_content_update`: Para actualizar un objeto existente de texto, vídeo, imagen o archivo. Incluye los parámetros `module_id` y `model_name` y un parámetro `id` para identificar el contenido que se está actualizando.

Crea un nuevo directorio dentro del directorio de plantillas `courses/manage/` y nómbralo `content`. Crea la plantilla `courses/manage/content/form.html` y añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}
    {% if object %}
        Edit content "{{ object.title }}"
    {% else %}
        Add new content
    {% endif %}
{% endblock %}

{% block content %}
    <h1>
        {% if object %}
            Edit content "{{ object.title }}"
        {% else %}
            Add new content
        {% endif %}
    </h1>
    <div class="module">
        <h2>Course info</h2>
        <form action="" method="post" enctype="multipart/form-data">
            {{ form.as_p }}
            {% csrf_token %}
            <p><input type="submit" value="Save content"></p>
        </form>
    </div>
{% endblock %}
```

Esta es la plantilla para la vista `ContentCreateUpdateView`. Incluyes `enctype="multipart/form-data"` en el elemento HTML `<form>` porque el formulario contiene subida de archivos para los modelos de contenido `File` e `Image`.

Ejecuta el servidor de desarrollo, abre `http://127.0.0.1:8000/course/mine/`, haz clic en **Edit modules** para un curso existente y crea un módulo.

Luego, abre la shell de Python con el siguiente comando:

```bash
python manage.py shell
```

Obtén el ID del módulo más recientemente creado:

```python
>>> from courses.models import Module
>>> Module.objects.latest('id').id
6
```

Ejecuta el servidor de desarrollo y abre `http://127.0.0.1:8000/course/module/6/content/image/create/` en tu navegador, reemplazando el ID del módulo por el que obtuviste antes:

> *Figura 13.9: El formulario Add new content del curso*

También necesitas una vista para eliminar contenido. Edita el archivo `views.py` de la aplicación `courses` y añade el siguiente código:

```python
class ContentDeleteView(View):
    def post(self, request, id):
        content = get_object_or_404(
            Content, id=id, module__course__owner=request.user
        )
        module = content.module
        content.item.delete()
        content.delete()
        return redirect('module_content_list', module.id)
```

La clase `ContentDeleteView` recupera el objeto de contenido con el ID dado. Elimina el objeto `Text`, `Video`, `Image` o `File` relacionado. Finalmente, elimina el objeto `Content` y redirige al usuario a la URL `module_content_list` para listar los otros contenidos del módulo.

Edita el archivo `urls.py` de la aplicación `courses` y añade el siguiente patrón de URL:

```python
path(
    'content/<int:id>/delete/',
    views.ContentDeleteView.as_view(),
    name='module_content_delete'
),
```

#### Gestión de módulos y sus contenidos

A continuación, necesitas una vista para mostrar todos los módulos de un curso y listar los contenidos de un módulo específico.

Edita el archivo `views.py` de la aplicación `courses` y añade el siguiente código:

```python
class ModuleContentListView(TemplateResponseMixin, View):
    template_name = 'courses/manage/module/content_list.html'

    def get(self, request, module_id):
        module = get_object_or_404(
            Module, id=module_id, course__owner=request.user
        )
        return self.render_to_response({'module': module})
```

Esta es la vista `ModuleContentListView`. Obtiene el objeto `Module` con el ID dado que pertenece al usuario actual y renderiza una plantilla con el módulo dado.

Edita el archivo `urls.py` de la aplicación `courses` y añade el siguiente patrón de URL:

```python
path(
    'module/<int:module_id>/',
    views.ModuleContentListView.as_view(),
    name='module_content_list'
),
```

Crea una nueva estructura de directorios y archivo para filtros de plantilla personalizados dentro del directorio de la aplicación `courses`:

```text
templatetags/
    __init__.py
    course.py
```

Edita el módulo `course.py` y añade el siguiente código:

```python
from django import template

register = template.Library()


@register.filter
def model_name(obj):
    try:
        return obj._meta.model_name
    except AttributeError:
        return None
```

Este es el filtro de plantilla `model_name`. Puedes aplicarlo en las plantillas como `object|model_name` para obtener el nombre del modelo de un objeto.

Crea una nueva plantilla dentro del directorio `templates/courses/manage/module/` y nómbrala `content_list.html`. Añade el siguiente código:

```html
{% extends "base.html" %}
{% load course %}

{% block title %}
    Module {{ module.order|add:1 }}: {{ module.title }}
{% endblock %}

{% block content %}
    {% with course=module.course %}
        <h1>Course "{{ course.title }}"</h1>
        <div class="contents">
            <h3>Modules</h3>
            <ul id="modules">
                {% for m in course.modules.all %}
                    <li data-id="{{ m.id }}" {% if m == module %} class="selected"{% endif %}>
                        <a href="{% url "module_content_list" m.id %}">
                            <span>
                                Module <span class="order">{{ m.order|add:1 }}</span>
                            </span>
                            <br>
                            {{ m.title }}
                        </a>
                    </li>
                {% empty %}
                    <li>No modules yet.</li>
                {% endfor %}
            </ul>
            <p><a href="{% url "course_module_update" course.id %}">Edit modules</a></p>
        </div>
        <div class="module">
            <h2>Module {{ module.order|add:1 }}: {{ module.title }}</h2>
            <h3>Module contents:</h3>
            <div id="module-contents">
                {% for content in module.contents.all %}
                    <div data-id="{{ content.id }}">
                        {% with item=content.item %}
                            <p>{{ item }} ({{ item|model_name }})</p>
                            <a href="{% url "module_content_update" module.id item|model_name item.id %}">
                                Edit
                            </a>
                            <form action="{% url "module_content_delete" content.id %}" method="post">
                                <input type="submit" value="Delete">
                                {% csrf_token %}
                            </form>
                        {% endwith %}
                    </div>
                {% empty %}
                    <p>This module has no contents yet.</p>
                {% endfor %}
            </div>
            <h3>Add new content:</h3>
            <ul class="content-types">
                <li>
                    <a href="{% url "module_content_create" module.id "text" %}">
                        Text
                    </a>
                </li>
                <li>
                    <a href="{% url "module_content_create" module.id "image" %}">
                        Image
                    </a>
                </li>
                <li>
                    <a href="{% url "module_content_create" module.id "video" %}">
                        Video
                    </a>
                </li>
                <li>
                    <a href="{% url "module_content_create" module.id "file" %}">
                        File
                    </a>
                </li>
            </ul>
        </div>
    {% endwith %}
{% endblock %}
```

Edita la plantilla `courses/manage/course/list.html` y añade un enlace a la URL `module_content_list`:

```html
<a href="{% url "course_module_update" course.id %}">Edit modules</a>
{% if course.modules.count > 0 %}
    <a href="{% url "module_content_list" course.modules.first.id %}">
        Manage contents
    </a>
{% endif %}
```

Reinicia el servidor de desarrollo para asegurarte de que se cargue el archivo de etiquetas de plantilla del curso:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/course/mine/` y haz clic en el enlace **Manage contents** para un curso que contenga al menos un módulo:

> *Figura 13.10: La página para gestionar los contenidos del módulo del curso*

Añade un par de contenidos de diferentes tipos al módulo:

> *Figura 13.11: Gestión de diferentes contenidos de módulos*

---

### Reordenación de módulos y sus contenidos

Implementaremos una funcionalidad de arrastrar y soltar (*drag-and-drop*) con JavaScript para permitir a los instructores reordenar los módulos de un curso arrastrándolos.

Para implementar esta función, utilizaremos la biblioteca HTML5 Sortable, que simplifica el proceso de creación de listas ordenables utilizando la API nativa de arrastrar y soltar de HTML5 (*HTML5 Drag and Drop API*).

Cuando los usuarios terminen de arrastrar un módulo, utilizarás la API Fetch de JavaScript para enviar una petición HTTP asíncrona al servidor que almacene el nuevo orden de los módulos.

#### Uso de mixins de django-braces

`django-braces` es un módulo de terceros que contiene una colección de mixins genéricos para Django.

Utilizarás los siguientes mixins de `django-braces`:

- `CsrfExemptMixin`: Se utiliza para evitar la verificación del token CSRF en las peticiones POST. Lo necesitas para realizar peticiones POST AJAX sin la necesidad de pasar un `csrf_token`.
- `JsonRequestResponseMixin`: Parsea los datos de la petición como JSON y también serializa la respuesta como JSON, devolviendo una respuesta HTTP con el tipo de contenido `application/json`.

Instala `django-braces` a través de pip:

```bash
python -m pip install django-braces==1.17.0
```

Edita el archivo `views.py` de la aplicación `courses` y añade el siguiente código:

```python
from braces.views import CsrfExemptMixin, JsonRequestResponseMixin


class ModuleOrderView(CsrfExemptMixin, JsonRequestResponseMixin, View):
    def post(self, request):
        for id, order in self.request_json.items():
            Module.objects.filter(
                id=id, course__owner=request.user
            ).update(order=order)
        return self.render_json_response({'saved': 'OK'})


class ContentOrderView(CsrfExemptMixin, JsonRequestResponseMixin, View):
    def post(self, request):
        for id, order in self.request_json.items():
            Content.objects.filter(
                id=id, module__course__owner=request.user
            ).update(order=order)
        return self.render_json_response({'saved': 'OK'})
```

Edita el archivo `urls.py` de la aplicación `courses` y añade los siguientes patrones de URL:

```python
path(
    'module/order/',
    views.ModuleOrderView.as_view(),
    name='module_order'
),
path(
    'content/order/',
    views.ContentOrderView.as_view(),
    name='content_order'
),
```

Edita la plantilla `base.html` ubicada en el directorio `templates/` de la aplicación `courses` y añade el bloque `include_js`:

```html
{% load static %}
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>{% block title %}Educa{% endblock %}</title>
    <link href="{% static "css/base.css" %}" rel="stylesheet">
</head>
<body>
    <div id="header">
        <a href="/" class="logo">Educa</a>
        <ul class="menu">
            {% if request.user.is_authenticated %}
                <li>
                    <form action="{% url "logout" %}" method="post">
                        <button type="submit">Sign out</button>
                    </form>
                </li>
            {% else %}
                <li><a href="{% url "login" %}">Sign in</a></li>
            {% endif %}
        </ul>
    </div>
    <div id="content">
        {% block content %}
        {% endblock %}
    </div>
    {% block include_js %}
    {% endblock %}
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

Edita la plantilla `courses/manage/module/content_list.html` y añade los bloques `include_js` y `domready` al final de la plantilla:

```html
{% block include_js %}
    <script src="https://cdnjs.cloudflare.com/ajax/libs/html5sortable/0.13.3/html5sortable.min.js"></script>
{% endblock %}

{% block domready %}
    var options = {
        method: 'POST',
        mode: 'same-origin'
    }
    const moduleOrderUrl = '{% url "module_order" %}';

    sortable('#modules', {
        forcePlaceholderSize: true,
        placeholderClass: 'placeholder'
    })[0].addEventListener('sortupdate', function(e) {
        modulesOrder = {};
        var modules = document.querySelectorAll('#modules li');
        modules.forEach(function (module, index) {
            // update module index
            modulesOrder[module.dataset.id] = index;
            // update index in HTML element
            module.querySelector('.order').innerHTML = index + 1;
        });
        // add new order to the HTTP request options
        options['body'] = JSON.stringify(modulesOrder);

        // send HTTP request
        fetch(moduleOrderUrl, options)
    });

    const contentOrderUrl = '{% url "content_order" %}';

    sortable('#module-contents', {
        forcePlaceholderSize: true,
        placeholderClass: 'placeholder'
    })[0].addEventListener('sortupdate', function(e) {
        contentOrder = {};
        var contents = document.querySelectorAll('#module-contents div');
        contents.forEach(function (content, index) {
            // update content index
            contentOrder[content.dataset.id] = index;
        });
        // add new order to the HTTP request options
        options['body'] = JSON.stringify(contentOrder);

        // send HTTP request
        fetch(contentOrderUrl, options)
    });
{% endblock %}
```

Abre `http://127.0.0.1:8000/course/mine/` en tu navegador y haz clic en **Manage contents** para cualquier curso. Ahora puedes arrastrar y soltar tanto los módulos de los cursos en la barra lateral izquierda como los contenidos de los módulos:

> *Figura 13.12: Reordenación de módulos con la funcionalidad de arrastrar y soltar*

> *Figura 13.13: Nuevo orden para los módulos tras reordenarlos con arrastrar y soltar*

> *Figura 13.14: Reordenación de contenidos de módulos con la funcionalidad de arrastrar y soltar*

---

### Resumen

En este capítulo, aprendiste a usar vistas basadas en clases y mixins para crear un CMS. Adquiriste conocimientos sobre reutilización y modularidad que puedes aplicar a tus futuras aplicaciones. También trabajaste con grupos y permisos para restringir el acceso a tus vistas, obteniendo información sobre seguridad y cómo controlar acciones sobre los datos. Aprendiste a usar formsets y model formsets para gestionar los módulos de los cursos y sus contenidos de manera flexible. También construiste una funcionalidad de arrastrar y soltar con JavaScript para reordenar los módulos de los cursos y sus contenidos con una interfaz de usuario mejorada.

En el próximo capítulo, crearás un sistema de registro de estudiantes y gestionarás la inscripción de estudiantes en los cursos. También aprenderás a renderizar diferentes tipos de contenido y a mejorar el rendimiento de tu aplicación almacenando en caché el contenido utilizando el framework de caché de Django.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter13](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter13)
- **Documentación sobre mixins de Django:** [https://docs.djangoproject.com/en/5.2/topics/class-based-views/mixins/](https://docs.djangoproject.com/en/5.2/topics/class-based-views/mixins/)
- **Paquete Neapolitan para crear vistas CRUD:** [https://github.com/carltongibson/neapolitan](https://github.com/carltongibson/neapolitan)
- **Creación de permisos personalizados en Django:** [https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#custom-permissions](https://docs.djangoproject.com/en/5.2/topics/auth/customizing/#custom-permissions)
- **Formsets de Django:** [https://docs.djangoproject.com/en/5.2/topics/forms/formsets/](https://docs.djangoproject.com/en/5.2/topics/forms/formsets/)
- **Model formsets de Django:** [https://docs.djangoproject.com/en/5.2/topics/forms/modelforms/#model-formsets](https://docs.djangoproject.com/en/5.2/topics/forms/modelforms/#model-formsets)
- **HTML5 Drag and Drop API:** [https://www.w3schools.com/html/html5_draganddrop.asp](https://www.w3schools.com/html/html5_draganddrop.asp)
- **Documentación de la biblioteca HTML5 Sortable:** [https://github.com/lukasoppermann/html5sortable](https://github.com/lukasoppermann/html5sortable)
- **Ejemplos de la biblioteca HTML5 Sortable:** [https://lukasoppermann.github.io/html5sortable/](https://lukasoppermann.github.io/html5sortable/)
- **Documentación de django-braces:** [https://django-braces.readthedocs.io/](https://django-braces.readthedocs.io/)
