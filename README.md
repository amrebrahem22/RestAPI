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
`2. serializers.py`

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
*and if you notice the serializer class is the same in forms.py, and it will convert it to json data*
### Create & Update through Serializers
*before we start using the shell we should know how to serialize and deserialize data*
*Serializers allow complex data such as querysets and model instances to be converted to native Python datatypes that can then be easily rendered into JSON*
**Serializing objects**
*We can now use CommentSerializer to serialize a comment, or list of comments. Again, using the Serializer class looks a lot like using a Form class.*
``` python
serializer = CommentSerializer(comment)
serializer.data
# {'email': 'leila@example.com', 'content': 'foo bar', 'created': '2016-01-27T15:17:10.375877'}
```
*At this point we've translated the model instance into Python native datatypes. To finalise the serialization process we render the data into json.*
``` python
from rest_framework.renderers import JSONRenderer

json = JSONRenderer().render(serializer.data)
json
# b'{"email":"leila@example.com","content":"foo bar","created":"2016-01-27T15:17:10.375877"}'
```
**Deserializing objects**
*Deserialization is similar. First we parse a stream into Python native datatypes...*
``` python
import io
from rest_framework.parsers import JSONParser

stream = io.BytesIO(json)
data = JSONParser().parse(stream)
```.
*..then we restore those native datatypes into a dictionary of validated data.*
``` python
serializer = CommentSerializer(data=data)
serializer.is_valid()
# True
serializer.validated_data
# {'content': 'foo bar', 'email': 'leila@example.com', 'created': datetime.datetime(2012, 08, 22, 16, 20, 09, 822243)}
```
*here i will use the shell to test some things*
`python manage.py shell`
*and in the shell*
``` python
from django.utils.six import BytesIO
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser

from status.api.serializers import StatusSerializer
from status.models import Status

# Serialize a single object

obj = Status.objects.first()
serializer = StatusSerializer(obj)
serializer.data
json_data = JSONRenderer().render(serializer.data)
print(json_data)

stream = BytesIO(json_data)
data = JSONParser().parse(stream)
print(data)
```
*so that's how to serialize a single object*
``` python
qs = Status.objects.all()
serializer2 = StatusSerializer(qs, many=True)
serializer2.data
json_data2 = JSONRenderer().render(serializer2.data)
print(json_data)

stream2 = BytesIO(json_data2)
data2 = JSONParser().parse(stream2)
print(data2)
```
*and that's how to serialize a queryset*
``` python
data = {'user': 1}
serializer = StatusSerializer(data=data)
serializer.is_valid()
serializer.save()

# if serializer.is_valid():
#     serializer.save()
```
*and here how to create an object*
``` python
obj = Status.objects.first()
data = {'content': 'some new content', "user": 1}
update_serializer = StatusSerializer(obj, data=data)
update_serializer.is_valid()
update_serializer.save()
```
*to update object*
``` python
data = {'user': 1, 'content': "please delete me"}
create_obj_serializer = StatusSerializer(data=data)
create_obj_serializer.is_valid()
create_obj = create_obj_serializer.save() # instance of the object
print(create_obj)

#data = {'id': 9}
obj = Status.objects.last()
get_data_serializer = StatusSerializer(obj)
# update_serializer.is_valid()
# update_serializer.save()
print(get_data_serializer.data)
```
*to delete object*
### Validation & Fields
*now in serializers.py i created the validate method*
``` python
class StatusSerializer(serializers.ModelSerializer):
    class Meta:
        model = Status 
        fields =[
            'user',
            'content',
            'image'
        ]

    # def validate_content(self, value):
    #     if len(value) > 10000:
    #         raise serializers.ValidationError("This is wayy too long.")
    #     return value

    def validate(self, data):
        content = data.get("content", None)
        if content == "":
            content = None
        image = data.get("image", None)
        if content is None and image is None:
            raise serializers.ValidationError("Content or image is required.")
        return data
 ```
 > and there's another way to serialize by creating serialized fields but we will not use it now it's just to know you can try it in the shell
 ``` python
 from rest_framework import serializers
class CustomSerializer(serializers.Serializer):
    content =      serializers.CharField()
    email       =  serializers.EmailField()


data = {'email': 'hello@teamcfe.com', 'content': "please delete me"}
create_obj_serializer = CustomSerializer(data=data)
if create_obj_serializer.is_valid():
    valid_data = create_obj_serializer.data
    print(valid_data)
  ```
  ### API Endpoints Overview
  *in **api** folder i created two files*
  
  1. views.py
  1. urls.py
  
  *and this will handle our CRUD and in urls.py*
  ``` python
  from django.conf.urls import url


urlpatterns = [
    url(r'^$', StatusListSearchAPIView.as_view()),
    url(r'^create/$', StatusCreateAPIView.as_view()),
    url(r'^(?P<id>.*)/$', StatusDetailAPIView.as_view()),
    url(r'^(?P<id>.*)/update/$', StatusUpdateAPIView.as_view()),
    url(r'^(?P<id>.*)/delete/$', StatusDeleteAPIView.as_view()),
    
]
```
*now that's how our urls will look like*
**We will Start with this urls**
```
# /api/status/ -> List
# /api/status/create -> Create
# /api/status/12/ -> Detail
# /api/status/12/update/ -> Update
# /api/status/12/delete/ -> Delete
```

**then End With this urls two urls will handle the CRUD**
```
# /api/status/ -> List -> CRUD
# /api/status/1/ -> Detail -> CRUD
```

**Finally we will end with one urls that will handle the CRUD**
```
# /api/status/ -> CRUD & LS
  ```
### List & Search API View



