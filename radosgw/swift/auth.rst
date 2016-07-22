======
 认证
======

需要认证的 Swift API 请求必须在请求头里带上 \
``X-Storage-Token`` 认证令牌。此令牌可以从 \
RADOS 网关、或别的认证器获取，要从 RADOS 网 \
关获取的话需创建用户，例如： ::

	sudo radosgw-admin user create --uid="{username}" --display-name="{Display Name}"

For details on RADOS Gateway administration, see `radosgw-admin`_. 

.. _radosgw-admin: ../../../man/8/radosgw-admin/ 

获取认证
--------

To authenticate a user, make a request containing an ``X-Auth-User`` and a
``X-Auth-Key`` in the header.
要对用户进行身份认证，需要构建一个请求头中有 ``X-Auth-User`` \
和 ``X-Auth-Key`` 的请求

语法
~~~~~~

::

    GET /auth HTTP/1.1
    Host: swift.radosgwhost.com
    X-Auth-User: johndoe
    X-Auth-Key: R7UUOLFDI2ZI9PRCQ53K

请求头
~~~~~~~~~~~~~~~

``X-Auth-User`` 

:描述: 用来认证的 RADOS 网关用户的用户名
:类型: String
:是否必需: Yes

``X-Auth-Key`` 

:描述: RADOS 网关用户的密钥
:类型: String
:是否必需: Yes


响应头
~~~~~~~~~~~~~~~~

服务器的响应应该包含一个 ``X-Auth-Token`` 值。\
响应也可能包含一个 ``X-Storage-Url`` ，它提供 \
``{api version}/{account}`` 前缀，在整个API \
文档的其他请求中指定的前缀。


``X-Storage-Token`` 

:描述:  在请求中 ``X-Auth-User`` 指定的用户的认证令牌
:类型: String


``X-Storage-Url`` 

:描述: URL 和为用户准备的 ``{api version}/{account}`` 路径.
:类型: String

一个典型的响应类似下面这样:: 

	HTTP/1.1 204 No Content
	Date: Mon, 16 Jul 2012 11:05:33 GMT
  	Server: swift
  	X-Storage-Url: https://swift.radosgwhost.com/v1/ACCT-12345
	X-Auth-Token: UOlCCC8TahFKlWuv9DB09TWHF0nDjpPElha0kAa
	Content-Length: 0
	Content-Type: text/plain; charset=UTF-8