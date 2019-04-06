# 简单的私有协议的实现方案

在学习了Netty的不同的编码器和解码器之后，我们可以通过编解码器实现简单的自定义协议，这个自定义的协议并没有实现心跳检测，三次握手以及断路重连等复杂的机制，仅仅是用于演示如何实现数据的传输和自定义解码。

## 1. 私有协议
在Netty中你可以根据需要自己编写自己的传输协议，比如第几个字节传递什么信息，第几个字节表示什么意思，这和其他协议暂时没有关系，所以你可以基于此完成非常好的企业私有协议.当然此私有协议也是基于TCP/IP 协议完成的。

## 2. 自定义私有协议
下面我们根据实际的需求来安排一个简单的私有协议。值得注意的是这里没有实现很复杂的协议信息，主要是各个字节的设计，后期的话有兴趣的话，会根据具体的学习过程，再此基础上晚上该内容。

### 2.1 需求说明
设定需求如下: 客户端连接成功后，发送一个题目(string)，和N个数字(int)，服务器端打印题目和N个数字的和，并返回这个计算结果(int)，客户端接到结果之后，完成输出并关闭连接。

协议的字节码
4个字节: 题目的长度m
m个字节: 题目内容
4个字节：数字的个数n
4*n个字节：n个数字
4个字节： 校验码


### 2.2 协议内容
需求很简单，那么我们来分析协议格式的说明，首先是题目，题目的内容是字符串，默认使用UTF-8编码。那么我们就需要写入一个整数(4个字节,代表问题字符串的字节码的长度，注意的是，这里的长度不是字符串String的长度，而是String的字节码长度), 然后写入字符串的字节码内容。其次要继续写入一个数字，表示后面有N个数字，，那么后面的N个数字占用的总字节数为 4*N ,最后在加上校验码，校验码是123，如果校验不通过，则返回数字0.

更直观的结构图如下所示:

[--4个字节--][--M个字节--][--4个字节--][--4*N个字节--][--4个字节--]
 
1. 第一块占用4个字节，是一个整数,其值为M,表示的是问题的内容的长度，readIndex ∈ [0,4)
2. 第二块表示的是具体的内容字节，readIndex ∈ [4,4+M)
3. 第三块占用4个字节是一个数字，其值为N，表示后面有N个数字，readindex ∈ [4+M, 8+M)
4. 第四块表示的数字内容，由于共有N个数字，那个其占用字节数为 4*N，readIndex ∈ [8+M,8+M+4N)
5. 第五块属于校验码，是个数字，占用一个字节表示数字校验值, readIndex ∈ [8+M+4N,12+M+4N)

## 3. 代码实现
字节的读取主要在编码器和解码器上，首先写一个消息实体`QuestionEntity` 内容如下:


```java
public class QuestionsEntity {
  
  // 代码省略了set/get方法，自行添加

  private int questionsLength;

  private String questions;

  private int numberCount;

  private int[] numbers;

  private int verificationCode;
}
```

同样的为了更好地使用`ReplayingDecoder` 类型，我们定义了ProtocolState枚举类来来标记状态.

```java
/** 协议状态枚举类型 */
public enum ProtocolState {
  READ_QUESTIONS_LENGTH("读取题目长度"),
  READ_QUESTIONS("读取题目"),
  READ_NUMBER_COUNT("读取数字个数"),
  READ_NUMBER("读取数字"),
  READ_VERITY_CODE("读取校验码");

  private String message;

  ProtocolState(String message) {
    this.message = message;
  }

  public String getMessage() {
    return message;
  }

  public void setMessage(String message) {
    this.message = message;
  }
}
```

### 3.1 客户端实现

首先看客户端的实现，客户端的引导程序，已经非常熟悉了，我们主要注意的是ChannelHandleAdapter的实现.

```java
  public static void main(String[] args) {
      EventLoopGroup workGroup = new NioEventLoopGroup();

      try {
          Bootstrap bootstrap =
                  new Bootstrap()
                          .group(workGroup)
                          .remoteAddress(new InetSocketAddress("localhost", 8080))
                          .channel(NioSocketChannel.class)
                          .handler(
                                  new ChannelInitializer<SocketChannel>() {
                                      @Override
                                      protected void initChannel(SocketChannel ch) throws Exception {
                                          ChannelPipeline pipeline = ch.pipeline();
                                          //解码器
                                          pipeline.addLast(new IntegerDecoder());
                                          // 编码器
                                          pipeline.addLast(new PrivateProtocolEncoder());
                                          // 最终处理器
                                          pipeline.addLast(new ClientHandler());
                                      }
                                  });
          ChannelFuture sync = bootstrap.connect().sync();
          sync.channel().closeFuture().sync();
      } catch (InterruptedException e) {
          e.printStackTrace();
      } finally {
          workGroup.shutdownGracefully();
      }
  }
```

其中`IntegerDecoder` 解码器是入站的时候，将字节码接收转换为Integer 类型，代码很简单，如下所示:

```java
public class IntegerDecoder extends ReplayingDecoder<Void> {
  @Override
  protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    out.add(in.readInt());
  }
}
```

私有协议编码器`PrivateProtocolEncoder` 则是我们处理的重点，在ClinetHandle生成QuestionEntity 对象的时候则需要传入实例对象，因此`PrivateProtocolEncoder` 是一个接受了QuestionEntity 类型的编码器，按照上面的设计一一向ByteBuf中写入数据.

```java
public class PrivateProtocolEncoder extends MessageToByteEncoder<QuestionsEntity> {
  @Override
  protected void encode(ChannelHandlerContext ctx, QuestionsEntity msg, ByteBuf out)
      throws Exception {
    byte[] questionBytes = msg.getQuestions().getBytes();
    // 写入题目长度
    out.writeInt(questionBytes.length);
    // 写入题目
    out.writeBytes(questionBytes);
    // 写入数字的个数
    out.writeInt(msg.getNumberCount());
    // 遍历写入数字
    for (int i = 0; i < msg.getNumberCount(); i++) {
      out.writeInt(msg.getNumbers()[i]);
    }
    // 写入校验码
    out.writeInt(msg.getVerificationCode());
    // 打印日志
    System.out.println("客户端编码完成");
  }
}
```


在ClientHandler处理器中，则是在通道激活的死后发送封装的数据，然后接收到结果打印消息,具体的内容，请看注释。

```java
public class ClientHandler extends SimpleChannelInboundHandler<Integer> {

  /**
   * 接受到结果后打印数据，并关闭上下文
   *
   * @param ctx
   * @param msg
   * @throws Exception
   */
  @Override
  protected void messageReceived(ChannelHandlerContext ctx, Integer msg) throws Exception {
    System.out.println("客户端接收到答案:" + msg);
    ctx.close();
  }

  /**
   * 连接成功后发送数据
   *
   * @param ctx
   * @throws Exception
   */
  @Override
  public void channelActive(ChannelHandlerContext ctx) throws Exception {
    String question = "给定的所有数据的和是多少?";
    QuestionsEntity entity = new QuestionsEntity();
    entity.setQuestions(question);
    entity.setNumberCount(5);
    entity.setNumbers(new int[] {13, 2, 3, 4, 5});
    entity.setQuestionsLength(123);
    entity.setVerificationCode(123);
    ctx.channel().writeAndFlush(entity);
    System.out.println("客户端发送数据完成");
  }
}
```

这样就完成了客户端的操作，重新整理下
> 客户端引导成功 -> 发送QuestionEntity  -> PrivateProtocolEncoder 开始编码  -> 发送数据到服务器  —> 服务器返回数据  —> IntegerDecoder 进行解码操作  ->  ClientHandler 打印数据   -> 关闭连接



### 3.2 服务端实现

同样的我们也需要完成服务的端的设计，服务端的方案和以前类似，也是基本的结构，如下:

```java
  public static void main(String[] args) {
    EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    EventLoopGroup workGroup = new NioEventLoopGroup();

    try {
      ServerBootstrap boot =
          new ServerBootstrap()
              .group(bossGroup, workGroup)
              .channel(NioServerSocketChannel.class)
              .childHandler(
                  new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                      ChannelPipeline pipeline = ch.pipeline();
                      // 解码器
                      pipeline.addLast(new PrivateProtocolDecoder());
                      // 编码器
                      pipeline.addLast(new IntegerEncoder());
                      // 最终处理器
                      pipeline.addLast(new ServiceHandler());
                    }
                  });
      ChannelFuture sync = boot.bind(8080).sync();
      System.out.println("服务器端启动在8080 端口");
      sync.channel().closeFuture().sync();
    } catch (InterruptedException e) {
      e.printStackTrace();
    } finally {
      bossGroup.shutdownGracefully();
      workGroup.shutdownGracefully();
    }
  }
```

在服务端的启动中，有三种编解码器，其中`PrivateProtocolDecoder` 解码器是用来完成Byte到QuestionEntity的转换，是服务端的重点，`ServiceHandler`是服务端的处理器，是用来完成数据的计算和返回，在返回结构之后，`IntegerEncoder` 编码器开始工作，将数字转换为ByteBuf 返回给客户端，完成操作,下面我们逐一来看。

```java

import com.zhoutao123.netty.private_protocol.QuestionsEntity;
import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.ReplayingDecoder;

import java.util.List;

/** 私有协议处理 */
public class PrivateProtocolDecoder extends ReplayingDecoder<ProtocolState> {

  private int numberCount = -1;

  private int questionLength = -1;

  private String question = null;

  private int[] numbers = null;

  private int verificationCode = 0;

  /** 将此编码器的初始状态设置为 {@link ProtocolState#READ_QUESTIONS_LENGTH} */
  public PrivateProtocolDecoder() {
    super(ProtocolState.READ_QUESTIONS_LENGTH);
  }

  /**
   * 解码数据
   *
   * <p>checkpoint(S s) 是将当前的状态设置为指定的类型，后续后新的数据更新的时候，会继续执行之前的逻辑
   *
   * @param ctx 当前处理器的上下文
   * @param in ByteBuf
   * @param out 写入的数据，将发送送给下个入站处理器
   * @throws Exception
   */
  @Override
  protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {

    switch (state()) {
      case READ_QUESTIONS_LENGTH:
        // 读取题目长度
        questionLength = in.readInt();
        checkpoint(ProtocolState.READ_QUESTIONS);
      case READ_QUESTIONS:
        // 读取题目
        if (in.hasArray()) {
          question = new String(in.array(), in.arrayOffset() + in.readerIndex(), questionLength);
        } else {
          byte[] bytes = new byte[questionLength];
          in.getBytes(in.readerIndex(), bytes);
          question = new String(bytes, 0, bytes.length);
          // 因为getBytes是绝对操作，不会移动readIndex，因此读取完成后，需要手动移动readIndex
          in.readBytes(questionLength);
        }

        checkpoint(ProtocolState.READ_NUMBER_COUNT);
      case READ_NUMBER_COUNT:
        // 读取数字长度
        numberCount = in.readInt();

        checkpoint(ProtocolState.READ_NUMBER);
      case READ_NUMBER:
        numbers = new int[numberCount];
        for (int i = 0; i < numberCount; i++) {
          numbers[i] = in.readInt();
        }

        checkpoint(ProtocolState.READ_VERITY_CODE);
      case READ_VERITY_CODE:
        // 读取校验码
        verificationCode = in.readInt();
        // 封装数据
        QuestionsEntity entity = new QuestionsEntity();
        entity.setQuestionsLength(questionLength);
        entity.setQuestions(question);
        entity.setNumbers(numbers);
        entity.setVerificationCode(verificationCode);
        out.add(entity);
        break;
      default:
        throw new IllegalArgumentException();
    }
  }
}

```


转换为实体之后，ServiceHandler开始进行业务处理,当然其数据类型肯定是PrivateProtocolDecoder的转换结果,也就是QuestionEntity类型。

```java
public class ServiceHandler extends SimpleChannelInboundHandler<QuestionsEntity> {

  /**
   * 读取数据，计算，并写回数据
   *
   * @param ctx
   * @param msg
   * @throws Exception
   */
  @Override
  protected void messageReceived(ChannelHandlerContext ctx, QuestionsEntity msg) throws Exception {
    // 判断校验码
    if (msg.getQuestionsLength() != 123) {
      ctx.channel().writeAndFlush(0);
    } else {
      // 处理业务逻辑
      int[] numbers = msg.getNumbers();
      int sum = 1;
      for (int i = 0; i < numbers.length; i++) {
        sum *= numbers[i];
      }
      // 打印问题和答案
      System.out.println(msg.getQuestions());
      System.out.println("答案是:" + sum);
      // 写回计算结果
      ctx.channel().writeAndFlush(sum);
    }
  }
}

```

写回数据使用的整形编码器，比价简单，不再细说.

```java
public class IntegerEncoder extends MessageToByteEncoder<Integer> {
  @Override
  protected void encode(ChannelHandlerContext ctx, Integer msg, ByteBuf out) throws Exception {
    out.writeInt(msg);
  }
}
```