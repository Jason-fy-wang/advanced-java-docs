---
tags:
  - nettty
  - http
---
http 的数据结构如下所述，针对此种数据， netty是如何进行解析的呢？
[[1-http-chunk-stream]]


```shell
## set up http1/1 server
static void setupHttp1Pipeline(ChannelPipeline p, NettyHandlerAdapter nettyHandlerAdapter, MuServerImpl server, String proto) {  
    p.addLast("decoder", new HttpRequestDecoder(server.settings().maxUrlSize + LENGTH_OF_METHOD_AND_PROTOCOL, server.settings().maxHeadersSize, 8192));  
    p.addLast("encoder", new HttpResponseEncoder() {  
        @Override  
        protected boolean isContentAlwaysEmpty(HttpResponse msg) {  
            return super.isContentAlwaysEmpty(msg) || msg instanceof NettyResponseAdaptor.EmptyHttpResponse;  
        }  
    });  
    if (server.settings().gzipEnabled) {  
        p.addLast("compressor", new SelectiveHttpContentCompressor(server.settings()));  
    }  
    p.addLast("keepalive", new HttpServerKeepAliveHandler());  
    p.addLast("flowControl", new FlowControlHandler());  
    p.addLast(BackPressureHandler.NAME, new BackPressureHandler());  
    p.addLast("preread", new PreReader());  
    p.addLast("muhandler", new Http1Connection(nettyHandlerAdapter, server, proto));  
}
```

其中主要是`HttpRequestDecoder`来解析http请求.
其中针对 正常的http请求， 其会通过 `content-length` 来解析一共需要读取的body的数据长度。

针对chunk，则是每次解析收到的body部分数据，chunk数据格式会写明数据的长度，`直到读取到长度为0的数据表明读取结束。`

真正解析的方法为：
>io.netty.handler.codec.http.HttpObjectDecoder#decode

```java
@Override  
protected void decode(ChannelHandlerContext ctx, ByteBuf buffer, List<Object> out) throws Exception {  
    if (resetRequested.get()) {  
        resetNow();  
    }  
    switch (currentState) {  
    case SKIP_CONTROL_CHARS:  
	    ...
    case READ_INITIAL: try {  
        ByteBuf line = lineParser.parse(buffer);  
        if (line == null) {  
            return;  
        }
        // read http first line
        // GET /PATH  http/1.1
        final String[] initialLine = splitInitialLine(line);  
        assert initialLine.length == 3 : "initialLine::length must be 3";  
  
        message = createMessage(initialLine);  
        currentState = State.READ_HEADER;  
    } catch (Exception e) {  
        out.add(invalidMessage(message, buffer, e));  
        return;  
    }  
    case READ_HEADER: try {  
	    // 读取 http header, 并获取下一个 状态
	    // readHeader 函数内容如下:
        State nextState = readHeaders(buffer);  
		...
        currentState = nextState;  
        switch (nextState) {  
        case SKIP_CONTROL_CHARS:  
			...
		// read chunk size
        case READ_CHUNK_SIZE:    
            addCurrentMessage(out);  
            return;  
        default:  
	        ...
        }  
    } catch (Exception e) {  
        out.add(invalidMessage(message, buffer, e));  
        return;  
    }  
    case READ_VARIABLE_LENGTH_CONTENT: {  
		...
    }  
    /// read conent-length body
    case READ_FIXED_LENGTH_CONTENT: {  
        int readLimit = buffer.readableBytes();  
		
		if (readLimit == 0) {  
            return;  
        }  
  
        int toRead = Math.min(readLimit, maxChunkSize);  
        if (toRead > chunkSize) {  
            toRead = (int) chunkSize;  
        }  
        ByteBuf content = buffer.readRetainedSlice(toRead);  
        chunkSize -= toRead;  
  
        if (chunkSize == 0) {  
            // Read all content.  
            out.add(new DefaultLastHttpContent(content, trailersFactory));  
            resetNow();  
        } else {  
            out.add(new DefaultHttpContent(content));  
        }  
        return;  
    }  
    case READ_CHUNK_SIZE: try {  
        ByteBuf line = lineParser.parse(buffer);  
        if (line == null) {  
            return;  
        }  
        int chunkSize = getChunkSize(line.array(), line.arrayOffset() + line.readerIndex(), line.readableBytes());  
        this.chunkSize = chunkSize;  
        if (chunkSize == 0) {  
            currentState = State.READ_CHUNK_FOOTER;  
            return;  
        }  
        currentState = State.READ_CHUNKED_CONTENT;  
        // fall-through  
    } catch (Exception e) {  
        out.add(invalidChunk(buffer, e));  
        return;  
    }  
    case READ_CHUNKED_CONTENT: {  
        assert chunkSize <= Integer.MAX_VALUE;  
        int toRead = Math.min((int) chunkSize, maxChunkSize);  
        if (!allowPartialChunks && buffer.readableBytes() < toRead) {  
            return;  
        }  
        toRead = Math.min(toRead, buffer.readableBytes());  
        if (toRead == 0) {  
            return;  
        }  
        HttpContent chunk = new DefaultHttpContent(buffer.readRetainedSlice(toRead));  
        chunkSize -= toRead;  
  
        out.add(chunk);  
  
        if (chunkSize != 0) {  
            return;  
        }  
        currentState = State.READ_CHUNK_DELIMITER;  
        // fall-through  
    }  
    case READ_CHUNK_DELIMITER: {  
		... 
        return;  
    }  
    case READ_CHUNK_FOOTER: try {  
        LastHttpContent trailer = readTrailingHeaders(buffer);  
        if (trailer == null) {  
            return;  
        }  
        out.add(trailer);  
        resetNow();  
        return;  
    } catch (Exception e) {  
        out.add(invalidChunk(buffer, e));  
        return;  
    }  
    case BAD_MESSAGE: {  
		...
        break;  
    }  
    case UPGRADED: {  
        ...
        break;  
    }  
    default:  
        break;  
    }  
}
```



```java
    private State readHeaders(ByteBuf buffer) {
        final HttpMessage message = this.message;
        final HttpHeaders headers = message.headers();

        final HeaderParser headerParser = this.headerParser;
        ByteBuf line = headerParser.parse(buffer);
		...
        int lineLength = line.readableBytes();

        while (lineLength > 0) {
            // read headers
        }
        ...
        // Done parsing initial line and headers. Set decoder result.
        HttpMessageDecoderResult decoderResult = new HttpMessageDecoderResult(lineParser.size, headerParser.size);

        message.setDecoderResult(decoderResult);

        //获取content-length的value

        List<String> contentLengthFields = headers.getAll(HttpHeaderNames.CONTENT_LENGTH);

        if (!contentLengthFields.isEmpty()) {
            HttpVersion version = message.protocolVersion();
            boolean isHttp10OrEarlier = version.majorVersion() < 1 || (version.majorVersion() == 1 && version.minorVersion() == 0);
            contentLength = HttpUtil.normalizeAndGetContentLength(contentLengthFields,
                    isHttp10OrEarlier, allowDuplicateContentLengths);

            if (contentLength != -1) {
                String lengthValue = contentLengthFields.get(0).trim();
                if (contentLengthFields.size() > 1 ||
                        !lengthValue.equals(Long.toString(contentLength))) {
                    headers.set(HttpHeaderNames.CONTENT_LENGTH, contentLength);

                }
            }
        } else {
			....
            contentLength = HttpUtil.getWebSocketContentLength(message);

        }
        ...
		// 如何 http header中由 Transfer-Encoding: Chunked
		// 则表明为 chunk 格式, 则把状态设置为 READ_CHUNK_SIZE,表示下一个为要 读取chunk的长度, 有了长度后,就可以读取chunk的 message
        if (HttpUtil.isTransferEncodingChunked(message)) {
            this.chunked = true;
            if (!contentLengthFields.isEmpty() && message.protocolVersion() == HttpVersion.HTTP_1_1) {
                handleTransferEncodingChunkedWithContentLength(message);
            }
            return State.READ_CHUNK_SIZE;
        }
        // 如果存在 content-length
        // 那么就读取 content-length 长度的 message
        if (contentLength >= 0) {
            return State.READ_FIXED_LENGTH_CONTENT;
        }
        // 某个就读取变量长度的数据
        // 即: 一直从 socket中读取数据, 直到读取不到数据为止
        return State.READ_VARIABLE_LENGTH_CONTENT;
    }
```






