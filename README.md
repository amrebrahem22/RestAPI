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
``` python
from rest_framework import generics
# from rest_framework.generics import List
from rest_framework.views import APIView
from rest_framework.response import Response
# from django.views.generic import View

from status.models import Status
from .serializers import StatusSerializer

class StatusListSearchAPIView(APIView):
    permission_classes          = []
    authentication_classes      = []

    def get(self, request, format=None):
        qs = Status.objects.all()
        serializer = StatusSerializer(qs, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        qs = Status.objects.all()
        serializer = StatusSerializer(qs, many=True)
        return Response(serializer.data)
```
*Now in api/views.py I imported the API view and the response then created a class based on the rest freamwork view api and we should define this two variables then define which method we will use here I defined the get method*
``` python
url(r'^api/updates/', include('updates.api.urls')), # api/updates/ --> list api/updates/1/ -->detail
```
*and in the main urls.py i included my urls*
*And if we didn't specify `serializer = StatusSerializer(qs, many=True)` then when we go to the urls it will give us error because it’s not serialized so we will call the serialize class and print it’s data*
*And when we go to the url it will print the data*
``` python
class StatusAPIView(generics.ListAPIView):
    permission_classes          = []
    authentication_classes      = []
    serializer_class            = StatusSerializer

    def get_queryset(self):
        qs = Status.objects.all()
        query = self.request.GET.get('q')
        if query is not None:
            qs = qs.filter(content__icontains=query)
        return qs
  ```
  *then try to do it with another class that will inherit from list api view and we also create the method we will use to make our query and it's required or you will get error and created the serializer class*
  ``` python
from .views import StatusAPIView

urlpatterns = [
    url(r'^$', StatusAPIView.as_view()),
    # url(r'^create/$', StatusCreateAPIView.as_view()),
    # url(r'^(?P<id>.*)/$', StatusDetailAPIView.as_view()),
    # url(r'^(?P<id>.*)/update/$', StatusUpdateAPIView.as_view()),
    # url(r'^(?P<id>.*)/delete/$', StatusDeleteAPIView.as_view()),
    
]
```
*and n api/urls.py*
*Now we have the view if you go to the url `api/updates/`*
### Create API View
``` python
class StatusCreateAPIView(generics.CreateAPIView):
    permission_classes          = []
    authentication_classes      = []
    queryset                    = Status.objects.all()
    serializer_class            = StatusSerializer

    # def perform_create(self, serializer):
    #     serializer.save(user=self.request.user)
```
*Now I created the api create view*
``` python
from .views import StatusAPIView, StatusCreateAPIView

urlpatterns = [
    url(r'^$', StatusAPIView.as_view()),
    url(r'^create/$', StatusCreateAPIView.as_view()),    
]
```
*And defined the url*
### Detail API View
``` python
class StatusDetailAPIView(generics.RetrieveAPIView):
    permission_classes          = []
    authentication_classes      = []
    queryset                    = Status.objects.all()
    serializer_class            = StatusSerializer
    #lookup_field                = 'id' # 'slug'

    # def get_object(self, *args, **kwargs):
    #     kwargs = self.kwargs
    #     kw_id = kwargs.get('abc')
    #     return Status.objects.get(id=kw_id)
```
*now i created the detail view that will inherit from `generics.RetrieveAPIView` now you should the pk in the url and if you passed the id in `urls.py` it will give you error so to use the id you should define `lookup_field` and you can use it to if you use the slug field or this method `get_object` but i commented it and will use the pk*
``` python
from .views import (
    StatusAPIView, 
    StatusCreateAPIView,
    StatusDetailAPIView,
    )

urlpatterns = [
    url(r'^$', StatusAPIView.as_view()),
    url(r'^create/$', StatusCreateAPIView.as_view()),
    url(r'^(?P<pk>.*)/$', StatusDetailAPIView.as_view()),
]
```
*And here in the urls I use the primary key*
### Update & Delete API Views
``` python
class StatusUpdateAPIView(generics.UpdateAPIView):
    permission_classes          = []
    authentication_classes      = []
    queryset                    = Status.objects.all()
    serializer_class            = StatusSerializer

class StatusDeleteAPIView(generics.DestroyAPIView):
    permission_classes          = []
    authentication_classes      = []
    queryset                    = Status.objects.all()
    serializer_class            = StatusSerializer
```
*now i created the delete and update view*
``` python
from .views import (
    StatusAPIView, 
    StatusCreateAPIView,
    StatusDetailAPIView,
    StatusUpdateAPIView,
    StatusDeleteAPIView
    )

urlpatterns = [
    url(r'^$', StatusAPIView.as_view()),
    url(r'^create/$', StatusCreateAPIView.as_view()),
    url(r'^(?P<pk>\d+)/$', StatusDetailAPIView.as_view()),
    url(r'^(?P<pk>\d+)/update/$', StatusUpdateAPIView.as_view()),
    url(r'^(?P<pk>\d+)/delete/$', StatusDeleteAPIView.as_view()),   
]
```
*and you can test it by going to `http://127.0.0.1:8000/status/api/1/update/` or `http://127.0.0.1:8000/status/api/1/delete/`*
### Mixins to Power Http Methods
`from rest_framework import generics, mixins`
*First import the mixin*
``` python
# CreateModelMixin --- POST method
# UpdateModelMixin --- PUT method
# DestroyModelMixin -- DELETE method

class StatusAPIView(mixins.CreateModelMixin, generics.ListAPIView): # Create List
    permission_classes          = []
    authentication_classes      = []
    serializer_class            = StatusSerializer

    def get_queryset(self):
        qs = Status.objects.all()
        query = self.request.GET.get('q')
        if query is not None:
            qs = qs.filter(content__icontains=query)
        return qs

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)

    # def perform_create(self, serializer):
    #     serializer.save(user=self.request.user)
```
*Then add the create model mixin to our class and create the post method that will create our object, now you can create and list objects just by this class*
``` python
class StatusDetailAPIView(mixins.DestroyModelMixin, mixins.UpdateModelMixin, generics.RetrieveAPIView):
    permission_classes          = []
    authentication_classes      = []
    queryset                    = Status.objects.all()
    serializer_class            = StatusSerializer
    
    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```
*And in the detail view I added the delete and update mixin and defined this two methods put and delete*
``` python
from .views import (
    StatusAPIView, 
    StatusDetailAPIView,
    )

urlpatterns = [
    url(r'^$', StatusAPIView.as_view()),
    url(r'^(?P<pk>\d+)/$', StatusDetailAPIView.as_view()),    
]
```
*and `urls.py` And I removed this all urls and just keep the api view and the detail view*
### One API Endpoint for CRUDL
``` python
class StatusAPIView(
    mixins.CreateModelMixin, 
    mixins.RetrieveModelMixin,
    mixins.UpdateModelMixin,
    mixins.DestroyModelMixin,
    generics.ListAPIView): 
    permission_classes          = []
    authentication_classes      = []
    serializer_class            = StatusSerializer

    def get_queryset(self):
        request = self.request
        qs = Status.objects.all()
        query = request.GET.get('q')
        if query is not None:
            qs = qs.filter(content__icontains=query)
        return qs

    def get_object(self):
        request         = self.request
        passed_id       = request.GET.get('id', None)
        queryset        = self.get_queryset()
        obj = None
        if passed_id is not None:
            obj = get_object_or_404(queryset, id=passed_id)
            self.check_object_permissions(request, obj)
        return obj

    def get(self, request, *args, **kwargs):
        passed_id       = request.GET.get('id', None)
        #request.body
        #request.data
        if passed_id is not None:
            return self.retrieve(request, *args, **kwargs)
        return super().get(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def patch(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```
*Now I will create just one class to handle all our methods Then I copied the get_queryset method and the get_object and modified it to get the id from the request and check if it’s passed and if true then get this object and use the check_objects_permissions method built in with rest freamwork that will check the permission And define the get method to retrieve it with that id* </br>
*and if you go to the url you will find all methods appear*
``` python
def get(self, request, *args, **kwargs):
        url_passed_id    = request.GET.get('id', None)
        json_data        = {}
        body_            = request.body
        if is_json(body_):
            json_data        = json.loads(request.body)
        new_passed_id    = json_data.get('id', None)
        #print(request.body)
        #request.data
        passed_id = url_passed_id or new_passed_id or None
        self.passed_id = passed_id
        if passed_id is not None:# or passed_id is not "":
            return self.retrieve(request, *args, **kwargs)
        return super().get(request, *args, **kwargs)
```
*and in the get method I defined the passed id variable that will display the json data or the normal id*
``` python
from .views import StatusAPIView

urlpatterns = [
    url(r'^$', StatusAPIView.as_view()),
 ]
```
*now we have one url*
``` python
import json
import requests

ENDPOINT = "http://127.0.0.1:8000/api/status/"

def do(method='get', data={}, is_json=True):
    headers = {}
    if is_json:
        headers['content-type'] = 'application/json'
        data = json.dumps(data)
    r = requests.request(method, ENDPOINT, data=data, headers=headers)
    print(r.text)
    print(r.status_code)
    return r

do(data={'id': 500})
```
*in `cfe_rest_framework_api.py` Then I created a folder out of our project to test it and in this file I imported the requests library and installed it then created a method call do and when i run it in the terminal it will print the object and I will use the is_json to convert it to string*

``` python
def is_json(json_data):
    try:
        real_json = json.loads(json_data)
        is_valid = True
    except ValueError:
        is_valid = False
    return is_valid
```
*Then I created this method that will check if it json or not*
``` python
class StatusAPIView(
    mixins.CreateModelMixin, 
    mixins.RetrieveModelMixin,
    mixins.UpdateModelMixin,
    mixins.DestroyModelMixin,
    generics.ListAPIView): 
    permission_classes          = []
    authentication_classes      = []
    serializer_class            = StatusSerializer
    passed_id                   = None
```
*And modified the passed id variable And assign the passed id to my id in the get method*
``` python
def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        url_passed_id    = request.GET.get('id', None)
        json_data        = {}
        body_            = request.body
        if is_json(body_):
            json_data        = json.loads(request.body)
        new_passed_id    = json_data.get('id', None)
        #print(request.body)
        #request.data
        passed_id = url_passed_id or new_passed_id or None
        self.passed_id = passed_id
        return self.update(request, *args, **kwargs)

    def patch(self, request, *args, **kwargs):
        url_passed_id    = request.GET.get('id', None)
        json_data        = {}
        body_            = request.body
        if is_json(body_):
            json_data        = json.loads(request.body)
        new_passed_id    = json_data.get('id', None)
        #print(request.body)
        #request.data
        passed_id = url_passed_id or new_passed_id or None
        self.passed_id = passed_id
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        url_passed_id    = request.GET.get('id', None)
        json_data        = {}
        body_            = request.body
        if is_json(body_):
            json_data        = json.loads(request.body)
        new_passed_id    = json_data.get('id', None)
```
*And add this code to the other methods But I didn’t add it to the post*
``` python
do(data={'id': 500}) # retrieve

do(method='delete', data={'id': 13}) # delete

do(method='put', data={'id': 13, "content": "some cool new content", 'user': 1}) # update

do(method='post', data={"content": "some cool new content", 'user': 1}) # create
```
*And I will try with all other methods It’s will work but the delete not work yet* </br>
*In order to make the delete method work we should define this method we will create this method*
``` python
def perform_destroy(self, instance):
        if instance is not None:
            return instance.delete()
        return None
```
####                And Here's the Whole Code 
``` python
import json
from rest_framework import generics, mixins
from rest_framework.views import APIView
from rest_framework.response import Response
from django.shortcuts import get_object_or_404

from status.models import Status
from .serializers import StatusSerializer

# # CreateModelMixin --- POST method
# # UpdateModelMixin --- PUT method
# # DestroyModelMixin -- DELETE method
def is_json(json_data):
    try:
        real_json = json.loads(json_data)
        is_valid = True
    except ValueError:
        is_valid = False
    return is_valid


class StatusAPIView(
    mixins.CreateModelMixin, 
    mixins.RetrieveModelMixin,
    mixins.UpdateModelMixin,
    mixins.DestroyModelMixin,
    generics.ListAPIView): 
    permission_classes          = []
    authentication_classes      = []
    serializer_class            = StatusSerializer
    passed_id                   = None

    def get_queryset(self):
        request = self.request
        qs = Status.objects.all()
        query = request.GET.get('q')
        if query is not None:
            qs = qs.filter(content__icontains=query)
        return qs

    def get_object(self):
        request         = self.request
        passed_id       = request.GET.get('id', None) or self.passed_id
        queryset        = self.get_queryset()
        obj = None
        if passed_id is not None:
            obj = get_object_or_404(queryset, id=passed_id)
            self.check_object_permissions(request, obj)
        return obj

    def perform_destroy(self, instance):
        if instance is not None:
            return instance.delete()
        return None

    def get(self, request, *args, **kwargs):
        url_passed_id    = request.GET.get('id', None)
        json_data        = {}
        body_            = request.body
        if is_json(body_):
            json_data        = json.loads(request.body)
        new_passed_id    = json_data.get('id', None)
        #print(request.body)
        #request.data
        passed_id = url_passed_id or new_passed_id or None
        self.passed_id = passed_id
        if passed_id is not None:# or passed_id is not "":
            return self.retrieve(request, *args, **kwargs)
        return super().get(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        url_passed_id    = request.GET.get('id', None)
        json_data        = {}
        body_            = request.body
        if is_json(body_):
            json_data        = json.loads(request.body)
        new_passed_id    = json_data.get('id', None)
        #print(request.body)
        #request.data
        passed_id = url_passed_id or new_passed_id or None
        self.passed_id = passed_id
        return self.update(request, *args, **kwargs)

    def patch(self, request, *args, **kwargs):
        url_passed_id    = request.GET.get('id', None)
        json_data        = {}
        body_            = request.body
        if is_json(body_):
            json_data        = json.loads(request.body)
        new_passed_id    = json_data.get('id', None)
        #print(request.body)
        #request.data
        passed_id = url_passed_id or new_passed_id or None
        self.passed_id = passed_id
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        url_passed_id    = request.GET.get('id', None)
        json_data        = {}
        body_            = request.body
        if is_json(body_):
            json_data        = json.loads(request.body)
        new_passed_id    = json_data.get('id', None)
        #print(request.body)
        #request.data
        passed_id = url_passed_id or new_passed_id or None
        self.passed_id = passed_id
        return self.destroy(request, *args, **kwargs)
```
### Uploading & Handling Images
``` python
def upload_status_image(instance, filename):
    return "status/{user}/{filename}".format(user=instance.user, filename=filename)
```
*In models.py I will upload my image to this path*
``` python
MEDIA_ROOT = os.path.join(os.path.dirname(BASE_DIR), 'static-server', 'media-root') # '/Users/cfe/dev/restapi/'
MEDIA_URL = '/media/'
```
*And add the path in settings.py to be in our virtual environment not in the project*
``` python
import json
import requests
import os

ENDPOINT = "http://127.0.0.1:8000/api/status/"

image_path = os.path.join(os.getcwd(), "logo.jpg")

def do_img(method='get', data={}, is_json=True, img_path=None):
    headers = {}
    if is_json:
        headers['content-type'] = 'application/json'
        data = json.dumps(data)
    if img_path is not None:
        with open(image_path, 'rb') as image:
            file_data = {
                'image': image
            }
            r = requests.request(method, ENDPOINT, data=data, files=file_data, headers=headers)
    else:
        r = requests.request(method, ENDPOINT, data=data, headers=headers)
    print(r.text)
    print(r.status_code)
    return r

do_img(
    method='post', 
    data={'user': 1, "content": ""}, 
    is_json=False, 
    img_path=image_path
    )
```
*Then in my test file I will get the image path in my folder and then will create the method for the image then create if statement that will open this image as binary file then create a dictionary with this image and make a request to this url with this data and add the files argument to have our image dictionary and add the headers else we will make our normal request then call our method with the post method so that’s mean we will create and add the data and make the is_json false because we will not use json and add the image path Now everytime I call this file it will create a new object with this image*
### 2 Views for CRUDL
*in views.py*
``` python
 def put(self, request, *args, **kwargs):
    url_passed_id    = request.GET.get('id', None)
    json_data        = {}
    body_            = request.body
    if is_json(body_):
        json_data        = json.loads(request.body)
    new_passed_id    = json_data.get('id', None)
    #print(request.body)
    #request.data
    print(request.data)
    requested_id = None #request.data.get('id')
    passed_id = url_passed_id or new_passed_id or requested_id or None
    self.passed_id = passed_id
    return self.update(request, *args, **kwargs)
```
*The reason for doesn’t update is the id is not set so now we try to print the request data, And it will return the whole object, So now we get the id from the data*
``` python
class StatusAPIDetailView(
    mixins.UpdateModelMixin,
    mixins.DestroyModelMixin, 
    generics.RetrieveAPIView):
    permission_classes          = []
    authentication_classes      = []
    serializer_class            = StatusSerializer
    queryset                    = Status.objects.all()
    lookup_field                = 'id'

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def patch(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```
*Then I created the detail api view that will have the update and delete method* </br>
*then in urls.py*
``` python
from .views import (
    StatusAPIView, 
    StatusAPIDetailView,
    )

urlpatterns = [
    url(r'^$', StatusAPIView.as_view()),
    url(r'^(?P<id>\d+)/$', StatusAPIDetailView.as_view()),
]
```
*in views.py*
``` python
class StatusAPIView(
    mixins.CreateModelMixin, 
    mixins.UpdateModelMixin,
    mixins.DestroyModelMixin,
    generics.ListAPIView): 
    permission_classes          = []
    authentication_classes      = []
    serializer_class            = StatusSerializer
    passed_id                   = None

    def get_queryset(self):
        request = self.request
        qs = Status.objects.all()
        query = request.GET.get('q')
        if query is not None:
            qs = qs.filter(content__icontains=query)
        return qs
    
    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```
*Then created the list api view that will return the list and the create method*
### Authentication & Permissions
*in views.py*
``` python
import json
from rest_framework import generics, mixins, permissions
from rest_framework.authentication import SessionAuthentication
from rest_framework.views import APIView
from rest_framework.response import Response
from django.shortcuts import get_object_or_404
from status.models import Status
from .serializers import StatusSerializer
```
``` python
class StatusAPIView(
    mixins.CreateModelMixin, 
    generics.ListAPIView): 
    permission_classes          = []
    authentication_classes      = [SessionAuthentication] #Oauth, JWT
    serializer_class            = StatusSerializer
    passed_id                   = None
```
*Then added it in our variable, And now if we print request.user it will print it*
``` python
def perform_create(self, serializer):
    serializer.save(user=self.request.user)
```
*Then I will activate the perform create method*
``` python
class StatusSerializer(serializers.ModelSerializer):
    class Meta:
        model = Status 
        fields =[
            'id', # ?
            'user',
            'content',
            'image'
        ]
        read_only_fields = ['user'] # GET  #readonly_fields
```
*And in our serializer I made the user read only*
``` python
permission_classes          = [permissions.IsAuthenticated]
```
*And this permission will make sure if is authenticated or not and will display or api if authenticated*
``` python
permission_classes          = [permissions.IsAuthenticatedOrReadOnly]
```
*And this will display or api if authenticated and read only data if not*

### Global Settings for Authentication & Permissions
*And I created a new folder call `restconf` in our main app and created two files one call `__init__.py` and the second `main.py` and imported it in our settings*
``` python
from cfeapi.restconf.main import *
```
*in settings.py at the end of the file*
``` python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        #'rest_framework.authentication.BasicAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ),
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    )
}
```
*And in main.py I created the default authentication classes and default permission classes*
``` python
class StatusAPIView(
    mixins.CreateModelMixin, 
    generics.ListAPIView): 
    permission_classes          = [permissions.IsAuthenticatedOrReadOnly]
    serializer_class            = StatusSerializer
    passed_id                   = None
```
*then i removed the authentication variable*

### Permission Tests with Python Requests
``` python
import json
import requests
import os

ENDPOINT = "http://127.0.0.1:8000/api/status/"

image_path = os.path.join(os.getcwd(), "logo.jpg")

get_endpoint =  ENDPOINT + str(12)
post_data = json.dumps({"content": "Some random content"})

r = requests.get(get_endpoint)
print(r.text)

r2 = requests.get(ENDPOINT)
print(r2.status_code)

post_headers = {
    'content-type': 'application/json'
}

post_response = requests.post(ENDPOINT, data=post_data, headers=post_headers)
print(post_response.text)
```
*Now I will check my urls with get and post*

### Implement JWT Authentication
*Now I will use the json web token*
`pip install djangorestframework-jwt` *now i will install the jwt*
``` python
import datetime

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        #'rest_framework.authentication.BasicAuthentication',
        'rest_framework_jwt.authentication.JSONWebTokenAuthentication',
        'rest_framework.authentication.SessionAuthentication', #Oauth, JWT
    ),
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    )
}
```
*And added it to main.py*
``` python
from rest_framework_jwt.views import refresh_jwt_token, obtain_jwt_token # accounts app

urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^api/auth/jwt/$', obtain_jwt_token),
    url(r'^api/auth/jwt/refresh/$', refresh_jwt_token),
    url(r'^api/status/', include('status.api.urls')),
    url(r'^api/updates/', include('updates.api.urls')), 
]
```
*And imported it and created the url for that, And if you go to this url and create a new object it will give you token*
``` python
AUTH_ENDPOINT = "http://127.0.0.1:8000/api/auth/jwt/"
REFRESH_ENDPOINT = AUTH_ENDPOINT + "refresh/"
ENDPOINT = "http://127.0.0.1:8000/api/status/"

image_path = os.path.join(os.getcwd(), "logo.jpg")

headers = {
    "Content-Type": "application/json"
}

data = {
    'username': 'cfe',
    'password': 'learncode'
}

r = requests.post(AUTH_ENDPOINT, data=json.dumps(data), headers=headers)

print(token)
```
*now we want to print just the token, Here’s it and everytime I print this it will give me a new token*
``` python
JWT_AUTH = {
    'JWT_ENCODE_HANDLER':
    'rest_framework_jwt.utils.jwt_encode_handler',

    'JWT_DECODE_HANDLER':
    'rest_framework_jwt.utils.jwt_decode_handler',

    'JWT_PAYLOAD_HANDLER':
    'rest_framework_jwt.utils.jwt_payload_handler',

    'JWT_PAYLOAD_GET_USER_ID_HANDLER':
    'rest_framework_jwt.utils.jwt_get_user_id_from_payload_handler',

    'JWT_RESPONSE_PAYLOAD_HANDLER':
    'rest_framework_jwt.utils.jwt_response_payload_handler',

    'JWT_SECRET_KEY': settings.SECRET_KEY,
    'JWT_GET_USER_SECRET_KEY': None,
    'JWT_PUBLIC_KEY': None,
    'JWT_PRIVATE_KEY': None,
    'JWT_ALGORITHM': 'HS256',
    'JWT_VERIFY': True,
    'JWT_VERIFY_EXPIRATION': True,
    'JWT_LEEWAY': 0,
    'JWT_EXPIRATION_DELTA': datetime.timedelta(seconds=300),
    'JWT_AUDIENCE': None,
    'JWT_ISSUER': None,

    'JWT_ALLOW_REFRESH': False,
    'JWT_REFRESH_EXPIRATION_DELTA': datetime.timedelta(days=7),

    'JWT_AUTH_HEADER_PREFIX': 'JWT',
    'JWT_AUTH_COOKIE': None,

}
#This packages uses the JSON Web Token Python implementation, PyJWT and allows to modify some of it's available options.
```
*And added this code you will find it in the documentation*
``` python
JWT_AUTH = {
    'JWT_ENCODE_HANDLER':
    'rest_framework_jwt.utils.jwt_encode_handler',

    'JWT_DECODE_HANDLER':
    'rest_framework_jwt.utils.jwt_decode_handler',

    'JWT_PAYLOAD_HANDLER':
    'rest_framework_jwt.utils.jwt_payload_handler',

    'JWT_PAYLOAD_GET_USER_ID_HANDLER':
    'rest_framework_jwt.utils.jwt_get_user_id_from_payload_handler',

    'JWT_RESPONSE_PAYLOAD_HANDLER':
    'rest_framework_jwt.utils.jwt_response_payload_handler',

    'JWT_ALLOW_REFRESH': True,
    'JWT_REFRESH_EXPIRATION_DELTA': datetime.timedelta(days=7),

    'JWT_AUTH_HEADER_PREFIX': 'JWT',
    'JWT_AUTH_COOKIE': None,

}
```
*but now i deleted some of these.*
``` python
import json
import requests
import os


AUTH_ENDPOINT = "http://127.0.0.1:8000/api/auth/jwt/"
REFRESH_ENDPOINT = AUTH_ENDPOINT + "refresh/"
ENDPOINT = "http://127.0.0.1:8000/api/status/"

image_path = os.path.join(os.getcwd(), "logo.jpg")

headers = {
    "Content-Type": "application/json"
}

data = {
    'username': 'cfe',
    'password': 'learncode'
}

r = requests.post(AUTH_ENDPOINT, data=json.dumps(data), headers=headers)
token = r.json()['token']

#print(token)

refresh_data = {
    'token': token
}

new_response = requests.post(REFRESH_ENDPOINT, data=json.dumps(refresh_data), headers=headers)
new_token = new_response.json()#['token']

print(new_token)
```
*to use the refresh*
###  JWT Authorization Header
``` python
AUTH_ENDPOINT = "http://127.0.0.1:8000/api/auth/jwt/"
REFRESH_ENDPOINT = AUTH_ENDPOINT + "refresh/"
ENDPOINT = "http://127.0.0.1:8000/api/status/"

image_path = os.path.join(os.getcwd(), "logo.jpg")

headers = {
    "Content-Type": "application/json"
}

data = {
    'username': 'cfe',
    'password': 'learncode'
}

r = requests.post(AUTH_ENDPOINT, data=json.dumps(data), headers=headers)
token = r.json()['token']

headers = {
    "Content-Type": "application/json",
    "Authorization": "JWT " + token,
}

data = {
    "content": "Updated description"
}
json_data = json.dumps(data)
posted_response = requests.put(ENDPOINT, data=data, headers=headers)
print(posted_response.text)
```
*In our test file I will make a request with post and to use the jwt we pass it in the headers and add our token to it, now you can create*
``` python
headers = {
    #"Content-Type": "application/json",
    "Authorization": "JWT " + token,
}

with open(image_path, 'rb') as image:
    file_data = {
        'image': image
    }
    data = {
        "content": "Updated description"
    }
    json_data = json.dumps(data)
    posted_response = requests.put(ENDPOINT, data=data, headers=headers, files=file_data)
    print(posted_response.text)
```
*Now I have an image I will open this image with the read binary and I commented the content-type because I will use image and then in our request I will send empty data with jwt token in header and the files will be our image, and you can create image with data*
``` python
json_data = json.dumps(data)
    posted_response = requests.put(ENDPOINT + str(37) + "/", data=data, headers=headers, files=file_data)
    print(posted_response.text)
```
*to update the detail view*

### Custom JWT Response Payload Handler

`python manage.py startapp accounts` *I created the accounts app*
``` python
JWT_AUTH = {
    'JWT_ENCODE_HANDLER':
    'rest_framework_jwt.utils.jwt_encode_handler',

    'JWT_DECODE_HANDLER':
    'rest_framework_jwt.utils.jwt_decode_handler',

    'JWT_PAYLOAD_HANDLER':
    'rest_framework_jwt.utils.jwt_payload_handler',

    'JWT_PAYLOAD_GET_USER_ID_HANDLER':
    'rest_framework_jwt.utils.jwt_get_user_id_from_payload_handler',

    'JWT_RESPONSE_PAYLOAD_HANDLER':
    'rest_framework_jwt.utils.jwt_response_payload_handler'

    'JWT_ALLOW_REFRESH': True,
    'JWT_REFRESH_EXPIRATION_DELTA': datetime.timedelta(days=7),

    'JWT_AUTH_HEADER_PREFIX': 'JWT', # Authorization: JWT <token>
    'JWT_AUTH_COOKIE': None,

}
```
*Now I want to create this jwt_response_payload_handler function and assign it to JWT_RESPONSE_PAYLOAD_HANDLER*
``` python
import datetime
from django.conf import settings
from django.utils import timezone

expire_delta = settings.JWT_AUTH['JWT_REFRESH_EXPIRATION_DELTA']

def jwt_response_payload_handler(token, user=None, request=None):
    return {
        'token': token,
        'user': user.username,
        'expires': timezone.now() + expire_delta - datetime.timedelta(seconds=200)
    }
```
*Now in our accounts app I created folder call api and in this folder I created two files the first is __init__.py and the second is utils.py and in this file I created this method that will take the token and the user and the request and return a dictionary with this data Then I get the jwt auth expire date from the settings.py then created a new expire*
``` python
'JWT_RESPONSE_PAYLOAD_HANDLER':
    'accounts.api.utils.jwt_response_payload_handler',
```
*and added the path for our new method* </br>
*Now when you go to the link it will return this data*
### Custom Authentication View
*Then in the api folder I created another two files one call `urls.py` and the second is `views.py`*
``` python
from django.conf.urls import url, include
from django.contrib import admin

from rest_framework_jwt.views import refresh_jwt_token, obtain_jwt_token # accounts app

from .views import AuthView
urlpatterns = [
    url(r'^jwt/$', obtain_jwt_token),
    url(r'^jwt/refresh/$', refresh_jwt_token),
]
```
*And I copied and paste this two urls to our new urls.py*
``` python
urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^api/auth/', include('accounts.api.urls')),
    url(r'^api/status/', include('status.api.urls')),
    url(r'^api/updates/', include('updates.api.urls')), 
]
```
*Then In our main app I removed them and include the new file*
``` python
from rest_framework.views import APIView
from rest_framework.response import Response

class AuthView(APIView):
    permission_classes      = [permissions.AllowAny]
    def post(self, request, *args, **kwargs):
        return Response({"token": "abc"})
```
*Then in our new views.py I created this view with test token*
``` python
from rest_framework_jwt.views import refresh_jwt_token, obtain_jwt_token # accounts app

from .views import AuthView
urlpatterns = [
    url(r'^$', AuthView.as_view()),
    url(r'^jwt/$', obtain_jwt_token),
    url(r'^jwt/refresh/$', refresh_jwt_token),
]
```
*And will make this view be the home*
``` python
AUTH_ENDPOINT = "http://127.0.0.1:8000/api/auth/"
REFRESH_ENDPOINT = AUTH_ENDPOINT + "refresh/"
ENDPOINT = "http://127.0.0.1:8000/api/status/"

image_path = os.path.join(os.getcwd(), "logo.jpg")

headers = {
    "Content-Type": "application/json"
}

data = {
    'username': 'cfe',
    'password': 'learncode'
}

r = requests.post(AUTH_ENDPOINT, data=json.dumps(data), headers=headers)
token = r.json() #['token']
print(token)
```
*Then will go to create a new one by sending this data and our header and will get the response as json and print it, And it will return our test token*
``` python
jwt_payload_handler             = api_settings.JWT_PAYLOAD_HANDLER
jwt_encode_handler              = api_settings.JWT_ENCODE_HANDLER

class AuthView(APIView):
    permission_classes      = [permissions.AllowAny]
    def post(self, request, *args, **kwargs):
        #print(request.user)
        if request.user.is_authenticated():
            return Response({'detail': 'You are already authenticated'}, status=400)
        data = request.data
        username = data.get('username') # username or email address
        password = data.get('password')
        user = authenticate(username=username, password=password)
        payload = jwt_payload_handler(user)
        token = jwt_encode_handler(payload)
        return Response({'token': token})
```
*Then I copied the payload and encode from the documentation then make the payload by the user and make our token by the payload and return this token*
``` python
from django.contrib.auth import authenticate, get_user_model
from django.db.models import Q
from rest_framework import permissions
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework_jwt.settings import api_settings

jwt_payload_handler             = api_settings.JWT_PAYLOAD_HANDLER
jwt_encode_handler              = api_settings.JWT_ENCODE_HANDLER
jwt_response_payload_handler    = api_settings.JWT_RESPONSE_PAYLOAD_HANDLER

User = get_user_model()

class AuthView(APIView):
    permission_classes      = [permissions.AllowAny]
    def post(self, request, *args, **kwargs):
        #print(request.user)
        if request.user.is_authenticated():
            return Response({'detail': 'You are already authenticated'}, status=400)
        data = request.data
        username = data.get('username') # username or email address
        password = data.get('password')
        user = authenticate(username=username, password=password)
        qs = User.objects.filter(
                Q(username__iexact=username)|
                Q(email__iexact=username)
            ).distinct()
        if qs.count() == 1:
            user_obj = qs.first()
            if user_obj.check_password(password):
                user = user_obj
                payload = jwt_payload_handler(user)
                token = jwt_encode_handler(payload)
                response = jwt_response_payload_handler(token, user, request=request)
                return Response(response)
        return Response({"detail": "Invalid credentials"}, status=401)
```
*Then I get the response method from the utils.py and make our response and return it back and modified the other two variables to get them from the same file, and when i run this file Now our token is here with the user and expire date*</br>
*Then I get the user model and imported the Q lookup*<br>
*Then I will make a query to our user model to get the user by username or email and use the distinct to return a unique value then if there’s one then get the first one and check the password that is send in the data if the same in database and make our toke otherwise return invalid*
``` python
import datetime
from django.conf import settings
from django.utils import timezone

from rest_framework_jwt.settings import api_settings

expire_delta             = api_settings.JWT_REFRESH_EXPIRATION_DELTA


def jwt_response_payload_handler(token, user=None, request=None):
    return {
        'token': token,
        'user': user.username,
        'expires': timezone.now() + expire_delta - datetime.timedelta(seconds=200)
    }
```
*And I replaced our variable to get it from the api_settings in utils.py*








