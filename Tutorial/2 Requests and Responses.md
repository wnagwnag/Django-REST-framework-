++From this point we're going to really start covering the core of REST framework. Let's introduce a couple of essential building blocks.++

从这一点开始，我们将真正开始介绍REST框架的核心。让我们介绍几个必要的构建块。
### Request objects

++REST framework introduces a `Request` object that extends the regular `HttpRequest`, and provides more flexible request parsing. The core functionality of the `Request` object is the `request.data` attribute, which is similar to `request.POST`, but more useful for working with Web APIs.++

REST框架介绍`Request`对象它扩展了常规的` HttpRequest `，提供更灵活的请求解析。`Request`对象的核心功能是`request.data`属性，它类似于`request.POST`，但更有用的是使用Web API。


```
request.POST  # Only handles form data.  Only works for 'POST' method.
request.data  # Handles arbitrary data.  Works for 'POST', 'PUT' and 'PATCH' methods.
```


### Response objects

REST framework also introduces a `Response` object, which is a type of `TemplateResponse` that takes unrendered content and uses content negotiation to determine the correct content type to return to the client.

REST框架也引入了一个`Response`对象，它是一种` TemplateResponse `以未着色的内容和使用内容协商确定返回给客户端的正确的内容类型。


```
return Response(data)  # Renders to content type as requested by the client.
```

### Status codes

++Using numeric HTTP status codes in your views doesn't always make for obvious reading, and it's easy to not notice if you get an error code wrong. REST framework provides more explicit identifiers for each status code, such as `HTTP_400_BAD_REQUEST` in the `status` module. It's a good idea to use these throughout rather than using numeric identifiers.++

在视图中使用数字HTTP状态代码并不是很方便，如果错误代码出错了也不容易发现。REST框架在`status`模块为每个状态码提供了更明确的标识，如` http_400_bad_request `。使用这些贯穿始终，而不是使用数字标识符是一个好主意。
### Wrapping API views

++REST framework provides two wrappers you can use to write API views.++

REST框架提供了两个可以用来编写API视图的包装器。

- ++The `@api_view` decorator for working with function based views.++
- `@api_view`装饰器用于处理基于函数的视图。 
- ++The `APIView` class for working with class-based views.++
- `APIView`类用于处理基于类的视图。

++These wrappers provide a few bits of functionality such as making sure you receive `Request` instances in your view, and adding context to `Response` objects so that content negotiation can be performed.++

这些包装器提供了一些功能，如确保在视图中接收`Request`实例，并将上下文添加到`Response`对象中，从而可以进行内容协商。

++The wrappers also provide behaviour such as returning `405 Method Not Allowed` responses when appropriate, and handling any `ParseError` exception that occurs when accessing `request.data` with malformed input.++

该包装还提供行为，如在适当的时候返回` 405 Method Not Allowed`响应，和处理任何
当访问`request.data`时，因为错误的数据输入，发生的` ParseError `异常。

### Pulling it all together

++Okay, let's go ahead and start using these new components to write a few views.++

好的，让我们继续使用这些新组件来编写一些视图。 

++We don't need our `JSONResponse` class in `view.py` any more, so go ahead and delete that. Once that's done we can start refactoring our views slightly.++

我们不需要我们的` JSONResponse `类`view.py`模块中，所以可以删除。一旦完成，我们就可以稍微重构我们的视图了。

```
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer


@api_view(['GET', 'POST'])
def snippet_list(request):
    """
    List all code snippets, or create a new snippet.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```


++Our instance view is an improvement over the previous example. It's a little more concise, and the code now feels very similar to if we were working with the Forms API. We're also using named status codes, which makes the response meanings more obvious.++

我们的实例视图比前面的示例有所改进。它稍微简洁一点，现在的代码与我们使用表单API时非常相似。我们还使用命名状态代码，这使得响应含义更加明显。

++Here is the view for an individual snippet, in the `views.py` module.++

这里是一个单独代码片段的视图，在`views.py`模块中。 

```
@api_view(['GET', 'PUT', 'DELETE'])
def snippet_detail(request, pk):
    """
    Retrieve, update or delete a code snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```


++This should all feel very familiar - it is not a lot different from working with regular Django views.++

这应该都很熟悉的——它与我们正常的视图代码编写没有什么不同。

++Notice that we're no longer explicitly tying our requests or responses to a given content type. `request.data` can handle incoming `json` requests, but it can also handle other formats. Similarly we're returning response objects with data, but allowing REST framework to render the response into the correct content type for us.++

注意，我们不再显式地将请求或响应绑定到给定的内容类型。`request.data`可以处理传入的`json`请求，但它也可以处理其他格式。类似地，我们用数据返回响应对象，但允许REST框架将响应呈现给我们正确的内容类型。

### Adding optional format suffixes to our URLs

++To take advantage of the fact that our responses are no longer hardwired to a single content type let's add support for format suffixes to our API endpoints. Using format suffixes gives us URLs that explicitly refer to a given format, and means our API will be able to handle URLs such as [http://example.com/api/items/4.json](http://note.youdao.com/).++

利用这个事实，我们的反应不再是硬连接到一个单一的内容类型添加后缀的形式让我们的API端点支持。使用格式后缀给我们明确引用给定格式的URL，意味着我们的API能够处理URL，例如[http://example.com/api/items/4.json](http://note.youdao.com/).

Start by adding a `format` keyword argument to both of the views, like so.

首先，向这两个视图添加一个`format`关键字参数，如下所示。

```
def snippet_list(request, format=None):
```


and


```
def snippet_detail(request, pk, format=None):
```


++Now update the `snippets/urls.py` file slightly, to append a set of `format_suffix_patterns` in addition to the existing URLs.++

现在稍微更新一下`snippets/urls.py`文件，附加一套` format_suffix_patterns `除了现有的URL。

```
from django.conf.urls import url
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    url(r'^snippets/$', views.snippet_list),
    url(r'^snippets/(?P<pk>[0-9]+)$', views.snippet_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```


++We don't necessarily need to add these extra url patterns in, but it gives us a simple, clean way of referring to a specific format.++

我们不一定需要在URL中添加这些额外的URL模式，但是它提供了一种简单、干净的方式来引用特定的格式。

### How's it looking?

++Go ahead and test the API from the command line, as we did in tutorial part 1. Everything is working pretty similarly, although we've got some nicer error handling if we send invalid requests.++

继续从命令行测试API，就像我们在教程第1部分中所做的那样。一切都非常相似，但如果发送无效请求，我们会得到更好的错误处理。

++We can get a list of all of the snippets, as before.++

我们可以像以前一样获得所有代码片段的列表。 

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


++We can control the format of the response that we get back, either by using the Accept header:++

我们可以通过使用接受头来控制我们返回的响应的格式：

```
http http://127.0.0.1:8000/snippets/ Accept:application/json  # Request JSON
http http://127.0.0.1:8000/snippets/ Accept:text/html         # Request HTML
```


++Or by appending a format suffix:++

或附加格式后缀：

```
http http://127.0.0.1:8000/snippets.json  # JSON suffix
http http://127.0.0.1:8000/snippets.api   #
```

### Browsable API suffix

++Similarly, we can control the format of the request that we send, using the Content-Type header.++

类似地，我们可以使用内容类型头控制我们发送的请求的格式。 

```
# POST using form data
http --form POST http://127.0.0.1:8000/snippets/ code="print 123"

{
  "id": 3,
  "title": "",
  "code": "print 123",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}

# POST using JSON
http --json POST http://127.0.0.1:8000/snippets/ code="print 456"

{
    "id": 4,
    "title": "",
    "code": "print 456",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```


++If you add a --debug switch to the http requests above, you will be able to see the request type in request headers.++

如果向上面的HTTP请求添加一个调试开关，您将能够看到请求标头中的请求类型。

++Now go and open the API in a web browser, by visiting http://127.0.0.1:8000/snippets/.
Browsability++

现在在Web浏览器中打开API，通过访问http://127.0.0.1:8000/snippets/
浏览功能。

++Because the API chooses the content type of the response based on the client request, it will, by default, return an HTML-formatted representation of the resource when that resource is requested by a web browser. This allows for the API to return a fully web-browsable HTML representation.++

因为API根据客户机请求选择响应的内容类型，默认情况下，当Web浏览器请求该资源时，它将返回资源的HTML格式化表示。这允许返回一个完全的Web浏览HTML表示API。

++Having a web-browsable API is a huge usability win, and makes developing and using your API much easier. It also dramatically lowers the barrier-to-entry for other developers wanting to inspect and work with your API.++

有一个Web浏览器API是一个巨大的可用性的胜利，使开发和使用你的API更容易。它还大大降低了希望检查和使用API的其他开发人员的进入障碍。

++See the browsable api topic for more information about the browsable API feature and how to customize it.++

看到浏览器API的话题有关浏览器API功能的更多信息以及如何自定义。
### What's next?

In tutorial part 3, we'll start using class-based views, and see how generic views reduce the amount of code we need to write.

在教程第3部分中，我们将开始使用基于类的视图，并查看泛型视图如何减少我们需要编写的代码量。
