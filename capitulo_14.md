# Parte 4: Creación de una plataforma de E-Learning

## Capítulo 14: Renderizado y almacenamiento en caché de contenido

### Introducción

En el capítulo anterior, utilizaste la herencia de modelos y las relaciones genéricas para crear modelos flexibles para el contenido de los cursos. Implementaste un campo de modelo personalizado y construiste un sistema de gestión de cursos (*CMS*) utilizando vistas basadas en clases. Finalmente, creaste una funcionalidad de arrastrar y soltar con JavaScript utilizando peticiones HTTP asíncronas para ordenar los módulos de los cursos y sus contenidos.

En este capítulo, construirás la funcionalidad para crear un sistema de registro de estudiantes y gestionar la inscripción de estudiantes en los cursos. Implementarás el renderizado de los diferentes tipos de contenido de los cursos y aprenderás a almacenar datos en caché utilizando el framework de caché de Django.

El renderizado de diversos tipos de contenido es esencial en las plataformas de e-learning, donde los cursos se estructuran típicamente con módulos flexibles que incluyen una combinación de texto, imágenes, vídeos y documentos. En este contexto, el almacenamiento en caché también se vuelve crucial. Dado que el contenido de los cursos suele permanecer sin cambios durante períodos prolongados (días, semanas o incluso meses), la caché ayuda a conservar la potencia de cálculo y reduce la necesidad de consultar la base de datos cada vez que los estudiantes acceden a los mismos materiales. Al almacenar datos en caché, no solo ahorras recursos del sistema, sino que también mejoras el rendimiento al entregar contenido a un gran número de estudiantes.

En este capítulo, aprenderás a:

- Crear vistas públicas para mostrar información de los cursos
- Construir un sistema de registro de estudiantes
- Gestionar la inscripción de estudiantes en los cursos
- Renderizar diversos contenidos para los módulos de los cursos
- Instalar y configurar Memcached
- Almacenar contenido en caché utilizando el framework de caché de Django
- Utilizar los backends de caché de Memcached y Redis
- Monitorizar tu servidor Redis en el sitio de administración de Django

---

### Visión general funcional

La Figura 14.1 muestra una representación de las vistas, plantillas y funcionalidades que se construirán en este capítulo:

> *Figura 14.1: Diagrama de las funcionalidades construidas en el Capítulo 14*

En este capítulo, implementarás la vista pública `CourseListView` para listar cursos y `CourseDetailView` para mostrar los detalles de un curso. Implementarás `StudentRegistrationView` para permitir que los estudiantes creen cuentas de usuario y `StudentEnrollCourseView` para que los estudiantes se inscriban en los cursos. Crearás `StudentCourseListView` para que los estudiantes vean la lista de cursos en los que están inscritos y `StudentCourseDetailView` para acceder a todo el contenido de un curso, organizado en los diferentes módulos del curso. También añadirás una caché a tus vistas utilizando el framework de caché de Django, primero con el backend de Memcached y luego reemplazándolo con el backend de caché de Redis.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter14](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter14).

Todos los módulos de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente que acompaña a este capítulo. Puedes seguir las instrucciones para instalar cada módulo de Python a continuación o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Visualización del catálogo de cursos

Comencemos con el catálogo de cursos. Para tu catálogo de cursos, debes construir las siguientes funcionalidades:

- Listar todos los cursos disponibles, opcionalmente filtrados por materia (*subject*)
- Mostrar una vista general (*overview*) de un curso individual

Esto permitirá a los estudiantes ver todos los cursos disponibles en la plataforma e inscribirse en aquellos que les interesen. Edita el archivo `views.py` de la aplicación `courses` y añade el siguiente código:

```python
from django.db.models import Count
from .models import Subject


class CourseListView(TemplateResponseMixin, View):
    model = Course
    template_name = 'courses/course/list.html'

    def get(self, request, subject=None):
        subjects = Subject.objects.annotate(
            total_courses=Count('courses')
        )
        courses = Course.objects.annotate(
            total_modules=Count('modules')
        )
        if subject:
            subject = get_object_or_404(Subject, slug=subject)
            courses = courses.filter(subject=subject)
        return self.render_to_response(
            {
                'subjects': subjects,
                'subject': subject,
                'courses': courses
            }
        )
```

Esta es la vista `CourseListView`. Hereda de `TemplateResponseMixin` y `View`. En esta vista, se realizan las siguientes tareas:

1. Recuperar todas las materias utilizando el método `annotate()` del ORM con la función de agregación `Count()` para incluir el número total de cursos de cada materia.
2. Recuperar todos los cursos disponibles, incluyendo el número total de módulos contenidos en cada curso.
3. Si se proporciona un parámetro de URL `subject` (slug de la materia), recuperar el objeto de materia correspondiente y limitar la consulta a los cursos que pertenecen a la materia dada.
4. Utilizar el método `render_to_response()` proporcionado por `TemplateResponseMixin` para renderizar los objetos en una plantilla y devolver una respuesta HTTP.

Creemos una vista de detalle para mostrar la vista general de un curso individual. Añade el siguiente código al archivo `views.py`:

```python
from django.views.generic.detail import DetailView


class CourseDetailView(DetailView):
    model = Course
    template_name = 'courses/course/detail.html'
```

Esta vista hereda de la vista genérica `DetailView` proporcionada por Django. Especificas los atributos `model` y `template_name`. La vista `DetailView` de Django espera un parámetro de URL de clave primaria (`pk`) o `slug` para recuperar un solo objeto del modelo dado. La vista renderiza la plantilla especificada en `template_name`, incluyendo el objeto `Course` en la variable de contexto de plantilla `object`.

Edita el archivo `urls.py` principal del proyecto `educa` y añade el siguiente patrón de URL:

```python
from courses.views import CourseListView

urlpatterns = [
    # ...
    path('', CourseListView.as_view(), name='course_list'),
]
```

Añades el patrón de URL `course_list` al archivo `urls.py` principal del proyecto porque deseas mostrar la lista de cursos en la URL `http://127.0.0.1:8000/`, y todas las demás URLs de la aplicación `courses` tienen el prefijo `/course/`.

Edita el archivo `urls.py` de la aplicación `courses` y añade los siguientes patrones de URL:

```python
path(
    'subject/<slug:subject>/',
    views.CourseListView.as_view(),
    name='course_list_subject'
),
path(
    '<slug:slug>/',
    views.CourseDetailView.as_view(),
    name='course_detail'
),
```

Defines los siguientes patrones de URL:

- `course_list_subject`: Para mostrar todos los cursos de una materia.
- `course_detail`: Para mostrar la vista general de un solo curso.

Construyamos las plantillas para las vistas `CourseListView` y `CourseDetailView`.

Crea la siguiente estructura de archivos dentro del directorio `templates/courses/` de la aplicación `courses`:

```text
course/
    list.html
    detail.html
```

Edita la plantilla `courses/course/list.html` de la aplicación `courses` y escribe el siguiente código:

```html
{% extends "base.html" %}

{% block title %}
    {% if subject %}
        {{ subject.title }} courses
    {% else %}
        All courses
    {% endif %}
{% endblock %}

{% block content %}
    <h1>
        {% if subject %}
            {{ subject.title }} courses
        {% else %}
            All courses
        {% endif %}
    </h1>
    <div class="contents">
        <h3>Subjects</h3>
        <ul id="modules">
            <li {% if not subject %}class="selected"{% endif %}>
                <a href="{% url "course_list" %}">All</a>
            </li>
            {% for s in subjects %}
                <li {% if subject == s %}class="selected"{% endif %}>
                    <a href="{% url "course_list_subject" s.slug %}">
                        {{ s.title }}
                        <br>
                        <span>
                            {{ s.total_courses }} course{{ s.total_courses|pluralize }}
                        </span>
                    </a>
                </li>
            {% endfor %}
        </ul>
    </div>
    <div class="module">
        {% for course in courses %}
            {% with subject=course.subject %}
                <h3>
                    <a href="{% url "course_detail" course.slug %}">
                        {{ course.title }}
                    </a>
                </h3>
                <p>
                    <a href="{% url "course_list_subject" subject.slug %}">{{ subject }}</a>.
                    {{ course.total_modules }} modules.
                    Instructor: {{ course.owner.get_full_name }}
                </p>
            {% endwith %}
        {% endfor %}
    </div>
{% endblock %}
```

Esta es la plantilla para listar los cursos disponibles. Creas una lista HTML para mostrar todos los objetos `Subject` y construyes un enlace a la URL `course_list_subject` para cada uno de ellos. También incluyes el número total de cursos de cada asignatura y utilizas el filtro de plantilla `pluralize` para añadir un sufijo plural a la palabra *course* cuando el número sea diferente de 1. Añades la clase HTML `selected` para resaltar la asignatura actual si hay una seleccionada. Iteras sobre cada objeto `Course`, mostrando el número total de módulos y el nombre del instructor.

Ejecuta el servidor de desarrollo y abre `http://127.0.0.1:8000/` en tu navegador:

> *Figura 14.2: La página de lista de cursos*

La barra lateral izquierda contiene todas las materias, incluyendo el número total de cursos de cada una. Puedes hacer clic en cualquier materia para filtrar los cursos mostrados.

Edita la plantilla `courses/course/detail.html` y añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}
    {{ object.title }}
{% endblock %}

{% block content %}
    {% with subject=object.subject %}
        <h1>
            {{ object.title }}
        </h1>
        <div class="module">
            <h2>Overview</h2>
            <p>
                <a href="{% url "course_list_subject" subject.slug %}">
                    {{ subject.title }}</a>.
                {{ object.modules.count }} modules.
                Instructor: {{ object.owner.get_full_name }}
            </p>
            {{ object.overview|linebreaks }}
        </div>
    {% endwith %}
{% endblock %}
```

Abre `http://127.0.0.1:8000/` en tu navegador y haz clic en uno de los cursos:

> *Figura 14.3: La página de descripción general del curso*

---

### Adición de registro de estudiantes

Necesitamos implementar el registro de estudiantes para permitir la inscripción en cursos y el acceso al contenido. Crea una nueva aplicación con el siguiente comando:

```bash
python manage.py startapp students
```

Edita el archivo `settings.py` del proyecto `educa` y añade la nueva aplicación a la configuración `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    # ...
    'students.apps.StudentsConfig',
]
```

#### Creación de una vista de registro de estudiantes

Edita el archivo `views.py` de la aplicación `students` y escribe el siguiente código:

```python
from django.contrib.auth import authenticate, login
from django.contrib.auth.forms import UserCreationForm
from django.urls import reverse_lazy
from django.views.generic.edit import CreateView


class StudentRegistrationView(CreateView):
    template_name = 'students/student/registration.html'
    form_class = UserCreationForm
    success_url = reverse_lazy('student_course_list')

    def form_valid(self, form):
        result = super().form_valid(form)
        cd = form.cleaned_data
        user = authenticate(
            username=cd['username'],
            password=cd['password1']
        )
        login(self.request, user)
        return result
```

Esta es la vista que permite a los estudiantes registrarse en tu sitio. Utilizas la vista genérica `CreateView`, que proporciona la funcionalidad para crear objetos de modelo. Esta vista requiere los siguientes atributos:

- `template_name`: La ruta de la plantilla para renderizar esta vista.
- `form_class`: El formulario para crear objetos, que debe ser un `ModelForm`. Utilizas `UserCreationForm` de Django como el formulario de registro para crear objetos `User`.
- `success_url`: La URL a la que redirigir al usuario cuando el formulario se envíe con éxito. Para esto, inviertes la URL llamada `student_course_list`, que crearemos en la sección *Acceso a los contenidos del curso* para listar los cursos en los que están inscritos los estudiantes.

El método `form_valid()` se ejecuta cuando se han enviado datos de formulario válidos. Sobrescribes este método para iniciar sesión automáticamente al usuario después de que se haya registrado correctamente.

Crea un nuevo archivo dentro del directorio de la aplicación `students` y nómbralo `urls.py`. Añade el siguiente código:

```python
from django.urls import path
from . import views

urlpatterns = [
    path(
        'register/',
        views.StudentRegistrationView.as_view(),
        name='student_registration'
    ),
]
```

Luego, edita el archivo `urls.py` principal del proyecto `educa` e incluye las URLs de la aplicación `students`:

```python
urlpatterns = [
    # ...
    path('students/', include('students.urls')),
]
```

Crea la siguiente estructura de archivos dentro del directorio de la aplicación `students`:

```text
templates/
    students/
        student/
            registration.html
```

Edita la plantilla `students/student/registration.html` y añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}
    Sign up
{% endblock %}

{% block content %}
    <h1>
        Sign up
    </h1>
    <div class="module">
        <p>Enter your details to create an account:</p>
        <form method="post">
            {{ form.as_p }}
            {% csrf_token %}
            <p><input type="submit" value="Create my account"></p>
        </form>
    </div>
{% endblock %}
```

Ejecuta el servidor de desarrollo y abre `http://127.0.0.1:8000/students/register/` en tu navegador:

> *Figura 14.4: El formulario de registro de estudiantes*

#### Inscripción en cursos

Para almacenar las inscripciones, necesitas crear una relación muchos a muchos (*many-to-many*) entre los modelos `Course` y `User`.

Edita el archivo `models.py` de la aplicación `courses` y añade el siguiente campo al modelo `Course`:

```python
    students = models.ManyToManyField(
        User,
        related_name='courses_joined',
        blank=True
    )
```

Crea y aplica la migración:

```bash
python manage.py makemigrations
```

```bash
python manage.py migrate
```

Crea un nuevo archivo `forms.py` dentro del directorio de la aplicación `students` y añade el siguiente código:

```python
from django import forms
from courses.models import Course


class CourseEnrollForm(forms.Form):
    course = forms.ModelChoiceField(
        queryset=Course.objects.none(),
        widget=forms.HiddenInput
    )

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.fields['course'].queryset = Course.objects.all()
```

Este formulario se utilizará para inscribir a los estudiantes en los cursos. El campo `course` es para el curso en el que se inscribirá el usuario; por lo tanto, es un `ModelChoiceField`. Utilizas un widget `HiddenInput` porque este campo no está destinado a ser visible para el usuario. Inicialmente, defines el QuerySet como `Course.objects.none()`. El uso de `none()` crea un QuerySet vacío que no devuelve ningún objeto y, lo que es más importante, no consulta la base de datos, evitando una carga innecesaria durante la inicialización del formulario. Poblas el QuerySet real en el método `__init__()` del formulario.

Edita el archivo `views.py` de la aplicación `students` y añade el siguiente código:

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic.edit import FormView
from .forms import CourseEnrollForm


class StudentEnrollCourseView(LoginRequiredMixin, FormView):
    course = None
    form_class = CourseEnrollForm

    def form_valid(self, form):
        self.course = form.cleaned_data['course']
        self.course.students.add(self.request.user)
        return super().form_valid(form)

    def get_success_url(self):
        return reverse_lazy(
            'student_course_detail',
            args=[self.course.id]
        )
```

Esta es la vista `StudentEnrollCourseView`. Gestiona la inscripción de estudiantes en los cursos. La vista hereda del mixin `LoginRequiredMixin` para que solo los usuarios autenticados puedan acceder a la vista. También hereda de `FormView` de Django, ya que maneja el envío de un formulario. Utilizas el formulario `CourseEnrollForm` para el atributo `form_class` y también defines un atributo `course` para almacenar el objeto `Course` dado. Cuando el formulario es válido, el usuario actual se añade a los estudiantes inscritos en el curso.

Edita el archivo `urls.py` de la aplicación `students` y añade el siguiente patrón de URL:

```python
path(
    'enroll-course/',
    views.StudentEnrollCourseView.as_view(),
    name='student_enroll_course'
),
```

Edita el archivo `views.py` de la aplicación `courses` y modifica `CourseDetailView`:

```python
from students.forms import CourseEnrollForm


class CourseDetailView(DetailView):
    model = Course
    template_name = 'courses/course/detail.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['enroll_form'] = CourseEnrollForm(
            initial={'course': self.object}
        )
        return context
```

Edita la plantilla `courses/course/detail.html` y sustituye `{{ object.overview|linebreaks }}` por:

```html
{{ object.overview|linebreaks }}
{% if request.user.is_authenticated %}
    <form action="{% url "student_enroll_course" %}" method="post">
        {{ enroll_form }}
        {% csrf_token %}
        <input type="submit" value="Enroll now">
    </form>
{% else %}
    <a href="{% url "student_registration" %}" class="button">
        Register to enroll
    </a>
{% endif %}
```

Abre `http://127.0.0.1:8000/` en tu navegador y haz clic en un curso. Si has iniciado sesión, verás un botón **ENROLL NOW**:

> *Figura 14.5: La página de descripción general del curso, incluyendo el botón ENROLL NOW*

---

### Renderizado de los contenidos del curso

Una vez que los estudiantes están inscritos en los cursos, necesitan una ubicación central para acceder a todos los cursos en los que están registrados y a sus contenidos.

#### Acceso a los contenidos del curso

Edita el archivo `views.py` de la aplicación `students` y añade el siguiente código:

```python
from django.views.generic.list import ListView
from courses.models import Course


class StudentCourseListView(LoginRequiredMixin, ListView):
    model = Course
    template_name = 'students/course/list.html'

    def get_queryset(self):
        qs = super().get_queryset()
        return qs.filter(students__in=[self.request.user])
```

Esta es la vista para ver los cursos en los que están inscritos los estudiantes.

Luego, añade la vista `StudentCourseDetailView` al archivo `views.py` de la aplicación `students`:

```python
from django.views.generic.detail import DetailView


class StudentCourseDetailView(LoginRequiredMixin, DetailView):
    model = Course
    template_name = 'students/course/detail.html'

    def get_queryset(self):
        qs = super().get_queryset()
        return qs.filter(students__in=[self.request.user])

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        # get course object
        course = self.get_object()
        if 'module_id' in self.kwargs:
            # get current module
            context['module'] = course.modules.get(
                id=self.kwargs['module_id']
            )
        else:
            # get first module
            context['module'] = course.modules.all()[0]
        return context
```

Edita el archivo `urls.py` de la aplicación `students` y añade los siguientes patrones de URL:

```python
path(
    'courses/',
    views.StudentCourseListView.as_view(),
    name='student_course_list'
),
path(
    'course/<pk>/',
    views.StudentCourseDetailView.as_view(),
    name='student_course_detail'
),
path(
    'course/<pk>/<module_id>/',
    views.StudentCourseDetailView.as_view(),
    name='student_course_detail_module'
),
```

Crea la siguiente estructura de archivos dentro del directorio `templates/students/` de la aplicación `students`:

```text
course/
    detail.html
    list.html
```

Edita la plantilla `students/course/list.html`:

```html
{% extends "base.html" %}

{% block title %}My courses{% endblock %}

{% block content %}
    <h1>My courses</h1>
    <div class="module">
        {% for course in object_list %}
            <div class="course-info">
                <h3>{{ course.title }}</h3>
                <p><a href="{% url "student_course_detail" course.id %}">
                    Access contents</a></p>
            </div>
        {% empty %}
            <p>
                You are not enrolled in any courses yet.
                <a href="{% url "course_list" %}">Browse courses</a> to enroll in a course.
            </p>
        {% endfor %}
    </div>
{% endblock %}
```

Edita el archivo `settings.py` del proyecto `educa` y añade `LOGIN_REDIRECT_URL`:

```python
from django.urls import reverse_lazy

LOGIN_REDIRECT_URL = reverse_lazy('student_course_list')
```

Edita la plantilla `students/course/detail.html` y añade el siguiente código:

```html
{% extends "base.html" %}

{% block title %}
    {{ object.title }}
{% endblock %}

{% block content %}
    <h1>
        {{ module.title }}
    </h1>
    <div class="contents">
        <h3>Modules</h3>
        <ul id="modules">
            {% for m in object.modules.all %}
                <li data-id="{{ m.id }}" {% if m == module %}class="selected"{% endif %}>
                    <a href="{% url "student_course_detail_module" object.id m.id %}">
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
    </div>
    <div class="module">
        {% for content in module.contents.all %}
            {% with item=content.item %}
                <h2>{{ item.title }}</h2>
                {{ item.render }}
            {% endwith %}
        {% endfor %}
    </div>
{% endblock %}
```

#### Renderizado de diferentes tipos de contenido

Para mostrar los contenidos del curso, necesitas renderizar los diferentes tipos de contenido que creaste: texto, imagen, vídeo y archivo.

Edita el archivo `models.py` de la aplicación `courses` y añade el siguiente método `render()` al modelo `ItemBase`:

```python
from django.template.loader import render_to_string


class ItemBase(models.Model):
    # ...
    def render(self):
        return render_to_string(
            f'courses/content/{self._meta.model_name}.html',
            {'item': self}
        )
```

Crea la siguiente estructura de archivos dentro del directorio `templates/courses/` de la aplicación `courses`:

```text
content/
    text.html
    file.html
    image.html
    video.html
```

Edita `courses/content/text.html`:

```html
{{ item.content|linebreaks }}
```

Edita `courses/content/file.html`:

```html
<p>
    <a href="{{ item.file.url }}" class="button">Download file</a>
</p>
```

Edita `courses/content/image.html`:

```html
<p>
    <img src="{{ item.file.url }}" alt="{{ item.title }}">
</p>
```

Para incrustar vídeos de YouTube o Vimeo, utilizaremos `django-embed-video`. Instala el paquete con:

```bash
python -m pip install django-embed-video==1.4.10
```

Añade `'embed_video'` a `INSTALLED_APPS` en `settings.py`:

```python
INSTALLED_APPS = [
    # ...
    'embed_video',
]
```

Edita `courses/content/video.html`:

```html
{% load embed_video_tags %}
{% video item.url "small" %}
```

Abre `http://127.0.0.1:8000/`, inscríbete en un curso y accede a sus contenidos:

> *Figura 14.6: Una página de contenidos de curso*

---

### Uso del framework de caché

El framework de caché de Django permite almacenar datos calculados, consultas o páginas renderizadas para evitar operaciones costosas en peticiones sucesivas.

#### Backends de caché disponibles

Django incluye los siguientes backends de caché:

- `backends.memcached.PyMemcacheCache` o `backends.memcached.PyLibMCCache`: Backends de Memcached. Memcached es un servidor de caché en memoria rápido y eficiente.
- `backends.redis.RedisCache`: Backend de caché de Redis (añadido en Django 4.0).
- `backends.db.DatabaseCache`: Utiliza la base de datos relacional como sistema de caché.
- `backends.filebased.FileBasedCache`: Utiliza el sistema de almacenamiento de archivos.
- `backends.locmem.LocMemCache`: Backend de caché en memoria local (predeterminado).
- `backends.dummy.DummyCache`: Backend de caché ficticio para desarrollo (no almacena nada).

---

### Instalación de Memcached

Memcached es un servidor de caché en memoria de alto rendimiento muy popular.

#### Instalación de la imagen Docker de Memcached

Descarga la imagen Docker de Memcached con el siguiente comando:

```bash
docker pull memcached:1.6.26
```

Ejecuta el contenedor Docker de Memcached:

```bash
docker run -it --rm --name memcached -p 11211:11211 memcached:1.6.26 -m 64
```

#### Instalación del enlace de Python para Memcached

Instala `pymemcache` vía pip:

```bash
python -m pip install pymemcache==4.0.0
```

---

### Configuración de caché de Django

Django proporciona las siguientes configuraciones de caché principales en `settings.py`:

- `CACHES`: Diccionario que contiene todas las cachés disponibles para el proyecto.
- `CACHE_MIDDLEWARE_ALIAS`: El alias de caché a utilizar para el almacenamiento.
- `CACHE_MIDDLEWARE_KEY_PREFIX`: Prefijo para evitar colisiones entre sitios.
- `CACHE_MIDDLEWARE_SECONDS`: Número predeterminado de segundos para almacenar páginas en caché.

#### Adición de Memcached a tu proyecto

Edita el archivo `settings.py` del proyecto `educa` y añade:

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': '127.0.0.1:11211',
    }
}
```

---

### Niveles de caché

Django proporciona los siguientes niveles de caché, ordenados de mayor a menor granularidad:

1. **API de caché de bajo nivel (*Low-level cache API*)**: Proporciona la mayor granularidad para almacenar consultas o cálculos específicos.
2. **Caché de fragmentos de plantilla (*Template cache*)**: Permite almacenar en caché fragmentos de plantilla.
3. **Caché por vista (*Per-view cache*)**: Proporciona almacenamiento en caché para vistas individuales.
4. **Caché por sitio (*Per-site cache*)**: Almacena en caché el sitio completo.

#### Uso de la API de caché de bajo nivel

Abre la shell de Django (`python manage.py shell`):

```python
>>> from django.core.cache import cache
>>> cache.set('musician', 'Django Reinhardt', 20)
>>> cache.get('musician')
'Django Reinhardt'
```

Almacenando QuerySets:

```python
>>> from courses.models import Subject
>>> subjects = Subject.objects.all()
>>> cache.set('my_subjects', subjects)
>>> cache.get('my_subjects')
<QuerySet [<Subject: Mathematics>, <Subject: Music>, <Subject: Physics>, <Subject: Programming>]>
```

Edita `courses/views.py` y añade el uso de la caché a `CourseListView`:

```python
from django.core.cache import cache


class CourseListView(TemplateResponseMixin, View):
    # ...
    def get(self, request, subject=None):
        subjects = cache.get('all_subjects')
        if not subjects:
            subjects = Subject.objects.annotate(
                total_courses=Count('courses')
            )
            cache.set('all_subjects', subjects)
        # ...
```

#### Comprobación de solicitudes de caché con Django Debug Toolbar

Instala Django Debug Toolbar:

```bash
python -m pip install django-debug-toolbar==5.0.1
```

Añade `'debug_toolbar'` a `INSTALLED_APPS` y su middleware a `MIDDLEWARE` en `settings.py`:

```python
INSTALLED_APPS = [
    # ...
    'debug_toolbar',
]

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

INTERNAL_IPS = [
    '127.0.0.1',
]
```

Edita `educa/urls.py` y añade la ruta de Debug Toolbar:

```python
path('__debug__/', include('debug_toolbar.urls')),
```

Abre `http://127.0.0.1:8000/` y revisa el panel de Cache en la barra lateral derecha:

> *Figura 14.7: El panel de Cache de Django Debug Toolbar mostrando solicitudes de caché en un fallo de caché (cache miss)*

> *Figura 14.8: Consultas SQL ejecutadas para CourseListView en un fallo de caché*

Al recargar la página:

> *Figura 14.9: El panel de Cache de Django Debug Toolbar mostrando un acierto de caché (cache hit)*

> *Figura 14.10: Consultas SQL ejecutadas para CourseListView en un acierto de caché*

#### Almacenamiento en caché de bajo nivel basado en datos dinámicos

Edita `CourseListView` en `courses/views.py`:

```python
class CourseListView(TemplateResponseMixin, View):
    model = Course
    template_name = 'courses/course/list.html'

    def get(self, request, subject=None):
        subjects = cache.get('all_subjects')
        if not subjects:
            subjects = Subject.objects.annotate(
                total_courses=Count('courses')
            )
            cache.set('all_subjects', subjects)
        all_courses = Course.objects.annotate(
            total_modules=Count('modules')
        )
        if subject:
            subject = get_object_or_404(Subject, slug=subject)
            key = f'subject_{subject.id}_courses'
            courses = cache.get(key)
            if not courses:
                courses = all_courses.filter(subject=subject)
                cache.set(key, courses)
        else:
            courses = cache.get('all_courses')
            if not courses:
                courses = all_courses
                cache.set('all_courses', courses)
        return self.render_to_response(
            {
                'subjects': subjects,
                'subject': subject,
                'courses': courses
            }
        )
```

#### Almacenamiento en caché de fragmentos de plantilla

Edita `templates/students/course/detail.html`:

```html
{% extends "base.html" %}
{% load cache %}
...
{% cache 600 module_contents module %}
    {% for content in module.contents.all %}
        {% with item=content.item %}
            <h2>{{ item.title }}</h2>
            {{ item.render }}
        {% endwith %}
    {% endfor %}
{% endcache %}
```

#### Almacenamiento en caché de vistas

Edita `students/urls.py` y aplica el decorador `cache_page`:

```python
from django.views.decorators.cache import cache_page

urlpatterns = [
    # ...
    path(
        'course/<pk>/',
        cache_page(60 * 15)(views.StudentCourseDetailView.as_view()),
        name='student_course_detail'
    ),
    path(
        'course/<pk>/<module_id>/',
        cache_page(60 * 15)(views.StudentCourseDetailView.as_view()),
        name='student_course_detail_module'
    ),
]
```

#### Uso de la caché por sitio

Para la caché por sitio, se añaden `UpdateCacheMiddleware` y `FetchFromCacheMiddleware` a `MIDDLEWARE` en `settings.py`:

```python
MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    # 'django.middleware.cache.UpdateCacheMiddleware',
    'django.middleware.common.CommonMiddleware',
    # 'django.middleware.cache.FetchFromCacheMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

CACHE_MIDDLEWARE_ALIAS = 'default'
CACHE_MIDDLEWARE_SECONDS = 60 * 15  # 15 minutes
CACHE_MIDDLEWARE_KEY_PREFIX = 'educa'
```

---

### Uso del backend de caché de Redis

Instala `redis-py`:

```bash
python -m pip install redis==5.2.1
```

Modifica la configuración `CACHES` en `settings.py`:

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379',
    }
}
```

Inicia el contenedor Docker de Redis:

```bash
docker run -it --rm --name redis -p 6379:6379 redis:7.2.4
```

#### Monitorización de Redis con Django Redisboard

Instala `django-redisboard`:

```bash
python -m pip install django-redisboard==8.4.0
```

Añade `'redisboard'` a `INSTALLED_APPS` en `settings.py` y aplica las migraciones:

```bash
python manage.py migrate redisboard
```

Abre `http://127.0.0.1:8000/admin/redisboard/redisserver/add/` y añade un servidor con la etiqueta `redis` y URL `redis://localhost:6379/0`:

> *Figura 14.11: El formulario para añadir un servidor Redis para Django Redisboard en el sitio de administración*

> *Figura 14.12: La monitorización de Redis de Django Redisboard en el sitio de administración*

---

### Resumen

En este capítulo, implementaste las vistas públicas para el catálogo de cursos. Construiste un sistema para que los estudiantes se registren e inscriban en los cursos. También creaste la funcionalidad para renderizar diferentes tipos de contenido para los módulos de los cursos. Finalmente, aprendiste a usar el framework de caché de Django y utilizaste los backends de caché de Memcached y Redis para tu proyecto.

En el próximo capítulo, construirás una API RESTful para tu proyecto utilizando Django REST framework y la consumirás utilizando la biblioteca Python Requests.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter14](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter14)
- **Documentación de django-embed-video:** [https://django-embed-video.readthedocs.io/en/latest/](https://django-embed-video.readthedocs.io/en/latest/)
- **Documentación del framework de caché de Django:** [https://docs.djangoproject.com/en/5.2/topics/cache/](https://docs.djangoproject.com/en/5.2/topics/cache/)
- **Imagen Docker de Memcached:** [https://hub.docker.com/_/memcached](https://hub.docker.com/_/memcached)
- **Descargas de Memcached:** [https://memcached.org/downloads](https://memcached.org/downloads)
- **Sitio web oficial de Memcached:** [https://memcached.org](https://memcached.org/)
- **Documentación de la configuración CACHES de Django:** [https://docs.djangoproject.com/en/5.2/ref/settings/#caches](https://docs.djangoproject.com/en/5.2/ref/settings/#caches)
- **Código fuente de pymemcache:** [https://github.com/pinterest/pymemcache](https://github.com/pinterest/pymemcache)
- **Backend de caché de Redis en Django:** [https://docs.djangoproject.com/en/5.2/topics/cache/#redis](https://docs.djangoproject.com/en/5.2/topics/cache/#redis)
- **Imagen Docker oficial de Redis:** [https://hub.docker.com/_/redis](https://hub.docker.com/_/redis)
- **Opciones de descarga de Redis:** [https://redis.io/download/](https://redis.io/download/)
- **Código fuente de Django Redisboard:** [https://github.com/ionelmc/django-redisboard](https://github.com/ionelmc/django-redisboard)

