## 1、环境

采用windows+docker desktop

通过docker desktop把资源都拉起来



docker原生是linux生态产物，一来linux内核技术，windows、macOS本身内核不支持容器底层能力，必须靠内嵌轻量linux环境才能跑docker，所以需要安装docker desktop，内部启用WSL2（微软内置极简linux子系统）

docker是容器引擎，负责打包成镜像、启动镜像生成容器作为独立进程运行、隔离资源，但要注意容器是一个临时的环境，删除容器会丢失内部所有数据，应通过数据卷挂载到宿主机

镜像、容器、仓库









