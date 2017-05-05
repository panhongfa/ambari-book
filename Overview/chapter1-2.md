# 基本架构
Ambari架构采用的是Server/Client的模式，主要由两部分组成：ambari-agent和ambari-server。还提供一个界面清亮的管理监控页面ambari-web，这些页面由ambari-server提供。ambari-server开放了REST API，这些API也主要分两大类，其中一类为ambari-web提供管理监控服务，另一类用于与ambari-agent交互，接受ambari-agent向ambari-server发送的心跳请求。下图是Ambari的系统架构。其中master模块接受API和Agent Interface的请求，完成ambari-server的集中式管理监控逻辑，而每个agent节点只负责所在节点的状态采集及维护。