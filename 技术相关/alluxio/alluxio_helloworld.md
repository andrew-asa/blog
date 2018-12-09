# 搭建第一个alluxio用例
+ 按照alluxio帮助文档流程下载,安装[alluxio](https://www.alluxio.org/docs/1.8/cn/Getting-Started.html)

+ maven 添加依赖依赖

```
<dependency>
    <groupId>org.alluxio</groupId>
    <artifactId>alluxio-core-client-fs</artifactId>
    <version>1.8.0</version>
</dependency>
```
+ hello world java 代码

```
public class AlluxioHelloWorldTest {

    @Test
    public void helloWorldTest() throws Exception {
        //System.setProperty(PropertyKey.MASTER_HOSTNAME.getName(), "localhost");
        //System.setProperty("HADOOP_USER_NAME", "hadoop");
        //System.setProperty("HADOOP_USER_NAME", "hadoop");
        //Configuration.set(PropertyKey.MASTER_HOSTNAME, "localhost");
        //Configuration.set(PropertyKey.USER_, "192.168.0.33");
        //
        String fpath = "/helloworld";
        FileSystem fs = FileSystem.Factory.get();
        AlluxioURI path = new AlluxioURI(fpath);
        boolean exists = fs.exists(path);
        if (exists) {
            System.out.println(StringUtils.messageFormat("path {} is exists", fpath));
            FileInStream inStream = fs.openFile(path);
            MemoryFileInputStreamReader reader = new MemoryFileInputStreamReader(inStream);
            String content = reader.readAsString();
            System.out.println(content);
        } else {
            // Create a file and get its output stream
            FileOutStream out = fs.createFile(path);
            // Write data
            String str = "helloworld";
            out.write(str.getBytes());
            // Close and complete file
            out.close();
        }
    }
}
```
+ 浏览http://localhost:19999/browse?path=%2F&offset=0&limit=1 可以看到多了一个helloworld文件

# 参考资料
+ 1:[Alluxio入门大全2 http://www.winseliu.com/blog/2016/04/15/alluxio-quickstart2/](http://www.winseliu.com/blog/2016/04/15/alluxio-quickstart2/)
+ 2:[Alluxio 中文帮助文档 https://www.alluxio.org/docs/master/cn/Overview.html](https://www.alluxio.org/docs/master/cn/Overview.html)
+ 3:[Alluxio（前Tachyon）北京Meetup视频整理（上） http://www.voidcn.com/article/p-dglaltbo-qr.html](http://www.voidcn.com/article/p-dglaltbo-qr.html)