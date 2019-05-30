# 部署
克隆完直接在idea运行

**已发现的bug都修复完成**

### [新功能](#1)

---

# 下面是debug记录


克隆完直接在idea运行，但是发现报错`sun.misc.BASE64Encoder找不到jar包`

解决方案：  
- 去掉导入包：
```
import sun.misc.BASE64Encoder;
import sun.misc.BASE64Decoder;
```
- 将原本的代码：
```
return new  BASE64Encoder().encode(encrypted);
```
- 替换为：
```
import org.apache.commons.codec.binary.Base64;
return Base64.encodeBase64String(encrypted);
```
- 将：
```
byte[] encrypted1 = new BASE64Decoder().decodeBuffer(text); 
```
- 替换为：
```
import org.apache.commons.codec.binary.Base64;
byte[] encrypted1 =Base64.decodeBase64(text); 
```

然后可以运行了。

### 七牛云配置
想要上传图片，就要配置七牛云，不然就是自己贴图片地址了。

令我惊讶的是七牛云 对象存储 **免费10G**。这有点爽到啊。
- [注册七牛云](https://portal.qiniu.com/)
- 新建一个对象存储bucket
- 拿密钥：个人中心->密钥管理
- 拿cdn(也就是外网访问的地址):对象存储->内容管理 中可以看到（ 注意：配置文件中cdn要加`http://`前缀 和 `/` 后缀不然图片无法访问），或者绑定个域名吧
- 在配置文件中配置好上面拿到的密钥、bucket名、cdn
- 在`api\QiniuCloudService`中更改下面的代码(这里改为zone2，因为zone2是华南的，我用的是华南的服务器。详情看后面的SDK文档)：
```
    Configuration cfg = new Configuration(Zone.zone0());
```
```
    Configuration cfg = new Configuration(Zone.zone2());
```
SDK文档：
> https://developer.qiniu.com/kodo/sdk/1239/java#upload-token

## 使用问题
写了解决方案的问题在我自己的部署中都fix了
### 新添文章在首页不显示
这个问题是第一次添加文章出现，貌似是一个bug，只有一个文章的时候，似乎是被前面的图片阻挡了看不到，发布第二篇文章之后成功了。

没有仔细看代码，目前不知道什么原因。

### 删除文章七牛云的图片没有删除
这个问题确实比较麻烦，不过考虑到博客发表了一般不会删除，先不进行处理。

**思路：**

- 将图片放在以文章id命名的文件夹
- 删除文章时删除七牛云中这个id的文件夹。

### 友链没有正确显示
友链管理中填入了链接地址、链接LOGO，但是却没有正确显示。

查看代码发现`templates\site\about.html`中
```
<div class="thumbnails-pan">
    <th:block th:each="link,linkStat: ${links}">
        <th:block th:if="${linkStat.index < 3}">
            <section class="col-xs-12 col-sm-4 col-md-4 col-lg-4 ">
                 <figure>
                    <img th:src="@{'/site/images/about-images/about-image-' + ${linkStat.index+1} + '.jpg'}" th:onclick="'javascript:linkFirend(\''+${link.slug}+'\')'" class="img-responsive"/>
                                    
```
首先这个each条件`linkStat`是用来统计数量，本来很苦恼为什么要统计，后来看到后面要用他来加载静态资源。静态资源只有3张图，所以在这里限制。因此，加载的图竟然是静态资源的图，所以友链添加的LOGO根本是白干。

回库查询发现，友链LOGO是`description`字段。这不是描述么。。并发现`about.html`后面有添加`description`。我猜是原设计被改了。

**解决方案：**  

在这里我改动使用`description`作为图片资源。然后删掉描述标签：

```
<div class="thumbnails-pan">
    <th:block th:each="link: ${links}">
        <section class="col-xs-12 col-sm-4 col-md-4 col-lg-4 ">
                <figure>
                    <img th:src="@{${link.description}}" th:onclick="'javascript:linkFirend(\''+${link.slug}+'\')'" class="img-responsive"/>

                    <figcaption>
                        <h3><th:block th:text="${link.name}"></th:block></h3>
                    </figcaption>
                </figure>
            </section>
    </th:block>
</div>
```
**debug过程中发现，这个`javascript:linkFirend`方法写在`templates\site\footer.html`中的百度收录自动推送模块中。。还有就是发现数据库`metas`表的`contentType`字段没用。不过都影响不大，暂不修改**

### 添加多个图片，却只显示一张图片
在增加作品时插入了多张图片，在查看的时候却只有第一张图片被显示，并且填写的文字都没了。

**解决方案**  

原因是正则表达式错了。找到`util\Commons.java`中的方法`show_all_thumb(String content)`改成如下代码：
```
List<String> rs = new LinkedList();
        content = TaleUtils.mdToHtml(content);
        if (content.contains("<img")) {
            String img = "";
            Pattern p_image;
            Matcher m_image;
            //图片链接地址
            String regEx_img = "<img.*src\\s*=\\s*(.*?)[^>]*?>";
            p_image = Pattern.compile
                    (regEx_img, Pattern.CASE_INSENSITIVE);
            m_image = p_image.matcher(content);
            while (m_image.find()) {
                // 得到<img />数据
                img = m_image.group();
                // 匹配<img>中的src数据
                Matcher m = Pattern.compile("src\\s*=\\s*\"?(.*?)(\"|>|\\s+)").matcher(img);
                while (m.find()) {
                    rs.add(m.group(1));
                }
            }
        }
        for (String s:rs
        ) {
            System.out.println(s);
        }
        return rs;
```
这样就可以加载多张图片了。但是还是没有文字，于是我添加多了一个实体类`WorkDomain`用于存储`<img>`标签中的`src`、`alt`、`title`属性。然后在前端遍历获取再拼接:  

- `Commons.java`的`show_all_thumb(String content)`:
```
    public static List<WorksDomain> show_all_thumb(String content) {
        List<WorksDomain> rs = new LinkedList();
        content = TaleUtils.mdToHtml(content);
        if (content.contains("<img")) {
            String img = "";
            Pattern p_image;
            Matcher m_image;
            //图片链接地址
            String regEx_img = "<img.*src\\s*=\\s*(.*?)[^>]*?>";
            p_image = Pattern.compile
                    (regEx_img, Pattern.CASE_INSENSITIVE);
            m_image = p_image.matcher(content);

            while (m_image.find()) {
                WorksDomain wd = new WorksDomain();
                // 得到<img />数据
                img = m_image.group();
                // 匹配<img>中的src数据
                Matcher m = Pattern.compile("src\\s*=\\s*\"?(.*?)(\"|>|\\s+)").matcher(img);
                while (m.find()) {
                    wd.setSrc(m.group(1));
                }
                // 匹配<img>中的alt数据
                Matcher a = Pattern.compile("alt\\s*=\\s*\"?(.*?)(\"|>|\\s+)").matcher(img);
                while (a.find()) {
                    wd.setAlt(a.group(1));
                }
                // 匹配<img>中的title数据
                Matcher t = Pattern.compile("title\\s*=\\s*\"?(.*?)(\"|>|\\s+)").matcher(img);
                while (t.find()) {
                    wd.setTitle(t.group(1));
                }
                rs.add(wd);
            }
        }
        return rs;
    }
```
- `work_detail.html`:
```
<div class="work-images grid">

    <ul class="grid-lod effect-2" id="grid">
        <th:block th:each="pic : ${commons.show_all_thumb(archive.content)}">
            <li><img th:src="${pic.src}" th:alt="${pic.alt}" th:title="${pic.title}" class="img-responsive"/></li>
            <br/>
        </th:block>
    </ul>

</div>
```
只剩下文字没有解决。想法：既然是发照片的，也不用那么多描述了，就有个标题就好了

# <span id="1">新功能</span>
2019/05/30 

添加了浏览次数显示功能，并修复浏览次数不更新的bug。