---
tags:
  - jenkins
  - plugin
  - plugin_result
  - result
---
在平时编写Jenkinsfile进行CI/CD, 很多场景下就需要获取一些 step的结果, 并根据结果进行下一步的处理.   不过很多时候, 其实不清楚step返回的结果是什么, 有些什么的字段会方法可以使用.

针对此种情况呢, 其实可以尝试一下方法:
1. 进行测试, 打印出结果值, 可以直接使用
此种方法可以在工作中使用, 用于快速解决问题, 并使用.
2. 阅读其源码, 知其所以然.
此方法用于了解其原理, 后续对于其他的plugin也会有一些思路.


方法一就不需要多说了, 本篇以两个plugin为例, 记录如何查看其返回值.

> NexusArtifactUploader

[nexusUploader](https://github.com/jenkinsci/nexus-artifact-uploader-plugin/blob/master/src/main/java/sp/sd/nexusartifactuploader/NexusArtifactUploader.java)

```java
public class NexusArtifactUploader extends Builder implements SimpleBuildStep, Serializable {
	....

    @Override
    public void perform(Run build, FilePath workspace, Launcher launcher, final TaskListener listener) throws IOException, InterruptedException {
		...
		//  由此可以看到, 执行是异步的, 调用一个 Callable 封装的方法.
		// 由 Callable可以看到其返回为Boolean, 表示是否上传成功
        workspace.act(new Callable<Boolean, IOException>() {
            private static final long serialVersionUID = 1L;

            @Override
            public Boolean call() throws IOException {
                ....
                // 具体上传方法
                return Utils.uploadArtifacts(listener, username, password,
                        nexusUrl, repository, protocol, nexusVersion,
                        nexusArtifacts.toArray(new org.sonatype.aether.artifact.Artifact[0]));
            }

            @Override
            public void checkRoles(RoleChecker checker) throws SecurityException {

            }
        });
    }



}
```




> httpRequest

[http request](https://github.com/jenkinsci/http-request-plugin/blob/master/src/main/java/jenkins/plugins/http_request/HttpRequest.java)

```java
public class HttpRequest extends Builder {
    @Override
    public boolean perform(AbstractBuild<?,?> build, Launcher launcher, BuildListener listener)
    throws InterruptedException, IOException
    {
    
		HttpRequestExecution exec = HttpRequestExecution.from(this, envVars, build, this.getQuiet() ? TaskListener.NULL : listener);
		VirtualChannel channel = launcher.getChannel();
		// 可以看到方法同样是 异步的
		channel.call(exec);
		// 此时的返回值 表示方法 提交成功
        return true;
    }
}
```

```java
public class HttpRequestExecution extends MasterToSlaveCallable<ResponseContentSupplier, RuntimeException> {
	@Override
	public ResponseContentSupplier call() throws RuntimeException {
	...
		try {
		// 方法执行
			return authAndRequest();
		} catch (IOException | InterruptedException |
				KeyStoreException | NoSuchAlgorithmException | KeyManagementException e) {
			throw new IllegalStateException(e);
		}
	}
}


	private ResponseContentSupplier authAndRequest()
			throws IOException, InterruptedException, KeyStoreException, NoSuchAlgorithmException, KeyManagementException {
		//only leave open if no error happen
		ResponseHandle responseHandle = ResponseHandle.NONE;
		CloseableHttpClient httpclient = null;
		try {
			//// 请求执行完成后, 返回 ResponseContentSupplier 作为结果
			ResponseContentSupplier response = executeRequest(httpclient, clientUtil, httpRequestBase, context);
			processResponse(response);
		
			return response;
		} finally {
			if (responseHandle != ResponseHandle.LEAVE_OPEN) {
				if (httpclient != null) {
					httpclient.close();
				}
			}
		}
	}
```

```java
public class ResponseContentSupplier implements Serializable, AutoCloseable {

	private static final long serialVersionUID = 1L;

	private final int status;
	private final Map<String, List<String>> headers = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);
	private String charset;

	private ResponseHandle responseHandle;
	private String content;
	@SuppressFBWarnings("SE_TRANSIENT_FIELD_NOT_RESTORED")
	private transient InputStream contentStream;
	@SuppressFBWarnings("SE_TRANSIENT_FIELD_NOT_RESTORED")
	private transient CloseableHttpClient httpclient;

	public ResponseContentSupplier(String content, int status) {
		this.content = content;
		this.status = status;
	}

	public ResponseContentSupplier(ResponseHandle responseHandle, HttpResponse response) {
		this.status = response.getStatusLine().getStatusCode();
		this.responseHandle = responseHandle;
		readHeaders(response);
		readCharset(response);

		try {
			HttpEntity entity = response.getEntity();
			InputStream entityContent = entity != null ? entity.getContent() : null;

			if (responseHandle == ResponseHandle.STRING && entityContent != null) {
				byte[] bytes = IOUtils.toByteArray(entityContent);
				contentStream = new ByteArrayInputStream(bytes);
				content = new String(bytes, charset == null || charset.isEmpty() ?
						Charset.defaultCharset().name() : charset);
			} else {
				contentStream = entityContent;
			}
		} catch (IOException e) {
			throw new IllegalStateException(e);
		}
	}
	// 响应码
	@Whitelisted
	public int getStatus() {
		return this.status;
	}
// 响应头
	@Whitelisted
	public Map<String, List<String>> getHeaders() {
		return this.headers;
	}

	// 编码
	@Whitelisted
	public String getCharset() {
		return charset;
	}
		// 获取 repsonseBody
	@Whitelisted
	public String getContent() {
		if (responseHandle == ResponseHandle.STRING) {
		// text 返回体
			return content;
		}
		if (content != null) {
		// content
			return content;
		}
		if(contentStream == null) {
			return null;
		}

		try (InputStreamReader in = new InputStreamReader(contentStream,
				charset == null || charset.isEmpty() ? Charset.defaultCharset().name() : charset)) {
			content = IOUtils.toString(in);
			// content
			return content;
		} catch (IOException e) {
			...
		}
	}
}
```

从上面的`ResponseContentSupplier`可以看到, 其中的 http请求结果可以通过`getStatus, getContent, getHeader` 等方法进行获取.  由此就可以直到 httpRequest step 执行完成后, 有哪些方法可以使用, 哪些 field可以使用.  


