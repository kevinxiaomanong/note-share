### 背景

centos7已经到达其声明周期终点EOL，即官方不再提供更新和支持，这意味着我们需要计划迁移到其他有技术支持的OS版本或发行版，携程这边从Node.js20开始 基础系统将迁移到ALmaLinux9（ALmaLinux是开源的linux发行版）



发现升级到node20.11.0后报错 ERR_OSSL_EVP_UNSUPPORTED 

原因：出现这个错误是因为 node.js V17及以后版本中最近发布的OpenSSL3.0,OpenSSL3.0对允许算法和密钥大小增加了严格的限制，可能会对生态系统造成一些影响.



发现流水线跑镜像跑不通 可以在本地用 这个是intall阶段的命令

npm install --proxy=$PROXY --https-proxy=$PROXY --noproxy=$NO_PROXY --unsafe-perm复现



这个应该是peer dependency的问题，我们通过npm的overrides配置去改

```
"overrides": {
  "react": "^17.0.2",
  "react-dom": "^17.0.2"
}
```

这是build这部分的问题 npm run build && npm prune --production



通过这个需求应该总结到的东西：

1、captain的CI CD流程 其实gitlab就是按照我们自己设置的东西在跑的 

以这次前端构建为例 首先是install拉取资源 然后是build构建 然后test测试流程

最后是打包镜像推给captain 以及sonar代码质量检查



2、总结一下这次来升级nodejs遇到的问题以及解决方案

- 发现升级到node20.11.0后报错 ERR_OSSL_EVP_UNSUPPORTED 

出现这个错误是因为 node.js V17及以后版本中最近发布的OpenSSL3.0,OpenSSL3.0对允许算法和密钥大小增加了严格的限制，可能会对生态系统造成一些影响.

我们在启动脚本里添加NODE_OPTIONS='--openssl-legacy-provider' 解决

- peer dependency问题

就是有些资源依赖

peer react@">=17.0.2" from @wangeditor/editor-for-react@1.0.6

peer react@"^16.6.3" from react-file-viewer@1.2.1

可以看到 我们项目依赖的资源其实一个需要react16.6.3

一个又要求react要17.0.2，react-file-viewer@1.2.1最新的依赖版本就到react16了

所以只能忽略依赖 我们在package-json里配置override解决



3、关于window和linux不同环境设置的问题

其实我一开始设置的是SET NODE_OPTIONS=--openssl-legacy-provider &&

这里其实有两点问题，一是set是windows的cmd命令，到linux上跑shell的时候识别不了，会报错

![image-20240708153315534](D:\note\工作文档\项目类\assets\image-20240708153315534.png)

这里我们去install一下cross-env的依赖 然后用cross-env来设置环境变量即可



还有就是不可以加&& 在shell里&&会在前一个命令执行成功后再执行下一命令

用来确保只有前一个命令执行成功时才执行后续命令



### 幽灵依赖检查

所谓的幽灵依赖是指 在应用中引用 但未在package.json的dependencies中声明的依赖

这会导致维护 安全 可移植性等问题 

并且可以发现在nodejs.20起 标准流水线为buil任务默认添加了npm prune --production指令

在构建完成后 清理部署产物的开发依赖 有效减少最终部署产物的大小 加快镜像构建和部署的效率

所以我们得准确定义好依赖 以确保npm prune命令不会误删生产依赖

depscheck本地复现命令



npm install @umijs/fabric jest-environment-node carlo detect-installer puppeteer puppeteer-core cross-port-killer @ant-design/pro-card quill @@/core mockjs











