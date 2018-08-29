# Okio的示例

## Okio的两个基本类`Source`(Input)和`Sink`(Output)

### `Source`的体系
![Source的体系](/OkioSource.png)

### `Sink`的体系
![Sink的体系](/OkioSink.png)

## Java IO的缺陷

__大量独立拓展的装饰者导致类爆炸__

Java的基础接口有四个`InputStream`、`OutputStream`、`Reader`、`Writer`。

Okio进行了简化，可以方便调用。
```java
try {
        Okio.buffer(Okio.sink(file))
            .writeUtf8("write string by utf-8.\n")
            .writeInt(1234).close();
    } catch (Exception e) {
        e.printStackTrace();
    }
```

## Okio的分析的文章
[深入理解okio的优化思想](https://blog.csdn.net/zoudifei/article/details/51232711)  
[OKio - 重新定义“短小精悍”](https://www.jianshu.com/p/5eaf26c58047)  
[大概是最完全的Okio源码解析文章](https://www.jianshu.com/p/f033a64539a1)    
[用轻和快定义优雅，Okio框架解析](https://www.jianshu.com/p/ea3ef6d7f01b)