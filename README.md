# PEP 333 - Python Web Server Gateway Interface v1.0 中文版
============
> 翻译自 `Python Web Server Gateway Interface v1.0`  [PEP 333 - Python Web Server Gateway Interface v1.0](https://www.python.org/dev/peps/pep-0333/)

##译者的话
>Python基础学完后，免不了要深入到Python的主流Web框架(Python科学计算那部分用不到可以先不管)，在学习Flask这些框架的过程中发现它们的底层都是WSGI协议，故决定先啃下WSGI，鉴于目前网上几乎没有(完整的)WSGI中文版，于是干脆自己翻译，这样也有助于加深自己的理解。

##内容
* [序言](#preface)
* [摘要](#abstract)
* [基本原理及目标](#基本原理及目标)  
* [规范概述](#规范概述)  
	* [应用程序/框架 端](#应用程序/框架端)
	* [服务器/网关 端](#服务器/网关端)
	* [中间件:可扮演两端角色的组件](#中间件：可扮演两端角色的组件)
* [规范细则](#规范细则)  
	* [`environ`变量](#environ变量)
		* [输入和错误流](#输入和错误流)
	* [`start_response() Callable`](#start_response()Callable)
		* [处理Content-Length头信息](#处理Content-Length头信息)
	* [缓冲和流](#缓冲和流)
		* [中间件处理块边界](#中间件处理块边界)
		* [可调用的`write()`函数](#可调用的write()函数)
	* [Unicode问题](#Unicode问题)
	* [错误处理](#错误处理)
	* [`HTTP 1.1 Expect/Continue`机制](#HTTP1.1Expect/Continue机制)
	* [HTTP的其他特性](#HTTP的其他特性)
	* [线程支持](#线程支持)
* [具体实现/应用程序](#具体实现/应用程序)
	* [服务器扩展API](#服务器扩展API)
	* [应用程序配置](#应用程序配置)
	* [URL重构](#URL重构)
	* [对Python2.2之前的版本的支持](#对Python2.2之前的版本的支)
	* [可选的平台相关的文件处理](#可选的平台相关的文件处理)
* [尚在讨论中的提议](#尚在讨论中的提议)
* [鸣谢](#鸣谢)
* [参考文献](#参考文献)
* [版权声明](#版权声明)

<a name="preface"/>
###序言
注意: 关于本规范的后续版本，请参照PEP 3333，它支持Python 3.x的更新版本及包含一些社区勘误，补充，更正的相关说明等。

<a name="preface"/>
###摘要
这份规范规定了一种在web服务器与web应用/框架之间推荐的标准接口，以确保web应用程序在不同的web服务器之间具有可移植性。

<a name="preface"/>
###基本原理及目标
Python目前拥有大量的web框架，比如 Zope, Quixote, Webware, SkunkWeb, PSO, 和Twisted Web--这里我仅列举出几个[参考文献 1](#参考文献)。这么多的选择让新手无所适从，因为总得来说，框架的选择都会反过来限制web服务器的选择。

对比之下，虽然java也拥有许多web框架，但是java的`servlet API`使得使用任何框架编写出来的应用程序都可以在任何支持` servlet API`的web服务器上运行。

服务器中这种针对python的API（不管服务器是用python写的(如: Medusa)，还是内嵌python(如: mod_python)，亦或是通过一种网关协议来调用Python(如:CGI, FastCGI等))的使用和普及，将人们从web框架的选择和web服务器的选择中分离开来，使他们可以任意选择适合他们的组合，而web服务器和web框架的开发者也能够把精力集中到各自的领域。  

基于此，这份PEP建议在web服务器和web应用/web框架之间建立一种简单通用的接口规范，即Python Web服务器网关接口(WSGI)。

但是光有这么一份规范对于改变web服务器和web应用/框架的现状是不够的，只有当那些web服务器和web框架的作者/维护者们真正地实现了WSGI，这份WSGI规范才将起到它该有的效果。  

然而，由于目前还没有任何框架或服务器实现了WSGI，而支持WSGI的框架作者们也不会得到任何直接的奖励和好处，因此，我们的WSGI必须拟定地尽可能容易实现，这样才能降低框架作者们在实现接口这件事上的初始投资成本。
 
由此可见，服务器和框架两端接口实现的简单性，对于WSGI的实用性来说，绝对是非常重要的，同时，这一点也是任何设计决策的首要依据。
 
然而需要注意的是，框架作者实现框架时的简单性和web应用程序开发者使用框架时的易用性是两回事。WSGI为框架作者提出了一套绝对只包含必需的最基本元素的接口，因为像响应对象和cookie处理等这些花哨的高级功能只会妨碍现有的框架对这些问题的处理。再说一次，WSGI的目标是使现有的web服务器和web框架之间更加方便地互联互通，而不是想创建一套新的web框架。

同时也要注意到，这个目标也限定了WSGI不会要求任何在现有的python版本里不存在的东西，因此，这一份规约中也不会推荐或要求任何新的Python标准模块，WSGI中规定的任何东西都不需要2.2.2以上版本的python支持。(当然，在未来版本的Python标准库中，若自带的标准库里的Web服务器能内嵌对这份接口的支持，那也是个不错的主意。)

除了要让现有的及将要出现的框架和服务器容易实现之外，也应该让创建诸如请求预处理器`(request preprocessors)`、响应处理器`(response postprocessors）`和其他基于WSGI的中间件组件这类事情变得简单，至于这里说的中间件组件，它们是这样一种东西：对于服务器来说他们是应用程序，而对于它们包含的应用程序来说它们又可以被看作是服务器。

如果中间件既简单又健壮，且WSGI能广泛地应用在在服务器和框架中间，那么就有可能出现全新的python web框架：一个由若干个WSGI中间件组件组成的松耦合的框架。事实上，现有框架的作者们甚至可能会选择重构他们框架中以有的服务，使它们变得更像是一些配合WSGI使用的库而不是一个巨大的框架。这样一来，web应用程序开发者们这就可以为他们想实现的特定功能选择最佳组合的组件，而不用再局限于某一个特定框架忍受它所有优缺点。

当然，就现在来说，这一天毫无疑问还要等很久。同时，对WSGI来说，让每一个框架都能在任何服务器上运行起来，又是一个充分的短期目标。
 
最后，需要指出的是，当前版本的WSGI对于一个应用程序具体应该以何种方式部署在web服务器或者服务器网关上并没有做具体说明。就现在来看，这个是需要由服务器或网关来负责定义怎么实现的。等到以后，等有了足够多的服务器/网关通过实现了WSGI并提供多样化的部署需求方面的领域经验，那么到时候也许可以产生另一份PEP来描述WSGI服务器和应用框架的部署标准。

<a name="preface"/>
###规范概述
WSGI接口可以分为两端：服务器/网关端和应用程序/Web框架端。服务器端调用一个由应用程序端提供的可调用的对象`(Callable)`，至于该对象是如何被调用的，这要取决于服务器/网关这一端。我们假定有一些服务器/网关会要求应用程序的部署人员编写一个简短的脚本来启动一个服务器/网关的实例，并提供给服务器/网关一个应用程序对象，而还有的一些服务器/网关则不需要这样，它们会需要一个配置文件又或者是其他机制来指定应该从哪里导入或者获得应用程序对象。
 
除了纯粹的服务器/网关和应用程序/框架，还可以创建叫做中间件的组件，中间件实现了这份规范当中的两端(服务器端和应用程序端)，我们可以这样解释中间件，对于包含它们的服务器，中间件是应用程序，而对于包含在服务器当中的应用程序来说，中间又扮演着服务器的角色。不仅如此，中间件还可以用来提供可扩展的API，内容转换，导航和其他有用的功能。
 
在这份规范说明书中，我们将使用的术语`"a callable`(可调用者)"它的意思是"**一个函数，方法，类，或者拥有 __call__ 方法的一个对象实例**",这取决于服务器，网关，或者应用程序根据需要而选择的合适实现技术。相反，服务器，网关，或者请求一个可调用者(callable)的应用程序必须不依赖callable(可调用对象)具体的提供方式。记住，可调用者(callable)只是被调用，不会自省(introspect)。[**译者注：introspect，自省，Python的强项之一，指的是代码可以在内存中象处理对象一样查找其它的模块和函数**]

<a name="preface"/>
####应用程序/框架 端
一个应用程序对象就是一个简单的接受2个参数的可调用对象(callable object)，这里的对象并不能理解为它真的需要一个对象实例：一个函数、方法、类、或者带有 `__call__` 方法的对象实例都可以用来当做应用程序对象。应用程序对象必须可以被多次调用，实质上所有的服务器/网关(除了CGI)都会产生这样的重复请求。
 
(注意：虽然我们把他叫做"应用程序"对象，但这并不意味着程序员要把WSGI当做API来调用！我们假定应用程序开发者将会仍然使用更高层的框架服务来开发它们的应用程序，WSGI只是一个提供给框架和服务器开发者使用的工具，它并没有打算直接向应用程序开发者提供支持。)
 
这里我们来看两个应用程序对象的示例：其中，一个是函数，另一个是类:
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

<a name="preface"/> 
####服务器/网关 端
每一次，当HTTP客户端(冲着应用程序来的)发来一个请求，服务器/网关都会调用应用程序可调用者(callable)。为了说明方便，这里有一个CGI网关，简单的说它就是个以应用程序对象为参数的函数实现，请注意，这个例子中只做了有限的错误处理，因为默认情况下没有被捕获到的异常都会被输出到`sys.stderr`并被服务器记录下来。

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

<a name="preface"/>
####中间件：可扮演两端角色的组件
注意到单个对象可以作为请求应用程序的服务器存在，也可以作为被服务器调用的应用程序存在。这样的“中间件”可以执行以下这些功能:

 - 在相应地重写`environ`变量之后，根据目标URL将请求路由到不同的应用程序对象
 - 允许多个应用程序或框架在同一个进程中并行运行
 - 通过在网络中转发请求和应答，实现负载均衡和远程处理
 - 对上下文(content)进行后加工，比如应用xsl样式表
 
中间件的存在对于"服务器/网关"和"应用程序/框架"来说是透明的，并不需要特殊的支持。希望在应用程序中加入中间件的用户只需简单得把中间件当作应用程序提供给服务器，并配置中间件组件以服务器的身份来调用应用程序。当然，中间件组件包裹的“应用程序”也可能是另外一个包裹应用程序的中间件组件，这样循环下去就构成了我们所说的"中间件栈"了。

最重要的别忘了，中间件必须遵循WSGI的服务器和应用程序两端提出的一些限制和要求，甚至有些时候,对中间件的要求比纯粹的服务器或应用程序还要严格，这些我们都会在这份规范文档中指出来。
 
这里有一个(有趣的)中间件组件的例子，它使用`Joe Strout`写的`piglatin.py`程序将text/plain的响应转换成pig latin **[译者注:意思是将英语词尾改成拉丁语式]** (注意：一个“真实”的中间件组件很可能会使用更加鲁棒的方式来检查上下文(content)的类型和下文(content)的编码。同样，这个简单的例子还忽略了一个单词还可能跨区块分割的可能性)。

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

<a name="preface"/>
###规格的详细说明
应用程序对象必须接受两个位置参数`(positional arguments)`，为了方便说明，我们不妨将它们分别命名为`environ`和`start_response`，但是这并不意味着它们必须取这两个名字。服务器或网关必须用这两个位置参数(注意不是关键字参数)来调用应用程序对象(比如，像上面展示的,调用`result = application(environ,start_response)`)

`environ`参数是一个字典对象，也是一个有着CGI风格的环境变量。这个对象必须是一个python内建的字典对象(不能是子类、用户字典(UserDict)或其他对字典对象的模仿)，应用程序必须允许以任何它需要的方式来修改这个字典， `environ`还必须包含一些特定的WSGI需要的变量(在后面章节里会提到)，有可以包含一些服务器特定的扩展变量，通过下面提到的命名规范来命名。

`start_response`参数是一个可调用者(callable)，它接受两个必须的位置参数和一个可选参数。为方便说明，我们分别将它们命名为`status`, `response_headers`,和 `exc_info` 。再强调一下，这并不是说它们必须取这些名字。应用程序必须用这些位置参数来请求可调用者(callable) start_response(比如像这样：start_response(status,response_headers))。

`status`参数是一个形式如"999 Message here"这样的状态字符串。而`response_headers`参数是包含有(header_name,header_value)的元组,用来描述HTTP响应头。可选的`exc_info`参数会在接下来的[The start_response() Callable](#The start_response() Callable)和 `错误处理` 两节中描述，它只有在应用程序捕获到了错误并试图在浏览器上显示错误的时候才有用。

`start_response` 可调用者(callable)必须返回一个 write(body_data) 可调用者(callable)，write(body_data)接受一个位置参数：一个将要被做为HTTP响应体的一部分输出的字符串(注意：提供可调用者 write() 只是为了支持某些现有框架的命令式输出APIs；新的应用程序或框架应当尽量避免使用，详细情况请看 Buffering and Streaming 一节。)

当应用程序被服务器调用的时候，它必须返回能生成0个或多个字符串的iterable。这可以通过几种方式来实现，比如通过返回一个包含一系列字符串的列表，又或者是通过让应用程序本身是一个能生成多个字符串的生成器(generator)，又或者是通过让应用程序本身是一个类并且它的实例是一个可迭代的(iterable)。总之，不论通过什么途径完成，应用程序对象必须总是返回一个能生成0个或多个字符串的可迭代的(iterable)。

服务器或者网关必须将产生的字符串以一种无缓冲的方式传输到客户端，总是在传完一个字符串之后再去请求下一个字符串。(换句话说，应用程序必须自己负责实现缓冲。更多关于应用程序输出应该如何处理的细节请阅读下面的 `Buffering and Streaming`章节。)

服务器或网关应该将产生的字符串当做二进制字节序列来对待：特别地，它必须确保行的结尾没有被修改。应用程序必须负责确保将要传至HTTP客户端的字符串是以与客户端匹配的编码输出(服务器/网关可能会附加HTTP传输编码，或者为了实现一些类似字节级传输(byte-range transmission)这样的HTTP的特性而进行一些转换，更多HTTP特性的细节请看下面的 Other HTTP Features )

服务器如果成功调用`len(iterable)`，它将认为此结果是正确的并且信赖这个结果。也就是说，如果应用程序返回的可迭代的(iterable)字符串提供了一个能用的`__len__()` 方法，那么服务器就肯定应用程序确实是返回了正确的结果(关于这个方法正常情况下如何被使用的请阅读 Handling the Content-Length Header )

如果应用程序返回的可迭代者(iterable)有一个叫做close()的方法，则不论当前的请求是正常结束还是由于错误而终止，服务器/网关都**必须**在结束该请求之前调用这个方法。（这么做是为了支持应用程序对资源的释放，这份规约将试图补充PEP 325中生成器的支持，以及其他有cloase()方法的通用的可迭代者(iterable))

(注意：应用程序必须在可迭代者(iterable)产生第一个主体(body)字符串之前请求`start_response()`可调用者(callable)，这样服务器才能在发送任何主体(body)内容之前发送响应头。然而这一步的调用也可能在可迭代者(iterable)第一次迭代的时候执行,所以服务器不能假定在它们开始迭代之前 start_response() 已经被调用过了)

最后要说的是，服务器和网关不能使用应用程序返回的可迭代者(iterable)的其他任何属性，除非是针对服务器或网关的特定类型的实例，比如wsgi.file_wrapper返回的“file wrapper”（阅读 Optional Platform-Specific File Handling )。通常情况下，只有在这里指定的属性，或者通过PEP 234 iteration APIs访问的属性才是可以接受的。


<a name="preface"/>
####environ变量

`environ`字典被用来包含这些CGI环境变量，这些变量定义可以在Common Gateway Interface specification [2]中找到。下面所列出的这些变量必须给定，除非它们的值是空字符串,在这种情况下如果下面没有特别指出的话他们会被忽略。

------
__REQUEST_METHOD__  
HTTP的请求方式, 比如 "GET" 或者 "POST"。这个参数永远不可能是空字符串，故必须给出。

------
__SCRIPT_NAME__  
URL请求中'路径'('path')的开始部分，对应了应用程序对象，这样应用程序就知道它的虚拟位置。如果该应用程序对应服务器的 根目录的话， 那么`SCRIPT_NAME`的值可能为空字符串。

------
__PATH_INFO__  
URL请求中'路径'('path')的其余部分，指定请求的目标在应用程序内部的虚拟位置。如果请求的目标是应用程序根目录并且末尾没有'/'符号结尾的话，那么`PATH_INFO`可能为空字符串 。

------
__QUERY_STRING__  
URL请求中紧跟在"?"后面的那部分，它可以为空或者不存在。

------
__CONTENT_TYPE__  
HTTP请求中`Content-Type`字段包含的所有内容，它可以为空或者不存在。

------
__CONTENT_LENGTH__  
HTTP请求中`Content-Length`字段包含的所有内容，它可以为空或者不存在。

------
__SERVER_NAME__, __SERVER_PORT__  
这两个变量可以和 SCRIPT\_NAME、PATH\_INFO  一起组成一个完整的URL。然而要注意的是，如果有出现 HTTP\_HOST，那么在重建URL请求的时候应当优先使用 HTTP\_HOST而非 SERVER\_NAME 。详细内容请阅读下面的 [URL重构](#URL重构) 这一章节 。SERVER\_NAME 和 SERVER\_PORT这两个变量永远不可能是空字符串，并且总是必须给定的。

------
__SERVER_PROTOCOL__  
客户端发送请求的时候所使用的协议版本。通常是类似"HTTP/1.0" 或 "HTTP/1.1"这样的字符串，可以被应用程序用来判断如何处理请求HTTP请求报头。（事实上我认为这个变量更应该被叫做 REQUEST\_PROTOCOL，因为这个变量代表的是在请求中使用的协议，而且看样子和服务器响应时使用的协议毫无关系。 。然而，为了保持和CGI的兼容性，这里我们还是沿用已有的名字SERVER\_PROTOCOL)

------
__HTTP_  变量组__  
这组变量对应着客户端提供的HTTP请求报头(即那些名字以 "HTTP_" 开头的变量)。这组变量的存在与否应该和HTTP请求中相对应的HTTP报头保持一致。

------
一个服务器或网关应该尽可能多地提供其他可用的CGI变量。另外，如果启用了SSL，服务器或网关应该也尽可能地提供可用的Apache SSL环境变量 [5] ，比如 HTTPS=on 和SSL\_PROTOCOL。不过要注意，任何使用了上面没有列出的变量的应用程序对不支持相关扩展的服务器来说就必然有点不可移植的缺点了。(比如，不发布文件的web服务器就没法提供一个有意义的 DOCUMENT\_ROOT 或 PATH\_TRANSLATED变量。)
 
一个支持WSGI的服务器或网关应该在文档中描述它们自己的定义的同时，适当地说明下它们可以提供些什么变量。而应用程序这边应该对所有它们要求的每一个变量的存在性进行检查，并且在检查到某些变量不存在时有备用的措施。
 
注意: 缺少变量 (比如在不需要验证的情况下的 REMOTE_USER ) 应该被排除在`environ`字典之外。同样需要注意，CGI定义的变量如果存在的话必须是字符串。任何除字符串类型以外的CGI变量的存在都是违反本规范的。

除了CGI定义的变量，`environ` 字典也可以包含任意操作系统的环境变量，并且必须包含下面这些WSGI定义的变量:

| 变量        | 变量值    |
| --------   | -----| 
| wsgi.version	| 元组tuple (1, 0)，代表WSGI版本 1.0 |   
|wsgi.url_scheme| 应用程序被调用过程中的一个字符串，表示URL中的"scheme"部分。正常情况下，它的值是"http"或者"https"，视场合而定。| 
|wsgi.input	|一个能被HTTP请求主体(body)读取的输入流(类文件对象)(服务器或者网关在读取的时候可能是按需读取，因为应用程序不定时发来请求。或者它们会预读取客户端的请求体然后缓存在内存或者磁盘中，又或者根据它自己的参数，利用其他任意的技术来提供这样一种输入流)|  
|wsgi.errors|输出流(类文件对象)，用来写错误信息的，目的是记录程序或者其他在标准化及可能的中心化错误。它应该是一个“文本模式”的流；举一个例子,应用程序应该用"\n"作为行结束符，并且默认服务器/网关能将它转换成正确的行结束符。对许多服务器来说，`wsgi.errors`是服务器的主要错误日志。当然也有其它选择，比如`sys.stderr`，或者干脆是某种日志文件。服务器的文档应当包含这类解释：比如该如何配置这些日志，又或者该从哪里去查找这些记录下来的输出。如果需要，一个服务器或网关还可以向不同的应用程序提供不同的错误流。 |
|wsgi.multithread	|如果应用程序对象同时被在同一个进程中的不同线程调用，则这个参数值应该为"true"，否则就为“false"|
|wsgi.multiprocess	|如果相同的应用程序对象同时被另外一个线程调用，则此参数值应该为"true"；否则就为"false"。|
|wsgi.run_once	|如果服务器/网关期待(但不保证)应用程序在它所在的进程生命期间只会被调用一次，则这个值应该为"true"。正常情况下，对于那些基于CGI(或类似的)的网关，这个值只会是"true"。|  

最后想说的是，这个`environ`字典有可能会包含服务器定义的变量。这些变量应该用小写，数字，点号及下划线来命名，并且必须定义一个该服务器/网关特有的前缀开头。举个例子，`mod_python`在定义变量的时候，就会使用类似`mod_python.some_variable`这样的名字。


<a name="preface"/>
#####输入和错误流
服务器提供的输入输出流必须提供以下的方法

|方法(Method)|流(Stream)|注释(Notes)|  
| --------| -----| ------|
|read(size)|input|1|
|readline()|input|1, 2|
|readlines(hint)|input|1, 3|
|\__iter__()|input| |
|flush()|errors|4|
|write(str)|errors|	
|writelines(seq)|errors|
	
以上所有方法的语义在Python Library Reference里已经写得很具体了，除了在注释栏特别标注的注意点之外。

1. 服务器读取的长度不一定非要超过客户端指定的`Content-length`, 并且如果应用程序尝试去读取超过那个点，则服务器可以模拟一个流结束（end-of-file）的条件。而应用程序这边则不应该去尝试读取比指定的CONTENT_LENGTH长度更多的数据。
2. 可选参数size是不支持用于readline()方法中的，因为它有可能给开发服务器的作者们增加复杂度，所以在实际中它不常用。
3. 请注意readlines()方法中的隐藏参数对于调用者和实现者都是可选。应用程序方可以自由选择不提供它，而服务器或网关这端也可以自由选择是否忽略它。
4. 由于错误流不能回转(rewound)，服务器和网关可以立即选择自由地继续向前写操作(forward write)，而不需要缓存。在这种情况下，flush()方法可能就是个空操作(no-op)。不过，可移植的应用程序不能假定这个输出是无缓冲的或者flush是空操作。可移植的应用程序如果需要确保输出确实已经被写入，则必须调用flush()方法。(例如：从多进程中写入同一个日志文件的时候，可以做到最小化的数据交织）
 
每一个遵循此规范的服务器都必须支持上表所列出的每一个方法。每一个遵循此规范的应用程序都不能使用除上表之外的其他方法或属性。特别需要指出的是，应用程序千万不要试图去关闭这些流，就算它们自己有对close()方法做处理。

<a name="preface"/>
####The start_response() Callable
传递给应用程序的第二个参数是一个可调用的形式，start_response（status, reponse_headers, exc_info=None）.(同所有的WSGI调用，参数位置必须是 
通过位置对应，而不是用key来对应)。 
start_response调用被用来开始HTTP相应，并且必须返回一个 write(body_data) callable(参考下面的buffering and streaming段)

此status参数是http的'status'字符，如"200 OK", "404 Not Found".也就是说,它是一个字符串组成的一个状态码和一个原因短语,按这样的顺序并用个空格分隔。 
两头不包含其他的字符或空格。（见RFC2616, 6.1.1段获取更多信息）,字符串不能包含控制字符，不能有终止符或换行符等其他组合结束的符号。

response_headers参数是个tuples（header_name, header_value）的列表list，必须是个python的list,即type(response_headers) is ListType. 
并且服务器可以随意更改其中的内容如果它需要的话，所有的header_name必须是合法的HTTP header field-name（参见RFC2616 4.2段），没有冒号或其他标点。 
所有的header_value不能包含任何控制字符，包括回车和换行。（这样的要求是为了方便那些必须检查相应头的服务器，网关，中间件，使他们所必须的解析工作复杂度降到最低）

一般来说，服务器或网关负责确保正确的头信息发送到客户端，如果应用程序(application)遗漏了必要的头信息（或其他相关的规范信息），服务器或网关须补上。 
比如：HTTP date:和Server:头信息通常是由服务器或网关提供。 
（一个要提醒给服务器/网关作者的事情: HTTP 头名称是区分大小写的，所以在检查application提供的头信息时一定要考虑大小写的问题）

应用程序和中间件禁止使用HTTP/1.1的'hop-by-hop'特性或头信息，任何在HTTP/1.0中等价或类似的特性或头信息，都会影响到客户端和服务器的持久的连接。 
这特性专属的目前的服务器，服务器/网关须要考虑考虑这一个致命错误,一个应用程序尝试发送它们,并抛出一个错误,如果他们是提供给start_response()。 
(为了解更多'hop-by-hop'的细节和特性，请参阅下面的Other HTTP Features段)。

start_response回调必须不实际地传输response头，代替的，它用来存贮头信息供服务器/网关传输，只有在应用程序第一次返回后第一次迭代的时候返回一个非空字符串， 
或者在应用程序的一个调用write()回调的时候。换句话说，response头不能被发送在没有实际的body数据前，或者是应用程序返回的迭代器终止前，（唯一的例外就是response头信息里明确包含了Content-Length为0） 
这样延迟的头信息传输是为了确保有缓存或异步的应用程序能用出错信息替换掉原来可能要发送的数据，直到最后一刻。例如应用程序可能会替换到头状态'200 OK'为'500 Internal Error', 如果当body数据是有应用程序缓存构成的但发送了错误。

exc_info参数是可选的，但如果被提供，必须是python sys.exc_info()tuple.此参数只有在start_response请求错误handler才要求被提供。如果exc_info提供了，并且还没有任何HTTP header被输出，start_response应该替换the currently-stored HTTP response headers with the newly-supplied ones,应此允许应用程序在有错误发生的情况下能够改变output。 
然而，如果exc_info被提供，并且HTTP头已经被发送，start_response必须抛出错误，并且抛出exc_info tuple，也就是下面的 
raise exc_info[0], exc_info[1], exc_info[2] 
抛出的异常会被应用程序重新捕获到，原则上应到终止应用程序。（应用程序会尝试发送error output到浏览器，一旦HTTP headers被发送，这样是不安全的）应用程序必须不捕获任何来自于start_response调用的异常，如果调用start_response用来exc_info参数，替代的做法应该允许这样的异常传回给服务器/网关。更多信息见下面的Error Handing。

应用程序可能多次调用start_response当且仅当exc_info参数提供的时候。更确切的说，如果start_response已经被当且应用程序调用过后，再次调用没有exc_info参数的start_response是个很致命的错误。（参考上面CGI网关示例,其中说明了正确的逻辑） 
注意：服务器/网关/中间件实现start_response应当确保exc_info没有持续指向任何引用，当start_response方法调用完成之后。为了避免通过traceback和frames involved创建了回环的引用，最简单的例子如下：

```python
def start_response(status, response_headers, exc_info=None):
    if exc_info:
         try:
             # do stuff w/exc_info here
         finally:
             exc_info = None    # Avoid circular ref.
 ```

<a name="preface"/>
####处理Content-Length头信息
如果应用程序没有提供Content-Length头，服务器/网关可以有好几种方式来处理它，其中最简单的就是在response完成的时候关闭客户端连接。

然而在某些情况下，服务器或网关可能会要么自己生成Content-Length头，要么至少避免了关闭客户端连接这件事。如果应用程序没有调用write()callable，并返回一个len()是1的iterable，那么服务器便可以自动地识别Content-Length长度，这是通过iterable产生出来的第一个的字符串的长度来判断的。

还有，如果服务器和客户端都支持HTTP/1.1 "分块传输编码(chunked encoding)"[3],那么服务器可以在每一次调用write()方法发送数据块(Chunk)或iterable迭代生成的字符串的时候，因此会为每个chunk数据块生成Content-Length头。这样可以让服务器保持客户端长连接，如果需要的话。注意如果真要这么做的话，服务器必须完全遵循RFC2616规范，要不然就回退到另外的策略来处理Content-Length的缺失。
 
(注意：应用程序和中间件的输出一定不能使用任何类型的Transfer-Encoding，比如chunking or gzipping等；因为在逐跳路由("hop-by-hop")操作中，这些encoding都是实际的服务器/网关的职权。详细信息参见下面的Other HTTP Features章节）


<a name="preface"/>
#####中间件处理块边界
为了更好地支持异步应用程序和服务器，中间件组件一定不能阻塞迭代，该迭代等待从应用程序的iterable中返回多个值。如果中间件需要从应用程序中累积更多的数据来生成一个输出流，那么它必须生成(yield)一个空字符串。

让我们换一种方式来表述这个需求，每一次底层应用程序生成(yields)一个值，中间件组件都必须至少生成(yields)一个值。如果中间件不能生成(yields)任何值，那么它也必须生成(yields)一个空字符串。

这个要求确保了异步的服务器和应用程序能共同协作，在需要同时提供多个应用程序实例的时候减少线程的数量。

注意,这样的要求同时也意味着一旦处于底层应用的程序返回了一个iterable，中间件就必须尽快的返回一个iterable。另外，中间件调用write() callable来传输由底层应用程序yielded的数据是不允许的。中间件仅可以使用它父服务器的write() callable来传输由底层应用程序利用中间
件提供的write() callable发送来的数据。

<a name="preface"/>
#####可调用的write()函数 
一些现有框架Apis与WSGI的一个不同处理方式是他们支持无缓存的输出，特别指出的是，他们提供一个write函数或方法来写一个无缓冲的块或数据，或者他们提供一个缓冲的write函数和一个flush机制来flush缓冲。
不幸的是，这样的APIS无法实现像WSGI这样应用程序可迭代返回值，除非使用多线程或其他的机制。
因此为了允许这些框架继续使用这些必要的API，WSGI包含一个特殊的write()调用,由start_response调用返回。
新的WSGI应用程序和框架不应该使用write()调用如果可以避免这样做。这个write()调用是完全hack用来支持必要的流能力的apis。一般来说，应用程序应该通过返回的iterable来生成输出流，因为这样可以让web服务器交错在同一个Python的其他线程上，总的来说可以为服务器提供更大的吞吐量。
这个write()调用是由start_response调用返回的，并且它接受一个单一的参数：一个将用来写入到HTTP 一部分response体中的字符串，它被看作是已经被迭代生成后的结果。换句话说，在writer()返回前，它必须保证传入的字符串要么是完全发送给客户机，或者已经缓冲了应用程序要向前的传输。（or that it is buffered for transmission while the application proceeds onward）
一个应用程序必须返回一个可迭代的对象，即使它使用write()来生成全部或部分response body。返回的可迭代可以是空的（例如一个空字符串），但是如果它不生成空字符串，那么output必须被处理，通常是由服务器/网关来做（例如它必须立即发送或加入队列中）。应用程序必须不在他们返回的iterable内调用write()。因此所有由iterable生成的字符串会在传递给write()之后发送到客户端。

<a name="preface"/>
####Unicode问题
HTTP不直接支持Unicode，同样的这份接口也不支持Unicode。所有的编码/解码应当由应用程序来处理；所有传递给服务器的或从服务器传出的字符串都必须是python标准的字节字符串而不能是Unicode对象。在要求使用字符串对象的地方使用Unicode对象，这会产生不可预料的结果。

注意 所有的作为状态或响应头传给start_response()方法的字符串在编码方面都必须遵循RFC2616的编码。也就是说，它们必须使用ISO-8859-1字符，或者使用RFC 2047 MIME编码。

Python平台的str或StringType实际上是基于Unicode(如jython,ironPython, python 3000等)，这份规约中提到的所有的"字符串"都只限定在ISO-8859-1编码规范中可表示的代码点（从\u0000-\u00FF）。如果应用程序提供的字符串包含任何其它的Unicode字符或编码点(code points)，这将是个致命的错误。同样地，服务器和网关也不能向应用程序提供任何Unicode字符。

再次声明，本规约中提到的所有的字符串都必须是str类型或StringType，不能是unicode类型或UnicodeType。并且，就本规约中所提到的“字符串”，即使一个平台支持str/StringType对象超过8比特每字符，也仅仅是该“字符串”的低8位比特被用到。

<a name="preface"/>
####错误处理

一般来说，应用程序应当自己负责捕获自己的内部错误，并向浏览器输出有用的信息。(在这一方面，应用程序自己来决定哪些信息是有用的。)

然而，要显示这些信息，并不是说应用程序真的向浏览器发送了数据，这样做的话有让response损坏的危险。因此WSGI提供了一种机制，要么允许应用程序发送它自己的错误信息，要么就自动地终止应用程序：start\_response的exc\_info参数。这里有个如何使用的例子。
```python
try:
    # regular application code here
    status = "200 Froody"
    response_headers = [("content-type", "text/plain")]
    start_response(status, response_headers)
    return ["normal body goes here"]
except:
    # XXX should trap runtime issues like MemoryError, KeyboardInterrupt
    #     in a separate handler before this bare 'except:'...
    status = "500 Oops"
    response_headers = [("content-type", "text/plain")]
    start_response(status, response_headers, sys.exc_info())
    return ["error body goes here"]
```
如果当有异常发生但是输出还没有被写入时，对start\_response的调用将正常返回，然后应用程序返回一个错误信息体发到浏览器。然而如果已经有部分输出发到浏览器了的话，则start_response会重新抛出准备好的异常。这个异常不能被应用程序捕获，所以应用程序它会异常终止。服务器/网关会捕获异常这个关键的异常并终止响应。  

服务器应该捕获任何促使应用程序终止的异常或者迭代返回的值，并记录日志。如果应用程序出错的时候已经有一部分response被发送到浏览器了，服务器或网关可以试图添加一个错误消息到output，当然前提是已经发送了的部分信息里面有指明text/* content 类型的头信息，因为那样的话服务器就知道应该如何利索地修改。

一些中间件可能会希望提供额外的异常处理服务，或拦截并替换应用程序的异常信息。在这种情况下，中间件可能选择不再重新抛出提供给start_response的exc\_info，换作抛出中间件自己特有的异常，或者在存储了提供的参数之后简单地返回而不包含异常。这将导致应用程序返回错误的body iterable（或调用write()）.允许中间件来捕获并修改错误输出。这些技术只有在应用程序的开发者们做到下面几点时才起效：  
1. 每一次当开始一个错误的响应的时候，都提供exc\_info。  
2. 当exc\_info已经提供的情况下，千万不要去捕获由start\_response产生的异常。 

<a name="preface"/>
####HTTP 1.1 Expect/Continue机制
实现了HTTP1.1的服务器/网关必须提供对HTTP1.1中"Expect/Continue"机制的透明支持，这可以通过以下几种方式来实现： 
 
1. 响应包含Expect属性的请求: 一个包含立即执行响应的"100 Continue" 100-continue请求，并且正常处理。
2. 正常处理请求，但是额外提供给应用程序一个`wsgi.input`流，当/如果应用程序第一次尝试从输入流中读取的时候就发送一个"100 Continue"响应。这个read请求必须保持阻塞状态直到客户端响应请求。  
3. 一直等待，直到客户端确认服务器不支持expect/continue特性，然后客户端自身发来的请求体。（这个方法较次，不推荐）  

注意，以上这些行为的限制不适用于HTTTP 1.O，也不适用于那些不发往应用程序对象的请求。更多HTTP 1.1 Except/Continue的信息，请参阅RFC 2616的8.2.3段和10.1.1段。

<a name="preface"/>
####HTTP的其他特性
通常来说，服务器和网关应当“装傻”并让应用程序对它们自己的output有100%的控制权。服务器/网关应当只做一些小修改并且不影响应用程序响应的语义。应用程序的开发者总是有可能通过添加中间件来额外提供一些特性，所以服务器/网关的开发者在实现服务器/网关的时候可以适当偏保守些。在某种意义上说，一个服务器应当将自己看作是一个`HTTP网关服务器(gateway server)`，应用程序则应当将自己看作是一个HTTP "源服务器(origin server)"(关于这些术语的定义，请参照RFC 2616的 1.3章节)
 
然而，由于WSGI服务器和应用程序并不通过HTTP通信，RFC 2616中提到的"逐跳路由(hop-by-hop)"并没有应用到WSGI内部通信中。因此，WSGI应用程序一定不能生成任何"逐跳路由(hop-by-hop)"头信息[4]，试图使用HTTP中要求它们生成这样的报头的特性，或者依赖任何传入的"逐跳路由(hop-by-hop)"`environ`字典中报头。WSGI服务器必须自己处理任何被支持入站的"逐跳路由(hop-by-hop)"头信息，比如为每一个传入的Transfer-Encoding解码，包括分块编码，如果有的话。
 
如果将这些原则应用到各种各样的HTTP特性中去，应当很容易搞清楚：服务器可以通过If-None-Match和If-Modified-Sine的请求头处理缓存验证这些If-None-Match和If-Modified-Sine的请求头，和一些Last-Modified和ETag响应头和一些Last-Modified和ETag响应头。然而它并不是必须要这样做，并且一个应用程序应到处理自己的缓存验证等，如果它自身相提供这样的特性，然后服务器/网关就不需要做这样的验证。
 
同样地，服务器可能会重新编码或传输编码一个应用程序的响应，但是应用程序应当对自己发送的内容做适当的编码，并且不提供transport encoding。如果客户端需要服务器可能以字节码的方式传输应用程序的响应，并且应用程序本身是不支持字节码方式。再次申明，如果需要，应用程序应当自己执行相应的方法。
 
注意，这些在应用程序上的限制不是要求应用程序为每个HTTP特性重新实现一次，许多HTTP特性可以完全或部分地由中间件来实现，这样便可以让服务器和应用程序作者在一遍又一遍的实现这些特性中解放出来。 

<a name="preface"/> 
####线程支持
线程的支持，除非不支持，否则也是取决于服务器自己的。服务器虽然可以同时并行处理多个请求，但也应当提供额外的选择让让应用程序以单线程的方式运行，这样一来 ，一些不是线程安全的应用程序或框架仍旧可以在这些服务器上运行。

<a name="preface"/>
###具体实现/应用程序
<a name="preface"/>
####服务器扩展API
一些服务器的作者可能希望暴露更多高级的API，让应用程序和框架的作者能用来做更特别的功能。例如，一个基于`mod_python`的网关可能就希望暴露部分Apache API作为WSGI的扩展。

在最简单的情况下，这只需要定义一个`environ`变量，其它的什么都不需要了，比如`mod_python.some_api`。但是，更多情况下，可能出现的中间件会就使情况变得复杂的多。例如，一个API，它提供了访问`environ`变量中出现的同一个HTTP报头，如果`environ`变量被中间件修改，则它很可能会返回不一样的值。

通常情况下，任何重复，取代或绕过部分WSGI功能的扩展API都会有与中间件组件不兼容的风险。服务器/网关开发者不能寄希望于没人使用中间件，因为有一些框架的作者们明确打算(重新)组织他们的框架，使之几乎完全就像各种中间件一样工作。

所以，为了提供最大的兼容性，提供了扩展API来取代部分WSGI功能的服务器/网关，必须设计这些API以便它们被部分替换过的API调用。例如:一个允许访问HTTP请求头的扩展API需必须要求应用程序传输当前的`environ`，以便服务器/网关可以验证那些能被API访问的HTTP头，验证它们没有被中间件修改过。如果该扩展的API不能保证它总是就HTTP报头内容同`environ`达成协议，它就必须拒绝向应用程序提供服务。例如，通过抛出一个错误，返回None来代替头信息集合，或者其它任何适合该API的东西。
 
同样地，如果扩展的API额外提供了一种方法来写response数据或是头信息，它应当要求`start_response` 这个 `callable`在应用程序能获得的扩展的服务之前被传入。如果传入的对象和最开始服务器/网关提供给应用程序的不一样，则它就不能保证正确运转并且必须拒绝给应用程序提供扩展的服务。
 
这些指南同样适用于中间件，中间件添加类似解析过的cookies信息，表单变量，会话sessions，或者类似`evniron`。特别地，这样的中间件提供的这些特性应当像操作`environ`的函数那样，而不仅仅是简单地往`evniron``里面填充值。这样有助于保证来自信息是从`evniron`里计算得来的，在所有中间件完成每一个URL重写或对`evniron`做的其它修改之后。
 
服务器/网关和中间件的开发者们遵守这些“安全扩展”规则是非常重要的，否则以后就可能出现中间件的开发者们为了确保应用程序使用他们扩展的中间件时不被绕过， 而不得不从`environ`中删除一些或者全部的扩展API这样的事情。

<a name="preface"/>
####应用程序配置
这份规格说明没有定服务器如何选择或获得一个应用程序来调用。因为这和其他一些配置选项都是高度取决于服务器的。我们期望那些服务器/网关的作者能关心并负责将这些事情文档化：比如如何配置服务器来执行一个特定的应用程序对象，以及需要带什么样的参数（如线程的选项）。
 
另一方面，Web框架的作者应当将关心这些事情并将它们文档化：比如应该怎样创建一个包裹了框架功能的应用程序对象。而已经选定了服务器和应用程序框架的用户，必须将这两者连接起来。然而，现在由于有了这两者共同的接口，使得这一切变成了一个机械式的问题，而不再是为了将新的应用程序和服务器配对组合的重大工程了。
 
最后，一些应用程序，框架，和中间件可能希望使用`evniron`字典来接受一些简单的字符串配置选项。服务器和网关应当通过允许应用程序部署者向`evniron`字典里指定特殊的名-值对(name-value pairs)对来支持这些。最简单的例子是，由于部署者原则上可以配置这些外部的信息到服务器上，或者在CGI的情况下他们可能是通过服务器的配置文件来设置。所以，可以仅仅从os.environ中复制操作系统提供的所有环境变量到environ字典中就可以了。
 
应用程序本身应该尽量保持这些需要的变量最少，因为并不是所有的服务器都支持简单地配置它们。当然，即使在最槽糕的情况下，部署一个应用程序的人还可以创建一个脚本来提供一些必要的选项值:
```python
from the_app import application

def new_app(environ, start_response):
    environ['the_app.configval1'] = 'something'
    return application(environ, start_response)
```
但是，大多数现有的应用程序和框架很大可能只需用到environ里面的唯一一个配置值，用来指示它们的应用程序或框架特有的配置文件位置。（当然，应用程序应当缓存这些配置，以避免每次调用都重复读取）

<a name="preface"/>
####URL的构建
如果应用程序希望重建一个请求的完整URL,可以使用下面的算法，该算法由lan Bicking**[译者注：此大神乃pip，virtualenv的作者]**提供：
```python
from urllib import quote
url = environ['wsgi.url_scheme']+'://'

if environ.get('HTTP_HOST'):
    url += environ['HTTP_HOST']
else:
    url += environ['SERVER_NAME']

    if environ['wsgi.url_scheme'] == 'https':
        if environ['SERVER_PORT'] != '443':
           url += ':' + environ['SERVER_PORT']
    else:
        if environ['SERVER_PORT'] != '80':
           url += ':' + environ['SERVER_PORT']

url += quote(environ.get('SCRIPT_NAME', ''))
url += quote(environ.get('PATH_INFO', ''))
if environ.get('QUERY_STRING'):
    url += '?' + environ['QUERY_STRING']
```
注意，通过这种方式重建出来的URL可能跟客户端真实发过来的URI有些差别。举个例子，服务器重写规则可能会修改客户端原始请求的URL以便让它看起来更规范。

<a name="preface"/>
####对Python2.2之前的版本的支持

一些服务器，网关或者应用程序可能希望对Python2.2之前的版本提供支持。尤其在目标平台是Jython时显得特别重要，因为在我写这篇文档的时候，还没有一个生产版本的Jython 2.2。

对于服务器和网关来说，这是相当直接的：打算使用Python 2.2之前版本的服务器和网关，只需简单地限定它们自己只使用标准的"for"循环来迭代应用程序返回来的任何iterable即可。这是能在代码级别确保2.2版本之前的迭代器协议(后续会讲)跟“现在的”迭代器协议(参照PEP234)兼容的唯一方法。

(需要注意的是，这个技巧当然只针对那些Python写的服务器，网关，或者中间件。置于如何正确地在其他语言中使用迭代器协议则不再我们这份PEP的讨论范围之内。)

而对于应用程序，要提供对Python2.2之前的版本的支持则会稍微复杂些：

由于Python 2.2之前，文件并不是可迭代的，故你不能返回一个文件对象并期望它能像iterable一样工作。(总体上，你也不能这么做，因为大部分情况下这样做的表现很糟糕)。可以使用`wsgi.file_wrapper`或者一个应用程序特有的文件包装类。(参照`Optional Platform-Specific File Handling`章节获取更多关于`sgi.file_wrapper`的信息，该章节包含了一个教你如何将一个文件包装成一个iterable）

如果你想返回一个定制的iterable，那么它必须实现2.2版本之前的迭代器协议。也就是说，提供一个` __getitem__`方法来接收一个整形的键值，然后在所有数据都取完的时候抛出一个IndexError异常。（注意，使用内置的序列类型也是可行的，因为它也实现了这个迭代器协议)

最后，如果中间件也希望对Python2.2之前的版本提供支持，迭代应用程序返回的所有值或它自己返回一个iterable(又或者两者都有),那么这些中间件必须遵循以上提到的这些建议。

(另外，为了支持Python2.2之前的版本，毫无疑问，任何服务器，网关，应用程序，或者中间件必须只能使用该版本有的语言特性，比如用1和0，而不是True和False，类似这样的)

<a name="preface"/>
###QA问答
1.为什么evniron必须是字典？用子类(subclass)不行吗？
用字典的原理是为了最大化地满足在服务器间的移植性。另一种选择就是定义一些字典方法的子集，并以字典的方法作为标准的便捷的接口。然而事实上，大多服务器可能只需要找到一个合适的字典就足够它们用了，并且框架的作者往往期待完整的字典特性可用，因为多半情况是这样的。但是如果有一些服务器选择不用字典，那么尽管这类服务器也“符合”规范，但会有互用性的问题出现。因此使用强制的字典简化了规范和并确保了互用性。。
 
注意，以上这些并不妨碍服务器或框架的开发者向`evnrion`字典里加入自定义的变量来提供特别的服务。我们推荐使用这种方式提供任何一种增值服务。
 
2.为什么你既能调用write()又能yield字符串/返回一个迭代器(iterable)？我们难道不应该只选择一种做法吗？
如果我们仅仅使用迭代的做法，那么现存的框架将遭受"push"可用性的折磨。但是，如果我们只支持通过write()推送，那么服务器在传输大文件的时候性能将恶化（如果一个工作线程(worker)没有将所有的output都发送完成，那么将无法进行下一个新的request请求）。因此，我们做这样的妥协，好处是允许应用程序支持两个方法，视情况而定，并且比起需要push-only的方式来说，只给那些服务器的实现者们多了一点负担而已。
 
3.close()方法是拿来做什么的？
在应用程序执行期间，当writes完成后，应用程序可以通过一个try/finally代码块来确保资源都被释放了。但是，如果应用程序返回一个迭代(iterable)，那么在迭代器被垃圾收集器收集之前任何资源都不会被释放。这里的close()惯用法允许应用程序在一个request请求完成阶段释放重要资源，并且它向前兼容PEP 325中提到的迭代器中的try/finally.
 
4.为什么这个接口要设计地这么初级？我希望添加更多酷炫的W功能!（例如 cookies, 会话(sessions), 持久性(persistence),...） (Done)
记住，这并不是另一个Python的web框架，这仅仅是一个框架向web服务器通信的方法，反之亦然。如果你想拥有上面你说的这些特性，你需要选择一个提供这些你想要的特性的框架。并且如果这个框架让你创建一个WSGI应用程序，你将可以让它在泡在大部份支持WSGI的服务器上面。同样，一些WSGI服务器或许会通过在他们的`environ`字典里的提供的对象来提供一些额外的服务；可以参阅这些服务器具体的文档了解详情。（当然，使用这样扩展的应用程序将无法移植到其他的WSGI-based服务器上）

5.为什么使用CGI的变量而不是旧的HTTP头呢？并且为什么将它们和WSGI定义的变量混在一起呢？
许多现有的框架很大程序上是建立在CGI规范的基础上的，并且现有的web服务器知道如何生成CGI变量。相比之下，另一种表示到达的HTTP信息的方式不仅分散破碎更缺乏市场支持。因此使用CGI“标准”看起来是个不错的办法来最大化发挥现有的实现。至于将它们同WSGI变量混合在一起，那是因为分离他们的话会导致需要传入两个字典参数，显然这样做没什么好处。
 
6.那关于状态字符串，我们可不可以仅仅使用数字来代替，比如说传入200而不是"200 OK"？
这样做会使服务器/网关被复杂化，因为那样的话服务器/网关就需要一个数值状态和相应信息的对照表。相比之下，让应用程序或框架的作者们在他们处理专门的响应代码的时候顺便输入一些额外的信息则显得要简单地多，并且经常是现有的框架已经有一个这样的表包含这些需要的信息了。总之，权衡之后，我们认为这个让应用程序/框架来负责要比服务器或网关来负责要合适。
 
7.为什么wsgi.run_once不能保证app只运行一次？
因为它仅仅只是是建议应用程序应当"装备但不经常运行(rig for infrequent running)"。这是因为应用程序框架在操作缓存，会话这些东西的时候有多种模式。在"multiple run"模式下，这样的框架可能会预先加载缓存，并且在每个request请求之后可能不会有写操作，如写入logs或session数据到硬盘上等。在"single run"模式下，这样的框架没有预加载，避免每个request请求之后flush所有必要的写操作。
 
然而，为了验证在后者的模式下应用程序或框架的正确操作，可能会必要地（或是权宜之计）不止一次调用它。因此，一个应用程序不应当仅仅因为设置了wsgi.run_once为True就假定它肯定不会再次运行，
 
8.在应用程序代码里使用Feature x(dictionaries, callables, etc.)这些特性很丑陋，难道我们不可以使用对象来代替吗？(done)
 
WSGI中所有这些特性的实现选择都是专门为了从另外一个特性中解耦合的；将这些特性重新组装到一个封装完好的对象中只会在一定程度上加大写服务器/网关的难度，并且会在希望写一个中间件来只代替/修改一小部分整体功能的时候，难度上会上升一个数量级。

本质上，中间件希望有个"职责连"模式，凭这个它可以在一些功能中看成是一个"handler"，而同时允许其他功能保持不变。这样的要求如果用普通的python对象是比较难实现的，如果接口想要保持可扩展性的话。例如，你必须使用`__getattr__`或者`__getattribut__`的重写来确保这些扩展（比如未来的WSGI版本定义的变量）是被通过的。
 
这种类型的代码是出了名的难以保证100%正确的，并且极少人愿意自己重写。他们倾向于简单地复制别人的实现，但当别人修改了另外一个极端情况时却未能及时更新自己的拷贝。
 
进一步讲，这样必的样本代码将是纯碎的消费税,一个纯粹由中间件开发者承担的开发者消费税，目的仅仅是为了能给应用程序框架开发者支持稍微”漂亮“点的API而已。但是，应用框架开发者往往只会更新一个框架来支持WSGI，这只占他们所有框架的非常有限的部分。这很可能是他们的第一个(也可能是唯一一个)WSGI实现，因此他们很有可能去实现这份现成的规范。这样，花时间利用对象的属性或诸如此类的东西让这些API看起来"更漂亮"，似乎对本文读者们来说是浪费时间。
 
我们鼓励那些希望在直接的Web应用程序编程(相对于web框架开发)中有更漂亮的(或是改进的)WSGI接口的人，鼓励他们去开发APIs或者框架来包装WSGI使WSGI对那些应用程序开发者更便利。这样的话，WSGI就不仅可以在底层维持对服务器或中间件的便利性，而同时对应用程序开发者来说又不会显得"丑陋"。

<a name="preface"/>
###尚在讨论中的提议
下面这些项都还正在Web-SIG或其他地方讨论中，或者说还在PEP作者的计划清单中：

- `wsgi.input`是否改成一个迭代器而不是一个文件？这对于那些异步应用程序和分块编码的输入流是有帮助的。 
- 我们正在讨论将可选的扩展，它们将用来暂停一个应用程序输出的迭代，直到输入可用或者发生一个回调。
- 添加一个章节，关于同步vs.异步应用程序和服务器，相关的线程模型，以及这方面的问题/设计目标

<a name="preface"/>
###鸣谢
感谢那些Web-SIG邮件组里面的人，没有他们周全的反馈，将不可能有我这篇修正草案。特别地，我要感谢：  
- mod_python的作者Gregory "Grisha" Trubetskoy，是他毫不留情地指出了我的第一版草案并没有提供任何比“普通旧版的CGI”有优势的地方，他的批评促进了我去寻找更好的方法。  
- Ian Bicking，是他总是唠叨着要我适当地提供多线程(multithreading)及多进程(multiprocess)相关选项，对了，他还不断纠缠我让我提供一种机制可以让服务器向应用程序提供自定义的扩展数据。  
- Tony Lownds，是他提出了`start_response`函数的概念，提供给它status和headers两个参数然后返回一个write函数。他的这个想法为我后来设计异常处理功能提供了灵感，尤其是在考虑到中间件复写(overrides)应用程序的错误信息这方面。  
- Alan Kennedy, 一个有勇气去尝试实现WSGI-on-Jython(在我的这份规范定稿之前)的人，他帮助我形成了“supporting older versions of Python”这一章节，以及可选的`wsgi.file_wrapper`套件。  
- Mark Nottingham，是他为这份规范的HTTP RFC 发行规范做了大量的后期检查工作，特别针对HTTP/1.1特性，没有他的指出，我甚至不知道有这东西存在。  

<a name="preface"/>
###参考文献
[1]	The Python Wiki "Web Programming" topic ( http://www.python.org/cgi-bin/moinmoin/WebProgramming )  
[2]	The Common Gateway Interface Specification, v 1.1, 3rd Draft ( http://ken.coar.org/cgi/draft-coar-cgi-v11-03.txt )  
[3]	"Chunked Transfer Coding" -- HTTP/1.1, section 3.6.1 ( http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.6.1 )  
[4]	"End-to-end and Hop-by-hop Headers" -- HTTP/1.1, Section 13.5.1 ( http://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html#sec13.5.1 )  
[5]	mod_ssl Reference, "Environment Variables" ( http://www.modssl.org/docs/2.8/ssl_reference.html#ToC25 )  

<a name="preface"/>
###版权声明
这篇文档被托管在Mercurial上面.  
原文链接: https://hg.python.org/peps/file/tip/pep-0333.txt    

---

###译者注
> - 更新时间：`2014-12-21`
- 本人翻译的初衷是为了自身学习和记录，翻译不好或有误的地方，欢迎在我的Github上 [Pull Request](https://github.com/mainframer/PEP333-zh-CN/pulls)。
