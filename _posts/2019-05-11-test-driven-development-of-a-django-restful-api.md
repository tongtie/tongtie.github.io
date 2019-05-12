---
layout: post
title: Test Driven Development of a Django RESTful API
date: 2019-05-12
typora-root-url: ..
---







> 此文属于翻译文，有修改， 原始链接：https://realpython.com/test-driven-development-of-a-django-restful-api/
>





### 安装依赖


```shell
mkdir django-puppy-store
cd django-puppy-store
# 使用conda创建3.6的虚拟环境
conda create -n py36env python=3.6
# 激活虚拟环境
conda activate py36env
# 安装django1.11版本
pip install django==1.11.0
# 创建django工程
django-admin startproject puppy_store
# 进入django工程
cd puppy_store
# 创建app
python manage.py startapp puppies
# 下载DRF
pip install djangorestframework==3.6.2
# 安装mysqlclient
pip install mysqlclient
```



### 目录格式

```html
├── manage.py
├── puppies
│   ├── admin.py
│   ├── apps.py
│   ├── __init__.py
│   ├── migrations
│   │   └── __init__.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
└── puppy_store
    ├── __init__.py
    ├── __pycache__
    │   ├── __init__.cpython-36.pyc
    │   └── settings.cpython-36.pyc
    ├── settings.py
    ├── urls.py
    └── wsgi.py
```



### 修改settings

```python
# vim puppy_store/settings.py

# 添加apps
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'puppies',
    'rest_framework'
]

# 添加restframework配置， 目的是取消默认访问接口的权限
REST_FRAMEWORK = {
    # Use Django's standard `django.contrib.auth` permissions,
    # or allow read-only access for unauthenticated users.
    'DEFAULT_PERMISSION_CLASSES': [],
    'TEST_REQUEST_DEFAULT_FORMAT': 'json'
}

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'puppy_store_drf',
        'USER': 'mysql用户名',
        'PASSWORD': 'mysql密码',
        'HOST': '127.0.0.1',
        'PORT': '3306'
    }
}
```



### 创建数据库

```shell
# 进入到mysql
mysql -uroot -p
# 创建数据库
create database puppy_store_drf;
```



### 创建model

```python
# puppies/models.py

from django.db import models


class Puppy(models.Model):
    """
    Puppy Model
    Defines the attributes of a puppy
    """
    name = models.CharField(max_length=255)
    age = models.IntegerField()
    breed = models.CharField(max_length=255)
    color = models.CharField(max_length=255)
    # 创建的时候将时间更新新为当前时间
    created_at = models.DateTimeField(auto_now_add=True)
    # 每次save的时候时间更新为当前时间
    updated_at = models.DateTimeField(auto_now=True)

    def get_breed(self):
        return self.name + ' belongs to ' + self.breed + ' breed.'

    def __repr__(self):
        return self.name + ' is added.'
```



### 创建表

```python
python manage.py makemigrations
python manage.py migrate
```



### 初步测试

```python
# 删除tests.py文件
rm puppies/tests.py
# 创建tests文件夹
mkdir puppies/tests
# 新建__init__.py文件，使tests目录变成包
touch puppies/tests/__init__.py
# 新建测试文件test_models.py
# vim puppies/tests/test_models.py
# 验证model，通过写入和查询验证
from django.test import TestCase
from ..models import Puppy


class PuppyTest(TestCase):
    """ Test module for Puppy model """

    def setUp(self):
        Puppy.objects.create(
            name='Casper', age=3, breed='Bull Dog', color='Black')
        Puppy.objects.create(
            name='Muffin', age=1, breed='Gradane', color='Brown')

    def test_puppy_breed(self):
        puppy_casper = Puppy.objects.get(name='Casper')
        puppy_muffin = Puppy.objects.get(name='Muffin')
        self.assertEqual(
            puppy_casper.get_breed(), "Casper belongs to Bull Dog breed.")
        self.assertEqual(
            puppy_muffin.get_breed(), "Muffin belongs to Gradane breed.")
```



### 测试结果

```python
python manage.py test
```

出现如下信息，代表测试成功。django会自动创建一个test数据库，测试完成后会删除该数据库。

```shell

Creating test database for alias 'default'...
System check identified no issues (0 silenced).
.
----------------------------------------------------------------------
Ran 1 test in 0.007s

OK
Destroying test database for alias 'default'...
```



### 创建序列化文件(Serializer)

```shell
# 创建文件
touch puppies/serializers.py

vim puppies/serializers.py
```



```python
from rest_framework import serializers
from .models import Puppy


class PuppySerializer(serializers.ModelSerializer):
    class Meta:
        model = Puppy
        fields = '__all__'
```



### 测试驱动开发

测试驱动开发指的是先编写测试文件，然后根据测试的内容实现程序逻辑

```shell
touch puppies/tests/test_views.py
vim puppies/tests/test_veiws.py
```

有没有发现test文件中的每个类每次都需要自己插入内容，因为django的单元测试每次都会创建一个以test开头的新的空库。

```python
import json
from rest_framework import status
from django.test import TestCase, Client
from django.urls import reverse
from ..models import Puppy
from ..serializers import PuppySerializer

# initialize the APIClient app
client = Client()


class GetAllPuppiesTest(TestCase):
    """ Test module for GET all puppies API """

    def setUp(self):
        Puppy.objects.create(name='Casper',
                             age=3,
                             breed='Bull Dog',
                             color='Black')
        Puppy.objects.create(name='Muffin',
                             age=1,
                             breed='Gradane',
                             color='Brown')
        Puppy.objects.create(name='Rambo',
                             age=2,
                             breed='Labrador',
                             color='Black')
        Puppy.objects.create(name='Ricky',
                             age=6,
                             breed='Labrador',
                             color='Brown')

    # 测试GET所有
    def test_get_all_puppies(self):
        # get API response
        # reverse（定义的urls.py的name)可以获取到完整的url
        response = client.get(reverse('get_post_puppies'))
        # get data from db
        puppies = Puppy.objects.all()
        serializer = PuppySerializer(puppies, many=True)
        self.assertEqual(response.data, serializer.data)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    # 测试GET单个，不存在的情况
    def test_get_invalid_single_puppy(self):
        response = client.get(
            reverse('get_delete_update_puppy', kwargs={'pk': 30}))
        self.assertEqual(response.status_code, status.HTTP_404_NOT_FOUND)


# 测试POST，分别测试茶创建成功和失败的情况
class CreateNewPuppyTest(TestCase):
    """ Test module for inserting a new puppy """

    def setUp(self):
        self.valid_payload = {
            'name': 'Muffin',
            'age': 4,
            'breed': 'Pamerion',
            'color': 'White'
        }
        self.invalid_payload = {
            'name': '',
            'age': 4,
            'breed': 'Pamerion',
            'color': 'White'
        }

    def test_create_valid_puppy(self):
        response = client.post(reverse('get_post_puppies'),
                               data=json.dumps(self.valid_payload),
                               content_type='application/json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)

    def test_create_invalid_puppy(self):
        response = client.post(reverse('get_post_puppies'),
                               data=json.dumps(self.invalid_payload),
                               content_type='application/json')
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)


# 验证更新成功和失败
class UpdateSinglePuppyTest(TestCase):
    """ Test module for updating an existing puppy record """

    def setUp(self):
        self.casper = Puppy.objects.create(name='Casper',
                                           age=3,
                                           breed='Bull Dog',
                                           color='Black')
        self.muffin = Puppy.objects.create(name='Muffy',
                                           age=1,
                                           breed='Gradane',
                                           color='Brown')
        self.valid_payload = {
            'name': 'Muffy',
            'age': 2,
            'breed': 'Labrador',
            'color': 'Black'
        }
        self.invalid_payload = {
            'name': '',
            'age': 4,
            'breed': 'Pamerion',
            'color': 'White'
        }

    def test_valid_update_puppy(self):
        response = client.put(reverse('get_delete_update_puppy',
                                      kwargs={'pk': self.muffin.pk}),
                              data=json.dumps(self.valid_payload),
                              content_type='application/json')
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)

    def test_invalid_update_puppy(self):
        response = client.put(reverse('get_delete_update_puppy',
                                      kwargs={'pk': self.muffin.pk}),
                              data=json.dumps(self.invalid_payload),
                              content_type='application/json')
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)


# 验证删除成功和失败的情况
class DeleteSinglePuppyTest(TestCase):
    """ Test module for deleting an existing puppy record """

    def setUp(self):
        self.casper = Puppy.objects.create(name='Casper',
                                           age=3,
                                           breed='Bull Dog',
                                           color='Black')
        self.muffin = Puppy.objects.create(name='Muffy',
                                           age=1,
                                           breed='Gradane',
                                           color='Brown')

    def test_valid_delete_puppy(self):
        response = client.delete(
            reverse('get_delete_update_puppy', kwargs={'pk': self.muffin.pk}))
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)

    def test_invalid_delete_puppy(self):
        response = client.delete(
            reverse('get_delete_update_puppy', kwargs={'pk': 30}))
        self.assertEqual(response.status_code, status.HTTP_404_NOT_FOUND)

```



### 根据测试的内容编写view逻辑

```shell
vim puppies/views.py
```

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework import status
from .models import Puppy
from .serializers import PuppySerializer


@api_view(['GET', 'DELETE', 'PUT'])
def get_delete_update_puppy(request, pk):
    try:
        puppy = Puppy.objects.get(pk=pk)
    except Puppy.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    # get details of a single puppy
    if request.method == 'GET':
        serializer = PuppySerializer(puppy)
        return Response(serializer.data)
    # delete a single puppy
    elif request.method == 'DELETE':
        puppy.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
    # update details of a single puppy
    elif request.method == 'PUT':
        serializer = PuppySerializer(puppy, data=request.data)
        if serializer.is_valid():
            serializer.save()
            # 注意这里更新成功，返回的状态码是204
            return Response(serializer.data, status=status.HTTP_204_NO_CONTENT)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


@api_view(['GET', 'POST'])
def get_post_puppies(request):
    # get all puppies
    if request.method == 'GET':
        puppies = Puppy.objects.all()
        serializer = PuppySerializer(puppies, many=True)
        return Response(serializer.data)
    # insert a new record for a puppy
    elif request.method == 'POST':
        data = {
            'name': request.data.get('name'),
            'age': int(request.data.get('age')),
            'breed': request.data.get('breed'),
            'color': request.data.get('color')
        }
        serializer = PuppySerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```



### 创建对应的URL地址

```shell
touch puppies/urls.py
vim puppies/urls.py
```



```python
from django.conf.urls import url, include
from . import views


urlpatterns = [
    url(
        r'^api/v1/puppies/(?P<pk>[0-9]+)$',
        views.get_delete_update_puppy,
        name='get_delete_update_puppy'
    ),
    url(
        r'^api/v1/puppies/$',
        views.get_post_puppies,
        name='get_post_puppies'
    )
]
```



### 完善总的url地址

```shell
vim puppy_store/urls.py
```



```python
from django.conf.urls import url, include
from django.contrib import admin

urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^', include('puppies.urls')),
    url(r'^api-auth/',
        include('rest_framework.urls', namespace='rest_framework')),
]
```



### 测试并启动服务

```shell
# 测试
python manage.py test
# 启动服务
python manange.py shell
# 如果有问题，可以通过以下的命令排查
python manage.py check
```



### 最终目录结构

```html
└── puppy_store
    ├── manage.py
    ├── puppies
    │   ├── admin.py
    │   ├── apps.py
    │   ├── __init__.py
    │   ├── migrations
    │   │   ├── 0001_initial.py
    │   │   ├── __init__.py
    │   │   
    │   ├── models.py
    │   ├── serializers.py
    │   ├── tests
    │   │   ├── __init__.py
    │   │   ├── test_models.py
    │   │   └── test_views.py
    │   ├── urls.py
    │   └── views.py
    └── puppy_store
        ├── __init__.py
        ├── settings.py
        ├── urls.py
        └── wsgi.py
```



### Browsable API验证

启动服务后， 输入地址：http://localhost:8000/api/v1/puppies/

可以看到如下的页面代表成功了

![1557646017610](/image/restful-browsable-api)





如图，输入json串， 点击post验证

```json
{"name":"Muffy","age":2,"breed":"Labrador","color":"Black"}
```

返回，如图所示代表成功

![1557651927024](/image/restful-browsable-api-test)



### 进阶

#### Serializer的隐藏功能

```python
# vim puppies/serializers.py
# 修改添加两个字段和一个函数

class PuppySerializer(serializers.ModelSerializer):
    # 展现的时候会自动调用get_desc方法
    desc = serializers.SerializerMethodField()
    # 展现的时候的值即为models里的Choices里的内容
    human_readable_gender = serializers.CharField(source='get_gender_display', required=False)
    
    def get_desc(self, obj):
        return obj.get_breed()
    
# vim puppies/models.py
# 在类下面添加这两个信息
class Puppy(models.Model):
    """
    Puppy Model
    Defines the attributes of a puppy
    """
    GENDER_CHOICES = (('M', 'Male'), ('F', 'Female'))
    gender = models.CharField(max_length=1, choices=GENDER_CHOICES, null=True)


# vim puppies/views.py
# 修改添加，保存的时候保存gender字段
@api_view(['GET', 'POST'])
def get_post_puppies(request):
    # get all puppies
    # insert a new record for a puppy
    elif request.method == 'POST':
        data = {
            'gender': request.data.get('gender')
        }

```



更新内容到数据库

```shell
python manage.py makemigrations
python manage.py migrate
```



启动服务

```shell
python manage.py runserver
```



打开网页，http://localhost:8000/api/v1/puppies/， 输入以下内容，post请求，返回的信息如下即成功。

```json
{
    "name": "MeiMei",
    "age": 1,
    "breed": "Labrador",
    "color": "Black",
    "gender": "M"
}
```

![1557656165277](/image/restful-browsable-api-advanced)



### 附

代码地址：https://github.com/tongtie/blogs/tree/master/django-puppy-store