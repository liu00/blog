***本文引用的源码为android 4.4.4版本***

[请尊重博主劳动成果，转载请标明原文链接。](http://blog.csdn.net/hwliu51/article/details/70216242)

使用WebView的load(data,"text/html", "utf-8")加载含有中文的网页时，页面上的中文字符显示为乱码。网页没有问题，使用PC浏览器查看显示正常。
load方法源码

```java
	public void loadData(String data, String mimeType, String encoding) {
        checkThread();
        if (DebugFlags.TRACE_API) Log.d(LOGTAG, "loadData");
        mProvider.loadData(data, mimeType, encoding);
    }
```
真正执行加载网页操作的是mProvider，而它是什么类型的对象呢?

```java
	private WebViewProvider mProvider;
	...
	private void ensureProviderCreated() {
        checkThread();
        if (mProvider == null) {
            // As this can get called during the base class constructor chain, pass the minimum
            // number of dependencies here; the rest are deferred to init().
            //调用方法getFactory获取WebViewFactoryProvider对象，然后使用该对象的createWebView方法创建。
            mProvider = getFactory().createWebView(this, new PrivateAccess());
        }
    }

	//生成WebViewFactoryProvider方法
	private static synchronized WebViewFactoryProvider getFactory() {
        return WebViewFactory.getProvider();
    }
```

WebViewFactoryProvider是一个接口，那么就查看WebViewFactory类。WebViewFactory的getProvider方法调用getFactoryClass()获取到字节码，然后通过反射创建了WebViewChromiumFactoryProvider对象。

```java
public final class WebViewFactory {
	//类路径
    private static final String CHROMIUM_WEBVIEW_FACTORY =
            "com.android.webview.chromium.WebViewChromiumFactoryProvider";

    ...

    private static class Preloader {
        static WebViewFactoryProvider sPreloadedProvider;
        //静态代码块，字节码加载进来便创建了WebViewFactoryProvider对象
        static {
            try {
	            //通过反射创建对象
                sPreloadedProvider = getFactoryClass().newInstance();
            } catch (Exception e) {
                Log.w(LOGTAG, "error preloading provider", e);
            }
        }
    }

    …

    static WebViewFactoryProvider getProvider() {
        synchronized (sProviderLock) {
            //存在对象，则返回
            if (sProviderInstance != null) return sProviderInstance;

            Class<WebViewFactoryProvider> providerClass;
            try {
	            //获取字节码对象
                providerClass = getFactoryClass();
            } catch (ClassNotFoundException e) {
                Log.e(LOGTAG, "error loading provider", e);
                throw new AndroidRuntimeException(e);
            }

            //对象存在且字节码相同
            if (Preloader.sPreloadedProvider != null &&
                Preloader.sPreloadedProvider.getClass() == providerClass) {
                //赋值
                sProviderInstance = Preloader.sPreloadedProvider;
                
                return sProviderInstance;
            }

            
            StrictMode.ThreadPolicy oldPolicy = StrictMode.allowThreadDiskReads();
            try {
	            //通过字节码创建对象
                sProviderInstance = providerClass.newInstance();
                if (DEBUG) Log.v(LOGTAG, "Loaded provider: " + sProviderInstance);
                return sProviderInstance;
            } catch (Exception e) {
                Log.e(LOGTAG, "error instantiating provider", e);
                throw new AndroidRuntimeException(e);
            } finally {
                StrictMode.setThreadPolicy(oldPolicy);
            }
        }
    }

	//获取字节码的方法
    private static Class<WebViewFactoryProvider> getFactoryClass() throws ClassNotFoundException {
        return (Class<WebViewFactoryProvider>) Class.forName(CHROMIUM_WEBVIEW_FACTORY);
    }
}
```
WebViewFactory被加载进来，便创建WebViewChromiumFactoryProvider对象。源码在frameworks/webview/chromium/java/com/android/webview/chromium/WebViewChromiumFactoryProvider.java。
那就看看WebViewChromiumFactoryProvider.loadData方法代码：

```java
public WebViewProvider createWebView(WebView webView, WebView.PrivateAccess privateAccess) {
        WebViewChromium wvc = new WebViewChromium(this, webView, privateAccess);

        synchronized (mLock) {
            if (mWebViewsToStart != null) {
                mWebViewsToStart.add(new WeakReference<WebViewChromium>(wvc));
            }
        }
        ResourceProvider.registerResources(webView.getContext());
        return wvc;
    }
```
该方法创建了WebViewChromium对象，并返回了该对象。在ensureProviderCreated方法中创建了该对象，并将其赋值给WebView中的mProvider属性。也就是说，在WebView中网页相关的操作都是WebViewChromium真正在执行。

WebViewChromium的源码在：frameworks/webview/chromium/java/com/android/webview/chromium/WebViewChromium.java。
看看WebViewChromium的loadData方法：

```java
public void loadData(String data, String mimeType, String encoding) {
        loadUrlOnUiThread(LoadUrlParams.createLoadDataParams(
                fixupData(data), fixupMimeType(mimeType), isBase64Encoded(encoding)));
    }
```
使用LoadUrlParams类的静态方法createLoadDataParams对传入的参数做了封装。
继续查看LoadUrlParams，源码在：external/chromium_org/content/public/android/java/src/org/chromium/content/browser/LoadUrlParams.java。

```java
	public static LoadUrlParams createLoadDataParams(
        String data, String mimeType, boolean isBase64Encoded) {
        return createLoadDataParams(data, mimeType, isBase64Encoded, null);
    }

	public static LoadUrlParams createLoadDataParams(
            String data, String mimeType, boolean isBase64Encoded, String charset) {
        StringBuilder dataUrl = new StringBuilder("data:");
        //类型
        dataUrl.append(mimeType);
        //编码类型
        if (charset != null && !charset.isEmpty()) {
            dataUrl.append(";charset=" + charset);
        }
        //是否为base64编码
        if (isBase64Encoded) {
            dataUrl.append(";base64");
        }
        //分割符
        dataUrl.append(",");
        //网页
        dataUrl.append(data);

        LoadUrlParams params = new LoadUrlParams(dataUrl.toString());
        params.setLoadType(LoadUrlParams.LOAD_TYPE_DATA);
        params.setTransitionType(PageTransitionTypes.PAGE_TRANSITION_TYPED);
        return params;
    }
```
由以上源码可知：WebView的loadData方法传入的参数encoding并没有被封装到LoadUrlParams中，所以导致中文显示乱码。
调用createLoadDataParams(
            String data, String mimeType, boolean isBase64Encoded, String charset)方法肯定是能够将编码类型封装到LoadUrlParams。往回看看，哪里调用了该方法。WebViewChromium中的方法loadDataWithBaseURL(String baseUrl, String data, String mimeType, String encoding, String historyUrl)有调用到。
```java
	public void loadDataWithBaseURL(String baseUrl, String data, String mimeType, String encoding,
            String historyUrl) {
        data = fixupData(data);
        mimeType = fixupMimeType(mimeType);
        LoadUrlParams loadUrlParams;
        baseUrl = fixupBase(baseUrl);
        historyUrl = fixupHistory(historyUrl);

        if (baseUrl.startsWith("data:")) {
            // For backwards compatibility with WebViewClassic, we use the value of |encoding|
            // as the charset, as long as it's not "base64".
            boolean isBase64 = isBase64Encoded(encoding);
            //如果是base64编码：传入的编码类型为null；不是则传入设置的编码类型
            loadUrlParams = LoadUrlParams.createLoadDataParamsWithBaseUrl(
                    data, mimeType, isBase64, baseUrl, historyUrl, isBase64 ? null : encoding);
        } else {
            // When loading data with a non-data: base URL, the classic WebView would effectively
            // "dump" that string of data into the WebView without going through regular URL
            // loading steps such as decoding URL-encoded entities. We achieve this same behavior by
            // base64 encoding the data that is passed here and then loading that as a data: URL.
            try {
	            //设置为utf-8编码
                loadUrlParams = LoadUrlParams.createLoadDataParamsWithBaseUrl(
                        Base64.encodeToString(data.getBytes("utf-8"), Base64.DEFAULT), mimeType,
                        true, baseUrl, historyUrl, "utf-8");
            } catch (java.io.UnsupportedEncodingException e) {
                Log.wtf(TAG, "Unable to load data string " + data, e);
                return;
            }
        }
        loadUrlOnUiThread(loadUrlParams);
    }
```
WebView中的loadDataWithBaseURL(String baseUrl, String data, String mimeType, String encoding, String historyUrl) 方法调用了该方法，所以使用该方法能够解决乱码问题。
使用这种方式便可以解决中文乱码。
```java
loadDataWithBaseURL(null, "html", "text/html", "UTF-8", null);
```

