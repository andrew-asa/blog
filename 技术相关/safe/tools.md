# 常用工具、命令

### airport 
+  mac 本身自带，但是没有放在执行路径里面需要手动建立连接 `sudo ln -s /System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport /usr/local/bin/airport`
+  扫描周围无线网络 `airprt en0 scan` 或 `airport -s`SSID 是 wifi名称，RSSI 是信号强度，CHANNEL 是信道。
+  airport en0 sniff 6  
        en0 代表网卡名称 6代表信道
+  airport -z 断开当前无线网络
+  显示当前网络信息：airport -I

### 直接源码编译aircrack-ng并安装
+ 从[https://github.com/aircrack-ng/aircrack-ng.git](https://github.com/aircrack-ng/aircrack-ng.git)获取源码
+ 安装依赖`brew install autoconf automake libtool openssl shtool pkg-config hwloc pcre sqlite3 libpcap cmocka`
+ 配置`autoreconf -i`
+ 编译`make`
+ 安装`make install`
+ 添加`/usr/local/sbin`到path中，因为`airodump-ng`等会放到该目录

### aircrack-ng
  + 密码破解  ` aircrack-ng -w 01.txt 01.cap`
    
### iwconfig
   + 查看网卡支持的格式`iwconfig wlan0 mode`

### tcpdump
   + tcpdump -Ine -i en0 打开监控模式监听网卡en0
   + -n不要转换一些数值, 比如把80端口转换成http显示.
   + -i需要监控的网卡. 如果不指定, 则监控所有有效的网卡数据.
   + -c抓取多少个包后自动停止抓取
   + -s默认是只抓取96bytes的数据, 如果想要抓取更多的数据, 则要通过这指定更大的数值. 比如-s 1500抓取1500byte
   + -S默认每个包的sequence是显示相对的值, 如果想显示绝对值, 通过此选项打开.
   + -w 保存到文件中去


### 参考连接
+ []()
+ []()
+ []()
+ []()
+ []()
+ []()