.. _java:

Java S3 样例
================

设置
-----

下面的样例可能需要一些或者所有这里导入 \
的 java 类

.. code-block:: java

	import java.io.ByteArrayInputStream;
	import java.io.File;
	import java.util.List;
	import com.amazonaws.auth.AWSCredentials;
	import com.amazonaws.auth.BasicAWSCredentials;
	import com.amazonaws.util.StringUtils;
	import com.amazonaws.services.s3.AmazonS3;
	import com.amazonaws.services.s3.AmazonS3Client;
	import com.amazonaws.services.s3.model.Bucket;
	import com.amazonaws.services.s3.model.CannedAccessControlList;
	import com.amazonaws.services.s3.model.GeneratePresignedUrlRequest;
	import com.amazonaws.services.s3.model.GetObjectRequest;
	import com.amazonaws.services.s3.model.ObjectListing;
	import com.amazonaws.services.s3.model.ObjectMetadata;
	import com.amazonaws.services.s3.model.S3ObjectSummary;


如果你只是测试 Ceph 对象存储服务，考虑使用 \
HTTP 协议而不是 HTTPS。

首先，导入 ``ClientConfiguration`` 和 ``Protocol`` 

.. code-block:: java

	import com.amazonaws.ClientConfiguration;
	import com.amazonaws.Protocol;


然后，定义客户端配置，并为 S3 客户端添加客户 \
端配置作为一个参数

.. code-block:: java

	AWSCredentials credentials = new BasicAWSCredentials(accessKey, secretKey);
			 
	ClientConfiguration clientConfig = new ClientConfiguration();
	clientConfig.setProtocol(Protocol.HTTP);

	AmazonS3 conn = new AmazonS3Client(credentials, clientConfig);
	conn.setEndpoint("endpoint.com");


新建一个连接
---------------------

下面的代码会新建一个连接，这样你就可以和服务器交互.

.. code-block:: java

	String accessKey = "insert your access key here!";
	String secretKey = "insert your secret key here!";

	AWSCredentials credentials = new BasicAWSCredentials(accessKey, secretKey);
	AmazonS3 conn = new AmazonS3Client(credentials);
	conn.setEndpoint("objects.dreamhost.com");


列出用户的所有 bucket
---------------------

下面的代码会列出你的 bucket 的列表。
这也会打印出每个bucket的 bucket 名和创建时间。

.. code-block:: java

	List<Bucket> buckets = conn.listBuckets();
	for (Bucket bucket : buckets) {
		System.out.println(bucket.getName() + "\t" +
			StringUtils.fromDate(bucket.getCreationDate()));
	}

输出形式类似下面这样::

   mahbuckat1	2011-04-21T18:05:39.000Z
   mahbuckat2	2011-04-21T18:05:48.000Z
   mahbuckat3	2011-04-21T18:07:18.000Z


新建一个 Bucket
-----------------

下面的代码会新建一个名为 ``my-new-bucket`` 的bucket。

.. code-block:: java

	Bucket bucket = conn.createBucket("my-new-bucket");


列出 bucket 的内容
--------------------------
下面的代码会输出 bucket 内的所有对象列表。
这也会打印出每一个对象的名字、文件尺寸和\
最近修改时间。

.. code-block:: java

	ObjectListing objects = conn.listObjects(bucket.getName());
	do {
		for (S3ObjectSummary objectSummary : objects.getObjectSummaries()) {
			System.out.println(objectSummary.getKey() + "\t" +
				ObjectSummary.getSize() + "\t" +
				StringUtils.fromDate(objectSummary.getLastModified()));
		}
		objects = conn.listNextBatchOfObjects(objects);
	} while (objects.isTruncated());

输出形式类似下面这样::

   myphoto1.jpg	251262	2011-08-08T21:35:48.000Z
   myphoto2.jpg	262518	2011-08-08T21:38:01.000Z


删除 Bucket
-----------------

.. note::
   Bucket必须为空！否则它不会工作!

.. code-block:: java

	conn.deleteBucket(bucket.getName());


强制删除非空 Buckets
-----------------------------------
.. attention::
   不支持


新建一个对象
------------------

下面的代码会新建一个内容是字符串``"Hello World!"`` 的文件 ``hello.txt``。

.. code-block:: java

	ByteArrayInputStream input = new ByteArrayInputStream("Hello World!".getBytes());
	conn.putObject(bucket.getName(), "hello.txt", input, new ObjectMetadata());


修改一个对象的 ACL
----------------------

下面的代码会将对象 ``hello.txt`` 的权限变为公开可读，而将
``secret_plans.txt`` 的权限设为私有。

.. code-block:: java

	conn.setObjectAcl(bucket.getName(), "hello.txt", CannedAccessControlList.PublicRead);
	conn.setObjectAcl(bucket.getName(), "secret_plans.txt", CannedAccessControlList.Private);


下载一个对象 (到文件)
------------------------------

下面的代码会下载对象 ``perl_poetry.pdf`` 并将它存到位置
``C:\Users\larry\Documents``

.. code-block:: java

	conn.getObject(
		new GetObjectRequest(bucket.getName(), "perl_poetry.pdf"),
		new File("/home/larry/documents/perl_poetry.pdf")
	);


删除一个对象
----------------

下面的代码会删除对象 ``goodbye.txt``

.. code-block:: java

	conn.deleteObject(bucket.getName(), "goodbye.txt");


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

.. code-block:: java

	GeneratePresignedUrlRequest request = new GeneratePresignedUrlRequest(bucket.getName(), "secret_plans.txt");
	System.out.println(conn.generatePresignedUrl(request));

输出形式类似下面这样::

   https://my-bucket-name.objects.dreamhost.com/secret_plans.txt?Signature=XXXXXXXXXXXXXXXXXXXXXXXXXXX&Expires=1316027075&AWSAccessKeyId=XXXXXXXXXXXXXXXXXXX

