# PEP 333-zh-CN 
============
> 翻译自 `Python Web Server Gateway Interface v1.0`  [PEP 333 - Python Web Server Gateway Interface v1.0](https://www.python.org/dev/peps/pep-0333/)

## 译者的话
```
Pass
```
## 内容
* [序言](#序言)  
* [摘要](#摘要)
* [基本原理及目标](#基本原理及目标)  
* [概述](#概述)  
	* [应用程序/框架](#应用程序/框架)
	* [服务器/网关](#服务器/网关)
	* [中间件:可扮演两边角色的组件](#中间件:可扮演两边角色的组件)
* [规格的详细说明](#规格的详细说明)  
	* [`environ`变量](#environ变量)
		* [输入和错误流](#输入和错误流)
	* [start_response() Callable](#start_response() Callable)
		* [处理Content-Length头信息](#处理Content-Length头信息)
	* [缓冲和流](#缓冲和流)
		* [中间件处理块边界](#中间件处理块边界)
		* [可调用的write()函数](#可调用的write()函数)
	* [Unicode问题](#Unicode问题)
	* [错误处理](#错误处理)
	* [HTTP 1.1 Expect/Continue请求头](#HTTP 1.1 ExpectContinue请求头)
	* [HTTP的其他特性](#HTTP的其他特性)
	* [线程支持](#线程支持)
* [实现/应用手册](#实现/应用手册)
	* Server Extension APIs
	* Application Configuration
	* URL Reconstruction
	* Supporting Older (<2.2) Versions of Python
	* Optional Platform-Specific File Handling
* [QA问答](#QA问答)
* Proposed/Under Discussion
* [鸣谢](#鸣谢)
* [参考文献](#参考文献)
* [版权声明](#版权声明)

###序言(Done)
注意: 关于支持Python 3.x的更新版本及包含一些社区勘误，补充，更正的相关说明，请参照PEP 3333.

###摘要(Done)
本文档描述一份在web服务器与web应用/web框架之间的标准接口，此接口的目的是使得web应用在不同web服务器之间具有可移植性。

###基本原理及目标
Python目前拥有大量的web框架，比如 Zope, Quixote, Webware, SkunkWeb, PSO, 和Twisted Web--这里我仅列举出几个[参考文献1]。这么多的选择让新手无所适从，因为总得来说，框架的选择都会限制web服务器的选择。  

对比之下，虽然java也拥有许多web框架，但是java的"servlet" API使得使用任何框架编写出来的应用程序都可以在任何支持" servlet" API的web服务器上运行。

服务器中这种针对python的API（不管服务器是用python写的(如: Medusa)，还是内嵌python(如: mod_python)，亦或是通过一种网关协议来调用Python(如:CGI, FastCGI等))的使用和普及，将人们从web框架的选择和web服务器的选择中分离开来，使得用户可以自由选择适合他们的组合，而web服务器和web框架的开发者也能够把精力集中到各自的领域。  

基于此，这份PEP建议在web服务器和web应用/web框架之间建立一种简单的通用的接口规范，即Python Web服务器网关接口(WSGI)。

但是光有这么一份规范对于改变web服务器和web应用/框架的现状是不够的，只有当那些web服务器和web框架的作者/维护者们真正地实现了WSGI，这份WSGI规范才将起到它该有的效果。  

然而，由于目前还没有任何框架或服务器实现了WSGI，那些支持WSGI的框架作者也没有什么直接的奖励，因此，我们的WSGI必须拟定地足够容易实现，这样才能降低框架作者们在实现接口这件事上的初始投资成本。
 
由此可见，服务器和框架两边接口的实现的简单性，对于WSGI的实用性来说，绝对是非常重要的，同时，这一点也是任何设计决策的首要依据。
```
pass
```
###概述(Done)
WSGI接口可以分为两端：服务器/网关和应用程序/Web框架。服务器端调用一个由应用程序端提供的可调用的对象，至于该对象是如何被调用的，这要取决于服务器/网关这一端。我们假定有一些服务器/网关会要求应用程序的部署人员编写一个简短的脚本来启动一个服务器/网关的实例，并提供给服务器/网关一个应用程序对象，而还有的一些服务器/网关则不需要这样，它们会需要一个配置文件又或者是其他机制来指定应该从哪里导入或者获得应用程序对象。
 
除了纯粹的服务器/网关和应用程序/框架，还可以创建叫做中间件的组件，中间件实现了这份规约当中的两端(服务器端和应用程序端)，我们可以这样解释中间件，对于包含它们的服务器，中间件是应用程序，而对于包含在服务器当中的应用程序来说，中间又扮演着服务器的角色。不仅如此，中间件还可以用来提供可扩展的API，内容转换，导航和其他有用的功能。
 
在整个规格说明书中，我们将使用的术语"a callable(可调用的)"意思是"一个函数，方法，类，或者拥有 __call__ 方法的一个对象实例",这取决于服务器，网关，或者应用程序根据需要而选择的合适实现技术。相反，服务器，网关，或者请求一个可调用对象(callable)的应用程序必须不依赖callable(可调用对象)具体的提供方式。记住，可调用对象(callable)只是被表用，不会自省(译者注：introspect，自省，Python的强项之一，指的是代码可以在内存中象处理对象一样查找其它的模块和函数）

###应用程序/框架(Done) 
一个应用程序对象就是一个简单的接受2个参数的可调用对象(callable object)，这里的对象并不能理解为它真的需要一个对象实例：一个函数、方法、类、或者带有 `__call__` 方法的对象实例都可以用来做应用程序对象。应用程序对象必须可以被多次调用，实质上所有的服务器/网关(除了CGI)都会产生这样的重复请求。
 
(注意：虽然我们把他叫做"应用程序"对象，但这并不意味着程序员要把WSGI当做API来调用！我们假定应用程序开发者将会仍然使用更高层的框架服务来开发它们的应用程序，WSGI只是一个提供给框架和服务器开发者使用的工具，它并没有打算直接向应用程序开发者提供支持)
 
这里我们来看两个应用程序对象的示例；其中，一个是函数，另一个是类:
```python
def simple_app(environ, start_response):
    """Simplest possible application object"""
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain')]
    start_response(status, response_headers)
    return ['Hello world!\n']


class AppClass:
    """Produce the same output, but using a class

    (Note: 'AppClass' is the "application" here, so calling it
    returns an instance of 'AppClass', which is then the iterable
    return value of the "application callable" as required by
    the spec.

    If we wanted to use *instances* of 'AppClass' as application
    objects instead, we would have to implement a '__call__'
    method, which would be invoked to execute the application,
    and we would need to create an instance for use by the
    server or gateway.
    """

    def __init__(self, environ, start_response):
        self.environ = environ
        self.start = start_response

    def __iter__(self):
        status = '200 OK'
        response_headers = [('Content-type', 'text/plain')]
        self.start(status, response_headers)
        yield "Hello world!\n"
 ```
####服务器/网关(Done)
每一次，当HTTP客户端(冲着应用程序来的)发来一个请求，服务器/网关都会调用应用程序可调用对象(callable)。为了说明方便，这里有一个CGI网关，简单的说它就是一个以一个应用程序对象为参数的函数实现，请注意，这个例子中包含有限的错误处理，因为默认情况下没有被捕获到的异常都会被输出到`sys.stderr`并被服务器记录下来。

```python
import os, sys

def run_with_cgi(application):

    environ = dict(os.environ.items())
    environ['wsgi.input']        = sys.stdin
    environ['wsgi.errors']       = sys.stderr
    environ['wsgi.version']      = (1, 0)
    environ['wsgi.multithread']  = False
    environ['wsgi.multiprocess'] = True
    environ['wsgi.run_once']     = True

    if environ.get('HTTPS', 'off') in ('on', '1'):
        environ['wsgi.url_scheme'] = 'https'
    else:
        environ['wsgi.url_scheme'] = 'http'

    headers_set = []
    headers_sent = []

    def write(data):
        if not headers_set:
             raise AssertionError("write() before start_response()")

        elif not headers_sent:
             # Before the first output, send the stored headers
             status, response_headers = headers_sent[:] = headers_set
             sys.stdout.write('Status: %s\r\n' % status)
             for header in response_headers:
                 sys.stdout.write('%s: %s\r\n' % header)
             sys.stdout.write('\r\n')

        sys.stdout.write(data)
        sys.stdout.flush()

    def start_response(status, response_headers, exc_info=None):
        if exc_info:
            try:
                if headers_sent:
                    # Re-raise original exception if headers sent
                    raise exc_info[0], exc_info[1], exc_info[2]
            finally:
                exc_info = None     # avoid dangling circular ref
        elif headers_set:
            raise AssertionError("Headers already set!")

        headers_set[:] = [status, response_headers]
        return write

    result = application(environ, start_response)
    try:
        for data in result:
            if data:    # don't send headers until body appears
                write(data)
        if not headers_sent:
            write('')   # send headers now if body was empty
    finally:
        if hasattr(result, 'close'):
            result.close()
```
### 中间件:可扮演两边角色的组件（Done)
注意到单个对象可以作为请求应用程序的服务器存在，也可以作为被服务器调用的应用程序存在。这样的“中间件”可以执行以下这些功能:
 - 在相应地重写environ变量之后，根据目标URL将请求路由到不同的应用程序对象
 - 允许多个应用程序或框架在同一个进程中并行运行
 - 通过在网络中转发请求和应答，实现负载均衡和远程处理
 - 对上下文(content)进行后加工，比如应用xsl样式表
 
中间件的存在对于"服务器/网关"和"应用程序/框架"来说是透明的，并不需要特殊的支持。希望在应用程序中加入中间件的用户只需简单得把中间件当作应用提供给服务器，并配置中间件组件以服务器的身份来调用应用程序。当然，中间件组件包裹的“应用程序”也可能是另外一个包裹应用程序的中间件组件，这样循环下去就构成了我们所说的"中间件栈"了。

最重要的别忘了，中间件必须遵循WSGI的服务器和应用程序两边提出的一些限制和要求，甚至有些时候,对中间件的要求比纯粹的服务器或应用程序还要严格，这些我们都会在这份规范文档中指出来。
 
这里有一个(好玩的)中间件组件的例子，它使用`Joe Strout`写的`piglatin.py`程序将text/plain的响应转换成pig latin(译者注:将英语词尾改成拉丁语式)(注意：一个“真实”的中间件组件很可能会使用更加鲁棒的方式来检查上下文(content)的类型和下文(content)的编码。同样，这个简单的例子还忽略了一个单词还可能跨区块分割的可能性)。

```python
from piglatin import piglatin

class LatinIter:

    """Transform iterated output to piglatin, if it's okay to do so

    Note that the "okayness" can change until the application yields
    its first non-empty string, so 'transform_ok' has to be a mutable
    truth value.
    """

    def __init__(self, result, transform_ok):
        if hasattr(result, 'close'):
            self.close = result.close
        self._next = iter(result).next
        self.transform_ok = transform_ok

    def __iter__(self):
        return self

    def next(self):
        if self.transform_ok:
            return piglatin(self._next())
        else:
            return self._next()

class Latinator:

    # by default, don't transform output
    transform = False

    def __init__(self, application):
        self.application = application

    def __call__(self, environ, start_response):

        transform_ok = []

        def start_latin(status, response_headers, exc_info=None):

            # Reset ok flag, in case this is a repeat call
            del transform_ok[:]

            for name, value in response_headers:
                if name.lower() == 'content-type' and value == 'text/plain':
                    transform_ok.append(True)
                    # Strip content-length if present, else it'll be wrong
                    response_headers = [(name, value)
                        for name, value in response_headers
                            if name.lower() != 'content-length'
                    ]
                    break

            write = start_response(status, response_headers, exc_info)

            if transform_ok:
                def write_latin(data):
                    write(piglatin(data))
                return write_latin
            else:
                return write

        return LatinIter(self.application(environ, start_latin), transform_ok)
# Run foo_app under a Latinator's control, using the example CGI gateway
from foo_app import foo_app
run_with_cgi(Latinator(foo_app))
```
### 规格的详细说明 (Done)
应用程序对象必须接受两个位置参数，为了方便说明，我们不妨将它们分别命名为`environ`和`start_response`，但是这并不意味着它们必须取这两个名字。服务器或网关必须用这两个位置参数(注意不是关键字参数)来调用应用程序对象(比如，像上面展示的,调用`result = application(environ,start_response)`)

`environ`参数是一个字典对象，也是一个有着CGI风格的环境变量。这个对象必须是一个python内建的字典对象(不能是子类、用户字典(UserDict)或其他对字典对象的模仿)，应用程序必须允许以任何它需要的方式来修改这个字典， `environ`还必须包含一些特定的WSGI需要的变量(在后面章节里会提到)，有可以包含一些服务器特定的扩展变量，通过下面提到的命名规范来命名。

`start_response`参数是一个可调用者(callable)，它接受两个必须的位置参数和一个可选参数。为方便说明，我们分别将它们命名为`status`, `response_headers`,和 `exc_info` 。在强调一下，这并不不是说它们必须取这些名字。应用程序必须用这些位置参数来请求可调用者(callable) start_response(比如像这样：start_response(status,response_headers))。

`status`参数是一个形式如"999 Message here"这样的状态字符串。而`response_headers`参数是包含有(header_name,header_value)的元组,用来描述HTTP响应头。可选的`exc_info`参数会在接下来的`The start_response() Callable`和 `错误处理` 两节中描述，它只有在应用程序捕获到了错误并试图在浏览器上显示错误的时候才有用。

`start_response` 可调用者(callable)必须返回一个 write(body_data) 可调用者(callable)，write(body_data)接受一个位置参数：一个将要被做为HTTP响应体的一部分输出的字符串(注意：提供可调用者 write() 只是为了支持某些现有框架的命令式输出APIs；新的应用程序或框架应当尽量避免使用，详细情况请看 Buffering and Streaming 一节。)

当应用程序被服务器调用的时候，它必须返回能生成0个或多个字符串的iterable。这可以通过几种方式来实现，比如通过返回一个包含一系列字符串的列表，又或者是通过让应用程序本身是一个能生成多个字符串的生成器(generator)，又或者是通过让应用程序本身是一个类并且它的实例是一个可迭代的(iterable)。总之，不论通过什么途径完成，应用程序对象必须总是返回一个能生成0个或多个字符串的可迭代的(iterable)。

服务器或者网关必须将产生的字符串以一种无缓冲的方式传输到客户端，总是在传完一个字符串之后再去请求下一个字符串。(换句话说，应用程序必须自己负责实现缓冲。更多关于应用程序输出应该如何处理的细节请阅读下面的 `Buffering and Streaming`章节。)

服务器或网关应该将产生的字符串当做二进制字节序列来对待：特别地，它必须确保行的结尾没有被修改。应用程序必须负责确保将要传至HTTP客户端的字符串是以与客户端匹配的编码输出(服务器/网关可能会附加HTTP传输编码，或者为了实现一些类似字节级传输(byte-range transmission)这样的HTTP的特性而进行一些转换，更多HTTP特性的细节请看下面的 Other HTTP Features )

服务器如果成功调用`len(iterable)`，它将认为此结果是正确的并且信赖这个结果。也就是说，如果应用程序返回的可迭代的(iterable)字符串提供了一个能用的`__len__()` 方法，那么服务器就肯定应用程序确实是返回了正确的结果(关于这个方法正常情况下如何被使用的请阅读 Handling the Content-Length Header )

如果应用程序返回的可迭代者(iterable)有一个叫做close()的方法，则不论当前的请求是正常结束还是由于错误而终止，服务器/网关都**必须**在结束该请求之前调用这个方法。（这么做是为了支持应用程序对资源的释放，这份规约将试图补充PEP 325中生成器的支持，以及其他有cloase()方法的通用的可迭代者(iterable))

(注意：应用程序必须在可迭代者(iterable)产生第一个主体(body)字符串之前请求`start_response()`可调用者(callable)，这样服务器才能在发送任何主体(body)内容之前发送响应头。然而这一步的调用也可能在可迭代者(iterable)第一次迭代的时候执行,所以服务器不能假定在它们开始迭代之前 start_response() 已经被调用过了)

最后要说的是，服务器和网关不能使用应用程序返回的可迭代者(iterable)的其他任何属性，除非是针对服务器或网关的特定类型的实例，比如wsgi.file_wrapper返回的“file wrapper”（阅读 Optional Platform-Specific File Handling )。通常情况下，只有在这里指定的属性，或者通过PEP 234 iteration APIs访问的属性才是可以接受的。

####environ变量
```
pass
```
HTTP_ 变量
  
  ```
pass
```

####输入和错误流
```
pass
```

####The start_response() Callable
```
pass
```

####处理Content-Length头信息
```
pass
```

#####中间件处理块边界
```
pass
```

#####可调用的write()函数 
```
pass
```

####Unicode Issues
```
pass
```

####错误处理  
```
pass
``` 

####Http1.1的长连接
```
pass
```

####其他的HTTP特性
```
pass
```
 
####线程支持(done)
```
pass
```

###实现/应用 事项
```
pass
```
####服务扩展API
```
pass
```

####应用程序配置 (done)
```
pass
```

#### URL的构建
```
pass
```
###QA问答
1.为什么evniron必须是字典？用子类(subclass)不行吗？ (Done)
用字典的原理是为了最大化地满足在服务器间的移植性。另一种选择就是定义一些字典方法的子集，并以字典的方法作为标准的便捷的接口。然而事实上，大多服务器可能只需要找到一个合适的字典就足够它们用了，并且框架的作者往往期待完整的字典特性可用，因为多半情况是这样的。但是如果有一些服务器选择不用字典，那么尽管这类服务器也“符合”规范，但会有互用性的问题出现。因此使用强制的字典简化了规范和并确保了互用性。。
 
注意，以上这些并不妨碍服务器或框架的开发者向`evnrion`字典里加入自定义的变量来提供特别的服务。我们推荐使用这种方式提供任何一种增值服务。
 
2.为什么你既能调用write()又能yield字符串/返回一个迭代器(iterable)？我们难道不应该只选择一种做法吗？ (Done)
如果我们仅仅使用迭代的做法，那么现存的框架将遭受"push"可用性的折磨。但是，如果我们只支持通过write()推送，那么服务器在传输大文件的时候性能将恶化（如果一个工作线程(worker)没有将所有的output都发送完成，那么将无法进行下一个新的request请求）。因此，我们做这样的妥协，好处是允许应用程序支持两个方法，视情况而定，并且比起需要push-only的方式来说，只给那些服务器的实现者们多了一点负担而已。
 
3.close()方法是拿来做什么的？(Done)
在应用程序执行期间，当writes完成后，应用程序可以通过一个try/finally代码块来确保资源都被释放了。但是，如果应用程序返回一个迭代(iterable)，那么在迭代器被垃圾收集器收集之前任何资源都不会被释放。这里的close()惯用法允许应用程序在一个request请求完成阶段释放重要资源，并且它向前兼容PEP 325中提到的迭代器中的try/finally.
 
4.为什么这个接口要设计地这么初级？我希望添加更多酷炫的W功能!（例如 cookies, 会话(sessions), 持久性(persistence),...） (Done)
记住，这并不是另一个Python的web框架，这仅仅是一个框架向web服务器通信的方法，反之亦然。如果你想拥有上面你说的这些特性，你需要选择一个提供这些你想要的特性的框架。并且如果这个框架让你创建一个WSGI应用程序，你将可以让它在泡在大部份支持WSGI的服务器上面。同样，一些WSGI服务器或许会通过在他们的`environ`字典里的提供的对象来提供一些额外的服务；可以参阅这些服务器具体的文档了解详情。（当然，使用这样扩展的应用程序将无法移植到其他的WSGI-based服务器上）

 
5.为什么使用CGI的变量而不是旧的HTTP头呢？并且为什么将它们和WSGI定义的变量混在一起呢？（Done)
许多现有的框架很大程序上是建立在CGI规范的基础上的，并且现有的web服务器知道如何生成CGI变量。相比之下，另一种表示到达的HTTP信息的方式不仅分散破碎更缺乏市场支持。因此使用CGI“标准”看起来是个不错的办法来最大化发挥现有的实现。至于将它们同WSGI变量混合在一起，那是因为分离他们的话会导致需要传入两个字典参数，显然这样做没什么好处。
 
6.那关于状态字符串，我们可不可以仅仅使用数字来代替，比如说传入200而不是"200 OK"？(Done)
这样做会使服务器/网关被复杂化，因为那样的话服务器/网关就需要一个数值状态和相应信息的对照表。相比之下，让应用程序或框架的作者们在他们处理专门的响应代码的时候顺便输入一些额外的信息则显得要简单地多，并且经常是现有的框架已经有一个这样的表包含这些需要的信息了。总之，权衡之后，我们认为这个让应用程序/框架来负责要比服务器或网关来负责要合适。
 
7.为什么wsgi.run_once不能保证app只运行一次？
```
pass
```
 
8.Feature x(dictionaries, callables, etc.)对于应用程序代码来说是很丑陋的，我们可以使用对象来代替吗？
 
```
pass
```

###还在讨论的提议
```
pass
```

###鸣谢 (Done)

感谢那些Web-SIG邮件组里面的人，没有他们周全的反馈，将不可能有我这篇修正草案。特别地，我要感谢：
- mod_python的作者Gregory "Grisha" Trubetskoy，是他毫不留情地指出了我的第一版草案并没有提供任何比“普通旧版的CGI”有优势的地方，他的批评促进了我去寻找更好的方法。
- Ian Bicking，是他总是唠叨着要我适当地提供多线程(multithreading)及多进程(multiprocess)相关选项，对了，他还不断纠缠我让我提供一种机制可以让服务器向应用程序提供自定义的扩展数据。
- Tony Lownds，是他提出了`start_response`函数的概念，提供给它status和headers两个参数然后返回一个write函数。他的这个想法为我后来设计异常处理功能提供了灵感，尤其是在考虑到中间件复写(overrides)应用程序的错误信息这方面。
- Alan Kennedy, 一个有勇气去尝试实现WSGI-on-Jython(在我的这份规约定稿之前)的人，他帮助我形成了“supporting older versions of Python”这一章节，以及可选的`wsgi.file_wrapper`套件。
- Mark Nottingham，是他为这份规约的HTTP RFC 发行规范做了大量的后期检查工作，特别针对HTTP/1.1特性，没有他的指出，我甚至不知道有这东西存在。

###参考文献 (Done)
[1]	The Python Wiki "Web Programming" topic ( http://www.python.org/cgi-bin/moinmoin/WebProgramming )
[2]	The Common Gateway Interface Specification, v 1.1, 3rd Draft ( http://ken.coar.org/cgi/draft-coar-cgi-v11-03.txt )
[3]	"Chunked Transfer Coding" -- HTTP/1.1, section 3.6.1 ( http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.6.1 )
[4]	"End-to-end and Hop-by-hop Headers" -- HTTP/1.1, Section 13.5.1 ( http://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html#sec13.5.1 )
[5]	mod_ssl Reference, "Environment Variables" ( http://www.modssl.org/docs/2.8/ssl_reference.html#ToC25 )
###版权声明(Done)
这篇文档被托管在Mercurial上面.
原文链接: https://hg.python.org/peps/file/tip/pep-0333.txt
