*本文2014/05/03 发布于OSChina*

（一）中文路径编码配置

应项目的需要，要将一个具有中文的文件夹导入到Tomcat服务器的工程中。于是在过滤器中加入代码对URI进行编码，因为工程统一为UTF-8，所以采用UTF-8。

重启工程，打开浏览器输入地址访问，结果报404错误。查看日志，打印的URI与访问的一致。想想系统为GBK编码，那么就采用GBK对URI进行编码。重启，测试，还是一样的错误，日志中的URI正常。估计可能是服务器内部默认的URI编码导致的吧，因为Tomcat的默认编码都是ISO的。

在Tomcat的文档中找了一会，没找到。于是到stackoverflow上搜一搜，立即找到了相同的问题。原来需要在server.xml的Connector处添加URIEncoding="UTF-8"。

如下所示：

```xml
<Connector port="8080" protocol="HTTP/1.1" 
               connectionTimeout="20000" 
               redirectPort="8443" URIEncoding="UTF-8"/>
```

保存修改，重启，测试，访问显示正常，日志中打印的URI正常。又想起web.xml中还要两处UTF-8的字符编码配置被注释了，于是把注释去掉。

除去注释所示：
第一处，

```xml
<filter>
        <filter-name>setCharacterEncodingFilter</filter-name>
        <filter-class>org.apache.catalina.filters.SetCharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
</filter>
```

第二处，

```xml
<filter-mapping>
        <filter-name>setCharacterEncodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
 </filter-mapping>
```

保存修改，重启，测试，访问，一切正常。

（二）外部映射配置

由于工程中一文件夹内容较多，为方便以后更新，于是将移动某一个空盘中。在server.xml中的Host内部加入Context配置，如下所示（使用XX代替了真实路径）：

```xml
<Context path="/XX/XX/" docBase="F:/XX/XX"/>
```

保存修改，重启，测试，访问，一切正常。