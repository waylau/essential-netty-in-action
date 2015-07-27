EventLog 的 POJO
====

在消息应用里面，数据一般以 POJO 形式呈现。这可能保存配置或处理信息除了实际的消息数据。在这个应用程序里，消息的单元是一个“事件”。由于数据来自一个日志文件，我们将称之为 LogEvent。

清单13.1显示了这个简单的POJO的细节。

Listing 13.1 LogEvent message

	public final class LogEvent {
	    public static final byte SEPARATOR = (byte) ':';
	
	    private final InetSocketAddress source;
	    private final String logfile;
	    private final String msg;
	    private final long received;
	
	    public LogEvent(String logfile, String msg) { //1
	        this(null, -1, logfile, msg);
	    }
	
	    public LogEvent(InetSocketAddress source, long received, String logfile, String msg) {  //2
	        this.source = source;
	        this.logfile = logfile;
	        this.msg = msg;
	        this.received = received;
	    }
	
	    public InetSocketAddress getSource() { //3
	        return source;
	    }
	
	    public String getLogfile() { //4
	        return logfile;
	    }
	
	    public String getMsg() {  //5
	        return msg;
	    }
	
	    public long getReceivedTimestamp() {  //6
	        return received;
	    }
	}

1. 构造器用于出站消息
2. 构造器用于入站消息
3. 返回发送 LogEvent 的 InetSocketAddress 的资源
4. 返回用于发送 LogEvent 的日志文件的名称
5. 返回消息的内容
6. 返回 LogEvent 接收到的时间

