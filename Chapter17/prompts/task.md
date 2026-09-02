**Contexto:** He desarrollado una plataforma de aprendizaje electrónico con Django que organiza el contenido del curso en diferentes módulos.

**Objetivo:** Deseo permitir que los alumnos reanuden un curso exactamente en el módulo donde lo dejaron la última vez. Para lograrlo, planeo utilizar Redis para almacenar el último módulo al que accedió un estudiante en un curso.

**Detalles de implementación:**
Deseo utilizar el siguiente código para establecer una conexión con Redis utilizando la librería de Python `redis`, y quiero seguir las convenciones de nombres de Redis para las claves.
```
import redis
from django.conf import settings

# Configuración de la conexión a Redis
r = redis.Redis(
    host=settings.REDIS_HOST,
    port=settings.REDIS_PORT,
    db=settings.REDIS_DB,
)
```

**Aquí está parte de mi configuración actual:**

Definición de los modelos `Course` y `Module` en `courses/models.py`:
```
class Course(models.Model):
    # ...

    class Meta:
        ordering = ['-created']

    def __str__(self):
        return self.title


class Module(models.Model):
    course = models.ForeignKey(
        Course, related_name='modules', on_delete=models.CASCADE
    )
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    order = OrderField(blank=True, for_fields=['course'])

    class Meta:
        ordering = ['order']

    def __str__(self):
        return f'{self.order}. {self.title}'
```

Vista para que los estudiantes accedan a los contenidos del módulo del curso en `students/views.py`:
```
class StudentCourseDetailView(LoginRequiredMixin, DetailView):
    model = Course
    template_name = 'students/course/detail.html'

    def get_queryset(self):
        qs = super().get_queryset()
        return qs.filter(students__in=[self.request.user])

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        course = self.get_object()
        if 'module_id' in self.kwargs:
            context['module'] = course.modules.get(
                id=self.kwargs['module_id']
            )
        else:
            context['module'] = course.modules.all()[0]
        return context
```
