### 1、环境介绍

携程搭建了三个Gitlab的服务，分别给携程、集团、外部合作使用

集团主推GitLab Flow的基本型，这不要求团队具备主干开发的苛刻条件，又能让团队高效持续集成



### 2、java开发相关

携程用Nexus作为maven组件的仓库，用来存放Java构建的jar包，我们在开发过程中可以去Nexus服务上去找要的jar组件

短域名 nexus/



### 3、maven使用指南

首先是使用携程标准的settings.xml



只要项目直接或间接继承Ctrip Super POM，enforcer规则就会生效



### 4、SonarQube

是sonarsource公司推出的一个自动代码审查工具，用于检测代码中的错误、漏洞

公司采用SonarQube的社区版本对代码进行静态分析

后来从去哪儿引入了规则，搭建了公共的Sonar服务，有两套Sonar静态检查环境，用于本地开发和上线发布两种场景

代码质量检查规则由技术委员会制定

