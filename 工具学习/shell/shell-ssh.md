# ssh 相关
## 免密登录节点
+ 目标节点 ssh-keygen -t rsa -P ""
+ 本机节点 ssh-copy-id root@nodeip

## 设置默认登录名
+ vim 〜/.ssh/config

```
Host example                  // 名称
    HostName example.net      // ip
    User example              // 某人用户名
```
或者

```
Host *                        // 名称
    User example              // 某人用户名
```

## 常用
+ 退出ssh `logout`
+ scp
    + 从远程复制资源到本地路径`scp root@10.6.159.147:/opt/soft/demo.tar /opt/soft/`
    + 上传本地文件到远程指定目录`scp /opt/soft/demo.tar root@10.6.159.147:/opt/soft/scptest`
    + 上传本地目录到远程机器指定目录 `scp -r /opt/soft/test root@10.6.159.147:/opt/soft/scptest`



## 参考连接
+ [https://blog.csdn.net/feinifi/article/details/78213297](https://blog.csdn.net/feinifi/article/details/78213297)
+ [https://codeday.me/bug/20170409/9747.html](https://codeday.me/bug/20170409/9747.html)