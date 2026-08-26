# Parte 4: Creación de una plataforma de E-Learning

## Capítulo 15: Creación de una API

### Introducción

En el capítulo anterior, construiste un sistema para el registro de estudiantes y la inscripción en cursos. Creaste vistas para mostrar los contenidos de los cursos y aprendiste a utilizar el framework de caché de Django.

En este capítulo, crearás una API RESTful para tu plataforma de e-learning. Una API es una interfaz programable común que se puede utilizar en múltiples plataformas, como sitios web, aplicaciones móviles, plugins, etc. Por ejemplo, puedes crear una API para que sea consumida por una aplicación móvil para tu plataforma de e-learning. Si proporcionas una API a terceros, podrán consumir información y operar con tu aplicación de manera programática. Una API permite a los desarrolladores automatizar acciones en tu plataforma e integrar tu servicio con otras aplicaciones o servicios en línea. Construirás una API con todas las funciones para tu plataforma de e-learning.

En este capítulo, aprenderás a:

- Instalar Django REST framework
- Crear serializadores para tus modelos
- Construir una API RESTful
- Implementar campos de método del serializador (*serializer method fields*)
- Crear serializadores anidados
- Implementar vistas ViewSet y enrutadores (*routers*)
- Construir vistas de API personalizadas
- Gestionar la autenticación de la API
- Añadir permisos a las vistas de la API
- Crear permisos personalizados
- Utilizar la biblioteca Requests para consumir la API

---

### Visión general funcional

La Figura 15.1 muestra una representación de las vistas y endpoints de la API que se construirán en este capítulo:

> *Figura 15.1: Diagrama de vistas y endpoints de la API que se construirán en el Capítulo 15*

En este capítulo, crearás dos conjuntos diferentes de vistas de API: `SubjectViewSet` y `CourseViewSet`. El primero incluirá las vistas de lista y detalle para las materias. El segundo incluirá las vistas de lista y detalle para los cursos. También implementarás la acción `enroll` en `CourseViewSet` para inscribir estudiantes en cursos. Esta acción solo estará disponible para usuarios autenticados mediante el permiso `IsAuthenticated`. Crearás la acción `contents` en `CourseViewSet` para acceder al contenido de un curso. Para acceder a los contenidos del curso, los usuarios deben estar autenticados e inscritos en el curso dado. Implementarás el permiso personalizado `IsEnrolled` para limitar el acceso a los contenidos a los usuarios inscritos en el curso.

Si no estás familiarizado con los endpoints de una API, solo necesitas saber que son las ubicaciones específicas dentro de una API que aceptan y responden a solicitudes. Cada endpoint corresponde a una URL que puede aceptar uno o más métodos HTTP, como GET, POST, PUT o DELETE.

El código fuente de este capítulo se puede encontrar en [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter15](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter15).

Todos los módulos de Python utilizados en este capítulo están incluidos en el archivo `requirements.txt` en el código fuente que acompaña a este capítulo. Puedes seguir las instrucciones para instalar cada módulo de Python a continuación, o puedes instalar todos los requisitos a la vez con el comando `python -m pip install -r requirements.txt`.

---

### Creación de una API RESTful

Al crear una API, existen varias formas de estructurar sus endpoints y acciones, pero se recomienda seguir los principios REST.

La arquitectura REST proviene de *Representational State Transfer* (Transferencia de Estado Representacional). Las APIs RESTful se basan en recursos; tus modelos representan recursos, y se utilizan métodos HTTP como GET, POST, PUT o DELETE para recuperar, crear, actualizar o eliminar objetos. Los códigos de respuesta HTTP también se utilizan en este contexto. Se devuelven diferentes códigos de respuesta HTTP para indicar el resultado de la solicitud HTTP, por ejemplo, códigos de respuesta 2XX para el éxito, 4XX para errores, etc.

Los formatos más comunes para intercambiar datos en APIs RESTful son JSON y XML. Construirás una API RESTful con serialización JSON para tu proyecto. Tu API proporcionará las siguientes funcionalidades:

- Recuperar materias (*subjects*)
- Recuperar cursos disponibles
- Recuperar contenidos de cursos
- Inscribirse en un curso

Puedes crear una API desde cero con Django creando vistas personalizadas. Sin embargo, existen varios módulos de terceros que simplifican la creación de una API para tu proyecto; el más popular de ellos es Django REST framework (DRF).

DRF proporciona un conjunto completo de herramientas para crear APIs RESTful para tus proyectos. Los siguientes son algunos de los componentes más relevantes que utilizaremos para construir nuestra API:

- **Serializadores (*Serializers*)**: Para transformar datos a un formato estandarizado que otros programas puedan entender, o para deserializar datos, convirtiendo datos a un formato que tu programa pueda procesar.
- **Parsers y renderers**: Para renderizar (o formatear) los datos serializados adecuadamente antes de devolverlos en una respuesta HTTP. De manera similar, para analizar los datos entrantes y garantizar que tengan el formato correcto.
- **Vistas de API (*API views*)**: Para implementar la lógica de la aplicación.
- **URLs**: Para definir los endpoints de la API que estarán disponibles.
- **Autenticación y permisos**: Para definir métodos de autenticación para la API y los permisos requeridos para cada vista.

Comenzaremos instalando DRF y, después de eso, aprenderemos más sobre estos componentes para construir nuestra primera API.

#### Instalación de Django REST framework

Puedes encontrar toda la información sobre DRF en [https://www.django-rest-framework.org/](https://www.django-rest-framework.org/).

Abre la consola e instala el framework con el siguiente comando:

```bash
python -m pip install djangorestframework==3.16
```

Edita el archivo `settings.py` del proyecto `educa` y añade `rest_framework` a la configuración `INSTALLED_APPS` para activar la aplicación, de la siguiente manera:

```python
INSTALLED_APPS = [
    # ...
    'rest_framework',
]
```

Luego, añade el siguiente código al archivo `settings.py`:

```python
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.DjangoModelPermissionsOrAnonReadOnly'
    ]
}
```

Puedes proporcionar una configuración específica para tu API utilizando la variable de configuración `REST_FRAMEWORK`. DRF ofrece una amplia gama de ajustes para configurar comportamientos predeterminados. El ajuste `DEFAULT_PERMISSION_CLASSES` especifica los permisos predeterminados para leer, crear, actualizar o eliminar objetos. Estableces `DjangoModelPermissionsOrAnonReadOnly` como la única clase de permiso predeterminada. Esta clase se basa en el sistema de permisos de Django para permitir a los usuarios crear, actualizar o eliminar objetos, al tiempo que proporciona acceso de solo lectura a los usuarios anónimos. Aprenderás más sobre los permisos más adelante, en la sección *Adición de permisos a las vistas*.

Para obtener una lista completa de los ajustes disponibles para DRF, puedes visitar [https://www.django-rest-framework.org/api-guide/settings/](https://www.django-rest-framework.org/api-guide/settings/).

#### Definición de serializadores

Después de configurar DRF, debes especificar cómo se serializarán tus datos. Los datos de salida deben serializarse en un formato específico y los datos de entrada se deserializarán para su procesamiento. El framework proporciona las siguientes clases para crear serializadores para objetos individuales:

- `Serializer`: Proporciona serialización para instancias de clases estándar de Python.
- `ModelSerializer`: Proporciona serialización para instancias de modelos.
- `HyperlinkedModelSerializer`: Igual que `ModelSerializer`, pero representa las relaciones entre objetos con enlaces en lugar de claves primarias.

Construyamos nuestro primer serializador. Crea la siguiente estructura de archivos dentro del directorio de la aplicación `courses`:

```text
api/
    __init__.py
    serializers.py
```

Construirás toda la funcionalidad de la API dentro del directorio `api` para mantener todo bien organizado. Edita el archivo `api/serializers.py` y añade el siguiente código:

```python
from rest_framework import serializers
from courses.models import Subject


class SubjectSerializer(serializers.ModelSerializer):
    class Meta:
        model = Subject
        fields = ['id', 'title', 'slug']
```

Este es el serializador para el modelo `Subject`. Los serializadores se definen de manera similar a las clases `Form` y `ModelForm` de Django. La clase `Meta` te permite especificar el modelo a serializar y los campos que se incluirán para la serialización. Todos los campos del modelo se incluirán si no estableces un atributo `fields`.

Probemos el serializador. Abre la línea de comandos e inicia la shell de Django con el siguiente comando:

```bash
python manage.py shell
```

Ejecuta el siguiente código:

```python
>>> from courses.models import Subject
>>> from courses.api.serializers import SubjectSerializer
>>> subject = Subject.objects.latest('id')
>>> serializer = SubjectSerializer(subject)
>>> serializer.data
{'id': 4, 'title': 'Programming', 'slug': 'programming'}
```

En este ejemplo, obtienes un objeto `Subject`, creas una instancia de `SubjectSerializer` y accedes a los datos serializados. Puedes ver que los datos del modelo se traducen a tipos de datos nativos de Python.

Puedes leer más sobre serializadores en [https://www.django-rest-framework.org/api-guide/serializers/](https://www.django-rest-framework.org/api-guide/serializers/).

#### Comprensión de parsers y renderers

Los datos serializados deben renderizarse en un formato específico antes de devolverlos en una respuesta HTTP. Del mismo modo, cuando recibes una solicitud HTTP, debes analizar (*parse*) los datos entrantes y deserializarlos antes de poder operar con ellos. DRF incluye renderers y parsers para gestionar esto.

Veamos cómo analizar datos entrantes. Ejecuta el siguiente código en la shell de Python:

```python
>>> from io import BytesIO
>>> from rest_framework.parsers import JSONParser
>>> data = b'{"id":4,"title":"Programming","slug":"programming"}'
>>> JSONParser().parse(BytesIO(data))
{'id': 4, 'title': 'Programming', 'slug': 'programming'}
```

Dada una cadena de entrada JSON, puedes usar la clase `JSONParser` proporcionada por DRF para convertirla en un objeto de Python.

DRF también incluye clases `Renderer` que te permiten formatear las respuestas de la API. El framework determina qué renderer usar a través de la negociación de contenido inspeccionando el encabezado `Accept` de la solicitud para determinar el tipo de contenido esperado para la respuesta. Opcionalmente, el renderer se determina mediante el sufijo de formato de la URL. Por ejemplo, la URL `http://127.0.0.1:8000/api/data.json` podría ser un endpoint que active el `JSONRenderer` para devolver una respuesta JSON.

Vuelve a la consola y ejecuta el siguiente código para renderizar el objeto serializador del ejemplo anterior:

```python
>>> from rest_framework.renderers import JSONRenderer
>>> JSONRenderer().render(serializer.data)
```

Verás la siguiente salida:

```python
b'{"id":4,"title":"Programming","slug":"programming"}'
```

Utilizas el `JSONRenderer` para renderizar los datos serializados a formato JSON. De forma predeterminada, DRF utiliza dos renderers diferentes: `JSONRenderer` y `BrowsableAPIRenderer`. Este último proporciona una interfaz web para navegar fácilmente por tu API. Puedes cambiar las clases de renderer predeterminadas con la opción `DEFAULT_RENDERER_CLASSES` del ajuste `REST_FRAMEWORK`.

Puedes encontrar más información sobre renderers y parsers en [https://www.django-rest-framework.org/api-guide/renderers/](https://www.django-rest-framework.org/api-guide/renderers/) y [https://www.django-rest-framework.org/api-guide/parsers/](https://www.django-rest-framework.org/api-guide/parsers/), respectivamente.

A continuación, vas a aprender a crear vistas de API y utilizar serializadores en las vistas.

#### Creación de vistas de lista y detalle

DRF viene con un conjunto de vistas genéricas y mixins que puedes usar para construir tus vistas de API. Has estado usando vistas genéricas a lo largo de este libro desde el Capítulo 2, Mejorar tu blog y añadir funciones sociales, y aprendiste sobre mixins en el Capítulo 13, Creación de un sistema de gestión de contenidos (CMS).

Las vistas base y los mixins proporcionan la funcionalidad para recuperar, crear, actualizar o eliminar objetos de modelo. Puedes ver todos los mixins y vistas genéricas proporcionados por DRF en [https://www.django-rest-framework.org/api-guide/generic-views/](https://www.django-rest-framework.org/api-guide/generic-views/).

Creemos vistas de lista y detalle para recuperar objetos `Subject`. Crea un nuevo archivo dentro del directorio `courses/api/` y nómbralo `views.py`. Añade el siguiente código:

```python
from rest_framework import generics
from courses.api.serializers import SubjectSerializer
from courses.models import Subject


class SubjectListView(generics.ListAPIView):
    queryset = Subject.objects.all()
    serializer_class = SubjectSerializer


class SubjectDetailView(generics.RetrieveAPIView):
    queryset = Subject.objects.all()
    serializer_class = SubjectSerializer
```

En este código, utilizas las vistas genéricas `ListAPIView` y `RetrieveAPIView` de DRF. Ambas vistas tienen los siguientes atributos:

- `queryset`: El QuerySet base a utilizar para recuperar objetos.
- `serializer_class`: La clase para serializar objetos.

Añadamos patrones de URL para tus vistas. Crea un nuevo archivo dentro del directorio `courses/api/`, nómbralo `urls.py` y estructúralo de la siguiente manera:

```python
from django.urls import path
from . import views

app_name = 'courses'

urlpatterns = [
    path(
        'subjects/',
        views.SubjectListView.as_view(),
        name='subject_list'
    ),
    path(
        'subjects/<pk>/',
        views.SubjectDetailView.as_view(),
        name='subject_detail'
    ),
]
```

En el patrón de URL para la vista `SubjectDetailView`, incluyes un parámetro de URL `pk` para recuperar el objeto con la clave primaria dada del modelo `Subject`, que es el campo `id`. Edita el archivo `urls.py` principal del proyecto `educa` e incluye los patrones de la API, de la siguiente manera:

```python
urlpatterns = [
    # ...
    path('api/', include('courses.api.urls', namespace='api')),
]
```

Utilizas el espacio de nombres `api` para las URLs de tu API. Nuestros endpoints iniciales de la API están listos para ser utilizados.

---

### Consumo de la API

Al hacer que nuestras vistas estén disponibles a través de URLs, hemos creado nuestros primeros endpoints de la API. Probemos ahora nuestra propia API. Asegúrate de que tu servidor se esté ejecutando con el siguiente comando:

```bash
python manage.py runserver
```

Vamos a usar `curl` para consumir la API. `curl` es una herramienta de línea de comandos que te permite transferir datos hacia y desde un servidor. Si estás usando Linux, macOS o Windows 10/11, es muy probable que `curl` esté incluido en tu sistema. Sin embargo, puedes descargar `curl` desde [https://curl.se/download.html](https://curl.se/download.html).

Abre la consola y recupera la URL `http://127.0.0.1:8000/api/subjects/` con `curl`, de la siguiente manera:

```bash
curl http://127.0.0.1:8000/api/subjects/
```

Obtendrás una respuesta similar a la siguiente:

```json
[
  {
    "id": 1,
    "title": "Mathematics",
    "slug": "mathematics"
  },
  {
    "id": 2,
    "title": "Music",
    "slug": "music"
  },
  {
    "id": 3,
    "title": "Physics",
    "slug": "physics"
  },
  {
    "id": 4,
    "title": "Programming",
    "slug": "programming"
  }
]
```

Para obtener una respuesta JSON más legible y bien indentada, puedes usar `curl` con la utilidad `json_pp`, de la siguiente manera:

```bash
curl http://127.0.0.1:8000/api/subjects/ | json_pp
```

La respuesta HTTP contiene una lista de objetos `Subject` en formato JSON.

En lugar de `curl`, también puedes usar cualquier otra herramienta para enviar solicitudes HTTP personalizadas, incluida una extensión de navegador o una aplicación como Postman, que puedes obtener en [https://www.getpostman.com/](https://www.getpostman.com/).

Abre `http://127.0.0.1:8000/api/subjects/` en tu navegador. Verás la API navegable (*browsable API*) de DRF, como se muestra a continuación:

> *Figura 15.2: La página Subject List en la API navegable de REST framework*

Esta interfaz HTML la proporciona el renderer `BrowsableAPIRenderer`. Muestra los encabezados y el contenido del resultado, y te permite realizar solicitudes. También puedes acceder a la vista de detalle de la API para un objeto `Subject` incluyendo su ID en la URL.

Abre `http://127.0.0.1:8000/api/subjects/1/` en tu navegador. Verás un único objeto `Subject` renderizado en formato JSON.

> *Figura 15.3: La página Subject Detail en la API navegable de REST framework*

Esta es la respuesta para `SubjectDetailView`. Aprendamos a enriquecer el contenido devuelto para cada materia. En la siguiente sección, profundizaremos en la extensión de serializadores con campos y métodos adicionales.

---

### Extensión de serializadores

Has aprendido a serializar los objetos de tus modelos; sin embargo, a menudo es posible que desees enriquecer la respuesta con datos relevantes adicionales o campos calculados. Echemos un vistazo a algunas de las opciones para extender los serializadores.

#### Adición de campos adicionales a los serializadores

Editemos las vistas de materias para incluir el número de cursos disponibles para cada una. Utilizarás las funciones de agregación de Django para anotar el recuento de cursos relacionados para cada materia.

Edita el archivo `api/views.py` de la aplicación `courses` y añade el siguiente código:

```python
from django.db.models import Count
# ...

class SubjectListView(generics.ListAPIView):
    queryset = Subject.objects.annotate(total_courses=Count('courses'))
    serializer_class = SubjectSerializer


class SubjectDetailView(generics.RetrieveAPIView):
    queryset = Subject.objects.annotate(total_courses=Count('courses'))
    serializer_class = SubjectSerializer
```

Ahora estás utilizando un QuerySet para `SubjectListView` y `SubjectDetailView` que utiliza la función de agregación `Count` para anotar el recuento de cursos relacionados.

Edita el archivo `api/serializers.py` de la aplicación `courses` y añade el siguiente código:

```python
from rest_framework import serializers
from courses.models import Subject


class SubjectSerializer(serializers.ModelSerializer):
    total_courses = serializers.IntegerField()

    class Meta:
        model = Subject
        fields = ['id', 'title', 'slug', 'total_courses']
```

Has añadido el campo `total_courses` a la clase `SubjectSerializer`. Este campo es un `IntegerField` para representar números enteros. Este campo obtendrá automáticamente su valor del atributo `total_courses` del objeto que se está serializando. Al usar `annotate()`, añadimos el atributo `total_courses` a los objetos resultantes del QuerySet.

Abre `http://127.0.0.1:8000/api/subjects/1/` en tu navegador. El objeto JSON serializado ahora incluye el atributo `total_courses`:

> *Figura 15.4: La página Subject Detail, incluyendo el atributo total_courses*

Has añadido con éxito el atributo `total_courses` a las vistas de lista y detalle de materias. Ahora, veamos cómo añadir atributos adicionales utilizando métodos de serializador personalizados.

#### Implementación de campos de método del serializador

DRF proporciona `SerializerMethodField`, que te permite implementar campos de solo lectura que obtienen su valor llamando a un método de la clase serializadora. Esto puede ser particularmente útil cuando deseas incluir algunos datos con formato personalizado en tu objeto serializado o realizar cálculos complejos que no son directamente parte de las instancias de tu modelo.

Crearemos un método que serialice los 3 cursos más populares para una materia. Clasificaremos los cursos por el número de estudiantes inscritos en ellos. Edita el archivo `api/serializers.py` de la aplicación `courses` y añade el siguiente código:

```python
from django.db.models import Count
from rest_framework import serializers
from courses.models import Subject


class SubjectSerializer(serializers.ModelSerializer):
    total_courses = serializers.IntegerField()
    popular_courses = serializers.SerializerMethodField()

    def get_popular_courses(self, obj):
        courses = obj.courses.annotate(
            total_students=Count('students')
        ).order_by('total_students')[:3]
        return [
            f'{c.title} ({c.total_students})' for c in courses
        ]

    class Meta:
        model = Subject
        fields = [
            'id',
            'title',
            'slug',
            'total_courses',
            'popular_courses'
        ]
```

En el nuevo código, añades el nuevo campo de método de serializador `popular_courses` a `SubjectSerializer`. El campo obtiene su valor del método `get_popular_courses()`. Puedes proporcionar el nombre del método de serializador a llamar con el argumento `method_name` de `SerializerMethodField`. Si no se incluye, este valor predeterminado es `get_<field_name>`.

Abre `http://127.0.0.1:8000/api/subjects/1/` en tu navegador. El objeto JSON serializado ahora incluye el atributo `popular_courses`:

> *Figura 15.5: La página de detalle de la materia, incluyendo el atributo popular_courses*

Has implementado con éxito un `SerializerMethodField`. Ten en cuenta que, ahora, se genera una consulta SQL adicional para cada uno de los resultados devueltos por `SubjectListView`. A continuación, vas a aprender a controlar el número de resultados devueltos añadiendo paginación a `SubjectListView`.

---

### Adición de paginación a las vistas

DRF incluye capacidades de paginación integradas para controlar cuántos objetos se envían en las respuestas de tu API. Cuando el contenido de tu sitio comience a crecer, podrías terminar con una gran cantidad de materias y cursos. La paginación puede ser especialmente útil para mejorar el rendimiento y la experiencia del usuario al tratar con conjuntos de datos grandes.

Actualicemos la vista `SubjectListView` para incluir paginación. Primero, definiremos una clase de paginación.

Crea un nuevo archivo dentro del directorio `courses/api/` y nómbralo `pagination.py`. Añade el siguiente código:

```python
from rest_framework.pagination import PageNumberPagination


class StandardPagination(PageNumberPagination):
    page_size = 10
    page_size_query_param = 'page_size'
    max_page_size = 50
```

En esta clase, heredamos de `PageNumberPagination`. Esta clase proporciona soporte para la paginación basada en números de página. Establecemos los siguientes atributos:

- `page_size`: Determina el tamaño de página predeterminado (el número de elementos devueltos por página) cuando no se proporciona ningún tamaño de página en la solicitud.
- `page_size_query_param`: Define el nombre del parámetro de consulta a utilizar para el tamaño de página.
- `max_page_size`: Indica el tamaño de página máximo solicitado permitido.

Ahora, edita el archivo `api/views.py` de la aplicación `courses` y añade las siguientes líneas:

```python
from django.db.models import Count
from rest_framework import generics
from courses.models import Subject
from courses.api.pagination import StandardPagination
from courses.api.serializers import SubjectSerializer


class SubjectListView(generics.ListAPIView):
    queryset = Subject.objects.annotate(total_courses=Count('courses'))
    serializer_class = SubjectSerializer
    pagination_class = StandardPagination
# ...
```

Ahora puedes paginar los objetos devueltos por `SubjectListView`. Abre `http://127.0.0.1:8000/api/subjects/` en tu navegador. Puedes ver que la estructura JSON devuelta por la vista ahora es diferente debido a la paginación. Verás la siguiente estructura:

```json
{
  "count": 4,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 1,
      "title": "Mathematics"
    }
  ]
}
```

Los siguientes elementos ahora forman parte del JSON devuelto:

- `count`: El número total de resultados.
- `next`: La URL para recuperar la página siguiente. El valor es `null` cuando no hay páginas siguientes.
- `previous`: La URL para recuperar la página anterior. El valor es `null` cuando no hay páginas anteriores.
- `results`: Una lista con los objetos serializados devueltos en esta página.

Abre `http://127.0.0.1:8000/api/subjects/?page_size=2&page=1` en tu navegador. Esto paginará los resultados en dos elementos por página y recuperará la primera página de resultados:

> *Figura 15.6: Primera página de resultados para la paginación de lista de materias, con un tamaño de página de 2*

Hemos implementado la paginación basada en el número de página, pero DRF también proporciona clases para implementar paginación basada en límites/desplazamientos (*limit/offset*) y basada en cursores (*cursor-based*). Puedes leer más sobre la paginación en [https://www.django-rest-framework.org/api-guide/pagination/](https://www.django-rest-framework.org/api-guide/pagination/).

Has creado los endpoints de la API para las vistas de materias. A continuación, añadirás endpoints de cursos a tu API.

---

### Creación del serializador del curso

Vamos a crear un serializador para el modelo `Course`. Edita el archivo `api/serializers.py` de la aplicación `courses` y añade el siguiente código:

```python
# ...
from courses.models import Course, Subject


class CourseSerializer(serializers.ModelSerializer):
    class Meta:
        model = Course
        fields = [
            'id',
            'subject',
            'title',
            'slug',
            'overview',
            'created',
            'owner',
            'modules'
        ]
```

Veamos cómo se serializa un objeto `Course`. Abre la consola y ejecuta el siguiente comando:

```bash
python manage.py shell
```

Ejecuta el siguiente código:

```python
>>> from rest_framework.renderers import JSONRenderer
>>> from courses.models import Course
>>> from courses.api.serializers import CourseSerializer
>>> course = Course.objects.latest('id')
>>> serializer = CourseSerializer(course)
>>> JSONRenderer().render(serializer.data)
```

Obtendrás un objeto JSON con los campos que incluiste en `CourseSerializer`. Puedes ver que los objetos relacionados del gestor `modules` se serializan como una lista de claves primarias:

```json
"modules": [6, 7, 9, 10]
```

Estos son los IDs de los objetos `Module` relacionados. A continuación, vas a aprender diferentes métodos para serializar objetos relacionados.

#### Serialización de relaciones

DRF viene con diferentes tipos de campos relacionales para representar las relaciones del modelo. Esto funciona para relaciones `ForeignKey`, `ManyToManyField` y `OneToOneField`, así como para relaciones genéricas de modelos.

Vamos a utilizar `StringRelatedField` para cambiar cómo se serializan los objetos `Module` relacionados. `StringRelatedField` representa el objeto relacionado utilizando su método `__str__()`.

Edita el archivo `api/serializers.py` de la aplicación `courses` y añade el siguiente código:

```python
# ...
class CourseSerializer(serializers.ModelSerializer):
    modules = serializers.StringRelatedField(many=True, read_only=True)

    class Meta:
        # ...
```

En el nuevo código, defines el campo `modules` que proporciona serialización para los objetos `Module` relacionados. Utilizas `many=True` para indicar que estás serializando múltiples objetos relacionados. El parámetro `read_only=True` indica que este campo es de solo lectura y no debe incluirse en ninguna entrada para crear o actualizar objetos.

Abre la shell y crea una instancia de `CourseSerializer` nuevamente. Renderiza el atributo `data` del serializador con `JSONRenderer`. Esta vez, los módulos listados se serializan utilizando su método `__str__()`:

```json
"modules": ["0. Installing Django", "1. Configuring Django"]
```

Ten en cuenta que DRF no optimiza los QuerySets automáticamente. Al serializar una lista de cursos, se generará una consulta SQL para cada resultado de curso con el fin de recuperar los objetos `Module` relacionados. Puedes reducir la cantidad de solicitudes SQL adicionales utilizando `prefetch_related()` en tu QuerySet, como `Course.objects.prefetch_related('modules')`. Cubriremos esto más adelante en la sección *Creación de ViewSets y enrutadores (routers)*.

Puedes leer más sobre las relaciones de serializadores en [https://www.django-rest-framework.org/api-guide/relations/](https://www.django-rest-framework.org/api-guide/relations/).

Avancemos más y definamos la serialización de objetos relacionados con un serializador anidado.

---

### Creación de serializadores anidados

Si queremos incluir más información sobre cada módulo, necesitamos serializar los objetos `Module` y anidarlos. Modifica el código anterior del archivo `api/serializers.py` de la aplicación `courses` para que se vea de la siguiente manera:

```python
from django.db.models import Count
from rest_framework import serializers
from courses.models import Course, Module, Subject


class ModuleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Module
        fields = ['order', 'title', 'description']


class CourseSerializer(serializers.ModelSerializer):
    modules = ModuleSerializer(many=True, read_only=True)

    class Meta:
        # ...
```

En el nuevo código, defines `ModuleSerializer` para proporcionar serialización para el modelo `Module`. Luego, modificas el atributo `modules` de `CourseSerializer` para anidar el serializador `ModuleSerializer`. Mantienes `many=True` para indicar que estás serializando múltiples objetos y `read_only=True` para mantener este campo como de solo lectura.

Abre la shell y crea una instancia de `CourseSerializer` nuevamente. Renderiza el atributo `data` del serializador con `JSONRenderer`. Esta vez, los módulos listados se serializan con el serializador `ModuleSerializer` anidado:

```json
"modules": [
  {
    "order": 0,
    "title": "Introduction to overview",
    "description": "A brief overview about the Web Framework."
  },
  {
    "order": 1,
    "title": "Configuring Django",
    "description": "How to install Django."
  }
]
```

---

### Creación de ViewSets y enrutadores (routers)

Los `ViewSets` te permiten definir las interacciones de tu API y permitir que DRF cree URLs dinámicamente con un objeto `Router`. Al usar ViewSets, puedes evitar repetir la lógica para múltiples vistas. Los ViewSets incluyen acciones para las siguientes operaciones estándar:

- Operación de creación: `create()`
- Operación de recuperación: `list()` y `retrieve()`
- Operación de actualización: `update()` y `partial_update()`
- Operación de eliminación: `destroy()`

Creemos un ViewSet para el modelo `Course`. Edita el archivo `api/views.py` y añade el siguiente código:

```python
from django.db.models import Count
from rest_framework import generics
from rest_framework import viewsets
from courses.api.pagination import StandardPagination
from courses.api.serializers import CourseSerializer, SubjectSerializer
from courses.models import Course, Subject


class CourseViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Course.objects.prefetch_related('modules')
    serializer_class = CourseSerializer
    pagination_class = StandardPagination
```

La nueva clase `CourseViewSet` hereda de `ReadOnlyModelViewSet`, que proporciona las acciones de solo lectura `list()` y `retrieve()` para listar objetos o recuperar un solo objeto, respectivamente. Especificas el QuerySet base para recuperar objetos. Utilizas `prefetch_related('modules')` para obtener los objetos `Module` relacionados de manera eficiente. Esto evitará consultas SQL adicionales al serializar módulos anidados para cada curso. En esta clase, también defines las clases de serializador y paginación que se utilizarán para el ViewSet.

Edita el archivo `api/urls.py` y crea un router para tu ViewSet, de la siguiente manera:

```python
from django.urls import include, path
from rest_framework import routers
from . import views

app_name = 'courses'

router = routers.DefaultRouter()
router.register('courses', views.CourseViewSet)

urlpatterns = [
    # ...
    path('', include(router.urls)),
]
```

Creas un objeto `DefaultRouter` y registras `CourseViewSet` con el prefijo `courses`. El router se encarga de generar URLs automáticamente para tu ViewSet.

Abre `http://127.0.0.1:8000/api/` en tu navegador. Verás que el router enumera el ViewSet de cursos en su URL base:

> *Figura 15.7: La página API Root de la API navegable de REST framework*

Puedes acceder a `http://127.0.0.1:8000/api/courses/` para recuperar la lista de cursos:

> *Figura 15.8: La página Course List en la API navegable de REST framework*

Convirtamos las vistas `SubjectListView` y `SubjectDetailView` en un único ViewSet. Edita el archivo `api/views.py` y elimina o comenta las clases `SubjectListView` y `SubjectDetailView`. Luego, añade el siguiente código:

```python
# ...
class SubjectViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Subject.objects.annotate(total_courses=Count('courses'))
    serializer_class = SubjectSerializer
    pagination_class = StandardPagination
```

Edita el archivo `api/urls.py` y elimina o comenta las siguientes URLs, ya que no las necesitas más:

```python
# path(
#     'subjects/',
#     views.SubjectListView.as_view(),
#     name='subject_list'
# ),
# path(
#     'subjects/<pk>/',
#     views.SubjectDetailView.as_view(),
#     name='subject_detail'
# ),
```

En el mismo archivo, añade el siguiente código:

```python
from django.urls import include, path
from rest_framework import routers
from . import views

app_name = 'courses'

router = routers.DefaultRouter()
router.register('courses', views.CourseViewSet)
router.register('subjects', views.SubjectViewSet)

urlpatterns = [
    path('', include(router.urls)),
]
```

Abre `http://127.0.0.1:8000/api/` en tu navegador. Verás que el router ahora incluye URLs para los ViewSets de cursos y materias:

> *Figura 15.9: La página API Root de la API navegable de REST framework*

Puedes obtener más información sobre ViewSets en [https://www.django-rest-framework.org/api-guide/viewsets/](https://www.django-rest-framework.org/api-guide/viewsets/). También puedes encontrar más información sobre routers en [https://www.django-rest-framework.org/api-guide/routers/](https://www.django-rest-framework.org/api-guide/routers/).

Las vistas genéricas de API y los ViewSets son muy útiles para crear APIs REST basadas en tus modelos y serializadores. Sin embargo, es posible que también debas implementar tus propias vistas con lógica personalizada. Aprendamos a crear una vista de API personalizada.

---

### Creación de vistas de API personalizadas

DRF proporciona una clase `APIView` que construye la funcionalidad de API sobre la clase `View` de Django. La clase `APIView` difiere de `View` al usar los objetos `Request` y `Response` personalizados de DRF y al manejar excepciones `APIException` para devolver las respuestas HTTP apropiadas. También tiene un sistema integrado de autenticación y autorización para gestionar el acceso a las vistas.

Vas a crear una vista para que los usuarios se inscriban en los cursos. Edita el archivo `api/views.py` de la aplicación `courses` y añade el siguiente código:

```python
from django.db.models import Count
from django.shortcuts import get_object_or_404
from rest_framework import generics
from rest_framework import viewsets
from rest_framework.response import Response
from rest_framework.views import APIView
from courses.api.pagination import StandardPagination
from courses.api.serializers import CourseSerializer, SubjectSerializer
from courses.models import Course, Subject

# ...


class CourseEnrollView(APIView):
    def post(self, request, pk, format=None):
        course = get_object_or_404(Course, pk=pk)
        course.students.add(request.user)
        return Response({'enrolled': True})
```

La vista `CourseEnrollView` maneja la inscripción de usuarios en cursos. El código anterior funciona de la siguiente manera:

1. Creas una vista personalizada que hereda de `APIView`.
2. Defines un método `post()` para acciones POST. No se permitirá ningún otro método HTTP para esta vista.
3. Esperas un parámetro de URL `pk` que contenga el ID de un curso. Recuperas el curso por el parámetro `pk` dado y lanzas una excepción 404 si no se encuentra.
4. Añades el usuario actual a la relación muchos a muchos `students` del objeto `Course` y devuelves una respuesta exitosa.

Edita el archivo `api/urls.py` y añade el siguiente patrón de URL para la vista `CourseEnrollView`:

```python
path(
    'courses/<pk>/enroll/',
    views.CourseEnrollView.as_view(),
    name='course_enroll'
),
```

Teóricamente, ahora podrías realizar una solicitud POST para inscribir al usuario actual en un curso. Sin embargo, necesitas poder identificar al usuario y evitar que los usuarios no autenticados accedan a esta vista. Veamos cómo funcionan la autenticación y los permisos de la API.

---

### Gestión de la autenticación

DRF proporciona clases de autenticación para identificar al usuario que realiza la solicitud. Si la autenticación es exitosa, el framework establece el objeto `User` autenticado en `request.user`. Si ningún usuario está autenticado, se establece una instancia de `AnonymousUser` de Django en su lugar.

DRF proporciona los siguientes backends de autenticación:

- `BasicAuthentication`: Autenticación básica HTTP. El usuario y la contraseña son enviados por el cliente en el encabezado HTTP `Authorization`, codificados en Base64. Puedes obtener más información en [https://en.wikipedia.org/wiki/Basic_access_authentication](https://en.wikipedia.org/wiki/Basic_access_authentication).
- `TokenAuthentication`: Autenticación basada en tokens. Se utiliza un modelo `Token` para almacenar los tokens de los usuarios. Los usuarios incluyen el token en el encabezado HTTP `Authorization` para la autenticación.
- `SessionAuthentication`: Utiliza el backend de sesión de Django para la autenticación. Este backend es útil para realizar solicitudes AJAX autenticadas a la API desde el frontend de tu sitio web.
- `RemoteUserAuthentication`: Te permite delegar la autenticación a tu servidor web, que establece una variable de entorno `REMOTE_USER`.

Puedes crear un backend de autenticación personalizado heredando de la clase `BaseAuthentication` proporcionada por DRF y sobrescribiendo el método `authenticate()`.

#### Implementación de autenticación básica

Puedes configurar la autenticación por vista o globalmente con el ajuste `DEFAULT_AUTHENTICATION_CLASSES`.

La autenticación solo identifica al usuario que realiza la solicitud. No permitirá ni denegará el acceso a las vistas. Debes usar permisos para restringir el acceso a las vistas.

Puedes encontrar toda la información sobre la autenticación en [https://www.django-rest-framework.org/api-guide/authentication/](https://www.django-rest-framework.org/api-guide/authentication/).

Añadamos `BasicAuthentication` a tu vista. Edita el archivo `api/views.py` de la aplicación `courses` y añade un atributo `authentication_classes` a `CourseEnrollView`, de la siguiente manera:

```python
# ...
from rest_framework.authentication import BasicAuthentication


class CourseEnrollView(APIView):
    authentication_classes = [BasicAuthentication]
    # ...
```

Los usuarios serán identificados por las credenciales establecidas en el encabezado `Authorization` de la solicitud HTTP.

---

### Adición de permisos a las vistas

DRF incluye un sistema de permisos para restringir el acceso a las vistas. Algunos de los permisos integrados de DRF son:

- `AllowAny`: Acceso sin restricciones, independientemente de si el usuario está autenticado o no.
- `IsAuthenticated`: Permite el acceso únicamente a usuarios autenticados.
- `IsAuthenticatedOrReadOnly`: Acceso completo a usuarios autenticados. A los usuarios anónimos solo se les permite ejecutar métodos de lectura como GET, HEAD u OPTIONS.
- `DjangoModelPermissions`: Permisos vinculados a `django.contrib.auth`. La vista requiere un atributo `queryset`. Solo los usuarios autenticados con permisos de modelo asignados tienen permiso.
- `DjangoObjectPermissions`: Permisos de Django por objeto.

Si a los usuarios se les niega el permiso, normalmente obtendrán uno de los siguientes códigos de error HTTP:

- **HTTP 401**: Unauthorized (*No autorizado*)
- **HTTP 403**: Permission denied (*Permiso denegado*)

Puedes leer más información sobre permisos en [https://www.django-rest-framework.org/api-guide/permissions/](https://www.django-rest-framework.org/api-guide/permissions/).

Edita el archivo `api/views.py` de la aplicación `courses` y añade un atributo `permission_classes` a `CourseEnrollView`:

```python
# ...
from rest_framework.authentication import BasicAuthentication
from rest_framework.permissions import IsAuthenticated


class CourseEnrollView(APIView):
    authentication_classes = [BasicAuthentication]
    permission_classes = [IsAuthenticated]
    # ...
```

Incluyes el permiso `IsAuthenticated`. Esto evitará que los usuarios anónimos accedan a la vista. Ahora, puedes realizar una solicitud POST a tu nuevo método de API.

Asegúrate de que el servidor de desarrollo se esté ejecutando. Abre la consola y ejecuta el siguiente comando:

```bash
curl -i -X POST http://127.0.0.1:8000/api/courses/1/enroll/
```

Obtendrás la siguiente respuesta:

```text
HTTP/1.1 401 Unauthorized
...
{"detail": "Authentication credentials were not provided."}
```

Obtuviste un código HTTP 401 como se esperaba, ya que no estás autenticado. Usemos la autenticación básica con uno de tus usuarios. Ejecuta el siguiente comando, reemplazando `student:password` con las credenciales de un usuario existente:

```bash
curl -i -X POST -u student:password http://127.0.0.1:8000/api/courses/1/enroll/
```

Obtendrás la siguiente respuesta:

```text
HTTP/1.1 200 OK
...
{"enrolled": true}
```

Puedes acceder al sitio de administración y verificar que el usuario ahora esté inscrito en el curso.

---

### Adición de acciones adicionales a los ViewSets

Puedes añadir acciones adicionales a los ViewSets. Cambiemos la vista `CourseEnrollView` a una acción de ViewSet personalizada. Edita el archivo `api/views.py` y modifica la clase `CourseViewSet` para que se vea de la siguiente manera:

```python
# ...
from rest_framework.decorators import action


class CourseViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Course.objects.prefetch_related('modules')
    serializer_class = CourseSerializer

    @action(
        detail=True,
        methods=['post'],
        authentication_classes=[BasicAuthentication],
        permission_classes=[IsAuthenticated]
    )
    def enroll(self, request, *args, **kwargs):
        course = self.get_object()
        course.students.add(request.user)
        return Response({'enrolled': True})
```

En el código anterior, añades un método personalizado `enroll()` que representa una acción adicional para este ViewSet. El código funciona de la siguiente manera:

1. Utilizas el decorador `action` del framework con el parámetro `detail=True` para especificar que esta es una acción que se realizará en un solo objeto.
2. El decorador te permite añadir atributos personalizados para la acción. Especificas que solo se permite el método `post()` para esta vista y estableces las clases de autenticación y permisos.
3. Utilizas `self.get_object()` para recuperar el objeto `Course`.
4. Añades el usuario actual a la relación muchos a muchos `students` y devuelves una respuesta de éxito personalizada.

Edita el archivo `api/urls.py` y elimina o comenta la siguiente URL, ya que no la necesitas más:

```python
# path(
#     'courses/<pk>/enroll/',
#     views.CourseEnrollView.as_view(),
#     name='course_enroll'
# ),
```

Luego, edita el archivo `api/views.py` y elimina o comenta la clase `CourseEnrollView`.

La URL para inscribirse en cursos ahora la genera automáticamente el router. La URL sigue siendo la misma, ya que se construye dinámicamente utilizando el nombre de la acción `enroll`.

Después de que los estudiantes se inscriben en un curso, necesitan acceder al contenido del curso. A continuación, vas a aprender a asegurarte de que solo los estudiantes que se hayan inscrito puedan acceder al curso.

---

### Creación de permisos personalizados

Deseas que los estudiantes puedan acceder a los contenidos de los cursos en los que están inscritos. Solo los estudiantes inscritos en un curso deben poder acceder a sus contenidos. La mejor manera de hacer esto es con una clase de permiso personalizada. DRF proporciona una clase `BasePermission` que te permite definir los siguientes métodos:

- `has_permission()`: Una comprobación de permisos a nivel de vista.
- `has_object_permission()`: Una comprobación de permisos a nivel de instancia.

Estos métodos deben devolver `True` para otorgar acceso o `False` en caso contrario.

Crea un nuevo archivo dentro del directorio `courses/api/` y nómbralo `permissions.py`. Añade el siguiente código:

```python
from rest_framework.permissions import BasePermission


class IsEnrolled(BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.students.filter(id=request.user.id).exists()
```

Heredas de la clase `BasePermission` y sobrescribes `has_object_permission()`. Compruebas que el usuario que realiza la solicitud esté presente en la relación `students` del objeto `Course`. Vas a utilizar el permiso `IsEnrolled` a continuación.

---

### Serialización de contenidos del curso

Necesitas serializar los contenidos del curso. El modelo `Content` incluye una clave foránea genérica que te permite asociar objetos de diferentes modelos de contenido. Sin embargo, agregaste un método común `render()` para todos los modelos de contenido en el capítulo anterior. Puedes usar este método para proporcionar contenido renderizado a tu API.

Edita el archivo `api/serializers.py` de la aplicación `courses` y añade el siguiente código:

```python
from courses.models import Content, Course, Module, Subject


class ItemRelatedField(serializers.RelatedField):
    def to_representation(self, value):
        return value.render()


class ContentSerializer(serializers.ModelSerializer):
    item = ItemRelatedField(read_only=True)

    class Meta:
        model = Content
        fields = ['order', 'item']
```

En este código, defines un campo personalizado heredando del campo serializador `RelatedField` proporcionado por DRF y sobrescribiendo el método `to_representation()`. Defines el serializador `ContentSerializer` para el modelo `Content` y utilizas el campo personalizado para la clave foránea genérica `item`.

Necesitas un serializador alternativo para el modelo `Module` que incluya sus contenidos, así como un serializador `Course` extendido. Edita el archivo `api/serializers.py` y añade el siguiente código:

```python
class ModuleWithContentsSerializer(serializers.ModelSerializer):
    contents = ContentSerializer(many=True)

    class Meta:
        model = Module
        fields = ['order', 'title', 'description', 'contents']


class CourseWithContentsSerializer(serializers.ModelSerializer):
    modules = ModuleWithContentsSerializer(many=True)

    class Meta:
        model = Course
        fields = [
            'id',
            'subject',
            'title',
            'slug',
            'overview',
            'created',
            'owner',
            'modules'
        ]
```

Creemos una vista que imite el comportamiento de la acción `retrieve()` pero que incluya los contenidos del curso. Edita el archivo `api/views.py` y añade el siguiente método a la clase `CourseViewSet`:

```python
# ...
from courses.api.permissions import IsEnrolled
from courses.api.serializers import CourseWithContentsSerializer


class CourseViewSet(viewsets.ReadOnlyModelViewSet):
    # ...
    @action(
        detail=True,
        methods=['get'],
        serializer_class=CourseWithContentsSerializer,
        authentication_classes=[BasicAuthentication],
        permission_classes=[IsAuthenticated, IsEnrolled]
    )
    def contents(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)
```

La descripción de este método es la siguiente:

1. Utilizas el decorador `action` con el parámetro `detail=True` para especificar una acción que se realiza en un solo objeto.
2. Especificas que solo se permite el método GET para esta acción.
3. Utilizas la nueva clase de serializador `CourseWithContentsSerializer` que incluye los contenidos renderizados del curso.
4. Utilizas tanto `IsAuthenticated` como tus permisos personalizados `IsEnrolled`. Al hacerlo, te aseguras de que solo los usuarios inscritos en el curso puedan acceder a sus contenidos.
5. Utilizas la acción `retrieve()` existente para devolver el objeto `Course`.

Abre `http://127.0.0.1:8000/api/courses/1/contents/` en tu navegador. Si accedes a la vista con las credenciales correctas, verás que cada módulo del curso incluye el HTML renderizado para los contenidos del curso:

```json
{
  "order": 0,
  "title": "Introduction to Django",
  "description": "Brief introduction to the Django Web Framework.",
  "contents": [
    {
      "order": 0,
      "item": "<p>Meet Django. Django is a high-level Python Web framework ...</p>"
    },
    {
      "order": 1,
      "item": "\n<iframe width=\"480\" height=\"360\" src=\"http://www.youtube.com/embed/bgV39DlmZ2U? wmode=opaque\" frameborder=\"0\" allowfullscreen></iframe>\n"
    }
  ]
}
```

Has creado una API simple que permite a otros servicios acceder a la aplicación del curso mediante programación. DRF también te permite gestionar la creación y edición de objetos con la clase `ModelViewSet`. Hemos cubierto los aspectos principales de DRF, pero encontrarás más información sobre sus funciones en su extensa documentación en [https://www.django-rest-framework.org/](https://www.django-rest-framework.org/).

---

### Consumo de la API RESTful

Ahora que has implementado una API, puedes consumirla de manera programática desde otras aplicaciones. Puedes interactuar con la API utilizando la API Fetch de JavaScript en el frontend de tu aplicación, de manera similar a las funcionalidades que construiste en el Capítulo 6, Compartir contenido en tu sitio web. También puedes consumir la API desde aplicaciones creadas con Python o cualquier otro lenguaje de programación.

Vas a crear una aplicación Python simple que utiliza la API RESTful para recuperar todos los cursos disponibles y luego inscribir a un estudiante en todos ellos. Aprenderás a autenticarte contra la API utilizando la autenticación básica HTTP y a realizar solicitudes GET y POST.

Usaremos la biblioteca Python Requests para consumir la API. Usamos Requests en el Capítulo 6, Compartir contenido en tu sitio web, para recuperar imágenes mediante su URL. Requests abstrae la complejidad de lidiar con solicitudes HTTP y proporciona una interfaz muy simple para consumir servicios HTTP. Puedes encontrar la documentación de la biblioteca Requests en [https://requests.readthedocs.io/en/master/](https://requests.readthedocs.io/en/master/).

Abre la consola e instala la biblioteca Requests con el siguiente comando:

```bash
python -m pip install requests==2.31.0
```

Crea un nuevo directorio junto al directorio del proyecto `educa` y nómbralo `api_examples`. Crea un nuevo archivo dentro del directorio `api_examples/` y nómbralo `enroll_all.py`. La estructura de archivos ahora debería verse así:

```text
api_examples/
    enroll_all.py
educa/
    ...
```

Edita el archivo `enroll_all.py` y añade el siguiente código:

```python
import requests

base_url = 'http://127.0.0.1:8000/api/'
url = f'{base_url}courses/'
available_courses = []

while url is not None:
    print(f'Loading courses from {url}')
    r = requests.get(url)
    response = r.json()
    url = response['next']
    courses = response['results']
    available_courses += [course['title'] for course in courses]

print(f'Available courses: {", ".join(available_courses)}')
```

En este código, realizas las siguientes acciones:

1. Importas la biblioteca Requests y defines la URL base para la API y la URL para el endpoint de la lista de cursos.
2. Defines la lista vacía `available_courses`.
3. Utilizas una declaración `while` para paginar sobre todas las páginas de resultados.
4. Utilizas `requests.get()` para recuperar datos de la API enviando una solicitud GET a la URL `http://127.0.0.1:8000/api/courses/`. Este endpoint de la API es de acceso público, por lo que no requiere ninguna autenticación.
5. Utilizas el método `json()` del objeto de respuesta para decodificar los datos JSON devueltos por la API.
6. Almacenas el atributo `next` en la variable `url` para recuperar la siguiente página de resultados en la instrucción `while`.
7. Añades el atributo `title` de cada curso a la lista `available_courses`.
8. Cuando la variable `url` es `None`, vas a la última página de resultados y no recuperas ninguna página adicional.
9. Imprimes la lista de cursos disponibles.

Inicia el servidor de desarrollo desde el directorio del proyecto `educa` con el siguiente comando:

```bash
python manage.py runserver
```

En otra consola, ejecuta el siguiente comando desde el directorio `api_examples/`:

```bash
python enroll_all.py
```

Verás una salida con una lista de todos los títulos de cursos, como esta:

```text
Available courses: Introduction to Django, Python for beginners, Algebra basics
```

Esta es la primera llamada automatizada a tu API.

Edita el archivo `enroll_all.py` y añade las siguientes líneas:

```python
import requests

username = ''
password = ''
base_url = 'http://127.0.0.1:8000/api/'
url = f'{base_url}courses/'
available_courses = []

while url is not None:
    print(f'Loading courses from {url}')
    r = requests.get(url)
    response = r.json()
    url = response['next']
    courses = response['results']
    available_courses += [course['title'] for course in courses]

print(f'Available courses: {", ".join(available_courses)}')

for course in courses:
    course_id = course['id']
    course_title = course['title']
    r = requests.post(
        f'{base_url}courses/{course_id}/enroll/',
        auth=(username, password)
    )
    if r.status_code == 200:
        # successful request
        print(f'Successfully enrolled in {course_title}')
```

Reemplaza los valores de las variables `username` y `password` con las credenciales de un usuario existente o carga los valores desde variables de entorno. Puedes usar `python-decouple`, como hicimos en la sección *Trabajar con variables de entorno* en el Capítulo 2, Mejorar tu blog con funciones avanzadas, para cargar credenciales desde variables de entorno.

Con el nuevo código, realizas las siguientes acciones:

1. Defines el nombre de usuario y la contraseña del estudiante que deseas inscribir en los cursos.
2. Iteras sobre los cursos disponibles recuperados de la API.
3. Almacenas el atributo ID del curso en la variable `course_id` y el atributo del título en la variable `course_title`.
4. Utilizas `requests.post()` para enviar una solicitud POST a la URL `http://127.0.0.1:8000/api/courses/[id]/enroll/` para cada curso. Esta URL corresponde a la vista de API `CourseEnrollView`, que te permite inscribir a un usuario en un curso. Construyes la URL para cada curso utilizando la variable `course_id`. La vista `CourseEnrollView` requiere autenticación. Utiliza el permiso `IsAuthenticated` y la clase de autenticación `BasicAuthentication`. La biblioteca Requests admite la autenticación básica HTTP de forma nativa. Utilizas el parámetro `auth` para pasar una tupla con el nombre de usuario y la contraseña para autenticar al usuario utilizando la autenticación básica HTTP.
5. Si el código de estado de la respuesta es 200 OK, imprimes un mensaje para indicar que el usuario se ha inscrito correctamente en el curso.

Puedes utilizar diferentes tipos de autenticación con Requests. Puedes encontrar más información sobre la autenticación con Requests en [https://requests.readthedocs.io/en/master/user/authentication/](https://requests.readthedocs.io/en/master/user/authentication/).

Ejecuta el siguiente comando desde el directorio `api_examples/`:

```bash
python enroll_all.py
```

Ahora verás una salida como esta:

```text
Available courses: Introduction to Django, Python for beginners, Algebra basics
Successfully enrolled in Introduction to Django
Successfully enrolled in Python for beginners
Successfully enrolled in Algebra basics
```

¡Excelente! Has inscrito al usuario con éxito en todos los cursos disponibles mediante la API. Verás un mensaje `Successfully enrolled` para cada curso en la plataforma. Como puedes ver, es muy fácil consumir la API desde cualquier otra aplicación.

---

### Resumen

En este capítulo, aprendiste a usar DRF para crear una API RESTful para tu proyecto. Creaste serializadores y vistas para modelos, y creaste vistas de API personalizadas. También añadiste autenticación a tu API y restringiste el acceso a las vistas de la API mediante permisos. A continuación, descubriste cómo crear permisos personalizados e implementaste ViewSets y routers. Finalmente, utilizaste la biblioteca Requests para consumir la API desde un script de Python externo.

El próximo capítulo te enseñará cómo crear un servidor de chat utilizando Django Channels. Implementarás comunicación asíncrona mediante WebSockets y utilizarás Redis para configurar una capa de canales (*channel layer*).

---

### Recursos adicionales

Los siguientes recursos proporcionan información adicional relacionada con los temas tratados en este capítulo:

- **Código fuente para este capítulo:** [https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter15](https://github.com/PacktPublishing/Django-5-by-example/tree/main/Chapter15)
- **Sitio web de REST framework:** [https://www.django-rest-framework.org/](https://www.django-rest-framework.org/)
- **Ajustes de REST framework:** [https://www.django-rest-framework.org/api-guide/settings/](https://www.django-rest-framework.org/api-guide/settings/)
- **Serializadores de REST framework:** [https://www.django-rest-framework.org/api-guide/serializers/](https://www.django-rest-framework.org/api-guide/serializers/)
- **Renderers de REST framework:** [https://www.django-rest-framework.org/api-guide/renderers/](https://www.django-rest-framework.org/api-guide/renderers/)
- **Parsers de REST framework:** [https://www.django-rest-framework.org/api-guide/parsers/](https://www.django-rest-framework.org/api-guide/parsers/)
- **Mixins y vistas genéricas de REST framework:** [https://www.django-rest-framework.org/api-guide/generic-views/](https://www.django-rest-framework.org/api-guide/generic-views/)
- **Descargar curl:** [https://curl.se/download.html](https://curl.se/download.html)
- **Plataforma de API Postman:** [https://www.getpostman.com/](https://www.getpostman.com/)
- **Paginación de REST framework:** [https://www.django-rest-framework.org/api-guide/pagination/](https://www.django-rest-framework.org/api-guide/pagination/)
- **Relaciones de serializadores en REST framework:** [https://www.django-rest-framework.org/api-guide/relations/](https://www.django-rest-framework.org/api-guide/relations/)
- **Autenticación básica HTTP:** [https://en.wikipedia.org/wiki/Basic_access_authentication](https://en.wikipedia.org/wiki/Basic_access_authentication)
- **Autenticación en REST framework:** [https://www.django-rest-framework.org/api-guide/authentication/](https://www.django-rest-framework.org/api-guide/authentication/)
- **Permisos en REST framework:** [https://www.django-rest-framework.org/api-guide/permissions/](https://www.django-rest-framework.org/api-guide/permissions/)
- **ViewSets en REST framework:** [https://www.django-rest-framework.org/api-guide/viewsets/](https://www.django-rest-framework.org/api-guide/viewsets/)
- **Routers en REST framework:** [https://www.django-rest-framework.org/api-guide/routers/](https://www.django-rest-framework.org/api-guide/routers/)
- **Documentación de la biblioteca Python Requests:** [https://requests.readthedocs.io/en/master/](https://requests.readthedocs.io/en/master/)
- **Autenticación con la biblioteca Requests:** [https://requests.readthedocs.io/en/master/user/authentication/](https://requests.readthedocs.io/en/master/user/authentication/)
