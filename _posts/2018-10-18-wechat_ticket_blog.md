---
layout: post
title: '微信抢票总结博客'
subtitle: '部分功能点实现及bug处理'
tags: django 微信公众号 单元测试
---

# 微信抢票开发

在给定框架内，通过实现要求相应的接口来完成一个有在线抢票，查票，退票等功能的测试公众号。博客将从以下几个方面进行经验总结：

## API实现

后端接口的实现在技术方面问题较少，实现接口需要继承的`APIView`类与`WeChatHandler`类已经实现基础的封装，在实现api时需先了解类已经提供的接口。

* `APIView ` : `do_dispatch`已经实现对请求内容的解析，对于请求中所给的参数，例如`openid`，使用`self.input['openid']`即可查看请求参数的内容。

  继承这个类的子类需要实现get和post方法以响应GET和POST请求，且定义的方法只需返回要求结果的data段即可。

* `WeChatHandler` : 同样已经实现对input的解析，此外还可以通过`self.user`获得当前微信用户的信息。

  继承这个类的子类需要实现check方法来定义使用此handler来处理请求的条件，实现handle方法来定义响应请求的具体方式。需要返回的数据可以通过已经实现的`reply_text`, `reply_news`等方法进行返回

此外，框架也提供了`BaseError`类来提供对异常信息的包装。

* `BaseError` : 框架已经提供三个继承自该类的Error来返回信息，开发者也可以直接通过错误代码`code`和`msg`直接实例化一个错误对象。

子类内部的操作主要为对已定义的三个数据库的操作，其中为了提高查询数据库的效率，将Ticket中的activity外键改为存储activity_id。请求响应的实现难度并不大，主要用到的方法有`Model.objects.get(key=value)`, `Model.objects.create()`, `Model.objects.save()`等。

此外实现过程中要注意对数据访问合法性的检查，在访问数据无权获得或不存在时应返回相应的信息；此外还要在方法内部对逻辑进行检查，如请求的电子票和和活动是否对应等。

##单元测试

本次开发中需要进行单元测试的主要有两类接口，即继承自`APIView`类与`WeChatHandler`类的子类，在进行测试时，遇到的错误少数来自子类内部实现逻辑的错误，此类错误通常比较容易通过修改内部逻辑进行解决，更多的错误来自django，MySQL，微信接口中的某些特性，此类错误需要花很多精力来进行更多对框架进行调研才能解决，由于遇到的错误种类较杂，因此大致分为对上述两个类进行测试时遇到的错误。

* `APIView`

  对继承此类的子类进行测试时需要用到的工具主要为`django.test.TestCase`, `django.test.Client`等，上述工具提供了模拟请求以及检查返回结果的方法。具体测试时，测试类应继承自`django.test.TestCase`类，并手动构造测试数据与测试输入，并利用`assertEqual`等方法测试返回信息。以下为一下在此部分测试中踩到的坑：

  * 数据库对多语言的支持

    由于编码问题，如果希望平台的请求，数据的储存等支持中文，则需要在创建MySQL数据库时设置编码为utf-8，且在WeChatTicket文件夹下的settings.py的`DATABASE['default']`中加入`'OPTIONS': {'charset': 'utf8'}`。

  * 在服务器存储图片路径错误

    在向域名后连接图片的存储路径时，路径无需从服务器的根路径下开始，且图片应存储到static文件夹下。

  * 时间戳存储失败

    无论请求体中发送的时间是整数还是时间格式类型，在后端接收到的参数均为字符串，因此在使用时间戳创建`datetime`时，应先进行类型转换。

  * 登出操作失败

    这是处理时间最长的一个问题…我们使用了`django.contrib.auth`来进行登录状态的确认以及登录登出的操作，但是当测试按照文档要求的形式发送请求时，一定会报出`You cannot access body after reading from request's data stream`的错误，经过调查，此错误和中间件的使用以及django自己对request的处理规则有关，而具体原理还不清楚，但存在可行的解决办法，即在logout的post请求中随便加入一部分信息，如`c.post('/api/a/logout',{'whatever':'somestuff'})`即可使logout接口通过测试（逻辑实现无误）。

* `WeChatHandler`

  此部分测试遇到的逻辑性错误不多，而很多时间花在寻找如何测试这些接口的方法上：

  继承此类的子类测试原理与上一种大致相同，但由于微信接口的设置，此类接口的request和response的数据部分均存为xml格式，因此在传入请求参数或检查返回结果时应自己定义方法将样例dict等数据转为xml格式的bytes数据。

对于开发中的测试方法编写，压力测试，集成开发环境搭建等方面，请参见本组其他同学的总结博客，下附链接：

[hongfz16](https://hongfz16.github.io)

[shadowiterator](https://shadowiterator.github.io)

[ydcfwzy](https://ycdfwzy.github.io)

