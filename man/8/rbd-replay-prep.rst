:orphan:

===============================================================
 rbd-replay-prep -- 对捕捉到的用于重放的 RBD 工作载荷进行预处理
===============================================================

.. program:: rbd-replay-prep

摘要
====

| **rbd-replay-prep** [ --window *seconds* ] [ --anonymize ] *trace_dir* *replay_file*


描述
====

**rbd-replay-prep** 可处理原始 RBD 映像跟踪文件，以便用于 **rbd-replay** 。


选项
====

.. option:: --window seconds

   超过 'seconds' 秒之后的请求被认为是独立的。

.. option:: --anonymize

   匿名化映像和快照名。

.. option:: --verbose

   把所有处理过的事件打印到控制台。


样例
====

预处理 workload1-trace 以便重放： ::

       rbd-replay-prep workload1-trace/ust/uid/1000/64-bit workload1


使用范围
========

**rbd-replay-prep** 是 Ceph 的一部分，这是个大规模可伸缩、开源、分布式的\
存储系统，更多信息参见 http://ceph.com/docs 或 http://docs.ceph.org.cn/。


另请参阅
========

:doc:`rbd-replay <rbd-replay>`\(8),
:doc:`rbd <rbd>`\(8)
