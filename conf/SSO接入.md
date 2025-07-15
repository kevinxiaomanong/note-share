1、引入maven依赖

如果是jdk21+springboot3 可以用1.0.1-Java21的版本

引入包之后会自动注册CtripSSOFilter，拦截路径进行登录认证，它是通过@WebFilter自助注册装配的，如果servlet版本过低，需要手动配置bean



发现了IAM权限中台



其实引入还是蛮简单的 引入依赖 然后注入进ctripSSOfilter 就可以通过

Assertion访问到map

访问不到的原因主要是看实际的请求有没有被走校验



