1. Heap BUffer

2. Direact Buffer

3. Composition Buffer

> A virtual buffer which shows multiple buffers as a single merged buffer.  It is recommended to use  ByteBufAllocator#compositeBuffer() or  Unpooled#wrappedBuffer(ByteBuf...) instead of calling the constructor explicitly.