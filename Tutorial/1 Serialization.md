### Introduction
本教程将介绍创建一个简单的API的代码高亮Pastebin网站。同时介绍了各种部件静止框架，让你全面了解整体效果。

本教程是相当深入，所以你应该得到的cookie和一杯煮你的咖啡再开始计算。如果你只想迅速概览，你应该去快速入门文档。
> 注：本教程中的代码是可用的REST框架/tomchristie-教程库在Github。在完成执行沙箱还在线版本的测试，可用在这里。
### Setting up a new environment

在做任何事情之前，我们将会创建一个新的虚拟环境中、使用virtualenv。这将确保我们保持好的封装配置的任何其他项目。

```
virtualenv env
source env/bin/activate
```
现在，我们进入virtualenv的环境，我们可以将我们的要求包装。

```
pip install django
pip install djangorestframework
pip install pygments  # We'll be using this for the code highlighting
```
注：退出virtualenv环境在任何时间，只需停用。欲了解更多信息参见virtualenv的文件

### Getting started
好吧，我们准备编码。开始行动吧，让我们创建一个新项目。

```
cd ~
django-admin.py startproject tutorial
cd tutorial
```
一旦完成,我们可以创建一个应用程序使用，我们将创建一个简单的WebAPI。

```
python manage.py startapp snippets
```
我们将需要编辑`tutorial/settings.py`文件，添加新的`snippets`APP和`rest_framework`APP到`INSTALLED_APPS`中。

```
INSTALLED_APPS = (
    ...
    'rest_framework',
    'snippets.apps.SnippetsConfig',
)
```
好吧，我们准备好出发了。
### Creating a model to work with
在本教程中，我们将会创建一个简单的`Snippet`，用来保存代码片断。去吧，请编辑文件`snippets/models.py`。注：良好的编程实践中是否包括注释。虽然你会发现他们在我们的配置库的版本代码本教程中，我们已经省略这里关注代码本身。

```
from django.db import models
from pygments.lexers import get_all_lexers
from pygments.styles import get_all_styles

LEXERS = [item for item in get_all_lexers() if item[1]]
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
STYLE_CHOICES = sorted((item, item) for item in get_all_styles())


class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

    class Meta:
        ordering = ('created',)
```
我们还需要创建一个初始片段迁移的模型、和同步的数据库。

```
python manage.py makemigrations snippets
python manage.py migrate
```
### Creating a Serializer class
首先我们需要在我们的WebAPI提供的snippet实例的序列化和反序列化,例如表示为`json`。我们可以通过声明串行化器工作，非常类似于Django的形式。的`snippets`目录中创建一个名为`serializers.py`的文件，并添加以下代码：

```
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={'base_template': 'textarea.html'})
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

    def create(self, validated_data):
        """
        Create and return a new `Snippet` instance, given the validated data.
        """
        return Snippet.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        Update and return an existing `Snippet` instance, given the validated data.
        """
        instance.title = validated_data.get('title', instance.title)
        instance.code = validated_data.get('code', instance.code)
        instance.linenos = validated_data.get('linenos', instance.linenos)
        instance.language = validated_data.get('language', instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance
```
第一部分定义了字段序列化器类，序列化/反序列化。`create()`和`update()`方法定义如何全面实例的创建或修改时调用`serializer.save()`

串化器类非常类似于Django类形式，并且包括类似的验证标志的各种领域，例如`required`，`max_length`和`default`。

场标志还可以控制串行器应当如何被显示在某些情况下，例如，当呈现为HTML。模板的`“{_base′:′′}textarea.html`标志为等值使用`widget=widgets.Textarea`在Django的`Form`类。这尤其可用于控制如何显示可浏览的API，我们会在本教程中。

我们其实也可以省下一些时间通过使用`modelserializer`类，以后我们会看到，但是你现在必须保持我们的显式定义串行器。
### Working with Serializers
在我们继续之前，我们将了解使用我们新的序列化器类。我们顺便到Django shell。

```
python manage.py shell
```
好，当我们有一些进口的方式，让我们创建的代码片段。

```
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser

snippet = Snippet(code='foo = "bar"\n')
snippet.save()

snippet = Snippet(code='print "hello, world"\n')
snippet.save()
```
现在我们有几个`snippet实例。让我们来看看一个序列化的那些实例。

```
serializer = SnippetSerializer(snippet)
serializer.data
# {'id': 2, 'title': u'', 'code': u'print "hello, world"\n', 'linenos': False, 'language': u'python', 'style': u'friendly'}
```
在这一点上我们的翻译模型实例转换成Python原生数据类型。最后，我们处理数据序列化为`json`。

```
content = JSONRenderer().render(serializer.data)
content
# '{"id": 2, "title": "", "code": "print \\"hello, world\\"\\n", "linenos": false, "language": "python", "style": "friendly"}'
```
反序列化类似。首先我们解析流的Python的本机数据类型。..

```
from django.utils.six import BytesIO

stream = BytesIO(content)
data = JSONParser().parse(stream)
```
一学就会恢复。那么我们这些原生数据类型的完全填充的对象实例。

```
serializer = SnippetSerializer(data=data)
serializer.is_valid()
# True
serializer.validated_data
# OrderedDict([('title', ''), ('code', 'print "hello, world"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])
serializer.save()
# <Snippet: Snippet object>
```
注意到类似的API来工作的形式。所述相似度将变得更加明显，当我们开始写观点，我们使用串行器。

我们还可以代替序列化querysets模型实例。大家只需添加一个`many=True`的参数标记到串行化器。

```
serializer = SnippetSerializer(Snippet.objects.all(), many=True)
serializer.data
# [OrderedDict([('id', 1), ('title', u''), ('code', u'foo = "bar"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 2), ('title', u''), ('code', u'print "hello, world"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 3), ('title', u''), ('code', u'print "hello, world"'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])]
```

### Using ModelSerializers
我们的`snippetserializer`类有大量的重复信息，包括代码段模型。如果我们能保持代码简洁一点就好了。

Django提供了`Form`类和`ModelForm`类，同样的REST FrameWork也提供了`Serializer`类和`ModelSerializer`

让我们看看我们重构串行器使用`modelserializer`类。打开文件`snippets/serializers.py`再次更换`snippetserializer`为以下。

```
class SnippetSerializer(serializers.ModelSerializer):
    class Meta:
        model = Snippet
        fields = ('id', 'title', 'code', 'linenos', 'language', 'style')
```
有一个不错的属性，串行化器是您可以检查所有的字段在序列化程序，例如通过命令行输入`Python manage.py shell`打开Django Shell ，然后进行以下尝试：

```
from snippets.serializers import SnippetSerializer
serializer = SnippetSerializer()
print(repr(serializer))
# SnippetSerializer():
#    id = IntegerField(label='ID', read_only=True)
#    title = CharField(allow_blank=True, max_length=100, required=False)
#    code = CharField(style={'base_template': 'textarea.html'})
#    linenos = BooleanField(required=False)
#    language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
#    style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...
```
重要的是要记住，这个`ModelSerializer`类不要做任何事，特别神奇的是它们仅仅是一种创建串行化类的捷径：

- 自动确定的一组字段。
- 简单的默认实现`create()`和`update()`方法。

### Writing regular Django views using our Serializer
让我们看看如何使用新的序列化程序类编写一些api视图。那一刻我们不会使用任何其他框架的其他功能，我们将把意见定期Django视图。

编辑`snippets/views.py`文件，并在其中添加。

```
from django.http import HttpResponse, JsonResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
```
我们的API的根将是一个视图，它支持列出所有现有的代码段，或者创建一个新的代码段。

```
@csrf_exempt
def snippet_list(request):
    """
    List all code snippets, or create a new snippet.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return JsonResponse(serializer.data, safe=False)

    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data, status=201)
        return JsonResponse(serializer.errors, status=400)
```
Note that because we want to be able to POST to this view from clients that won't have a CSRF token we need to mark the view as csrf_exempt.这不是您通常想要做的事情，REST框架视图实际上使用的是更明智的行为，但现在它将为我们的目标服务。

我们还需要一个视图对应于单独的片段，可用于检索、更新或删除片段。

```
@csrf_exempt
def snippet_detail(request, pk):
    """
    Retrieve, update or delete a code snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return JsonResponse(serializer.data)

    elif request.method == 'PUT':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(snippet, data=data)
        if serializer.is_valid():
            serializer.save()
            return JsonResponse(serializer.data)
        return JsonResponse(serializer.errors, status=400)

    elif request.method == 'DELETE':
        snippet.delete()
        return HttpResponse(status=204)
```
最后我们需要各方面的意见。创建`snippets/urls.py`文件：

```
from django.conf.urls import url
from snippets import views

urlpatterns = [
    url(r'^snippets/$', views.snippet_list),
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.snippet_detail),
]
```
我们还而要更改项目根目录中的URL配置，在`tutorial/urls.py`文件中引入snippet应用的url

```
from django.conf.urls import url, include

urlpatterns = [
    url(r'^', include('snippets.urls')),
]
```
值得注意的是，有一些边缘的情况下，我们不在正确的时刻。如果我们发送畸形的`JSON`，或如果提出请求的方法的视图，没有妥善处理，那么我们就拥有了500的“服务器错误”响应。不过，看我现在。
### Testing our first attempt at a Web API
现在我们可以开始了，我们的服务器摘要。

退出shell。..

```
quit()
```
然后开启Django开发服务器

```
python manage.py runserver

Validating models...

0 errors found
Django version 1.11, using settings 'tutorial.settings'
Development server is running at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```
在另一个终端窗口，我们可以测试服务器。

我们可以测试API使用`httpie`或`curl`。httpie是用户友好的HTTP客户端，它在Python编写的。让我们安装。

你可以使用pip来安装

```
pip install httpie
```
最后，我们可以得到所有的摘录：

```
http http://127.0.0.1:8000/snippets/

HTTP/1.1 200 OK
...
[
  {
    "id": 1,
    "title": "",
    "code": "foo = \"bar\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  },
  {
    "id": 2,
    "title": "",
    "code": "print \"hello, world\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  }
]
```
我们也可以通过参考其特定的片段ID：

```
http http://127.0.0.1:8000/snippets/2/

HTTP/1.1 200 OK
...
{
  "id": 2,
  "title": "",
  "code": "print \"hello, world\"\n",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}
```
类似地，可以显示具有相同URL的JSON数据访问的web浏览器。
### Where are we now
我们干得不错，我们的感觉相当的序列化器API，类似于Django的API形式，一些常规看法和Django。

我们的API并不做什么特别的，只不过发出JSON响应、错误处理和边缘有一些案件还是要清理干净，但它的功能的WebAPI。

我们将看到我们如何能改善第2部分的教程。
