# spring mvc 跨域问题
Spring mvc 4.2 以上的版本才提供CrossOrigin注解来方便支持，但是一开始引入的是spring 4.x 版本。
基于稳定的考虑，决定深入了解CrossOrigin的原理来自己定制注解支持


HandlerMethodReturnValueHandlerComposite 
HandlerMethodReturnValueHandler
中调用
真正会调用RequestResponseBodyMethodProcessor

初始化
RequestMappingHandlerAdapter.afterPropertiesSet





# 参考资料
+ [https://my.oschina.net/sugarZone/blog/704579](https://my.oschina.net/sugarZone/blog/704579)