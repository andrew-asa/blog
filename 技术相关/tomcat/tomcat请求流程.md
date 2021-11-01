# tomcat 请求流程 

既然到最后了，我们来总结下tomcat是如何处理HTTP请求的：
Socket-->Http11ConnectionHandler-->Http11Processor-->CoyoteAdapter-->StandardEngineValve-->StandardHostValve-->StandardContextValve-->ApplicationFilterChain-->Servlet