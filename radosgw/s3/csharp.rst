.. _csharp:

C# S3 样例
==============

新建一个连接
---------------------

下面的代码会新建一个连接，这样你就可以和服务器交互.

.. code-block:: csharp

	using System;
	using Amazon;
	using Amazon.S3;
	using Amazon.S3.Model;

	string accessKey = "put your access key here!";
	string secretKey = "put your secret key here!";

	AmazonS3Config config = new AmazonS3Config();
	config.ServiceURL = "objects.dreamhost.com";

	AmazonS3Client s3Client = new AmazonS3Client(
		accessKey,
		secretKey,
		config
		);


列出用户的所有 bucket
---------------------

下面的代码会列出你的 bucket 的列表。
这也会打印出每个bucket的 bucket 名和创建时间。

.. code-block:: csharp

	ListBucketsResponse response = client.ListBuckets();
	foreach (S3Bucket b in response.Buckets)
	{
		Console.WriteLine("{0}\t{1}", b.BucketName, b.CreationDate);
	}

输出形式类似下面这样::

   mahbuckat1	2011-04-21T18:05:39.000Z
   mahbuckat2	2011-04-21T18:05:48.000Z
   mahbuckat3	2011-04-21T18:07:18.000Z


新建一个 Bucket
-----------------
下面的代码会新建一个名为 ``my-new-bucket`` 的bucket。

.. code-block:: csharp

	PutBucketRequest request = new PutBucketRequest();
	request.BucketName = "my-new-bucket";
	client.PutBucket(request);

列出 bucket 的内容
--------------------------

下面的代码会输出 bucket 内的所有对象列表。
这也会打印出每一个对象的名字、文件尺寸和\
最近修改时间。

.. code-block:: csharp

	ListObjectsRequest request = new ListObjectsRequest();
	request.BucketName = "my-new-bucket";
	ListObjectsResponse response = client.ListObjects(request);
	foreach (S3Object o in response.S3Objects)
	{
		Console.WriteLine("{0}\t{1}\t{2}", o.Key, o.Size, o.LastModified);
	}

输出形式类似下面这样::

   myphoto1.jpg	251262	2011-08-08T21:35:48.000Z
   myphoto2.jpg	262518	2011-08-08T21:38:01.000Z


删除 Bucket
-----------------

.. note::

   Bucket必须为空！否则它不会工作!

.. code-block:: csharp

	DeleteBucketRequest request = new DeleteBucketRequest();
	request.BucketName = "my-new-bucket";
	client.DeleteBucket(request);


强制删除非空 Buckets
-----------------------------------

.. attention::

   不支持


新建一个对象
------------------

下面的代码会新建一个内容是字符串``"Hello World!"`` 的文件 ``hello.txt``。

.. code-block:: csharp

	PutObjectRequest request = new PutObjectRequest();
	request.BucketName      = "my-new-bucket";
	request.Key         = "hello.txt";
	request.ContentType = "text/plain";
	request.ContentBody = "Hello World!";
	client.PutObject(request);


修改一个对象的 ACL
----------------------

下面的代码会将对象 ``hello.txt`` 的权限变为公开可读，而将
``secret_plans.txt`` 的权限设为私有。

.. code-block:: csharp

	PutACLRequest request = new PutACLRequest();
	request.BucketName = "my-new-bucket";
	request.Key        = "hello.txt";
	request.CannedACL  = S3CannedACL.PublicRead;
	client.PutACL(request);

	PutACLRequest request2 = new PutACLRequest();
	request2.BucketName = "my-new-bucket";
	request2.Key        = "secret_plans.txt";
	request2.CannedACL  = S3CannedACL.Private;
	client.PutACL(request2);


下载一个对象 (到文件)
------------------------------

下面的代码会下载对象 ``perl_poetry.pdf`` 并将它存到位置
``C:\Users\larry\Documents``

.. code-block:: csharp

	GetObjectRequest request = new GetObjectRequest();
	request.BucketName = "my-new-bucket";
	request.Key        = "perl_poetry.pdf";
	GetObjectResponse response = client.GetObject(request);
	response.WriteResponseStreamToFile("C:\\Users\\larry\\Documents\\perl_poetry.pdf");


删除一个对象
----------------

下面的代码会删除对象 ``goodbye.txt``

.. code-block:: csharp

	DeleteObjectRequest request = new DeleteObjectRequest();
	request.BucketName = "my-new-bucket";
	request.Key        = "goodbye.txt";
	client.DeleteObject(request);


生成对象的下载 URLs (带签名和不带签名)
---------------------------------------------------

下面的代码会为 ``hello.txt`` 生成一个无签名为下载URL。 \
这个操作是生效是因为前面我们已经设置 ``hello.txt`` 的 \
ACL 为公开可读。下面的代码同时会为 ``secret_plans.txt`` \
生成一个有效时间是一个小时的带签名的下载 URL。带签名的下载 \
URL 在这个时间内是可用的，即使对象的权限是私有(当时间到期后 \
URL 将不可用)。

.. note::

    C# S3 库不支持生成不带签名的URLs，因此下面的实例只会展 \
    示如何生成代签名的 URLs.

.. code-block:: csharp

	GetPreSignedUrlRequest request = new GetPreSignedUrlRequest();
	request.BucketName = "my-bucket-name";
	request.Key        = "secret_plans.txt";
	request.Expires    = DateTime.Now.AddHours(1);
	request.Protocol   = Protocol.HTTP;
	string url = client.GetPreSignedURL(request);
	Console.WriteLine(url);

输出形式类似下面这样::

   http://objects.dreamhost.com/my-bucket-name/secret_plans.txt?Signature=XXXXXXXXXXXXXXXXXXXXXXXXXXX&Expires=1316027075&AWSAccessKeyId=XXXXXXXXXXXXXXXXXXX

