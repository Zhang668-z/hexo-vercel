# WebMagic的简单使用

## WebMagic介绍

### 1.1 WebMagic概览

[toc]WebMagic是一个简单灵活的Java爬虫框架。基于WebMagic，你可以快速开发出一个高效、易维护的爬虫。

WebMagic项目代码分为核心和扩展两部分。核心部分(webmagic-core)是一个精简的、模块化的爬虫实现，而扩展部分则包括一些便利的、实用性的功能。WebMagic的架构设计参照了Scrapy，目标是尽量的模块化，并体现爬虫的功能特点。

这部分提供非常简单、灵活的API，在基本不改变开发模式的情况下，编写一个爬虫。

扩展部分(webmagic-extension)提供一些便捷的功能，例如注解模式编写爬虫等。同时内置了一些常用的组件，便于爬虫开发。

### 1.2 特性：

- 简单的API，可快速上手
- 模块化的结构，可轻松扩展
- 提供多线程和分布式支持

### 1.3 总体架构

WebMagic的结构分为`Downloader`、`PageProcessor`、`Scheduler`、`Pipeline`四大组件，并由Spider将它们彼此组织起来。这四大组件对应爬虫生命周期中的下载、处理、管理和持久化等功能。WebMagic的设计参考了Scapy，但是实现方式更Java化一些。

而Spider则将这几个组件组织起来，让它们可以互相交互，流程化的执行，可以认为Spider是一个大的容器，它也是WebMagic逻辑的核心。

一般情况下我们只需要实现`PageProcessor`接口，自定义我们的爬虫抽取逻辑即可。

### 1.3.1 WebMagic的四个组件

#### 1.Downloader

Downloader负责从互联网上下载页面，以便后续处理。WebMagic默认使用了[Apache HttpClient](http://hc.apache.org/index.html)作为下载工具。

#### 2.PageProcessor

PageProcessor负责解析页面，抽取有用信息，以及发现新的链接。WebMagic使用[Jsoup](http://jsoup.org/)作为HTML解析工具，并基于其开发了解析XPath的工具[Xsoup](https://github.com/code4craft/xsoup)。

在这四个组件中，`PageProcessor`对于每个站点每个页面都不一样，是需要使用者定制的部分。

#### 3.Scheduler

Scheduler负责管理待抓取的URL，以及一些去重的工作。WebMagic默认提供了JDK的内存队列来管理URL，并用集合来进行去重。也支持使用Redis进行分布式管理。

除非项目有一些特殊的分布式需求，否则无需自己定制Scheduler。

#### 4.Pipeline

Pipeline负责抽取结果的处理，包括计算、持久化到文件、数据库等。WebMagic默认提供了“输出到控制台”和“保存到文件”两种结果处理方案。

`Pipeline`定义了结果保存的方式，如果你要保存到指定数据库，则需要编写对应的Pipeline。对于一类需求一般只需编写一个`Pipeline`。

### 参考

- Github地址：https://github.com/code4craft/webmagic
- 中文文档：http://webmagic.io/docs/zh/

## WebMagic的彼岸图网的DEMO

### 一、打开爬取的页面进行分析

#### 1.下载路径分析

以此网址<http://pic.netbian.com>为默认首页，打开页面如下所示：

![彼岸图网首页](https://s2.loli.net/2021/12/25/HlItCZwQBRUW8Mp.png)

按`F12`打开如下页面 选中一张图片可以看到如下便签

![](https://s2.loli.net/2021/12/25/GFCnrTftYoAyXLq.png)

我们可以看到很鲜明的标签规律  一个`<li>标签`是一张图片 每个里面包含一个`<a>标签`和一个`<span>标签`  `<a>标签`中的href属性为图片的详情地址后缀，将其拼接在首页的网址后面即可打开当前图片的详情页面

`<span>标签`中的src属性的为当前图片的缩略图地址 我们暂不考虑 4K图片才是我们要的 :smile: 

此页面没有4k图的下载地址 打开图片的详情页面 按下`F12` 选中下载原图 查看标签 如下

![](https://s2.loli.net/2021/12/25/cTAbJPukp4tQNdF.png)

点击下载原图 可以看到需要登录才可以下载 所以后续编辑代码时需要在代码模拟请求中将`Cookie信息添加到请求头`中 

F12选中下载原图后可以看到这个`<a>`标签 `<a href="javascript:;" data-id="21617">下载原图(4000x2357)</a>` 暂时qq登录 进行一元的充值 获取长达7天的下载时间后 :mask:  我们可以看到点击下载原图就可以触发浏览器下载  在href属性中我们并没有看到对应的下载链接 打开`network`后可以看到

![](https://s2.loli.net/2021/12/25/zSxgGuOw4XKakNo.png)

有两条请求记录存在  看请求名字可以推算到 是图片的下载请求 分别是<http://pic.netbian.com/e/extend/downpic.php?id=21617&t=0.7504811318948537> 以及<http://pic.netbian.com/downpic.php?id=21617&classid=66>

查看 第一条和第二条的响应头中的`Content-Type`可以看到

![](https://s2.loli.net/2021/12/25/TFVSsMyHOGDudB8.png)

![](https://s2.loli.net/2021/12/25/M8CObtXd4xJqnNZ.png)

第二个请求的响应头`content-Type`类型为`image/jpg` 因此第二条请求为真正的图片下载链接	

在查看第一条请求的`Response` 我们可以看到 

![](https://s2.loli.net/2021/12/25/TRgJ8eDpWZC1E2d.png)

响应数据是第二条的请求的请求路径  也就是说我们需要请求第一条路径拿到返回的路径信息 再请求真正的图片下载路径  但是经测试发现第二条的请求路径格式固定 只有其中的参数`id`代表不同的图片 而且其中的第二个参数`classid` 完全不用传入 仍然可以进行下载。而参数`id`的值就是详情页的下载原图按钮中的的`<a>标签`中的`data-id属性的值` 如下

![](https://s2.loli.net/2021/12/25/ajyu5NnGFMlHvJ3.png)

因此我们的最终下载链接即为<http://pic.netbian.com/downpic.php?id=21617>

我们的只需要通过爬虫获取`data-id`的值拼接到此路径中即可  `一定要模拟登录状态，即加入cookie信息,不然下载下来的照片大小为0 打开为损坏状态`

#### 2.翻页分析

首页为<http://pic.netbian.com>

第二页为<http://pic.netbian.com/index_2.html>

第三页为<http://pic.netbian.com/index_3.html>

因此翻页的路径变化就是页码的变化 

因此编写爬虫逻辑的时候需要区分首页和之后的页面 对应不同的路径

### 二、代码实现

#### 1.创建枚举类`SiteUrls`

```java
public enum SiteUrls {

    SITEURL("http://pic.netbian.com"), // 首页网址前缀
    PAGEURLPREFIX("http://pic.netbian.com/4kdongman/index_"), // 多页网址前缀(第一页、第二页等)
    DETAILPAGE("http://pic.netbian.com/tupian"), // 当前图片详情页网址前缀
    DOWNURL("http://pic.netbian.com/downpic.php?id="); // 图片下载地址前缀

    private String url;

    SiteUrls(String url) {
        this.url = url;
    }

    public String getUrl() {
        return this.url;
    }

}
```

#### 2.创建爬虫逻辑抽取类`PicProcesso`

```java
public class PicProcessor implements PageProcessor {

    private Site site = Site.me().setRetryTimes(3).setRetrySleepTime(1000);

    @Override
    // process是定制爬虫逻辑的核心接口，在这里编写抽取逻辑
    public void process(Page page) {
        if (page.getUrl().toString().startsWith(SiteUrls.DETAILPAGE.getUrl())) { // 详情页
            // 通过css样式定位到具体所需要的信息标签 获取属性中的信息
            String picId = page.getHtml().css("div.downpic a", "data-id").get();
            // 调用下载方法 传入下载路径
            DownloadUtil.downloadPic(SiteUrls.DOWNURL.getUrl() + picId, picId);
        } else {  // 表示列表页
            // 拿到一个列表页中多个图片的不同图片详情网址列表
            List<String> hrefs = page.getHtml().css("ul.clearfix>li>a", "href").all();
            for (String href : hrefs) {
                // 加入任务队列
                page.addTargetRequest(href);
            }
            // 获取当前列表url 进行字符串切割 获取当前页码
            String url = page.getUrl().get();
            String s = StringUtils.substringAfter(url,"_");
            String i = StringUtils.substringBefore(s, ".");
            if ("".equals(i)) {
                // 为空表示当前页为首页 进行第二页跳转
                i = "2";
            }else {
                // 页码加1
                i = (Integer.parseInt(i)+1)+"";
            }
            // 将下一页列表页加入到任务队列
            page.addTargetRequest(SiteUrls.PAGEURLPREFIX.getUrl() + i + ".html");
        }
    }

    @Override
    public Site getSite() {
        return site;
    }
}

```

#### 3.创建图片下载工具类`DownloadUtil`

```java
public class DownloadUtil {

    public static void downloadPic(String url, String picId) {
            FileOutputStream stream = null;
        try {
            RestTemplate template = new RestTemplate();
            // 设置请求头信息 cookie和referer
            HttpHeaders headers = new HttpHeaders();
            headers.add("Cookie","");
            headers.add("User-Agent","");
            headers.add("Referer","");
            HttpEntity<String> httpEntity = new HttpEntity<String>(null, headers);
            ResponseEntity<byte[]> entity = template.exchange(url, HttpMethod.GET, httpEntity, byte[].class);
            // 获取图片信息
            byte[] picData = entity.getBody();
            // 设置图片下载路径
            stream = new FileOutputStream(new File("C:\\Users\\Dell\\Desktop\\图片\\"+picId+".jpg"));
            // 信息写入文件
            stream.write(picData);
            } catch (Exception e) {
                e.printStackTrace();
                System.out.println("图片下载失败");
            }finally {
            try {
                stream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }

}
```

`下载类的9、10、11空的信息请自行填写，如有帮助或者需要超级会员无限下载的账号信息请联系我`

#### 4.代码启动入口类`Run`

```java
public class Run {
    public static void main(String[] args) {
		// 通过抽取逻辑对象创建自定义爬虫对象 添加首页网址以及设置爬取的线程数量 默认5个  
        Spider.create(new PicProcessor()).addUrl(SiteUrls.SITEURL.getUrl())
                .thread(5).run();
    }
}
```

`此文章仅供学习参考，如有问题，欢迎下方评论谢谢！！！`