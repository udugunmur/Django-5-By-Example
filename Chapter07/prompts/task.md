**Contexto:** He desarrollado una aplicación de Django llamada account, que incluye un modelo Profile. Este modelo extiende el modelo User de autenticación predeterminado de Django.

**Objetivo:** Mi meta es utilizar signals de Django para crear automáticamente un objeto `Profile` asociado cada vez que se crea un objeto `User`.

**Aquí está parte de mi configuración actual:**

Definición del modelo `Profile` en `account/models.py`:
```
class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE
    )
    date_of_birth = models.DateField(blank=True, null=True)
    photo = models.ImageField(
        upload_to='users/%Y/%m/%d/',
        blank=True
    )

    def __str__(self):
        return f'Profile of {self.user.username}'
```
