# wifi相关

## 零散笔记
+ 破解步骤
   + 干扰，使对方手机重新连接为了更方便的进行抓包 
   + 抓包。主要是抓握手包。成功握手的包。
      + tcpdump -Ine -i en0 -w dump.cap
   + 密码破解。
      + 查找有握手的ssid ` aircrack-ng dump.cap` 
      + 密码破解`aircrack-ng -w dic.txt -b xxx-xxx-sid ./dump.cap`
   

        
## Q & A
+ WPA和WPA2的区别
+ 什么抓握手包需要调试成为监控模式
   + A：内核ieee80211驱动会将数据包转换成802.3，然后才上传到网络协议栈，但是，当无线网卡为monitor模式时，不会做转换。[5]
+  各种不同的抓包模式都有什么用？
+  是否是抓成功连接的包的时候里面有私钥以及客户端成功加密之后的密码，然后主要是运行自己的密码进行相应的算法进行解密即可？
+  常用弱密码?
  A:[2]
+ 断开wifi连接，发送假数据包？
  A:[3]
+ 查看接入点上的连接电脑情况？
+ 抓包抓到有一次握手信息就停止，同时不断的进行干扰网络？同时过渡掉没有握手的包
+ 能否一边抓包一边进行网络干扰？
+ 密码生成器
+ 哪些信息是有用的，可以请求远程服务器进行密码的破解?
+ IVs是什么？和handshake一样有用？
+ WEP and WPA PSK (WPA 1 and 2)区别？是否都能够进行快速破解

  
  
  
##  参考资料
+ 1:[https://blog.csdn.net/jpiverson/article/details/22663101](https://blog.csdn.net/jpiverson/article/details/22663101)
+ 2:[https://github.com/danielmiessler/SecLists](https://github.com/danielmiessler/SecLists)
+ 3:[https://github.com/unixpickle/JamWiFi](https://github.com/unixpickle/JamWiFi)
+ 4:[https://martinsjean256.wordpress.com/2018/02/12/hacking-aircrack-ng-on-mac-cracking-wi-fi-without-kali-in-parallels/](https://martinsjean256.wordpress.com/2018/02/12/hacking-aircrack-ng-on-mac-cracking-wi-fi-without-kali-in-parallels/)
+ 5:[https://www.jianshu.com/p/f6bef3635ee4](https://www.jianshu.com/p/f6bef3635ee4)
+ 6:[Wi-Fi渗透测试工具 http://sec-redclub.com/archives/553/](http://sec-redclub.com/archives/553/)
+ []()
+ []()
+ []()
+ []()
+ []()