=========================
 认证和 ACLs
=========================

到 RADOS 网关的 Requests 请求可能已经经过验证也可能没有经 \
过验证。RGW 假设所有未经验证的请求都是由匿名用户发送的。RGW \
支持预置的 ACLs。

认证
--------------
对请求进行身份验证需要在它发送到 RGW 服务器之前就包含一个 \
access 密钥和基于散列消息验证码(HMAC)。RGW 使用兼容 S3 \
的身份验证方法。

::

	HTTP/1.1
	PUT /buckets/bucket/object.mpeg
	Host: cname.domain.com
	Date: Mon, 2 Jan 2012 00:01:01 +0000
	Content-Encoding: mpeg	
	Content-Length: 9999999

	Authorization: AWS {access-key}:{hash-of-header-and-secret}

在上述例子中，使用你自己的 access 密钥代替 ``{access-key}`` ，在其后跟着加入一个冒 \
号 (``:``)。将头字符串和 secret 密钥 hash 之后的结果替换 ``{hash-of-header-and-secret}`` ，\
这里的 secret 必须是跟你前面使用的 access 配对的密钥。

要生成头字符串和 secret 密钥的 hash 结果，你必须：

#. 获取头字符串
#. 将请求头字符串格式化为规范格式 
#. 使用 SHA-1 hashing 算法生成 HMAC
   查看 `RFC 2104`_ 和 `HMAC`_ 获取详细信息
#. 使用 base-64 将 ``hmac`` 结果再次编码

将请求头字符串格式化为规范格式: 

#. 获取所有以 ``x-amz-`` 开头的所有字段
#. 确保这些字段都是小写字母
#. 按照字母顺序将这些字段排序 
#. 将多个实例中相同的字段名称合并到一个单个字段中
   在这个字段中使用逗号分隔这些值
#. 将字段的值中的空格和换行符替换为单个空格
#. 删除冒号前后的空格
#. 在每个字段后追加一个空行
#. 将所有字段合并到头字符串中

将 HMAC 字符串使用 base-64编码的结果替换 ``{hash-of-header-and-secret}`` 

访问控制列表 (ACLs)
---------------------------

RGW 支持兼容 S3 ACL 的功能。ACL 是一个授权访问控制 \
列表，它指定了用户针对一个 bucket 或者一个对象能够执 \
行哪些操作。针对一个 bucket 或者一个对象的每次授权都 \
有不同的含义:

+------------------+--------------------------------------------------------+----------------------------------------------+
| Permission       | Bucket                                                 | Object                                       |
+==================+========================================================+==============================================+
| ``READ``         | Grantee can list the objects in the bucket.            | Grantee can read the object.                 |
+------------------+--------------------------------------------------------+----------------------------------------------+
| ``WRITE``        | Grantee can write or delete objects in the bucket.     | N/A                                          |
+------------------+--------------------------------------------------------+----------------------------------------------+
| ``READ_ACP``     | Grantee can read bucket ACL.                           | Grantee can read the object ACL.             |
+------------------+--------------------------------------------------------+----------------------------------------------+
| ``WRITE_ACP``    | Grantee can write bucket ACL.                          | Grantee can write to the object ACL.         |
+------------------+--------------------------------------------------------+----------------------------------------------+
| ``FULL_CONTROL`` | Grantee has full permissions for object in the bucket. | Grantee can read or write to the object ACL. |
+------------------+--------------------------------------------------------+----------------------------------------------+

.. _RFC 2104: http://www.ietf.org/rfc/rfc2104.txt
.. _HMAC: http://en.wikipedia.org/wiki/HMAC