# CoyoteAdapter

Tomcat 中维护了一个 Mapper，在收到 Http 请求后，会去 Mapper 中查找对应的 Host、Context 和 Wrapper，放到 request 的 MappingData 中一路带下去。

postParseRequest

```
connector.getService().getMapper().map(serverName, decodedURI,
                    version, request.getMappingData());
```


## 参考连接
[Tomcat的Mapper机制](https://fdx321.github.io/2017/06/29/%E3%80%90Tomcat%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E3%80%9112-Tomcat%E7%9A%84Mapper%E6%9C%BA%E5%88%B6/)