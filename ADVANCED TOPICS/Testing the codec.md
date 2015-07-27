测试编解码器
====

编码器和解码器完成,但仍有一些缺失:测试。

没有测试你只看到如果编解码器工作对一些真正的服务器运行时,这并不是你应该是依靠什么。第十章所示,为一个自定义编写测试 ChannelHandler通常是通过 EmbeddedChannel。

所以这正是现在做测试我们定制的编解码器,其中包括一个编码器和解码器。让重新开始编码器。后面的清单显示了简单的编写单元测试。

Listing 14.5 MemcachedRequestEncoderTest class

public class MemcachedRequestEncoderTest {

    @Test
    public void testMemcachedRequestEncoder() {
        MemcachedRequest request = new MemcachedRequest(Opcode.SET, "key1", "value1"); //1

        EmbeddedChannel channel = new EmbeddedChannel(new MemcachedRequestEncoder());  //2
        channel.writeOutbound(request); //3

        ByteBuf encoded = (ByteBuf) channel.readOutbound();

        Assert.assertNotNull(encoded);  //4
        Assert.assertEquals(request.magic(), encoded.readUnsignedByte());  //5
        Assert.assertEquals(request.opCode(), encoded.readByte());  //6
        Assert.assertEquals(4, encoded.readShort());//7
        Assert.assertEquals((byte) 0x08, encoded.readByte()); //8
        Assert.assertEquals((byte) 0, encoded.readByte());//9
        Assert.assertEquals(0, encoded.readShort());//10
        Assert.assertEquals(4 + 6 + 8, encoded.readInt());//11
        Assert.assertEquals(request.id(), encoded.readInt());//12
        Assert.assertEquals(request.cas(), encoded.readLong());//13
        Assert.assertEquals(request.flags(), encoded.readInt()); //14
        Assert.assertEquals(request.expires(), encoded.readInt()); //15

        byte[] data = new byte[encoded.readableBytes()]; //16
        encoded.readBytes(data);
        Assert.assertArrayEquals((request.key() + request.body()).getBytes(CharsetUtil.UTF_8), data);
        Assert.assertFalse(encoded.isReadable());  //17

        Assert.assertFalse(channel.finish());
        Assert.assertNull(channel.readInbound());
    }
}


1. 新建 MemcachedRequest 用于编码为 ByteBuf
2. 新建 EmbeddedChannel 用于保持 MemcachedRequestEncoder 到测试
3. 写请求到 channel 并且判断是否产生了编码的消息
4. 检查 ByteBuf 是否 null
5. 判断 magic 是否正确写入 ByteBuf
6. 判断 opCode (SET) 是否写入正确
7. 检查 key 是否写入长度正确
8. 检查写入的请求是否额外包含
9. 检查数据类型是否写
10. 检查是否保留数据插入
11. 检查 body 的整体大小 计算方式是 key.length + body.length + extras
12. 检查是否正确写入 id
13. 检查是否正确写入 Compare and Swap (CAS)
14. 检查是否正确的 flag
15. 检查是否正确设置到期时间的
16. 检查 key 和 body 是否正确
17. 检查是否可读

Listing 14.6 MemcachedResponseDecoderTest class

	public class MemcachedResponseDecoderTest {
	
	    @Test
	    public void testMemcachedResponseDecoder() {
	        EmbeddedChannel channel = new EmbeddedChannel(new MemcachedResponseDecoder());  //1
	
	        byte magic = 1;
	        byte opCode = Opcode.SET;
	
	        byte[] key = "Key1".getBytes(CharsetUtil.US_ASCII);
	        byte[] body = "Value".getBytes(CharsetUtil.US_ASCII);
	        int id = (int) System.currentTimeMillis();
	        long cas = System.currentTimeMillis();
	
	        ByteBuf buffer = Unpooled.buffer(); //2
	        buffer.writeByte(magic);
	        buffer.writeByte(opCode);
	        buffer.writeShort(key.length);
	        buffer.writeByte(0);
	        buffer.writeByte(0);
	        buffer.writeShort(Status.KEY_EXISTS);
	        buffer.writeInt(body.length + key.length);
	        buffer.writeInt(id);
	        buffer.writeLong(cas);
	        buffer.writeBytes(key);
	        buffer.writeBytes(body);
	        
	        Assert.assertTrue(channel.writeInbound(buffer));  //3
	
	        MemcachedResponse response = (MemcachedResponse) channel.readInbound();
	        assertResponse(response, magic, opCode, Status.KEY_EXISTS, 0, 0, id, cas, key, body);//4
	    }
	
	    @Test
	    public void testMemcachedResponseDecoderFragments() {
	        EmbeddedChannel channel = new EmbeddedChannel(new MemcachedResponseDecoder()); //5
	
	        byte magic = 1;
	        byte opCode = Opcode.SET;
	
	        byte[] key = "Key1".getBytes(CharsetUtil.US_ASCII);
	        byte[] body = "Value".getBytes(CharsetUtil.US_ASCII);
	        int id = (int) System.currentTimeMillis();
	        long cas = System.currentTimeMillis();
	
	        ByteBuf buffer = Unpooled.buffer(); //6
	        buffer.writeByte(magic);
	        buffer.writeByte(opCode);
	        buffer.writeShort(key.length);
	        buffer.writeByte(0);
	        buffer.writeByte(0);
	        buffer.writeShort(Status.KEY_EXISTS);
	        buffer.writeInt(body.length + key.length);
	        buffer.writeInt(id);
	        buffer.writeLong(cas);
	        buffer.writeBytes(key);
	        buffer.writeBytes(body);
	
	        ByteBuf fragment1 = buffer.readBytes(8); //7
	        ByteBuf fragment2 = buffer.readBytes(24);
	        ByteBuf fragment3 = buffer;
	
	        Assert.assertFalse(channel.writeInbound(fragment1));  //8
	        Assert.assertFalse(channel.writeInbound(fragment2));  //9
	        Assert.assertTrue(channel.writeInbound(fragment3));  //10
	
	        MemcachedResponse response = (MemcachedResponse) channel.readInbound();
	        assertResponse(response, magic, opCode, Status.KEY_EXISTS, 0, 0, id, cas, key, body);//11
	    }
	
	    private static void assertResponse(MemcachedResponse response, byte magic, byte opCode, short status, int expires, int flags, int id, long cas, byte[] key, byte[] body) {
	        Assert.assertEquals(magic, response.magic());
	        Assert.assertArrayEquals(key, response.key().getBytes(CharsetUtil.US_ASCII));
	        Assert.assertEquals(opCode, response.opCode());
	        Assert.assertEquals(status, response.status());
	        Assert.assertEquals(cas, response.cas());
	        Assert.assertEquals(expires, response.expires());
	        Assert.assertEquals(flags, response.flags());
	        Assert.assertArrayEquals(body, response.data().getBytes(CharsetUtil.US_ASCII));
	        Assert.assertEquals(id, response.id());
	    }
	}

1. 新建 EmbeddedChannel ，持有 MemcachedResponseDecoder 到测试
2. 创建一个新的 Buffer 并写入数据，与二进制协议的结构相匹配
3. 写缓冲区到 EmbeddedChannel 和检查是否一个新的MemcachedResponse 创建由声明返回值
4. 判断 MemcachedResponse 和预期的值
5. 创建一个新的 EmbeddedChannel 持有 MemcachedResponseDecoder 到测试
6. 创建一个新的 Buffer 和写入数据的二进制协议的结构相匹配
7. 缓冲分割成三个片段
8. 写的第一个片段 EmbeddedChannel 并检查,没有新的MemcachedResponse 创建,因为并不是所有的数据都是准备好了
9. 写第二个片段 EmbeddedChannel 和检查,没有新的MemcachedResponse 创建,因为并不是所有的数据都是准备好了
10. 写最后一段到 EmbeddedChannel 和检查新的 MemcachedResponse 
是否创建，因为我们终于收到所有数据
11. 判断 MemcachedResponse 与预期的值

