### 知识点
+  打包并执行jar中的某个class
```
 mvn exec:java -Dexec.mainClass="com.xxx.xxx" -Dexec.args="-r xxx"
```
+ Maven中的SNAPSHOT版本和正式版本
  + 见[Maven中的SNAPSHOT版本和正式版本](https://www.cnblogs.com/huang0925/p/5169624.html)
  
+ 依赖本地jar
```
<dependency>
            <groupId>com.asa</groupId>
            <artifactId>com.asa.third</artifactId>
            <version>1.0.0-SNAPSHORT</version>
            <scope>system</scope>
            <systemPath>${system.lib.dir}/com.asa.third-1.0-SNAPSHOT.jar</systemPath>
        </dependency>
```
+  模板

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.xx.xx</groupId>
    <artifactId>xxxx</artifactId>
    <version>1.0</version>
    <packaging>jar</packaging>
    <dependencies>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.4</version>
        </dependency>
        <dependency>
            <groupId>commons-cli</groupId>
            <artifactId>commons-cli</artifactId>
            <version>1.3.1</version>
        </dependency>
        <dependency>
            <groupId>net.lingala.zip4j</groupId>
            <artifactId>zip4j</artifactId>
            <version>1.3.2</version>
        </dependency>
        <dependency>
            <groupId>javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.12.1.GA</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-core</artifactId>
            <version>2.6.7</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-annotations</artifactId>
            <version>2.6.7</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.6.7</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.6</source>
                    <target>1.6</target>
                </configuration>
            </plugin>

            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                <!--  可以没有-->
                    <archive>
                        <manifest>
                            <mainClass>***</mainClass>
                        </manifest>
                    </archive>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

+ 指定编译语言等级

```
<plugin>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.1</version>
      <configuration>
             <source>1.8</source>
             <target>1.8</target>
       </configuration>
</plugin>

```

+ 指定构建的资源文件

```
<build>  
  <resources>  
    <resource>  
      <directory>src/main/java</directory>  
        <includes>  
          <include>**/*.xml</include>  
        </includes>  
    </resource>  
  </resources>  
</build> 
```
或者

```
<resource>
     <directory>src/main/java</directory>
     <excludes>
           <exclude>**/*.java</exclude>
     </excludes>
</resource>
```

# Q
+ mvn package 和 mvn install 的区别