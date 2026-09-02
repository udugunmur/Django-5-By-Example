**Contexto:** Estoy creando un blog con Django junto con la librería `django-taggit` para etiquetar las entradas de mi blog.

**Objetivo:** Quiero extender la funcionalidad del sitemap para incluir URLs para cada etiqueta utilizada en el blog, listando de forma efectiva las publicaciones filtradas por dichas etiquetas en el `sitemap.xml`.

**Esto es lo que he realizado hasta ahora:**
- Tengo configurado un sitemap para las publicaciones utilizando el framework de sitemaps de Django.
- He creado patrones de URL que permiten visualizar las publicaciones filtradas por etiquetas.
- Quiero añadir estas vistas filtradas por etiquetas a mi sitemap pero no estoy seguro de cómo hacerlo.

**Aquí está parte de mi configuración actual:**

Para el sitemap de publicaciones: `blog/sitemaps.py`
```
from django.contrib.sitemaps import Sitemap
from .models import Post


class PostSitemap(Sitemap):
    changefreq = 'weekly'
    priority = 0.9

    def items(self):
        return Post.published.all()

    def lastmod(self, obj):
        return obj.updated
```

El patrón de URL para listar publicaciones por etiqueta: `blog/urls.py`
```
urlpatterns = [
    # ...
    path('tag/<slug:tag_slug>/', views.post_list, name='post_list_by_tag'),
]
```

Y la configuración de URLs del sitio principal: `mysite/urls.py`
```
from django.contrib import admin
from django.contrib.sitemaps.views import sitemap
from django.urls import include, path
from blog.sitemaps import PostSitemap

sitemaps = {
    'posts': PostSitemap,
}

urlpatterns = [
    path('admin/', admin.site.urls),
    path('blog/', include('blog.urls', namespace='blog')),
    path(
        'sitemap.xml',
        sitemap,
        {'sitemaps': sitemaps},
        name='django.contrib.sitemaps.views.sitemap',
    ),
]
```

Por favor, explica las modificaciones necesarias en el código para añadir las páginas de etiquetas al sitemap.
