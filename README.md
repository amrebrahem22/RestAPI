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
 *and in models.py*
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
> Now In models.py I created my model and added the  method for the image location  and created the class for the model manager that will control our queryset class

### Model Form Validation
*now i created a file call forms.py in the status app*
``` python
from django import forms

from .models import Status

class StatusForm(forms.ModelForm):
    class Meta:
        model = Status
        fields = [
            'user',
            'content',
            'image'
        ]

    def clean_content(self, *args, **kwargs):
        content = self.cleaned_data.get('content')
        if len(content) > 240:
            raise forms.ValidationError("Content is too long")
        return content

    def clean(self, *args, **kwargs):
        data = self.cleaned_data
        content = data.get('content', None)
        if content == "":
            content = None
        image = data.get("image", None)
        if content is None and image is None:
            raise forms.ValidationError('Content or image is required.')
        return super().clean(*args, **kwargs)
 ```
 *i cleaned the fields and the content field and this form will use it in the admin*
 *so in admin.py*
 ``` python
 from django.contrib import admin

from .forms import StatusForm
from .models import Status


class StatusAdmin(admin.ModelAdmin):
    list_display = ['user', '__str__', 'image']
    form = StatusForm
    
    # another way
    # class Meta:
    #     model = Status


admin.site.register(Status, StatusAdmin)
```
### Creating a Serializer
*now in the **status** app i created a new folder call **api** and inside this folder i created two files:*
`1. __init__.py`
`1. serializers.py`
*and in serializers.py*
``` python
from rest_framework import serializers
from status.models import Status

# Serializers -> JSON
# Serializers -> validate data

class StatusSerializer(serializers.ModelSerializer):
    class Meta:
        model = Status 
        fields =[
            'user',
            'content',
            'image'
        ]
```
*and if you notice the serialzer class is the same in forms.py, and itw ill convert it to json data*




