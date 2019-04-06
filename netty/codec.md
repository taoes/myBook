# 编解码器

我认为Netty 最棒的一点就是Netty 设计的编解码链，这一优秀的设计，可以很方便的实现二进制流->ByteBuf-> Message对象,反之编码器也是如此.那么下面有一些总结的记录。
> 
1. 各种编解码器本质上是一个ChannelHandle处理器
2. 解码器(又称之为入站处理器)的顶层父类是ChannelInboundHandler, 编码器(又称之为出站处理器)的顶层父类是ChannelOutboundHandler.
3. 我们向网络中写入数据的时候，不管是什么类型，基本类型或者是类类型，最终都是转换为字节流的形式传递。那么在Netty中，其预置了一个ByteBuf的编解码器，他可以将网络中的二进制转换为ByteBuf(或者反之)，这样我们的最终目的只要将Message转换为ByteBuf(或者反之)即可,将Message转换为ByteBuf的过程称之为编码`Encoder`，ByteBuf转换为Message的过程称之为解码`Decoder`, 两者统称为编解码，codec 
4. 编码本质上是一种出站操作,因此编码一定是一个ChannelOutboundHandler，同样的解码本质是一个入站操作，那么其一定是一个ChannelInboundHandler.
5. 在Netty的编解码执行链中，一定注意链中某个处理器，其输入是上个处理器的输出，其输出是下个处理器的输入，如果类型不匹配，将无法发送数据(Netty 会抛出异常，但是此异常已被内部处理).


## 1. 解码器
解码器是一种入站操作，其将二进制流转换为响应的Message,解码器的最终父类一定是ChannelInboundHandle.

### 1.1 抽象类ByteToMessageDecoder

通过继承ByteTOMessageDecoder 来实现自定义解码器，重写其抽象方法`decoder(ChannelHandleContext,ByteBut,List)` 

一个简单的将ByteBuf转换为Long类型的数据如下:

```java
/** Long 类型数据解码器 */
public class LongDecoder extends ByteToMessageDecoder {
  @Override
  protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    System.out.println("调用解码器的decode方法");
    System.out.println("ByteBuf可读字节数:" + in.readableBytes());
    if (in.readableBytes() >= 8) {
      out.add(in.readLong());
    }
  }
}
```

> 需要注意的是: 在解码的过程中，一定要判断解码的可读字节是否足够，否则的话可能产生问题，如上面的示例中，需要读取一个Long类型的数据，那么在读取之前需要判断当前的ByteBuf中是否还有8个字节，否则的话，则不予处理。

在 ByetToMessageDecoder 的 `decode(ChannelHandleContext,ByteBuf,List)` 方法中
+ ChannelHandleContext 表示的是当前处理器的上下文，包含该处理链
+ ByteBuf 值得是Netyy为我么自动将Byte转换出来的ByteBuf对象，我们只要从这个对象中读取数据即可.
+ List list传递的是一个Object 集合,当前ByteBuf 中数据已经读取完成的时候，若list集合的数据不为空，则解码器会把List集合的数据交个下一个处理器操作。

在某些场景下，可能需要使用`decodeLast(ChannelHandleContext,ByteBuf,List)` 方法，这个方法在Channel状态变为非活动的时候，会被调用一次，重写该方法可以提供一些特殊的处理，比如在产生一个LastHttpContent。

### 1.2 抽象类ReplayingDecoder<S>

ReplayingDecoder 继承了ByteToMessageDecoder，它可以是我们不必使用readableBytes() 方法，它使用的了一个自定义的ByteBuf 实现，RepalyingDecoderByteBuf ，包装传入ByteBuf对象实例，其将在内部执行该调用(指定readableBytes() 方法);

> 其中泛型S 表示状态，在不需要状态管理的情况下，我们传入 `Void`。

#### 1.2.1 实现原理
ReplayingDecoder 自己集成ByteBuf抽象类实现了 ReplayingDecoderByteBuf(Netty 4.x 5.x不是继承ByteBuf),在每次readBytes()出现错误的时候，会自动抛出 Signal 实例(注意：该实例属于共用的示例，并不是出现异常创建新的实例)，该对象是Netty集成Error 实现的。

```java
/**
 * A special {@link Error} which is used to signal some state or request by throwing it.
 * {@link Signal} has an empty stack trace and has no cause to save the instantiation overhead.
 */
public final class Signal extends Error implements Constant<Signal> {...}

```

再捕获该错误之后，Netty并不继续向上抛出，而是忽略了该错误，当下次ByteBuf有新的数据过来的时候继续执行decode()方法。但是这样的实现也会造成性能上的降低，设想一下，当网络状态不好的时候，客户端发送数据很慢，每次都是一个字节一个自己的发送，那么服务端这边，每次接到一个自己就解析一次，然后解析到某个地方的时候，不足与read(),那么会抛出Error，等待下次继续有新的数据过来，那么当下次新的数据过来之后，还是很少的字节，依然不足以read(),如此循环，造成性能大大的下降，所以有个简单的使用准则：在使用`ByteToMessageDecoder` 没有多少复杂性的情况下，请使用`ByteToMessageDecoder`, 否则请使用`ReplayingDecoder`;

### 1.2.2 性能优化

每当有新的字节写入的话，处理器都会调用decode重新解码，这样造成了性能的浪费，ReplayingDecoide提供了一个状态的处理机制，该泛型一般是枚举类，表明各个处理阶段，当有新的数据过来的根据阶段执行更新，如：

```java
    switch (state()) {
     case READ_LENGTH:
        length = buf.readInt();
        checkpoint(MyDecoderState.READ_CONTENT);
    case READ_CONTENT:
       ByteBuf frame = buf.readBytes(length);
      checkpoint(MyDecoderState.READ_LENGTH);
       out.add(frame);
        break;
      default:
        throw new Error("Shouldn't reach here.");
      }
```


### 1.3 抽象类MessageToMessageDecoder
此解码器类似于`ByteToMessageDecoder<T>` 其中泛型T表示传入的数据类型，其余和ByteToMessageDecoder 类似。 



## 2. 编码器

编码器Encoder, 一般属于出站操作，是将Message转换为Byte的操作，其主要继承MessageToByteEncoder 实现，实现 encode()方法.

### 2.1 MessageToByteBufEncoder
继承此类需要传入泛型T，该泛型是表明传入的数据类型。实现编码器和实现解码器的过程类似，只是解码器是readBytes, 而编码器是writeBytes,通过向ByteBuf 写入数据完成操作，比较简单，具体看下面的示例.

```java
public class IntegerEncoder extends MessageToByteEncoder<Integer> {
  @Override
  protected void encode(ChannelHandlerContext ctx, Integer msg, ByteBuf out) throws Exception {
    out.writeInt(msg);
  }
}

```

### 2.2 其他编码器
同解码器类似，编码器也存在`MessageToMessageEncoder<I>`,是将消息A转换为消息B.
