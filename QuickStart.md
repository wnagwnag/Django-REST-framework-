我们将创建一个简单的API，以允许管理员用户查看和编辑的系统中的用户和组
### Projectsetup
创建一个新的Django项目命名为`tutorial`, 然后开启一个新的应用起名为`quickstart`.
```
# Create the project directory
mkdir tutorial
cd tutorial

# Create a virtualenv to isolate our package dependencies locally
virtualenv env
source env/bin/activate  # On Windows use `env\Scripts\activate`

# Install Django and Django REST framework into the virtualenv
pip install django
pip install djangorestframework

# Set up a new project with a single application
django-admin.py startproject tutorial .  # Note the trailing '.' character
cd tutorial
django-admin.py startapp quickstart
cd ..
```
在项目布局应该是：

```
$ pwd
<some path>/tutorial
$ find .
.
./manage.py
./tutorial
./tutorial/__init__.py
./tutorial/quickstart
./tutorial/quickstart/__init__.py
./tutorial/quickstart/admin.py
./tutorial/quickstart/apps.py
./tutorial/quickstart/migrations
./tutorial/quickstart/migrations/__init__.py
./tutorial/quickstart/models.py
./tutorial/quickstart/tests.py
./tutorial/quickstart/views.py
./tutorial/settings.py
./tutorial/urls.py
./tutorial/wsgi.py
```
它可能看起来不寻常的应用程序中创建项目目录。该项目使用的名称空间，避免名称冲突的外部模块(超出主题的范围的情况下快速启动)。

现在同步数据库：

```
python manage.py migrate
```
我们还将创建一个初始用户名为admin，密码password123。我们将为认证用户，后来在我们的例子。

```
python manage.py createsuperuser --email admin@example.com --username admin
```
一旦你建立了一个数据库和用户初始创建和准备出门，打开应用程序目录，我们会拿到编码。..
### Serializers
首先我们要明确一些。让我们创建一个新模块教程/quickstart/serializers.py，我们用于表示数据。

```
from django.contrib.auth.models import User, Group
from rest_framework import serializers


class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = User
        fields = ('url', 'username', 'email', 'groups')


class GroupSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Group
        fields = ('url', 'name')
```
注意，我们使用超链接关系在这种情况下，与hyperlinkedmodelserializer。还可以使用主密钥和其他各种关系，但是良好的RESTful设计是超链接。
### Views.
好了，最好写一些看法。//views.pyQuickStart开放教程并分型。

```
from django.contrib.auth.models import User, Group
from rest_framework import viewsets
from tutorial.quickstart.serializers import UserSerializer, GroupSerializer


class UserViewSet(viewsets.ModelViewSet):
    """
    API endpoint that allows users to be viewed or edited.
    """
    queryset = User.objects.all().order_by('-date_joined')
    serializer_class = UserSerializer


class GroupViewSet(viewsets.ModelViewSet):
    """
    API endpoint that allows groups to be viewed or edited.
    """
    queryset = Group.objects.all()
    serializer_class = GroupSerializer
```
写入多个视图而不是我们常见的所有行为分解成分别名为viewsets。

我们可以轻易地打破这些不同意见，如果我们需要，但是使用viewsets保持视图逻辑组织好，也很简洁。
### URLs
好吧，现在让我们连线API的URL。在为教程/urls.py。..

```
from django.conf.urls import url, include
from rest_framework import routers
from tutorial.quickstart import views

router = routers.DefaultRouter()
router.register(r'users', views.UserViewSet)
router.register(r'groups', views.GroupViewSet)

# Wire up our API using automatic URL routing.
# Additionally, we include login URLs for the browsable API.
urlpatterns = [
    url(r'^', include(router.urls)),
    url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
]
```
因为我们使用viewsets视图的替代，可以自动生成该confURL的API，通过简单的注册，路由器viewsets类。

再次，如果我们需要更多的控制API的URL可以简单地下降到正常的基于类的使用意见，并明确写入URLconf。

最后，我们默认的登录和注销视图包括用于浏览的API。可选的，但是如果你需要认证的API，并希望使用可浏览的API。
### Settings
Add ```'rest_framework'``` to ```INSTALLED_APPS```. The settings module will be in ```tutorial/settings.py```

```
INSTALLED_APPS = (
    ...
    'rest_framework',
)
```
OK,我们完成了！

---
### Testing our API
我们现在准备好测试API而建造。让我们启动服务器的命令行。

```
python manage.py runserver
```
我们现在可以访问我们的API、从命令行、使用工具，比如```curl```。..

```
bash: curl -H 'Accept: application/json; indent=4' -u admin:password123 http://127.0.0.1:8000/users/
{
    "count": 2,
    "next": null,
    "previous": null,
    "results": [
        {
            "email": "admin@example.com",
            "groups": [],
            "url": "http://127.0.0.1:8000/users/1/",
            "username": "admin"
        },
        {
            "email": "tom@example.com",
            "groups": [                ],
            "url": "http://127.0.0.1:8000/users/2/",
            "username": "tom"
        }
    ]
}
```
或使用httpie，命令行工具。..

```
bash: http -a admin:password123 http://127.0.0.1:8000/users/

HTTP/1.1 200 OK
...
{
    "count": 2,
    "next": null,
    "previous": null,
    "results": [
        {
            "email": "admin@example.com",
            "groups": [],
            "url": "http://localhost:8000/users/1/",
            "username": "paul"
        },
        {
            "email": "tom@example.com",
            "groups": [                ],
            "url": "http://127.0.0.1:8000/users/2/",
            "username": "tom"
        }
    ]
}
```
或直接通过浏览器去该网址```http://127.0.0.1:8000/users/```...
![image](http://www.django-rest-framework.org/img/quickstart.png)

如果你通过浏览器，请确保登录时使用控制在屏幕的右上角。

很棒，非常方便！

如果你想更深入了解REST框架装配在一起的关于我们的教程，或浏览该API指南开始。
