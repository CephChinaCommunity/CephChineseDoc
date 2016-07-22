.. _php:

PHP S3 样例
===============

新建一个连接
---------------------

下面的代码会新建一个连接，这样你就可以和服务器交互.

.. code-block:: php

	<?php
	define('AWS_KEY', 'place access key here');
	define('AWS_SECRET_KEY', 'place secret key here');
	define('AWS_CANONICAL_ID', 'your DHO Username');
	define('AWS_CANONICAL_NAME', 'Also your DHO Username!');
	$HOST = 'objects.dreamhost.com';

	// require the amazon sdk for php library
	require_once 'AWSSDKforPHP/sdk.class.php';

	// Instantiate the S3 class and point it at the desired host
	$Connection = new AmazonS3(array(
		'key' => AWS_KEY,
		'secret' => AWS_SECRET_KEY,
		'canonical_id' => AWS_CANONICAL_ID,
		'canonical_name' => AWS_CANONICAL_NAME,
	));
	$Connection->set_hostname($HOST);
	$Connection->allow_hostname_override(false);

	// Set the S3 class to use objects.dreamhost.com/bucket
	// instead of bucket.objects.dreamhost.com
	$Connection->enable_path_style();


列出用户的所有 bucket
---------------------
下面的代码会生成 CFSimpleXML 类型的对象列表，\
它代表你拥有的bucket。这也会打印出每个bucket \
的 bucket 名和创建时间。

.. code-block:: php

	<?php
	$ListResponse = $Connection->list_buckets();
	$Buckets = $ListResponse->body->Buckets->Bucket;
	foreach ($Buckets as $Bucket) {
		echo $Bucket->Name . "\t" . $Bucket->CreationDate . "\n";
	}

输出形式类似下面这样::

   mahbuckat1	2011-04-21T18:05:39.000Z
   mahbuckat2	2011-04-21T18:05:48.000Z
   mahbuckat3	2011-04-21T18:07:18.000Z


新建一个 Bucket
-----------------

下面的代码会新建一个名为 ``my-new-bucket`` 的bucket，\
并返回一个 ``CFResponse`` 对象

.. note::

   这个命令需要指定 region 作为第二个参数，
   所以我们使用 ``AmazonS3::REGION_US_E1``, 因为它的内容是 ``''``

.. code-block:: php

	<?php
	$Connection->create_bucket('my-new-bucket', AmazonS3::REGION_US_E1);


列出 bucket 的内容
-----------------------

下面的代码会输出 ``CFSimpleXML`` 对象的一个数组，\
它代表了bucket内的对象。这也会打印出每一个对象的名字、\
文件尺寸和最近修改时间。

.. code-block:: php

	<?php
	$ObjectsListResponse = $Connection->list_objects($bucketname);
	$Objects = $ObjectsListResponse->body->Contents;
	foreach ($Objects as $Object) {
		echo $Object->Key . "\t" . $Object->Size . "\t" . $Object->LastModified . "\n";
	}

.. note::

   如果在这个 bucket 中有超过1000个对象，你需要检查 \
   $ObjectListResponse->body->isTruncated 的值，\
   然后使用列出的最后一个 key 的名字再次运行。一直这样 \
   做直到 isTruncated 的值不再是 true。

如果该 bucket 内有文件，输出形式类似下面这样::

   myphoto1.jpg	251262	2011-08-08T21:35:48.000Z
   myphoto2.jpg	262518	2011-08-08T21:38:01.000Z


删除 Bucket
-----------------

下面的代码会删除名为 ``my-old-bucket`` 的 bucket，并返回一个
``CFResponse`` 对象。

.. note::

   Bucket必须为空！否则它不会工作!

.. code-block:: php

	<?php
	$Connection->delete_bucket('my-old-bucket');


强制删除非空 Buckets
----------------------------------

下面的代码会删除一个 bucket 即使它不是空的。

.. code-block:: php

	<?php
	$Connection->delete_bucket('my-old-bucket', 1);


新建一个对象
------------------

下面的代码会新建一个内容是字符串``"Hello World!"`` 的文件 ``hello.txt``。

.. code-block:: php

	<?php
	$Connection->create_object('my-bucket-name', 'hello.txt', array(
		'body' => "Hello World!",
	));


修改一个对象的 ACL
----------------------

下面的代码会将对象 ``hello.txt`` 的权限变为公开可读，而将
``secret_plans.txt`` 的权限设为私有。

.. code-block:: php

	<?php
	$Connection->set_object_acl('my-bucket-name', 'hello.txt', AmazonS3::ACL_PUBLIC);
	$Connection->set_object_acl('my-bucket-name', 'secret_plans.txt', AmazonS3::ACL_PRIVATE);


删除一个对象
----------------

下面的代码会删除对象 ``goodbye.txt``

.. code-block:: php

	<?php
	$Connection->delete_object('my-bucket-name', 'goodbye.txt');


下载一个对象 (到文件)
------------------------------

下面的代码会下载对象 ``perl_poetry.pdf`` 并将它存到位置
``C:\Users\larry\Documents``

.. code-block:: php

	<?php
	$FileHandle = fopen('/home/larry/documents/poetry.pdf', 'w+');
	$Connection->get_object('my-bucket-name', 'poetry.pdf', array(
		'fileDownload' => $FileHandle,
	));


生成对象的下载 URLs (带签名和不带签名)
---------------------------------------------------

下面的代码会为 ``hello.txt`` 生成一个无签名为下 \
载URL。这个操作会生效是因为前面我们已经设置 \
``hello.txt`` 的ACL 为公开可读。下面的代码同时会 \
为 ``secret_plans.txt`` 生成一个有效时间是一个小 \
时的带签名的下载 URL。带签名的下载URL 在这个时间内 \
是可用的，即使对象的权限是私有(当时间到期后URL 将不 \
可用)。

.. code-block:: php

	<?php
	my $plans_url = $Connection->get_object_url('my-bucket-name', 'hello.txt');
	echo $plans_url . "\n";
	my $secret_url = $Connection->get_object_url('my-bucket-name', 'secret_plans.txt', '1 hour');
	echo $secret_url . "\n";

输出形式类似下面这样::

   http://objects.dreamhost.com/my-bucket-name/hello.txt
   http://objects.dreamhost.com/my-bucket-name/secret_plans.txt?Signature=XXXXXXXXXXXXXXXXXXXXXXXXXXX&Expires=1316027075&AWSAccessKeyId=XXXXXXXXXXXXXXXXXXX

