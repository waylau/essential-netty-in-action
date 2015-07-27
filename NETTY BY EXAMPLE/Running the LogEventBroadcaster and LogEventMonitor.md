运行 LogEventBroadcaster 和 LogEventMonitor
====

如上所述,我们将使用 Maven 来运行应用程序。这一次你需要打开两个控制台窗口给每个项目。用 Ctrl-C 可以停止它。

首先我们将启动 LogEventBroadcaster 如清单13.4所示,除了已经构建项目以下命令即可(使用默认值):

	$ mvn exec:exec -Pchapter13-LogEventBroadcaster

和之前一样,这将通过 UDP 广播日志消息。

现在,在一个新窗口,构建和启动 LogEventMonitor 接收和显示广播消息。

Listing 13.9 Compile and start the LogEventBroadcaster

	$ mvn clean package exec:exec -Pchapter13-LogEventMonitor
	[INFO] Scanning for projects...
	[INFO]
	[INFO] --------------------------------------------------------------------
	[INFO] Building netty-in-action 0.1-SNAPSHOT
	[INFO] --------------------------------------------------------------------
	...
	[INFO]
	[INFO] --- maven-jar-plugin:2.4:jar (default-jar) @ netty-in-action ---
	[INFO] Building jar: /Users/norman/Documents/workspace-intellij/netty-in-actionprivate/
	target/netty-in-action-0.1-SNAPSHOT.jar
	[INFO]
	[INFO] --- exec-maven-plugin:1.2.1:exec (default-cli) @ netty-in-action ---
	LogEventMonitor running

当看到 “LogEventMonitor running” 说明程序运行成功了。

控制台将显示任何事件被添加到日志文件中,如下所示。消息的格式是由LogEventHandler 创建。

Listing 13.10 LogEventMonitor output

	1364217299382 [/192.168.0.38:63182] [/var/log/messages] : Mar 25 13:55:08 dev-linux
	dhclient: DHCPREQUEST of 192.168.0.50 on eth2 to 192.168.0.254 port 67
	1364217299382 [/192.168.0.38:63182] [/var/log/messages] : Mar 25 13:55:08 dev-linux
	dhclient: DHCPACK of 192.168.0.50 from 192.168.0.254
	1364217299382 [/192.168.0.38:63182] [/var/log/messages] : Mar 25 13:55:08 dev-linux
	dhclient: bound to 192.168.0.50 -- renewal in 270 seconds.
	1364217299382 [/192.168.0.38:63182] [[/var/log/messages] : Mar 25 13:59:38 dev-linux
	dhclient: DHCPREQUEST of 192.168.0.50 on eth2 to 192.168.0.254 port 67
	1364217299382 [/192.168.0.38:63182] [/[/var/log/messages] : Mar 25 13:59:38 dev-linux
	dhclient: DHCPACK of 192.168.0.50 from 192.168.0.254
	1364217299382 [/192.168.0.38:63182] [/var/log/messages] : Mar 25 13:59:38 dev-linux
	dhclient: bound to 192.168.0.50 -- renewal in 259 seconds.
	1364217299383 [/192.168.0.38:63182] [/var/log/messages] : Mar 25 14:03:57 dev-linux
	dhclient: DHCPREQUEST of 192.168.0.50 on eth2 to 192.168.0.254 port 67
	1364217299383 [/192.168.0.38:63182] [/var/log/messages] : Mar 25 14:03:57 dev-linux
	dhclient: DHCPACK of 192.168.0.50 from 192.168.0.254
	1364217299383 [/192.168.0.38:63182] [/var/log/messages] : Mar 25 14:03:57 dev-linux
	dhclient: bound to 192.168.0.50 -- renewal in 285 seconds.

若你没有访问 UNIX syslog 的权限，可以创建 自定义的文件，手动填入内容。下面是 UNIX 命令用 touch 创建一个空文件

	$ touch ~/mylog.log

再次启动 LogEventBroadcaster，设置系统属性

	$ mvn exec:exec -Pchapter13-LogEventBroadcaster -Dlogfile=~/mylog.log

当 LogEventBroadcaster 运行时，你可以手动的添加消息到文件来查看广播到 LogEventMonitor 控制台的内容。使用 echo 和输出的文件

	$ echo ’Test log entry’ >> ~/mylog.log

你可以启动任意个监视器实例，他们都会收到相同的消息。