.. _perl:

Perl S3 样例
================

新建一个连接
---------------------

下面的代码会新建一个连接，这样你就可以和服务器交互.

.. code-block:: perl

	use Amazon::S3;
	my $access_key = 'put your access key here!';
	my $secret_key = 'put your secret key here!';

	my $conn = Amazon::S3->new({
		aws_access_key_id     => $access_key,
		aws_secret_access_key => $secret_key,
		host                  => 'objects.dreamhost.com',
		secure                => 1,
		retry                 => 1,
	});


列出用户的所有 bucket
---------------------

下面的代码会获取你拥有的 `Amazon::S3::Bucket`_ 类型的对象列表。
这也会打印出每个bucket的 bucket 名和创建时间。

.. code-block:: perl

	my @buckets = @{$conn->buckets->{buckets} || []};
	foreach my $bucket (@buckets) {
		print $bucket->bucket . "\t" . $bucket->creation_date . "\n";
	}

输出形式类似下面这样::

   mahbuckat1	2011-04-21T18:05:39.000Z
   mahbuckat2	2011-04-21T18:05:48.000Z
   mahbuckat3	2011-04-21T18:07:18.000Z


新建一个 Bucket
-----------------

下面的代码会新建一个名为 ``my-new-bucket`` 的bucket。

.. code-block:: perl

	my $bucket = $conn->add_bucket({ bucket => 'my-new-bucket' });


列出 bucket 的内容
--------------------------

下面的代码会输出 bucket 内的所有对象列表。
这也会打印出每一个对象的名字、文件尺寸和\
最近修改时间。

.. code-block:: perl

	my @keys = @{$bucket->list_all->{keys} || []};
	foreach my $key (@keys) {
		print "$key->{key}\t$key->{size}\t$key->{last_modified}\n";
	}

输出形式类似下面这样::

   myphoto1.jpg	251262	2011-08-08T21:35:48.000Z
   myphoto2.jpg	262518	2011-08-08T21:38:01.000Z


删除 Bucket
-----------------

.. note::
   Bucket必须为空！否则它不会工作!

.. code-block:: perl

	$conn->delete_bucket($bucket);


强制删除非空 Buckets
-----------------------------------

.. attention::

   perl 模块 `Amazon::S3`_ 不支持


新建一个对象
------------------

下面的代码会新建一个内容是字符串``"Hello World!"`` 的文件 ``hello.txt``。

.. code-block:: perl

	$bucket->add_key(
		'hello.txt', 'Hello World!',
		{ content_type => 'text/plain' },
	);


修改一个对象的 ACL
----------------------

下面的代码会将对象 ``hello.txt`` 的权限变为公开可读，而将
``secret_plans.txt`` 的权限设为私有。

.. code-block:: perl

	$bucket->set_acl({
		key       => 'hello.txt',
		acl_short => 'public-read',
	});
	$bucket->set_acl({
		key       => 'secret_plans.txt',
		acl_short => 'private',
	});


下载一个对象 (到文件)
------------------------------

下面的代码会下载对象 ``perl_poetry.pdf`` 并将它存到位置
``C:\Users\larry\Documents``

.. code-block:: perl

	$bucket->get_key_filename('perl_poetry.pdf', undef,
		'/home/larry/documents/perl_poetry.pdf');


删除一个对象
----------------

下面的代码会删除对象 ``goodbye.txt``

.. code-block:: perl

	$bucket->delete_key('goodbye.txt');

生成对象的下载 URLs (带签名和不带签名)
---------------------------------------------------
下面的代码会为 ``hello.txt`` 生成一个无签名为下载URL。 \
这个操作是生效是因为前面我们已经设置 ``hello.txt`` 的 \
ACL 为公开可读。下面的代码同时会为 ``secret_plans.txt`` \
生成一个有效时间是一个小时的带签名的下载 URL。带签名的下载 \
URL 在这个时间内是可用的，即使对象的权限是私有(当时间到期后 \
URL 将不可用)。

.. note::
   `Amazon::S3`_ 模块还不支持生成下载 URL ,所以 \
   我们要使用另一个模块。不幸的是，大多数生成这些  \ 
   URL 的模块都假设你使用的是亚马逊，所以我们不得不 \
   使用一个更模糊的模块 `Muck::FS::S3`_ 。这应该和 \
   Amazon S3 的 perl 模块的样例一样，但是这个样例模 \
   块不在CPAN 中。所以，你可以使用CPAN安装 `Muck::FS::S3`_，\
   或手动安装 Amazon 的 S3模块示例。如果你遵循手册的 \
   路线，你可以在下面的示例中删除 ``Muck::FS::``。

.. code-block:: perl

	use Muck::FS::S3::QueryStringAuthGenerator;
	my $generator = Muck::FS::S3::QueryStringAuthGenerator->new(
		$access_key,
		$secret_key,
		0, # 0 means use 'http'. set this to 1 for 'https'
		'objects.dreamhost.com',
	);

	my $hello_url = $generator->make_bare_url($bucket->bucket, 'hello.txt');
	print $hello_url . "\n";

	$generator->expires_in(3600); # 1 hour = 3600 seconds
	my $plans_url = $generator->get($bucket->bucket, 'secret_plans.txt');
	print $plans_url . "\n";

输出形式类似下面这样::

   http://objects.dreamhost.com:80/my-bucket-name/hello.txt
   http://objects.dreamhost.com:80/my-bucket-name/secret_plans.txt?Signature=XXXXXXXXXXXXXXXXXXXXXXXXXXX&Expires=1316027075&AWSAccessKeyId=XXXXXXXXXXXXXXXXXXX


.. _`Amazon::S3`: http://search.cpan.org/~tima/Amazon-S3-0.441/lib/Amazon/S3.pm
.. _`Amazon::S3::Bucket`: http://search.cpan.org/~tima/Amazon-S3-0.441/lib/Amazon/S3/Bucket.pm
.. _`Muck::FS::S3`: http://search.cpan.org/~mike/Muck-0.02/

