# Failure Detector (FD)协议用于识别组内崩溃的成员

+ 如果在timeout指定的毫秒内没有接收到某个成员的应答，并且在尝试了max_tries 指定的次数后，那么这个成员会被标记为可疑，并将被GMS协议从集群中清除。
+ 每个成员向其右侧的邻居（当前view的成员列表中，该成员的下一个成员。列表中最后的成员的右侧邻居是列表的第一个成员）发送带有 FdHeader.HEARTBEAT头的消息。
+ 当邻居收到这个消息后，它会应答带有FdHeader.HEARTBEAT_ACK头的消息。
+ 每当收到应 答时，FD协议的last_ack属性会被更新成当前的时间，num_tries也会设置为0。
+ 如果当前时间和last_ack之差大于timeout指 定的毫秒数，那么FD协议会最多尝试max_tries 指定的次数，如果仍然没有收到应答，那么这个邻居会被标记为可疑。