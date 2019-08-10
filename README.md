# Rest Freamwork
> that's the guide for using the rest freamwork in Django

### Requirements
*REST framework requires the following:*
  1. Python (3.5, 3.6, 3.7)
  1. Django (1.11, 2.0, 2.1, 2.2)
  
### Installing

*Install using pip, including any optional packages you want...*
```
pip install djangorestframework
pip install markdown       # Markdown support for the browsable API.
pip install django-filter  # Filtering support  
```
*Add 'rest_framework' to your INSTALLED_APPS setting.*
``` python
INSTALLED_APPS = [
    ...
    'rest_framework',
]
```
#### Status Model & App
`python manage.py startapp status`
> and in models.py
``` python
from django.conf import settings
from django.db import models


def upload_status_image(instance, filename):
    return "updates/{user}/{filename}".format(user=instance.user, filename=filename)


class StatusQuerySet(models.QuerySet):
    pass


class StatusManager(models.Manager):
    def get_queryset(self):
        return StatusQuerySet(self.model, using=self._db)


class Status(models.Model): # fb status, instagram post, tweet, linkedin post
    user        = models.ForeignKey(settings.AUTH_USER_MODEL)
    content     = models.TextField(null=True, blank=True)
    image       = models.ImageField(upload_to=upload_status_image, null=True, blank=True)
    updated     = models.DateTimeField(auto_now=True)
    timestamp   = models.DateTimeField(auto_now_add=True)

    objects = StatusManager()

    def __str__(self):
        return str(self.content)[:50]
```
