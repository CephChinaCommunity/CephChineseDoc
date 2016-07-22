.. _ruby:

Ruby `AWS::SDK`_ 样例 (aws-sdk gem ~>2)
===========================================

设置
----

你可以用全局方法配置连接：

.. code-block:: ruby

	Aws.config.update(
		endpoint: 'https://objects.dreamhost.com.',
		access_key_id: 'my-access-key',
		secret_access_key: 'my-secret-key',
		force_path_style: true, 
		region: 'us-east-1'
	)


并把客户端对象实例化：

.. code-block:: ruby

    	s3_client = Aws::S3::Client.new

列出用户的所有 bucket
-------------------------------

下面的代码会列出你的 bucket 的列表。
这也会打印出每个bucket的 bucket 名和创建时间。

.. code-block:: ruby

	s3_client.list_buckets.buckets.each do |bucket|
		puts "#{bucket.name}\t#{bucket.creation_date}"
	end

输出形式类似下面这样::

   mahbuckat1	2011-04-21T18:05:39.000Z
   mahbuckat2	2011-04-21T18:05:48.000Z
   mahbuckat3	2011-04-21T18:07:18.000Z


新建一个 Bucket
-------------------

下面的代码会新建一个名为 ``my-new-bucket`` 的bucket。

.. code-block:: ruby

	s3_client.create_bucket(bucket: 'my-new-bucket')

如果你想要新建一个私有bucket: 

`acl` 选项接收的参数有: # private, public-read, public-read-write, authenticated-read

.. code-block:: ruby

	s3_client.create_bucket(bucket: 'my-new-bucket', acl: 'private')


列出 bucket 的内容
--------------------------

下面的代码会输出 bucket 内的所有对象列表。
这也会打印出每一个对象的名字、文件尺寸和\
最近修改时间。

.. code-block:: ruby

	s3_client.get_objects(bucket: 'my-new-bucket').contents.each do |object|
		puts "#{object.key}\t#{object.size}\t#{object.last-modified}"
	end

如果 bucket 内有文件，输出形式类似下面这样::

   myphoto1.jpg	251262	2011-08-08T21:35:48.000Z
   myphoto2.j‰g	262518	2011-08-08T21:38:01.000Z


删除桶
------
.. note::
   Bucket必须为空！否则它不会工作!

.. code-block:: ruby

	s3_client.delete_bucket(bucket: 'my-new-bucket')


强行删除非空 bucket
-------------------------
首先，你需要清空这个 bucket:

.. code-block:: ruby

	Aws::S3::Bucket.new('my-new-bucket', client: s3_client).clear!
	
然后删除这个 bucket

.. code-block:: ruby

	s3_client.delete_bucket(bucket: 'my-new-bucket')


新建一个对象
------------------

下面的代码会新建一个内容是字符串``"Hello World!"`` 的文件 ``hello.txt``。

.. code-block:: ruby

	s3_client.put_object(
		key: 'hello.txt',
		body: 'Hello World!',
		bucket: 'my-new-bucket',
		content_type: 'text/plain'
	)


修改一个对象的 ACL
----------------------

下面的代码会将对象 ``hello.txt`` 的权限变为公开可读，而将
``secret_plans.txt`` 的权限设为私有。

.. code-block:: ruby

	s3_client.put_object_acl(bucket: 'my-new-bucket', key: 'hello.txt', acl: 'public-read')

	s3_client.put_object_acl(bucket: 'my-new-bucket', key: 'private.txt', acl: 'private')


下载一个对象 (到文件)
------------------------------

下面的代码会下载对象 ``perl_poetry.pdf`` 并将它存到位置
``C:\Users\larry\Documents``

.. code-block:: ruby

	s3_client.get_object(bucket: 'my-new-bucket', key: 'poetry.pdf', response_target: '/home/larry/documents/poetry.pdf')


删除一个对象
----------------

下面的代码会删除对象 ``goodbye.txt``

.. code-block:: ruby

	s3_client.delete_object(key: 'goodbye.txt', bucket: 'my-new-bucket')


生成对象的下载 URLs (带签名和不带签名)
---------------------------------------------------

下面的代码会为 ``hello.txt`` 生成一个无签名为下载URL。 \
这个操作是生效是因为前面我们已经设置 ``hello.txt`` 的 \
ACL 为公开可读。下面的代码同时会为 ``secret_plans.txt`` \
生成一个有效时间是一个小时的带签名的下载 URL。带签名的下载 \
URL 在这个时间内是可用的，即使对象的权限是私有(当时间到期后 \
URL 将不可用)。

.. code-block:: ruby

	puts Aws::S3::Object.new(
		key: 'hello.txt',
		bucket_name: 'my-new-bucket',
		client: s3_client
	).public_url

	puts Aws::S3::Object.new(
		key: 'secret_plans.txt',
		bucket_name: 'hermes_ceph_gem',
		client: s3_client
	).presigned_url(:get, expires_in: 60 * 60)

输出形式类似下面这样::

   http://objects.dreamhost.com/my-bucket-name/hello.txt
   http://objects.dreamhost.com/my-bucket-name/secret_plans.txt?Signature=XXXXXXXXXXXXXXXXXXXXXXXXXXX&Expires=1316027075&AWSAccessKeyId=XXXXXXXXXXXXXXXXXXX

.. _`AWS::SDK`: http://docs.aws.amazon.com/sdkforruby/api/Aws/S3/Client.html



Ruby `AWS::S3`_ 样例 (aws-s3 gem)
=====================================

新建一个连接
---------------------

下面的代码会新建一个连接，这样你就可以和服务器交互.

.. code-block:: ruby

	AWS::S3::Base.establish_connection!(
		:server            => 'objects.dreamhost.com',
		:use_ssl           => true,
		:access_key_id     => 'my-access-key',
		:secret_access_key => 'my-secret-key'
	)


列出用户的所有 bucket
---------------------

下面的代码会列出一个 `AWS::S3::Bucket`_  对象类型的列表，这代 \
表你拥有的bucket。这也会打印出每个bucket的 bucket 名和创建时间。

.. code-block:: ruby

	AWS::S3::Service.buckets.each do |bucket|
		puts "#{bucket.name}\t#{bucket.creation_date}"
	end

输出形式类似下面这样::

   mahbuckat1	2011-04-21T18:05:39.000Z
   mahbuckat2	2011-04-21T18:05:48.000Z
   mahbuckat3	2011-04-21T18:07:18.000Z


新建一个 Bucket
-----------------

下面的代码会新建一个名为 ``my-new-bucket`` 的bucket。

.. code-block:: ruby

	AWS::S3::Bucket.create('my-new-bucket')


列出 bucket 的内容
--------------------------

下面的代码会输出 bucket 内的所有对象列表。
这也会打印出每一个对象的名字、文件尺寸和\
最近修改时间。

.. code-block:: ruby

	new_bucket = AWS::S3::Bucket.find('my-new-bucket')
	new_bucket.each do |object|
		puts "#{object.key}\t#{object.about['content-length']}\t#{object.about['last-modified']}"
	end

如果 bucket 内有文件，输出形式类似下面这样::

   myphoto1.jpg	251262	2011-08-08T21:35:48.000Z
   myphoto2.jpg	262518	2011-08-08T21:38:01.000Z


删除 Bucket
-----------------
.. note::
   Bucket必须为空！否则它不会工作!

.. code-block:: ruby

	AWS::S3::Bucket.delete('my-new-bucket')


强制删除非空 Buckets
-----------------------------------

.. code-block:: ruby

	AWS::S3::Bucket.delete('my-new-bucket', :force => true)


新建一个对象
------------------

下面的代码会新建一个内容是字符串``"Hello World!"`` 的文件 ``hello.txt``。

.. code-block:: ruby

	AWS::S3::S3Object.store(
		'hello.txt',
		'Hello World!',
		'my-new-bucket',
		:content_type => 'text/plain'
	)


修改一个对象的 ACL
----------------------

下面的代码会将对象 ``hello.txt`` 的权限变为公开可读，而将
``secret_plans.txt`` 的权限设为私有。

.. code-block:: ruby

	policy = AWS::S3::S3Object.acl('hello.txt', 'my-new-bucket')
	policy.grants = [ AWS::S3::ACL::Grant.grant(:public_read) ]
	AWS::S3::S3Object.acl('hello.txt', 'my-new-bucket', policy)

	policy = AWS::S3::S3Object.acl('secret_plans.txt', 'my-new-bucket')
	policy.grants = []
	AWS::S3::S3Object.acl('secret_plans.txt', 'my-new-bucket', policy)


下载一个对象 (到文件)
------------------------------

下面的代码会下载对象 ``perl_poetry.pdf`` 并将它存到位置
``C:\Users\larry\Documents``

.. code-block:: ruby

	open('/home/larry/documents/poetry.pdf', 'w') do |file|
		AWS::S3::S3Object.stream('poetry.pdf', 'my-new-bucket') do |chunk|
			file.write(chunk)
		end
	end


删除一个对象
----------------

下面的代码会删除对象 ``goodbye.txt``

.. code-block:: ruby

	AWS::S3::S3Object.delete('goodbye.txt', 'my-new-bucket')


生成对象的下载 URLs (带签名和不带签名)
---------------------------------------------------

下面的代码会为 ``hello.txt`` 生成一个无签名为下载URL。 \
这个操作是生效是因为前面我们已经设置 ``hello.txt`` 的 \
ACL 为公开可读。下面的代码同时会为 ``secret_plans.txt`` \
生成一个有效时间是一个小时的带签名的下载 URL。带签名的下载 \
URL 在这个时间内是可用的，即使对象的权限是私有(当时间到期后 \
URL 将不可用)。

.. code-block:: ruby

	puts AWS::S3::S3Object.url_for(
		'hello.txt',
		'my-new-bucket',
		:authenticated => false
	)

	puts AWS::S3::S3Object.url_for(
		'secret_plans.txt',
		'my-new-bucket',
		:expires_in => 60 * 60
	)

输出形式类似下面这样::

   http://objects.dreamhost.com/my-bucket-name/hello.txt
   http://objects.dreamhost.com/my-bucket-name/secret_plans.txt?Signature=XXXXXXXXXXXXXXXXXXXXXXXXXXX&Expires=1316027075&AWSAccessKeyId=XXXXXXXXXXXXXXXXXXX

.. _`AWS::S3`: http://amazon.rubyforge.org/
.. _`AWS::S3::Bucket`: http://amazon.rubyforge.org/doc/

