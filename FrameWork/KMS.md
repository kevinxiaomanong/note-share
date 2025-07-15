### 概述

KMS是一款安全管理类服务，主要是帮助创建和管理秘钥

可以将敏感的凭证信息托管到KMS平台，保护数据安全性



### 需求案例--token放入kms

首先我们去九凤信安平台上看，可以看到具体哪些地方被扫描到了 然后做整改 

这里我们看到 是qconfig上的token和url被扫到了

接着看应用哪里用到了

{"access_token":"${token}","request_body":{"indexAlias":"itdb_emloyee","queryJson":{"query":{"multi_match":{"query":"${query}","type":"best_fields","fields":["empcode","empaccount","displayname","c_name","businessname","leadercode","pinyin"],"operator":"and"}}},"type":"emloyee"}}

这里的token是用 qconfig里替换的 所以就是我们的token和url应该放进kms里即可

token：bc09364b283bacf4785de2f148939882

url: http://osg.ops.ctripcorp.com/api/esQuery

注意区分开外部token和内部token















