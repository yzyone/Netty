# Netty 实现http协议

这里简单介绍下，项目中使用netty在main方法中启动项目，实现http协议。

[maven](https://app.yinxiang.com/OutboundRedirect.action?dest=https%3A%2F%2Fso.csdn.net%2Fso%2Fsearch%3Fq%3Dmaven%26spm%3D1001.2101.3001.7020)依赖的包

```
<dependency>
	<groupId>io.netty</groupId>
	<artifactId>netty-all</artifactId>
	<version>4.1.27.Final</version>
</dependency>
```

1、netty启动入口：

```
package com.fotile.cloud.ruleengin;
 
import javax.servlet.ServletException;
 
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.mock.web.MockServletConfig;
import org.springframework.web.context.support.XmlWebApplicationContext;
import org.springframework.web.servlet.DispatcherServlet;
 
import com.fotile.cloud.ruleengin.falsework.NettyHttpServer;
 
/**
 * Hello world!
 *
 */
public class RuleApplication
{
 
    // 引擎端口
    private final static int ENGINE_PORT = 8086;
 
    /**
     * http prot is 8085,
     */
    public static void main(String[] args)
    {
	// 加载spring配置
	ApplicationContext ctx = new ClassPathXmlApplicationContext("spring-config.xml");
	DispatcherServlet servlet = getDispatcherServlet(ctx);
	NettyHttpServer server = new NettyHttpServer(ENGINE_PORT, servlet);
	server.start();
 
    }
 
    public static DispatcherServlet getDispatcherServlet(ApplicationContext ctx)
    {
 
	XmlWebApplicationContext mvcContext = new XmlWebApplicationContext();
	// 加载spring-mvc配置
	mvcContext.setConfigLocation("classpath:spring-mvc.xml");
	mvcContext.setParent(ctx);
	MockServletConfig servletConfig = new MockServletConfig(mvcContext.getServletContext(), "dispatcherServlet");
	DispatcherServlet dispatcherServlet = new DispatcherServlet(mvcContext);
	try
	{
	    dispatcherServlet.init(servletConfig);
	} catch (ServletException e)
	{
	    e.printStackTrace();
	}
	return dispatcherServlet;
    }
}
```

2、编写NettyHttpServer

```
package com.fotile.cloud.openplatform.falsework;
 
import org.apache.log4j.Logger;
import org.springframework.web.servlet.DispatcherServlet;
 
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
 
public class NettyHttpServer implements Runnable
{
 
    private Logger LOGGER = Logger.getLogger(this.getClass());
 
    private int port;
    private DispatcherServlet servlet;
 
    public NettyHttpServer(Integer port)
    {
	this.port = port;
    }
 
    public NettyHttpServer(Integer port, DispatcherServlet servlet)
    {
	this.port = port;
	this.servlet = servlet;
    }
 
    public void start()
    {
	EventLoopGroup bossGroup = new NioEventLoopGroup();
	EventLoopGroup workerGroup = new NioEventLoopGroup();
	try
	{
	    ServerBootstrap b = new ServerBootstrap();
	    b.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class)
		    .childHandler(new HttpServerInitializer(servlet)).option(ChannelOption.SO_BACKLOG, 128)
		    .childOption(ChannelOption.SO_KEEPALIVE, true);
 
	    LOGGER.info("NettyHttpServer Run successfully");
	    // 绑定端口，开始接收进来的连接
	    ChannelFuture f = b.bind(port).sync();
	    // 等待服务器 socket 关闭 。在这个例子中，这不会发生，但你可以优雅地关闭你的服务器。
	    f.channel().closeFuture().sync();
	} catch (Exception e)
	{
	    System.out.println("NettySever start fail" + e);
	} finally
	{
	    workerGroup.shutdownGracefully();
	    bossGroup.shutdownGracefully();
	}
    }
 
    @Override
    public void run()
    {
	start();
    }
}
```

3、处理http请求、处理、返回

```
package com.fotile.cloud.ruleengin.falsework;
 
import java.net.URLDecoder;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Map.Entry;
 
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.*;
import io.netty.handler.codec.http.multipart.DefaultHttpDataFactory;
import io.netty.handler.codec.http.multipart.HttpPostRequestDecoder;
import io.netty.handler.codec.http.multipart.InterfaceHttpData;
import io.netty.handler.codec.http.multipart.InterfaceHttpData.HttpDataType;
import io.netty.handler.codec.http.multipart.MemoryAttribute;
import io.netty.util.CharsetUtil;
 
import org.apache.commons.lang3.StringUtils;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;
import org.springframework.web.servlet.DispatcherServlet;
import org.springframework.web.util.UriComponents;
import org.springframework.web.util.UriComponentsBuilder;
import org.springframework.web.util.UriUtils;
 
public class HttpRequestHandler extends SimpleChannelInboundHandler<FullHttpRequest>
{
 
    private DispatcherServlet servlet;
 
    public HttpRequestHandler(DispatcherServlet servlet)
    {
	this.servlet = servlet;
    }
 
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest fullHttpRequest) throws Exception
    {
	boolean flag = HttpMethod.POST.equals(fullHttpRequest.method())
		|| HttpMethod.GET.equals(fullHttpRequest.method()) || HttpMethod.DELETE.equals(fullHttpRequest.method())
		|| HttpMethod.PUT.equals(fullHttpRequest.method());
 
	Map<String, String> parammap = getRequestParams(ctx, fullHttpRequest);
	if (flag && ctx.channel().isActive())
	{
	    // HTTP请求、GET/POST
	    MockHttpServletResponse servletResponse = new MockHttpServletResponse();
	    MockHttpServletRequest servletRequest = new MockHttpServletRequest(
		    servlet.getServletConfig().getServletContext());
	    // headers
	    for (String name : fullHttpRequest.headers().names())
	    {
		for (String value : fullHttpRequest.headers().getAll(name))
		{
		    servletRequest.addHeader(name, value);
		}
	    }
	    String uri = fullHttpRequest.uri();
	    uri = new String(uri.getBytes("ISO8859-1"), "UTF-8");
	    uri = URLDecoder.decode(uri, "UTF-8");
	    UriComponents uriComponents = UriComponentsBuilder.fromUriString(uri).build();
	    String path = uriComponents.getPath();
	    path = URLDecoder.decode(path, "UTF-8");
	    servletRequest.setRequestURI(path);
	    servletRequest.setServletPath(path);
	    servletRequest.setMethod(fullHttpRequest.method().name());
 
	    if (uriComponents.getScheme() != null)
	    {
		servletRequest.setScheme(uriComponents.getScheme());
	    }
	    if (uriComponents.getHost() != null)
	    {
		servletRequest.setServerName(uriComponents.getHost());
	    }
	    if (uriComponents.getPort() != -1)
	    {
		servletRequest.setServerPort(uriComponents.getPort());
	    }
 
	    ByteBuf content = fullHttpRequest.content();
	    content.readerIndex(0);
	    byte[] data = new byte[content.readableBytes()];
	    content.readBytes(data);
	    servletRequest.setContent(data);
 
	    if (uriComponents.getQuery() != null)
	    {
		String query = UriUtils.decode(uriComponents.getQuery(), "UTF-8");
		servletRequest.setQueryString(query);
	    }
	    if (parammap != null && parammap.size() > 0)
	    {
		for (String key : parammap.keySet())
		{
		    servletRequest.addParameter(UriUtils.decode(key, "UTF-8"),
			    UriUtils.decode(parammap.get(key) == null ? "" : parammap.get(key), "UTF-8"));
		}
	    }
	    servlet.service(servletRequest, servletResponse);
 
	    HttpResponseStatus status = HttpResponseStatus.valueOf(servletResponse.getStatus());
	    String result = servletResponse.getContentAsString();
	    result = StringUtils.isEmpty(result) ? status.toString() : result;
	    FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, status,
		    Unpooled.copiedBuffer(result, CharsetUtil.UTF_8));
	    response.headers().set("Content-Type", "text/json;charset=UTF-8");
	    response.headers().set("Access-Control-Allow-Origin", "*");
	    response.headers().set("Access-Control-Allow-Headers",
		    "Content-Type,Content-Length, Authorization, Accept,X-Requested-With,X-File-Name");
	    response.headers().set("Access-Control-Allow-Methods", "PUT,POST,GET,DELETE,OPTIONS");
	    response.headers().set("Content-Length", Integer.valueOf(response.content().readableBytes()));
	    response.headers().set("Connection", "keep-alive");
	    ChannelFuture writeFuture = ctx.writeAndFlush(response);
	    writeFuture.addListener(ChannelFutureListener.CLOSE);
	}
    }
 
    /**
     * 获取post请求、get请求的参数保存到map中
     */
    private Map<String, String> getRequestParams(ChannelHandlerContext ctx, HttpRequest req)
    {
	Map<String, String> requestParams = new HashMap<String, String>();
	// 处理get请求
	if (req.method() == HttpMethod.GET)
	{
	    QueryStringDecoder decoder = new QueryStringDecoder(req.uri());
	    Map<String, List<String>> parame = decoder.parameters();
	    Iterator<Entry<String, List<String>>> iterator = parame.entrySet().iterator();
	    while (iterator.hasNext())
	    {
		Entry<String, List<String>> next = iterator.next();
		requestParams.put(next.getKey(), next.getValue().get(0));
	    }
	}
	// 处理POST请求
	if (req.method() == HttpMethod.POST)
	{
	    HttpPostRequestDecoder decoder = new HttpPostRequestDecoder(new DefaultHttpDataFactory(false), req);
	    List<InterfaceHttpData> postData = decoder.getBodyHttpDatas(); //
	    for (InterfaceHttpData data : postData)
	    {
		if (data.getHttpDataType() == HttpDataType.Attribute)
		{
		    MemoryAttribute attribute = (MemoryAttribute) data;
		    requestParams.put(attribute.getName(), attribute.getValue());
		}
	    }
	}
	return requestParams;
    }
 
}
```

启来后，使用postman，调用本地接口。