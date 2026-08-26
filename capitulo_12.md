# Parte 4: Creación de una plataforma de E-Learning

## Capítulo 12: Creación de una plataforma de E-Learning

### Introducción

En el capítulo anterior, aprendiste los aspectos fundamentales de la internacionalización y localización de proyectos Django, adaptando tu proyecto para cumplir con los formatos y lenguajes locales de tus usuarios.

En este capítulo, comenzarás un nuevo proyecto de Django que consistirá en una plataforma de e-learning con tu propio sistema de gestión de contenidos (*CMS*, por sus siglas en inglés). Las plataformas de aprendizaje online son un excelente ejemplo de aplicaciones que requieren herramientas para el manejo avanzado de contenidos. Aprenderás a crear modelos de datos flexibles que admitan diversos tipos de datos y descubrirás cómo implementar funcionalidades de modelos personalizadas que podrás aplicar a tus futuros proyectos de Django.

En este capítulo, aprenderás a:

- Crear modelos para el CMS
- Crear fixtures para tus modelos y aplicarlas
- Utilizar la herencia de modelos para crear modelos de datos para contenido polimórfico
- Crear campos de modelo personalizados
- Ordenar contenidos de cursos y módulos
- Construir vistas de autenticación para el CMS

---

### Visión general funcional

En capítulos anteriores, los diagramas al inicio representaban vistas, plantillas y funcionalidades de extremo a extremo. Este capítulo, sin embargo, cambia el enfoque hacia la implementación de la herencia de modelos y la creación de campos de modelo personalizados, temas que no se capturan fácilmente en nuestros diagramas habituales. En su lugar, verás diagramas específicos para ilustrar estos conceptos a lo largo del capítulo.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter12](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter12).

Todos los módulos de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente que acompaña a este capítulo. Puedes seguir las instrucciones para instalar cada módulo de Python a continuación, o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Configuración del proyecto de e-learning

Tu proyecto práctico final será una plataforma de e-learning. Primero, crea un entorno virtual para tu nuevo proyecto dentro del directorio `env/` con el siguiente comando:

```bash
python -m venv env/educa
```

Si estás utilizando Linux o macOS, ejecuta el siguiente comando para activar tu entorno virtual:

```bash
source env/educa/bin/activate
```

Si estás utilizando Windows, utiliza el siguiente comando en su lugar:

```cmd
.\env\educa\Scripts\activate
```

Instala Django en tu entorno virtual con el siguiente comando:

```bash
python -m pip install Django~=5.2.0
```

Vas a gestionar la subida de imágenes en tu proyecto, por lo que también necesitas instalar Pillow con el siguiente comando:

```bash
python -m pip install Pillow==11.2.1
```

Crea un nuevo proyecto usando el siguiente comando:

```bash
django-admin startproject educa
```

Entra en el nuevo directorio `educa` y crea una nueva aplicación usando los siguientes comandos:

```bash
cd educa
django-admin startapp courses
```

Edita el archivo `settings.py` del proyecto `educa` y añade `courses` a la configuración `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    'courses.apps.CoursesConfig',
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
```

La aplicación `courses` ahora está activa para el proyecto. A continuación, vamos a preparar nuestro proyecto para servir archivos multimedia (*media files*) y definiremos los modelos para los cursos y los contenidos de los cursos.

---

### Servir archivos multimedia (Media files)

Antes de crear los modelos para cursos y contenidos de cursos, prepararemos el proyecto para servir archivos multimedia. Los instructores de los cursos podrán subir archivos multimedia al contenido del curso utilizando el CMS que construiremos. Por lo tanto, configuraremos el proyecto para servir archivos multimedia.

Edita el archivo `settings.py` del proyecto y añade las siguientes líneas:

```python
MEDIA_URL = 'media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

Esto permitirá a Django gestionar las subidas de archivos y servir archivos multimedia. `MEDIA_URL` es la URL base utilizada para servir los archivos multimedia subidos por los usuarios. `MEDIA_ROOT` es la ruta local donde residen. Las rutas y URLs de los archivos se construyen dinámicamente anteponiendo la ruta del proyecto o la URL multimedia para mayor portabilidad.

Ahora, edita el archivo `urls.py` principal del proyecto `educa` y modifica el código de la siguiente manera:

```python
from django.conf import settings
from django.conf.urls.static import static
from django.contrib import admin
from django.urls import path

urlpatterns = [
    path('admin/', admin.site.urls),
]

if settings.DEBUG:
    urlpatterns += static(
        settings.MEDIA_URL,
        document_root=settings.MEDIA_ROOT
    )
```

Hemos añadido la función auxiliar `static()` para servir archivos multimedia con el servidor de desarrollo de Django durante el desarrollo (es decir, cuando la configuración `DEBUG` se establece en `True`).

> [!IMPORTANT]
> Recuerda que la función auxiliar `static()` es adecuada para el desarrollo, pero no para su uso en producción. Django es ineficiente sirviendo archivos estáticos. Nunca sirvas tus archivos estáticos con el servidor de desarrollo de Django en un entorno de producción. Aprenderás a servir archivos estáticos en un entorno de producción en el Capítulo 17, *Puesta en producción (Going Live)*.

El proyecto ahora está listo para servir archivos multimedia. Creemos los modelos para los cursos y los contenidos de los cursos.

---

### Creación de los modelos del curso

Tu plataforma de e-learning ofrecerá cursos sobre diversos temas. Cada curso se dividirá en un número configurable de módulos, y cada módulo contendrá un número configurable de contenidos. Los contenidos serán de varios tipos: texto, archivos, imágenes o vídeos. El siguiente ejemplo muestra cómo será la estructura de datos de tu catálogo de cursos:

```text
Subject 1
    Course 1
        Module 1
            Content 1 (image)
            Content 2 (text)
        Module 2
            Content 3 (text)
            Content 4 (file)
            Content 5 (video)
...
```

Construyamos los modelos del curso. Edita el archivo `models.py` de la aplicación `courses` y añade el siguiente código:

```python
from django.contrib.auth.models import User
from django.db import models


class Subject(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)

    class Meta:
        ordering = ['title']

    def __str__(self):
        return self.title


class Course(models.Model):
    owner = models.ForeignKey(
        User,
        related_name='courses_created',
        on_delete=models.CASCADE
    )
    subject = models.ForeignKey(
        Subject,
        related_name='courses',
        on_delete=models.CASCADE
    )
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    overview = models.TextField()
    created = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created']

    def __str__(self):
        return self.title


class Module(models.Model):
    course = models.ForeignKey(
        Course,
        related_name='modules',
        on_delete=models.CASCADE
    )
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)

    def __str__(self):
        return self.title
```

Estos son los modelos iniciales `Subject`, `Course` y `Module`. Los campos del modelo `Course` son los siguientes:

- `owner`: El instructor que creó este curso.
- `subject`: La materia o temática a la que pertenece este curso. Es un campo `ForeignKey` que apunta al modelo `Subject`.
- `title`: El título del curso.
- `slug`: El slug del curso. Esto se usará en las URLs más adelante.
- `overview`: Una columna `TextField` para almacenar una descripción general del curso.
- `created`: La fecha y hora en que se creó el curso. Django lo establecerá automáticamente al crear nuevos objetos debido a `auto_now_add=True`.

Cada curso se divide en varios módulos. Por lo tanto, el modelo `Module` contiene un campo `ForeignKey` que apunta al modelo `Course`.

Abre la consola y ejecuta el siguiente comando para crear la migración inicial para esta aplicación:

```bash
python manage.py makemigrations
```

Verás la siguiente salida:

```text
Migrations for 'courses':
  courses/migrations/0001_initial.py:
    - Create model Course
    - Create model Module
    - Create model Subject
    - Add field subject to course
```

Luego, ejecuta el siguiente comando para aplicar todas las migraciones a la base de datos:

```bash
python manage.py migrate
```

Deberías ver una salida que incluye todas las migraciones aplicadas, incluidas las de Django. La salida contendrá la siguiente línea:

```text
Applying courses.0001_initial... OK
```

Los modelos de tu aplicación `courses` se han sincronizado con la base de datos. A continuación, vamos a añadir los modelos de cursos al sitio de administración.

#### Registro de los modelos en el sitio de administración

Registremos los modelos de cursos en el sitio de administración para que podamos gestionar los datos fácilmente. Edita el archivo `admin.py` dentro del directorio de la aplicación `courses` y añádele el siguiente código:

```python
from django.contrib import admin
from .models import Subject, Course, Module


@admin.register(Subject)
class SubjectAdmin(admin.ModelAdmin):
    list_display = ['title', 'slug']
    prepopulated_fields = {'slug': ('title',)}


class ModuleInline(admin.StackedInline):
    model = Module


@admin.register(Course)
class CourseAdmin(admin.ModelAdmin):
    list_display = ['title', 'subject', 'created']
    list_filter = ['created', 'subject']
    search_fields = ['title', 'overview']
    prepopulated_fields = {'slug': ('title',)}
    inlines = [ModuleInline]
```

Los modelos para la aplicación `courses` ahora están registrados en el sitio de administración. Recuerda que utilizas el decorador `@admin.register()` para registrar modelos en el sitio de administración.

En la siguiente sección, aprenderás cómo crear datos iniciales para poblar tus modelos.

#### Uso de fixtures para proporcionar datos iniciales a los modelos

A veces, es posible que desees precargar tu base de datos con datos predefinidos. Esto es útil para incluir automáticamente datos iniciales en la configuración del proyecto, en lugar de tener que añadirlos manualmente. Django incluye una forma sencilla de cargar y volcar datos de la base de datos en archivos llamados **fixtures**. Django admite fixtures en formatos JSON, XML o YAML. La estructura de una fixture se asemeja mucho a la representación de la API de un modelo, lo que facilita la traducción de datos entre los formatos internos de la base de datos y las aplicaciones externas. Vas a crear una fixture para incluir varios objetos `Subject` iniciales para tu proyecto.

Primero, crea un superusuario utilizando el siguiente comando:

```bash
python manage.py createsuperuser
```

Luego, ejecuta el servidor de desarrollo usando el siguiente comando:

```bash
python manage.py runserver
```

Abre `http://127.0.0.1:8000/admin/courses/subject/` en tu navegador. Crea varias materias utilizando el sitio de administración. La página de lista de cambios debería verse así:

> *Figura 12.1: La vista de lista de cambios de asignaturas en el sitio de administración*

Ejecuta el siguiente comando desde la consola:

```bash
python manage.py dumpdata courses --indent=2
```

Verás una salida similar a la siguiente:

```json
[
  {
    "model": "courses.subject",
    "pk": 1,
    "fields": {
      "title": "Mathematics",
      "slug": "mathematics"
    }
  },
  {
    "model": "courses.subject",
    "pk": 2,
    "fields": {
      "title": "Music",
      "slug": "music"
    }
  },
  {
    "model": "courses.subject",
    "pk": 3,
    "fields": {
      "title": "Physics",
      "slug": "physics"
    }
  },
  {
    "model": "courses.subject",
    "pk": 4,
    "fields": {
      "title": "Programming",
      "slug": "programming"
    }
  }
]
```

El comando `dumpdata` vuelca los datos de la base de datos en la salida estándar, serializados en formato JSON de forma predeterminada. La estructura de datos resultante incluye información sobre el modelo y sus campos para que Django pueda cargarla en la base de datos.

Puedes limitar la salida a los modelos de una aplicación proporcionando los nombres de las aplicaciones al comando o especificando modelos individuales utilizando el formato `app.Model`.

También puedes especificar el formato utilizando la bandera `--format`. De forma predeterminada, `dumpdata` envía los datos serializados a la salida estándar. Sin embargo, puedes indicar un archivo de salida mediante la bandera `--output`. La bandera `--indent` te permite especificar sangrías. Para obtener más información sobre los parámetros de `dumpdata`, ejecuta `python manage.py dumpdata --help`.

Guarda este volcado en un archivo de fixtures en un nuevo directorio `fixtures/` en la aplicación `courses` usando los siguientes comandos:

```bash
mkdir courses/fixtures
python manage.py dumpdata courses --indent=2 --output=courses/fixtures/subjects.json
```

Ejecuta el servidor de desarrollo y utiliza el sitio de administración para eliminar las materias que creaste:

> *Figura 12.2: Eliminación de todas las asignaturas existentes*

Después de eliminar todas las materias, carga la fixture en la base de datos usando el siguiente comando:

```bash
python manage.py loaddata subjects.json
```

Todos los objetos `Subject` incluidos en la fixture se cargan de nuevo en la base de datos:

> *Figura 12.3: Las asignaturas de la fixture se han cargado en la base de datos*

De forma predeterminada, Django busca archivos en el directorio `fixtures/` de cada aplicación, pero puedes especificar la ruta completa al archivo de fixture para el comando `loaddata`. También puedes utilizar la configuración `FIXTURE_DIRS` para indicarle a Django directorios adicionales donde buscar fixtures.

Las fixtures no solo son útiles para configurar datos iniciales, sino también para proporcionar datos de muestra para tu aplicación o datos requeridos para tus pruebas. También puedes usar fixtures para poblar los datos necesarios para entornos de producción.

---

### Creación de modelos para contenido polimórfico

Planeas añadir diferentes tipos de contenido a los módulos de los cursos, como texto, imágenes, archivos y vídeos. El **polimorfismo** es la provisión de una interfaz única para entidades de diferentes tipos. Necesitas un modelo de datos versátil que te permita almacenar contenido diverso que sea accesible a través de una interfaz única. En el Capítulo 7, *Seguimiento de acciones de usuario*, aprendiste sobre la conveniencia de usar relaciones genéricas para crear claves foráneas que puedan apuntar a los objetos de cualquier modelo. Vas a crear un modelo `Content` que represente los contenidos de los módulos y definirás una relación genérica para asociar cualquier objeto con el objeto de contenido.

Edita el archivo `models.py` de la aplicación `courses` y añade las siguientes importaciones:

```python
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType
```

Luego, añade el siguiente código al final del archivo:

```python
class Content(models.Model):
    module = models.ForeignKey(
        Module,
        related_name='contents',
        on_delete=models.CASCADE
    )
    content_type = models.ForeignKey(
        ContentType,
        on_delete=models.CASCADE
    )
    object_id = models.PositiveIntegerField()
    item = GenericForeignKey('content_type', 'object_id')
```

Este es el modelo `Content`. Un módulo contiene múltiples contenidos, por lo que defines un campo `ForeignKey` que apunta al modelo `Module`. También configuras una relación genérica para asociar objetos de diferentes modelos que representan diferentes tipos de contenido. Recuerda que necesitas tres campos diferentes para configurar una relación genérica. En tu modelo `Content`, estos son:

- `content_type`: Un campo `ForeignKey` al modelo `ContentType`.
- `object_id`: Un `PositiveIntegerField` para almacenar la clave primaria del objeto relacionado.
- `item`: Un campo `GenericForeignKey` al objeto relacionado combinando los dos campos anteriores.

Solo los campos `content_type` y `object_id` tienen una columna correspondiente en la tabla de la base de datos de este modelo. El campo `item` te permite recuperar o establecer el objeto relacionado directamente, y su funcionalidad se basa en los otros dos campos.

Vas a utilizar un modelo distinto para cada tipo de contenido: texto, imagen, vídeo y documento. Tus modelos de contenido compartirán algunos campos comunes, pero variarán en los datos específicos que almacenan. Para lograr esto, necesitarás emplear la herencia de modelos.

#### Uso de la herencia de modelos

Django admite la herencia de modelos. Funciona de manera similar a la herencia de clases estándar en Python.

Django ofrece las siguientes tres opciones para usar la herencia de modelos:

- **Modelos abstractos (*Abstract models*):** Útiles cuando deseas colocar información común en varios modelos.
- **Herencia de modelos multi-tabla (*Multi-table model inheritance*):** Aplicable cuando cada modelo en la jerarquía se considera un modelo completo por sí mismo.
- **Modelos proxy (*Proxy models*):** Útiles cuando necesitas cambiar el comportamiento de un modelo, por ejemplo, incluyendo métodos adicionales, cambiando el manager predeterminado o utilizando diferentes opciones de metadatos.

Veamos cada una de ellas más de cerca.

##### Modelos abstractos

Un modelo abstracto es una clase base en la que defines los campos que deseas incluir en todos los modelos hijos. Django no crea ninguna tabla de base de datos para los modelos abstractos. Se crea una tabla de base de datos para cada modelo hijo, incluyendo los campos heredados de la clase abstracta y los definidos en el modelo hijo.

Para marcar un modelo como abstracto, debes incluir `abstract=True` en su clase `Meta`:

```python
from django.db import models


class BaseContent(models.Model):
    title = models.CharField(max_length=100)
    created = models.DateTimeField(auto_now_add=True)

    class Meta:
        abstract = True


class Text(BaseContent):
    body = models.TextField()
```

En este caso, Django crearía una tabla solo para el modelo `Text`, incluyendo los campos `title`, `created` y `body`.

> *Figura 12.4: Modelos de ejemplo y tablas de base de datos para la herencia usando modelos abstractos*

##### Herencia de modelos multi-tabla

En la herencia multi-tabla, cada modelo corresponde a una tabla de base de datos. Django crea un campo `OneToOneField` para la relación entre el modelo hijo y su modelo padre:

```python
from django.db import models


class BaseContent(models.Model):
    title = models.CharField(max_length=100)
    created = models.DateTimeField(auto_now_add=True)


class Text(BaseContent):
    body = models.TextField()
```

Django incluirá un campo `OneToOneField` generado automáticamente en el modelo `Text` que apunta al modelo `BaseContent` (llamado `basecontent_ptr`). Se crea una tabla de base de datos para cada modelo.

> *Figura 12.5: Modelos de ejemplo y tablas de base de datos para la herencia de modelos multi-tabla*

##### Modelos proxy

Un modelo proxy cambia el comportamiento de un modelo sin crear una nueva tabla de base de datos. Ambos modelos operan sobre la tabla de base de datos del modelo original:

```python
from django.db import models
from django.utils import timezone


class BaseContent(models.Model):
    title = models.CharField(max_length=100)
    created = models.DateTimeField(auto_now_add=True)


class OrderedContent(BaseContent):
    class Meta:
        proxy = True
        ordering = ['created']

    def created_delta(self):
        return timezone.now() - self.created
```

Ambos modelos, `BaseContent` y `OrderedContent`, operan en la misma tabla de base de datos.

> *Figura 12.6: Modelos de ejemplo y tablas de base de datos para la herencia usando modelos proxy*

#### Creación de los modelos de contenido (Content models)

Usemos modelos abstractos para implementar modelos polimórficos. Edita el archivo `models.py` de la aplicación `courses` y añade el siguiente código:

```python
class ItemBase(models.Model):
    owner = models.ForeignKey(
        User,
        related_name='%(class)s_related',
        on_delete=models.CASCADE
    )
    title = models.CharField(max_length=250)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True

    def __str__(self):
        return self.title


class Text(ItemBase):
    content = models.TextField()


class File(ItemBase):
    file = models.FileField(upload_to='files')


class Image(ItemBase):
    file = models.FileField(upload_to='images')


class Video(ItemBase):
    url = models.URLField()
```

En este código, defines un modelo abstracto llamado `ItemBase` con `abstract=True` en su clase `Meta`.

Django te permite especificar un marcador de posición para el nombre de la clase del modelo en el atributo `related_name` como `%(class)s`. Al hacerlo, el `related_name` para cada modelo hijo se generará automáticamente: `text_related`, `file_related`, `image_related` y `video_related`, respectivamente.

Has definido cuatro modelos de contenido diferentes que heredan del modelo abstracto `ItemBase`:

- `Text`: Para almacenar contenido de texto.
- `File`: Para almacenar archivos, como PDFs.
- `Image`: Para almacenar archivos de imagen.
- `Video`: Para almacenar vídeos; utilizas un campo `URLField` para proporcionar una URL de vídeo y poder incrustarlo.

La Figura 12.7 muestra los modelos de contenido y las tablas de base de datos asociadas:

> *Figura 12.7: Modelos de contenido y tablas de base de datos asociadas*

Edita el modelo `Content` que creaste anteriormente y modifica su campo `content_type`:

```python
    content_type = models.ForeignKey(
        ContentType,
        on_delete=models.CASCADE,
        limit_choices_to={
            'model__in': ('text', 'video', 'image', 'file')
        }
    )
```

Añades un argumento `limit_choices_to` para limitar los objetos `ContentType` que se pueden usar para la relación genérica.

Crea y aplica la nueva migración:

```bash
python manage.py makemigrations
```

```bash
python manage.py migrate
```

#### Creación de campos de modelo personalizados

Crearás un campo de orden personalizado que hereda de `PositiveIntegerField` y proporciona un comportamiento adicional:

1. **Asignar automáticamente un valor de orden cuando no se proporciona ninguno**: Cuando se guarda un nuevo objeto sin un orden específico, el campo asigna automáticamente el número que sigue al último objeto ordenado existente.
2. **Ordenar objetos con respecto a otros campos**: Los módulos del curso se ordenarán con respecto al curso al que pertenecen y los contenidos del módulo con respecto al módulo al que pertenecen.

Crea un nuevo archivo `fields.py` dentro del directorio de la aplicación `courses` y añade el siguiente código:

```python
from django.core.exceptions import ObjectDoesNotExist
from django.db import models


class OrderField(models.PositiveIntegerField):
    def __init__(self, for_fields=None, *args, **kwargs):
        self.for_fields = for_fields
        super().__init__(*args, **kwargs)

    def pre_save(self, model_instance, add):
        if getattr(model_instance, self.attname) is None:
            # no current value
            try:
                qs = self.model.objects.all()
                if self.for_fields:
                    # filter by objects with the same field values
                    # for the fields in "for_fields"
                    query = {
                        field: getattr(model_instance, field)
                        for field in self.for_fields
                    }
                    qs = qs.filter(**query)
                # get the order of the last item
                last_item = qs.latest(self.attname)
                value = getattr(last_item, self.attname) + 1
            except ObjectDoesNotExist:
                value = 0
            setattr(model_instance, self.attname, value)
            return value
        else:
            return super().pre_save(model_instance, add)
```

Este es el `OrderField` personalizado. Toma un parámetro opcional `for_fields`, que te permite indicar los campos utilizados para ordenar los datos. Sobrescribe el método `pre_save()`, que se ejecuta antes de guardar el campo en la base de datos.

#### Adición de ordenación a los objetos Module y Content

Añadamos el nuevo campo a tus modelos. Edita el archivo `models.py` de la aplicación `courses`:

```python
from .fields import OrderField


class Module(models.Model):
    course = models.ForeignKey(
        Course,
        related_name='modules',
        on_delete=models.CASCADE
    )
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    order = OrderField(blank=True, for_fields=['course'])

    class Meta:
        ordering = ['order']

    def __str__(self):
        return f'{self.order}. {self.title}'


class Content(models.Model):
    module = models.ForeignKey(
        Module,
        related_name='contents',
        on_delete=models.CASCADE
    )
    content_type = models.ForeignKey(
        ContentType,
        on_delete=models.CASCADE,
        limit_choices_to={
            'model__in': ('text', 'video', 'image', 'file')
        }
    )
    object_id = models.PositiveIntegerField()
    item = GenericForeignKey('content_type', 'object_id')
    order = OrderField(blank=True, for_fields=['module'])

    class Meta:
        ordering = ['order']
```

Crea las migraciones para los nuevos campos de orden:

```bash
python manage.py makemigrations courses
```

Cuando Django solicite un valor predeterminado para las filas existentes, selecciona la opción `1` e introduce `0` como valor por defecto tanto para `Content` como para `Module`.

Aplica las migraciones con:

```bash
python manage.py migrate
```

Probemos el nuevo campo en la shell de Python (`python manage.py shell`):

```python
>>> from django.contrib.auth.models import User
>>> from courses.models import Subject, Course, Module
>>> user = User.objects.last()
>>> subject = Subject.objects.last()
>>> c1 = Course.objects.create(subject=subject, owner=user, title='Course 1', slug='course1')
>>> m1 = Module.objects.create(course=c1, title='Module 1')
>>> m1.order
0
>>> m2 = Module.objects.create(course=c1, title='Module 2')
>>> m2.order
1
>>> m3 = Module.objects.create(course=c1, title='Module 3', order=5)
>>> m3.order
5
>>> m4 = Module.objects.create(course=c1, title='Module 4')
>>> m4.order
6
>>> c2 = Course.objects.create(subject=subject, title='Course 2', slug='course2', owner=user)
>>> m5 = Module.objects.create(course=c2, title='Module 1')
>>> m5.order
0
```

---

### Adición de vistas de autenticación

Ahora crearemos un sistema de autenticación para el CMS.

#### Adición de un sistema de autenticación

Edita el archivo `urls.py` principal del proyecto `educa` e incluye las vistas de inicio y cierre de sesión del framework de autenticación de Django:

```python
from django.conf import settings
from django.conf.urls.static import static
from django.contrib import admin
from django.contrib.auth import views as auth_views
from django.urls import path

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
]

if settings.DEBUG:
    urlpatterns += static(
        settings.MEDIA_URL,
        document_root=settings.MEDIA_ROOT
    )
```

#### Creación de las plantillas de autenticación

Crea la siguiente estructura de archivos dentro del directorio de la aplicación `courses`:

```text
templates/
    base.html
    registration/
        login.html
        logged_out.html
```

Edita `templates/base.html`:

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

Copia los archivos CSS estáticos ubicados en [https://github.com/PacktPublishing/Django-5-by-Example/tree/main/Chapter12/educa/courses/static](https://github.com/PacktPublishing/Django-5-by-Example/tree/main/Chapter12/educa/courses/static) en el directorio `courses/static/` de tu proyecto.

Edita `templates/registration/login.html`:

```html
{% extends "base.html" %}

{% block title %}Log-in{% endblock %}

{% block content %}
    <h1>Log-in</h1>
    <div class="module">
        {% if form.errors %}
            <p>Your username and password didn't match. Please try again.</p>
        {% else %}
            <p>Please, use the following form to log-in:</p>
        {% endif %}
        <div class="login-form">
            <form action="{% url 'login' %}" method="post">
                {{ form.as_p }}
                {% csrf_token %}
                <input type="hidden" name="next" value="{{ next }}" />
                <p><input type="submit" value="Log-in"></p>
            </form>
        </div>
    </div>
{% endblock %}
```

Edita `templates/registration/logged_out.html`:

```html
{% extends "base.html" %}

{% block title %}Logged out{% endblock %}

{% block content %}
    <h1>Logged out</h1>
    <div class="module">
        <p>
            You have been successfully logged out. You can <a href="{% url "login" %}">log-in again</a>.
        </p>
    </div>
{% endblock %}
```

Ejecuta el servidor de desarrollo y abre `http://127.0.0.1:8000/accounts/login/` en tu navegador:

> *Figura 12.8: La página de inicio de sesión de la cuenta*

Inicia sesión con tus credenciales de superusuario. Luego, abre `http://127.0.0.1:8000/accounts/login/` de nuevo y haz clic en **Sign out**:

> *Figura 12.9: La página de sesión cerrada de la cuenta*

---

### Resumen

En este capítulo, aprendiste a usar fixtures para proporcionar datos iniciales para los modelos. Al utilizar la herencia de modelos, creaste un sistema flexible para gestionar diferentes tipos de contenido para los módulos de los cursos. También implementaste un campo de modelo personalizado para ordenar objetos y creaste un sistema de autenticación para la plataforma de e-learning.

En el próximo capítulo, implementarás la funcionalidad del CMS para gestionar el contenido de los cursos utilizando vistas basadas en clases. Utilizarás el sistema de grupos y permisos de Django para restringir el acceso a las vistas, e implementarás formsets para editar el contenido de los cursos. También crearás una funcionalidad de arrastrar y soltar (*drag-and-drop*) para reordenar los módulos de los cursos y su contenido utilizando JavaScript y Django.

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter12](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter12)
- **Uso de fixtures de Django para pruebas:** [https://docs.djangoproject.com/en/5.2/topics/testing/tools/#fixture-loading](https://docs.djangoproject.com/en/5.2/topics/testing/tools/#fixture-loading)
- **Migraciones de datos en Django:** [https://docs.djangoproject.com/en/5.2/topics/migrations/#data-migrations](https://docs.djangoproject.com/en/5.2/topics/migrations/#data-migrations)
- **Herencia de clases en Python:** [https://docs.python.org/3/tutorial/classes.html#inheritance](https://docs.python.org/3/tutorial/classes.html#inheritance)
- **Herencia de modelos en Django:** [https://docs.djangoproject.com/en/5.2/topics/db/models/#model-inheritance](https://docs.djangoproject.com/en/5.2/topics/db/models/#model-inheritance)
- **Creación de campos de modelo personalizados:** [https://docs.djangoproject.com/en/5.2/howto/custom-model-fields/](https://docs.djangoproject.com/en/5.2/howto/custom-model-fields/)
- **Directorio de estáticos para el proyecto de e-learning:** [https://github.com/PacktPublishing/Django-5-by-Example/tree/main/Chapter12/educa/courses/static](https://github.com/PacktPublishing/Django-5-by-Example/tree/main/Chapter12/educa/courses/static)
