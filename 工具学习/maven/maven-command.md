# maven 常用命令

+ 打包不运行测试 `mvn package -Dmaven.test.skip=true`
+ maven 打包指定路径 `maven package -f pom.xml`
+ maven 打包并打包依赖 `mvn install dependency:copy-dependencies`