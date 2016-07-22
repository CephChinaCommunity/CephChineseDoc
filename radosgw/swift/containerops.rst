======================
 容器操作
======================

一个容器是一种用来存储数据对象的机制。一个帐户可 \
以有很多容器，但容器名称必须是唯一的。这个API允 \
许客户端创建一个容器，设置访问控制和元数据，检索 \
一个容器的内容，和删除一个容器。因为这个 API 的 \
请求涉及到一个特定用户的帐户相关信息，因此在这个 \
API 中的所有请求都必须经过身份验证，除非一个容器 \
的访问控制故意设置为公开访问。(即允许匿名的请求)。

.. note:: Amazon S3 API 使用 'bucket' 这个词来描述一个 \
   数据容器。当你听到有人在 Swift API 中提到一个 'bucket' \
   时，'bucket' 这个词可以被理解为相当于术语 'container‘ 。
   
对象存储的一个特点就是它不支持分层路径或目录。相反，\
它支持由一个或者多个容器组成一个层级，其中每个容器包 \
含若干对象。RADOS 网关的 兼容 Swift API支持 \
'pseudo-hierarchical containers' 的概念，\
就是一个使用对象的命名方式来模拟一个容器(或目录)实际 \
上没有实现一个存储系统层次。你可以使用 pseudo-hierarchical \
名字来给对象命名。
(比如：photos/buildings/empire-state.jpg)，但容 \
器的名字不能包含一个正斜杠 (``/``) 字符。


新建一个容器
==================

要创建一个新容器，需要构建一个 ``PUT`` 请求，带上 \
API 版本、账户和新容器的名称。容器名称必须是唯一的，\
不能包含斜杠 (/)字符，且应该小于256字节。在请求头信 \
息中你也可以包括访问控制和元数据等头信息。这个操作是 \
幂等的，也就是说,如果你构建一个请求来创建一个已经存在 \
的容器，它将返回一个 HTTP 202 返回代码，但不会创建另 \
一个容器。

语法
~~~~~~

::

	PUT /{api version}/{account}/{container} HTTP/1.1
	Host: {fqdn}
	X-Auth-Token: {auth-token}
	X-Container-Read: {comma-separated-uids}
	X-Container-Write: {comma-separated-uids}
	X-Container-Meta-{key}: {value}


头信息
~~~~~~~

``X-Container-Read``

:描述: 有该容器的读权限的用户的 IDs 
:类型: 逗号分隔的用户 ID 字符串值.
:是否必需: No

``X-Container-Write``

:描述: 有该容器的写权限的用户的 IDs 
:类型: 逗号分隔的用户 ID 字符串值.
:是否必需: No

``X-Container-Meta-{key}``

:描述:  用户定义元数据的 key，一个任意的字符串值。
:类型: String
:是否必需: No


HTTP 响应
~~~~~~~~~~~~~

如果具有相同名称的容器已经存在，并且该用户是 \
该容器的所有者，则这个操作会成功。否则操作将 \
会失败。

``409``

:描述: 容器已经存在，且归一个不同的用户的所有
:状态码: ``BucketAlreadyExists``




列出容器内的对象
==========================

要列出一个容器内的所有对象，构建一个 ``GET`` 请求，带有 \
API 版本、帐户和容器的名称。您可以指定查询参数来过滤完整 \
列表，或置空该参数以返回该容器内存储的前10000个对象的名称 \
的列表。


语法
~~~~~~

::

   GET /{api version}/{container} HTTP/1.1
  	Host: {fqdn}
	X-Auth-Token: {auth-token}


参数
~~~~~~~~~~

``format``

:描述: 定义结果的格式 
:类型: String
:有效值: ``json`` | ``xml``
:是否必需: No

``prefix``

:描述: 限制所有结果的对象名以这个指定的前缀开始
:类型: String
:是否必需: No

``marker``

:描述: 返回的结果的列表大于 marker 的值
:类型: String
:是否必需: No

``limit``

:描述: 限制返回的结果的列表为这个指定的值
:类型: Integer
:有效范围: 0 - 10,000
:是否必需: No

``delimiter``

:描述: 前缀和剩余对象名之间的分隔符
:类型: String
:是否必需: No

``path``

:描述: 对象的 pseudo-hierarchical 路径
:类型: String
:是否必需: No


响应实体
~~~~~~~~~~~~~~~~~

``container``

:描述: 容器. 
:类型: Container

``object``

:描述: 容器内的对象
:类型: Container

``name``

:描述: 容器内的对象的名称
:类型: String

``hash``

:描述: 对象内容的 hash 代码
:类型: String

``last_modified``

:描述: T对象内容的最近修改时间
:类型: Date

``content_type``

:描述: 对象内容的类型
:类型: String



更新一个容器的 ACLs
=========================

当用户创建一个容器的时候，默认情况下用户有读和写这个容器 \
的权限。要允许其他用户读一个容器的内容或者写如对象到一个 \
容器，必须专门为该用户启用相关权限。你也可以在 ``X-Container-Read`` \
或者  ``X-Container-Write`` 设置中指定他们的值为 ``*`` 。\
这样可以有效地使所有用户都可以读或写这个容器。设置 ``*`` \
使得容器变为公开。也就是说允许匿名用户从容器中读取或想容器 \
写入到数据。


语法
~~~~~~

::

   POST /{api version}/{account}/{container} HTTP/1.1
   Host: {fqdn}
	X-Auth-Token: {auth-token}
	X-Container-Read: *
	X-Container-Write: {uid1}, {uid2}, {uid3}

请求头
~~~~~~~~~~~~~~~

``X-Container-Read``

:描述: 有该容器的读权限的用户的 IDs 
:类型: 逗号分隔的用户 ID 字符串值.
:是否必需: No

``X-Container-Write``

:描述: 有该容器的写权限的用户的 IDs 
:类型: 逗号分隔的用户 ID 字符串值.
:是否必需: No


添加/更新容器元数据
=============================

要向容器添加元数据，需要构建一个 ``POST`` 请求，\
带上 API 版本、账户和容器的名字。你必须有对你想 \
添加或者更新元数据的容器的写权限。

语法
~~~~~~

::

   POST /{api version}/{account}/{container} HTTP/1.1
   Host: {fqdn}
	X-Auth-Token: {auth-token}
	X-Container-Meta-Color: red
	X-Container-Meta-Taste: salty

请求头
~~~~~~~~~~~~~~~

``X-Container-Meta-{key}``

:描述:  用户定义的元数据的 key,一个任意的字符串值。
:类型: String
:是否必需: No



删除一个容器
==================

要删除一个容器，构建一个使 ``DELETE`` 请求，带上 \
API 版本、账户和容器的名称。容器必须是空的。如果你 \
想检查容器是否为空，对容器执行一个 ``HEAD`` 请求。\
一旦你确认成功删除了容器，就可以重用容器的名字。

语法
~~~~~~

::

	DELETE /{api version}/{account}/{container} HTTP/1.1
	Host: {fqdn}
	X-Auth-Token: {auth-token}    


HTTP 响应
~~~~~~~~~~~~~

``204``

:描述: 容器已经被删除
:状态码: ``NoContent``

