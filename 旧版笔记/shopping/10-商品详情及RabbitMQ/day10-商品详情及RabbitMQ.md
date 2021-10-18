# 0.学习目标

- 了解Thymeleaf的基本使用
- 实现商品详情页面
- 实现页面静态化功能
- 了解RabbitMQ的基本使用



# 1.商品详情

## 1.0.商品详情实现思路

当用户搜索到商品，肯定会点击查看，就会进入商品详情页，接下来我们完成商品详情页的展示，商品详情页在leyou-portal中对应的页面是：item.html

但是不同的商品，到达item.html需要展示的内容不同，该怎么做呢？

- 思路1：统一跳转到item.html页面，然后异步加载商品数据，渲染页面
- 思路2：将请求交给tomcat处理，在服务端完成数据渲染，给不同商品生成不同页面后，返回给用户

我们该选哪一种思路？

思路1：

- 优点：页面加载快，异步处理，用户体验好
- 缺点：会向服务端发起多次数据请求，增加服务端压力

思路2：

- 优点：服务端处理页面后返回，用户拿到是最终页面，不会再次向服务端发起数据请求。
- 缺点：在服务端处理页面，服务端压力过大，tomcat并发能力差



对于大型电商网站而言，必须要考虑的就是服务的高并发问题，因此要尽可能减少服务端压力，提高服务响应速度，所以这里我们两个方案都不会用，我们采用方案3：

方案3：页面静态化

页面静态化：顾名思义，就是把本来需要动态渲染的页面提前渲染完成，生成静态的HTML，当用户访问时直接读取静态HTML，提高响应速度，减轻服务端压力。

这种方式与方案2类似，都是要为每个商品生成独立页面，不同之处在于方案2需要每次都重新渲染，而静态化不需要这么频繁。不过，其中还有很多细节问题需要解决。

我们可以先实现方式2的方式，再来对比两者区别，改造为页面静态化方案。



后两种方案都是在服务端完成页面渲染，以前服务端渲染我们都使用的JSP，不过在SpringBoot中已经不推荐使用Jsp了，因此我们会使用另外的模板引擎技术：Thymeleaf

与其类似的模板渲染技术有：Freemarker、Velocity等都可以。

## 1.1.Thymeleaf入门

在商品详情页中，我们会使用到Thymeleaf来渲染页面，如果需要先了解Thymeleaf的语法。详见课前资料中《Thymeleaf语法入门.md》



## 1.2.商品详情页服务

我们创建一个微服务，用来完成商品详情页的渲染。

### 1.2.1.创建module

商品的详情页服务，命名为：`ly-page`

![1553221580631](assets/1553221580631.png)

目录：

![1553221600298](assets/1553221600298.png) 

### 1.2.2.pom依赖

```XML
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>leyou</artifactId>
        <groupId>com.leyou</groupId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>ly-page</artifactId>

    <dependencies>
        <!--eureka-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <!--web-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!--thymeleaf-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <!--feign-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <!-- 商品服务接口 -->
        <dependency>
            <groupId>com.leyou</groupId>
            <artifactId>ly-item-interface</artifactId>
            <version>1.0.0-SNAPSHOT</version>
        </dependency>
        <!--单元测试-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### 1.2.3.编写启动类：

```java
@EnableDiscoveryClient
@EnableFeignClients
@SpringBootApplication
public class LyPageApplication {
    public static void main(String[] args) {
        SpringApplication.run(LyPageApplication.class, args);
    }
}
```

### 1.2.4.application.yml文件

```yaml
server:
  port: 8084
spring:
  application:
    name: page-service
  thymeleaf:
    cache: false
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:10086/eureka
```



### 1.2.5.页面模板：

我们从课前资料中复制item.html模板：

 ![1535356585076](assets/1535356585076.png)

到当前项目resource目录下的template中：

  ![1535356618281](assets/1535356618281.png)



## 1.3.页面跳转

### 1.3.1.修改页面跳转路径

首先我们需要修改搜索结果页的商品地址，目前所有商品的地址都是：http://www.leyou.com/item.html

 ![1526955707685](assets/1526955707685.png)

我们应该跳转到对应的商品的详情页才对。

那么问题来了：商品详情页是一个SKU？还是多个SKU的集合？

![1526955852490](assets/1526955852490.png)

通过详情页的预览，我们知道它是多个SKU的集合，即SPU。

所以，页面跳转时，我们应该携带SPU的id信息。

例如：http://www.leyou.com/item/2314123.html

这里就采用了路径占位符的方式来传递spu的id，我们打开`search.html`，修改其中的商品路径：

 ![1526972476737](assets/1526972476737.png)

刷新页面后在看：

 ![1526972581134](assets/1526972581134.png)

### 1.3.2.nginx反向代理

接下来，我们要把这个地址指向我们刚刚创建的服务：`ly-page`，其端口为8084

我们在nginx.conf中添加一段逻辑：

 ```nginx
server {
	listen       80;
	server_name  www.leyou.com;
	location /item {
		proxy_pass	http://127.0.0.1:8084;
	}
	location / {
		proxy_pass   http://leyou-portal;
	}
}
 ```

把以/item开头的请求，代理到我们的8084端口

### 1.3.3.编写跳转controller

在`ly-page`中编写controller，接收请求，并跳转到商品详情页：

```java
@Controller
public class PageController {

    /**
     * 跳转到商品详情页
     * @param model
     * @param id
     * @return
     */
    @GetMapping("item/{id}.html")
    public String toItemPage(Model model, @PathVariable("id")Long id){

        return "item";
    }
}
```



### 1.3.4.测试

启动`ly-page`，点击搜索页面商品，看是能够正常跳转：

![1535422129984](assets/1535422129984.png)

现在看到是500，因为Thymeleaf模板渲染失败了，缺少模板渲染需要的Model数据：

![1535422177254](assets/1535422177254.png)

因此接下来，必须提供Model数据，完成页面渲染。

## 1.4.完成页面渲染

### 1.4.1.页面数据分析

首先我们一起来分析一下，在这个页面中需要哪些数据，这个可以查看`item.html`中的Thymeleaf语法得知，例如有下面的部分：

![1553245845879](assets/1553245845879.png)

逐个查看，发现需要下面的变量：

- categories：商品分类对象集合
- brand：品牌对象
- spuName：应该 是spu表中的name属性
- subTitle：spu中 的副标题
- detail：商品详情SpuDetail
- skus：商品spu下的sku集合
- specs：规格参数这个比较 特殊：
  - ![1553246049965](assets/1553246049965.png)
  - 通过上述代码，可以知道specs是规格组的集合，组内要包含params参数，就是该组下的规格参数的集合



我们已知的条件是传递来的spu的id，因此我们需要下面的一些查询接口：

- 根据id查询spu，最好同时查询出spuDetail和skus，这样比分三次查询效率较高。（没有这样的接口）
- 根据品牌id查询品牌（有）
- 根据分类id集合查询商品分类集合（有）
- 根据分类id查询规格组及组内规格参数没有这样的接口）

因此接下来，我们需要在商品微服务补充2个接口。

### 1.4.2.商品微服务提供接口

#### 查询spu接口

以上所需数据中，查询spu的接口目前还没有，我们需要在商品微服务中提供这个接口：

首先在`ly-item-interface`中对外提供的Feign客户端（`ItemClient`）中定义接口：

> ItemClient

```java
/**
 * 根据spu的id查询spu
 * @param id
 * @return
 */
@GetMapping("spu/{id}")
SpuDTO querySpuById(@PathVariable("id") Long id);
```

> GoodsController

```java
/**
     * 根据spu的id查询spu
     * @param id
     * @return
     */
@GetMapping("spu/{id}")
public ResponseEntity<SpuDTO> querySpuById(@PathVariable("id") Long id){
    return ResponseEntity.ok(goodsService.querySpuById(id));
}
```

> GoodsService

```java
public SpuDTO querySpuById(Long id) {
    // 查询spu
    Spu spu = spuMapper.selectByPrimaryKey(id);
    SpuDTO spuDTO = BeanHelper.copyProperties(spu, SpuDTO.class);
    // 查询spuDetail
    spuDTO.setSpuDetail(querySpuDetailById(id));
    // 查询sku
    spuDTO.setSkus(querySkuListBySpuId(id));
    return spuDTO;
}
```

#### 查询规格参数组

我们在页面展示规格时，需要按组展示：

 ![1535423451465](assets/1535423451465.png)

组内有多个参数，为了方便展示。我们提供一个接口，查询规格组，同时在规格组中持有组内的所有参数。

> 拓展`SpecGroupDTO`类：

我们在`SpecGroupDTO`中添加一个`SpecParamDTO`的集合，保存改组下所有规格参数

```java
@Data
public class SpecGroupDTO {
    private Long id;

    private Long cid;

    private String name;
    
    private List<SpecParamDTO> params;
}
```

在`ly-item-interface`中对外提供的Feign客户端（`ItemClient`）中定义接口：

> ItemClient

```java
/**
     * 查询规格参数组，及组内参数
     * @param id 商品分类id
     * @return 规格组及组内参数
     */
@GetMapping("/spec/of/category")
List<SpecGroupDTO> querySpecsByCid(@RequestParam("id") Long id);
```

> SpecController

```java
/**
     * 查询规格参数组，及组内参数
     * @param id 商品分类id
     * @return 规格组及组内参数
     */
@GetMapping("/of/category")
public ResponseEntity<List<SpecGroupDTO>> querySpecsByCid(@RequestParam("id") Long id){
    return ResponseEntity.ok(specService.querySpecsByCid(id));
}
```

> SpecificationService

```java
public List<SpecGroupDTO> querySpecsByCid(Long id) {
    // 查询规格组
    List<SpecGroupDTO> groupList = queryGroupByCategory(id);
    // 查询分类下所有规格参数
    List<SpecParamDTO> params = querySpecParams(null, id, null);
    // 将规格参数按照groupId进行分组，得到每个group下的param的集合
    Map<Long, List<SpecParamDTO>> paramMap = params.stream()
        .collect(Collectors.groupingBy(SpecParamDTO::getGroupId));
    // 填写到group中
    for (SpecGroupDTO groupDTO : groupList) {
        groupDTO.setParams(paramMap.get(groupDTO.getId()));
    }
    return groupList;
}
```

在service中，我们调用之前编写过的方法，查询规格组，和规格参数，然后封装返回。

### 1.4.3.完善页面跳转controller

在之前的页面跳转controller中，缺乏模型数据，导致页面渲染失败 。我们需要准备好数据，存入Model中，传递给页面模板。

修改controller代码：

```java
@Controller
public class PageController {
    
    @Autowired
    private PageService pageService;

    /**
     * 跳转到商品详情页
     * @param model
     * @param id
     * @return
     */
    @GetMapping("item/{id}.html")
    public String toItemPage(Model model, @PathVariable("id")Long id){
        // 查询模型数据
        Map<String,Object> itemData = pageService.loadItemData(id);
        // 存入模型数据，因为数据较多，直接存入一个map
        model.addAllAttributes(itemData);
        return "item";
    }
}
```



注意我们把数据的查询和加载放到了一个PageService中去完成，数据较多，因此这里的PageService方法返回的是一个Map。



### 1.4.4.封装Model数据

我们创建一个PageService，在里面来封装数据模型。

> Service代码

```java
package com.leyou.page.service;

import com.leyou.item.client.ItemClient;
import com.leyou.item.dto.BrandDTO;
import com.leyou.item.dto.CategoryDTO;
import com.leyou.item.dto.SpecGroupDTO;
import com.leyou.item.dto.SpuDTO;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * @author 黑马程序员
 */
@Service
public class PageService {

    @Autowired
    private ItemClient itemClient;

    public Map<String, Object> loadItemData(Long id) {
        // 查询spu
        SpuDTO spu = itemClient.querySpuById(id);
        // 查询分类集合
        List<CategoryDTO> categories = itemClient.queryCategoryByIds(spu.getCategoryIds());
        // 查询品牌
        BrandDTO brand = itemClient.queryBrandById(spu.getBrandId());
        // 查询规格
        List<SpecGroupDTO> specs = itemClient.querySpecsByCid(spu.getCid3());
        // 封装数据
        Map<String, Object> data = new HashMap<>();
        data.put("categories", categories);
        data.put("brand", brand);
        data.put("spuName", spu.getName());
        data.put("subTitle", spu.getSubTitle());
        data.put("skus", spu.getSkus());
        data.put("detail", spu.getSpuDetail());
        data.put("specs", specs);
        return data;
    }
}
```



### 1.4.5.页面测试数据

页面的数据渲染代码已经帮大家写好了，因此启动项目可以直接看到最终的效果：

![1535357366958](assets/1535357366958.png)



# 2.页面静态化

## 2.1.简介

### 2.1.1.问题分析

现在，我们的页面是通过Thymeleaf模板引擎渲染后返回到客户端。在后台需要大量的数据查询，而后渲染得到HTML页面。会对数据库造成压力，并且请求的响应时间过长，并发能力不高。

大家能想到什么办法来解决这个问题？

- 缓存，后端可以对要查询的数据进行缓存，提高查询效率，减小数据库压力。
  - 优点：以后无论是页面查询数据、移动的查询数据，都走缓存，更通用
  - 缺点：请求依然会到达服务端，造成服务端压力。
- 静态化，把页面渲染后写入HTML文件，以后不再查询服务端数据
  - 优点：不访问服务端，效率更高
  - 缺点：适应于H5，APP中需要另外的静态化方案

### 2.1.2.什么是静态化

静态化是指把动态生成的HTML页面变为静态内容保存，以后用户的请求到来，直接访问静态页面，不再经过服务的渲染。

而静态的HTML页面可以部署在nginx中，从而大大提高并发能力，减小tomcat压力。



### 2.1.3.如何实现静态化

目前，静态化页面都是通过模板引擎来生成，而后保存到nginx服务器来部署。常用的模板引擎比如：

- Freemarker
- Velocity
- Thymeleaf

我们之前就使用的Thymeleaf，来渲染html返回给用户。Thymeleaf除了可以把渲染结果写入Response，也可以写到本地文件，从而实现静态化。

## 2.2.Thymeleaf生成静态页

### 2.2.1.概念

先说下Thymeleaf中的几个概念：

- Context：运行上下文
- TemplateResolver：模板解析器
- TemplateEngine：模板引擎

> Context

上下文： 用来保存模型数据，当模板引擎渲染时，可以从Context上下文中获取数据用于渲染。

当与SpringBoot结合使用时，我们放入Model的数据就会被处理到Context，作为模板渲染的数据使用。

> TemplateResolver

模板解析器：用来读取模板相关的配置，例如：模板存放的位置信息，模板文件名称，模板文件的类型等等。

当与SpringBoot结合时，TemplateResolver已经由其创建完成，并且各种配置也都有默认值，比如模板存放位置，其默认值就是：templates。比如模板文件类型，其默认值就是html。

> TemplateEngine

模板引擎：用来解析模板的引擎，需要使用到上下文、模板解析器。分别从两者中获取模板中需要的数据，模板文件。然后利用内置的语法规则解析，从而输出解析后的文件。来看下模板引起进行处理的函数：

```java
templateEngine.process("模板名", context, writer);
```

三个参数：

- 模板名称
- 上下文：里面包含模型数据
- writer：输出目的地的流

在输出时，我们可以指定输出的目的地，如果目的地是Response的流，那就是网络响应。如果目的地是本地文件，那就实现静态化了。

而在SpringBoot中已经自动配置了模板引擎，因此我们不需要关心这个。现在我们做静态化，就是把输出的目的地改成本地文件即可！

### 2.2.2.具体实现

我们在PageService中，编写方法，实现静态化功能：

```java
/**
 * @author 黑马程序员
 */
@Slf4j
@Service
public class PageService {

    @Autowired
    private ItemClient itemClient;

    @Autowired
    private SpringTemplateEngine templateEngine;

    @Value("${ly.static.itemDir}")
    private String itemDir;
    @Value("${ly.static.itemTemplate}")
    private String itemTemplate;

    public Map<String, Object> loadItemData(Long id) {
        // 查询spu
        SpuDTO spu = itemClient.querySpuById(id);
        // 查询分类集合
        List<CategoryDTO> categories = itemClient.queryCategoryByIds(spu.getCategoryIds());
        // 查询品牌
        BrandDTO brand = itemClient.queryBrandById(spu.getBrandId());
        // 查询规格
        List<SpecGroupDTO> specs = itemClient.querySpecsByCid(spu.getCid3());
        // 封装数据
        Map<String, Object> data = new HashMap<>();
        data.put("categories", categories);
        data.put("brand", brand);
        data.put("spuName", spu.getName());
        data.put("subTitle", spu.getSubTitle());
        data.put("skus", spu.getSkus());
        data.put("detail", spu.getSpuDetail());
        data.put("specs", specs);
        return data;
    }

    public void createItemHtml(Long id) {
        // 上下文，准备模型数据
        Context context = new Context();
        // 调用之前写好的加载数据方法
        context.setVariables(loadItemData(id));
        // 准备文件路径
        File dir = new File(itemDir);
        if (!dir.exists()) {
            if (!dir.mkdirs()) {
                // 创建失败，抛出异常
                log.error("【静态页服务】创建静态页目录失败，目录地址：{}", dir.getAbsolutePath());
                throw new LyException(ExceptionEnum.DIRECTORY_WRITER_ERROR);
            }
        }
        File filePath = new File(dir, id + ".html");
        // 准备输出流
        try (PrintWriter writer = new PrintWriter(filePath, "UTF-8")) {
            templateEngine.process(itemTemplate, context, writer);
        } catch (IOException e) {
            log.error("【静态页服务】静态页生成失败，商品id：{}", id, e);
            throw new LyException(ExceptionEnum.FILE_WRITER_ERROR);
        }
    }
}
```

在application.yml中配置生成静态文件的目录：

```yaml
ly:
  static:
    itemDir: C:\\develop\\nginx-1.12.2\\html\\item
    itemTemplate: item
```



### 2.2.4.单元测试：

我们编写一个单元测试：

```java
package com.leyou.page.service;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import static org.junit.Assert.*;

@RunWith(SpringRunner.class)
@SpringBootTest
public class PageServiceTest {

    @Autowired
    private PageService pageService;

    @Test
    public void createItemHtml() throws InterruptedException {
        Long[] arr = {96L, 114L, 124L, 125L, 141L};
        for (Long id : arr) {
            pageService.createItemHtml(id);
            Thread.sleep(500);
        }
    }
}
```

然后到nginx下查看：

![1553252794316](assets/1553252794316.png) 





## 2.3.nginx代理静态页面

接下来，我们修改nginx，让它对商品请求进行监听，指向本地静态页面，如果本地没找到，才进行反向代理：

```nginx
server {
	listen       80;
	server_name  www.leyou.com;
	location /item {
		# 先找本地
        root html;
        if (!-f $request_filename) { #请求的文件不存在，就反向代理
            proxy_pass http://127.0.0.1:8084;
            break;
        }
	}
	location / {
		proxy_pass   http://leyou-portal;
	}
}
```

重启测试：

发现请求速度得到了极大提升：

![1553252883092](assets/1553252883092.png)



**`注意！！！`**，这里为了演示方便我们采用了if判断文件是否存在，不存在则反向代理到服务端，服务端生成页面返回。这种做法有`缓存穿透`的风险，建议不要加入if判断，如果nginx中不存在商品的静态页，直接返回404即可。

即：

```nginx
server {
	listen       80;
	server_name  www.leyou.com;
	location /item {
        root html;
	}
	location / {
		proxy_pass   http://leyou-portal;
	}
}
```



## 2.4.静态化的数据同步问题

我们思考这样几个问题：

- 商品详情页应该在什么时候生成呢？不能每次都用单元测试生成吧？
- 如果商品数据修改以后，静态页的内容与商品实际内容不符，该如何完成同步？
- 商品下架后，不应该再展示商品页面，静态页如何处理？



思考上面问题的同时，我们会想起一件事情，其实商品数据如果发生了增、删、改，不仅仅静态页面需要处理，我们的索引库数据也需要同步！！这又该如何解决？



因为商品新增后需要上架用户才能看到，商品修改需要先下架，然后修改，再上架。因此上述问题可以统一的设计成这样的逻辑处理：

- 商品上架：
  - 生成静态页
  - 新增索引库数据
- 商品下架：
  - 删除静态页
  - 删除索引库数据

这样既可保证数据库商品与索引库、静态页三者之间的数据同步。

那么，如何实现上述逻辑呢？



# 3.RabbitMQ

## 3.1.搜索与商品服务的问题

上一节中，我们要实现这样的效果：商品上架或下架时，静态页及索引库要做出相应的处理。我们思考一下，是否存在问题？

- 商品数据的上架或下架都在商品微服务（`item-service`）中完成。
- 索引库数据的增删改是在搜索微服务（`search-service`）中完成。
- 商品详情做了页面静态化，静态页面数据的操作是在静态页服务（`page-service`）中完成。

搜索服务和静态页服务，并不知晓商品微服务对商品数据的！！如何解决？



这里有两种解决方案：

- 方案1：在商品微服务的上下架业务后，加入修改索引库数据及静态页面的代码

- 方案2：搜索服务和静态页服务对外提供操作索引库和静态页接口，商品微服务在商品上下架后，调用接口。



以上两种方式都有同一个严重问题：就是代码耦合，后台服务中需要嵌入搜索和商品页面服务，违背了微服务的`独立`原则，而且严重违背了开闭原则。

所以，我们会通过另外一种方式来解决这个问题：消息队列

## 3.2.消息队列（MQ）

### 3.2.1.什么是消息队列

消息队列，即MQ，Message Queue。

![1527063872737](assets/1527063872737.png)



消息队列是典型的：生产者、消费者模型。生产者不断向消息队列中生产消息，消费者不断的从队列中获取消息。因为消息的生产和消费都是异步的，而且只关心消息的发送和接收，没有业务逻辑的侵入，这样就实现了生产者和消费者的解耦。

结合前面所说的问题：

- 商品服务对商品上下架以后，无需去操作索引库或静态页面，只是发送一条消息，也不关心消息被谁接收。
- 搜索服务和静态页面服务接收消息，分别去处理索引库和静态页面。

如果以后有其它系统也依赖商品服务的数据，同样监听消息即可，商品服务无需任何代码修改。



### 3.2.2.AMQP和JMS

MQ是消息通信的模型，并发具体实现。现在实现MQ的有两种主流方式：AMQP、JMS。

![1527064480681](assets/1527064480681.png)

![1527064487042](assets/1527064487042.png)



两者间的区别和联系：

- JMS是定义了统一的接口，来对消息操作进行统一；AMQP是通过规定协议来统一数据交互的格式
- JMS限定了必须使用Java语言；AMQP只是协议，不规定实现方式，因此是跨语言的。
- JMS规定了两种消息模型；而AMQP的消息模型更加丰富



### 3.2.3.常见MQ产品

![1527064606029](assets/1527064606029.png)



- ActiveMQ：基于JMS， Apache

- RabbitMQ：基于AMQP协议，erlang语言开发，稳定性好
- RocketMQ：基于JMS，阿里巴巴产品，目前交由Apache基金会
- Kafka：分布式消息系统，高吞吐量

### 3.2.4.RabbitMQ

RabbitMQ是基于AMQP的一款消息管理系统

官网： http://www.rabbitmq.com/

官方教程：http://www.rabbitmq.com/getstarted.html

![1527064881869](assets/1527064881869.png)



 ![1527064762982](assets/1527064762982.png)

RabbitMQ基于Erlang语言开发：

![1527065024587](assets/1527065024587.png)



## 3.3.下载和安装

### 3.3.1.下载

官网下载地址：http://www.rabbitmq.com/download.html

 ![1527065157430](assets/1527065157430.png)

目前最新版本是：3.7.5

我们的课程中使用的是：3.4.1版本

课前资料提供了安装包：

  ![1527066149086](assets/1527066149086.png)

### 3.3.2.安装

详见课前资料中的：

 ![1527066164552](assets/1527066164552.png)

## 3.4.五种消息模型

RabbitMQ提供了6种消息模型，但是第6种其实是RPC，并不是MQ，因此不予学习。那么也就剩下5种。

但是其实3、4、5这三种都属于订阅模型，只不过进行路由的方式不同。

![1527068544487](assets/1527068544487.png)



### 3.4.1.导入demo工程

我们通过一个demo工程来了解下RabbitMQ的工作方式：

导入工程：

 ![1527090513200](assets/1527090513200.png)

导入后：

 ![1527070384668](assets/1527070384668.png)



### 3.4.2.基本消息模型

#### 说明

官方文档说明：

> RabbitMQ是一个消息的代理者（Message Broker）：它接收消息并且传递消息。
>
> 你可以认为它是一个邮局：当你投递邮件到一个邮箱，你很肯定邮递员会终究会将邮件递交给你的收件人。与此类似，RabbitMQ 可以是一个邮箱、邮局、同时还有邮递员。
>
> 不同之处在于：RabbitMQ不是传递纸质邮件，而是二进制的数据

基本消息模型图：

 ![1527070619131](assets/1527070619131.png)

在上图的模型中，有以下概念：

- P：生产者，也就是要发送消息的程序
- C：消费者：消息的接受者，会一直等待消息到来。
- queue：消息队列，图中红色部分。类似一个邮箱，可以缓存消息；生产者向其中投递消息，消费者从其中取出消息。

#### 生产者

连接工具类：

```java
public class ConnectionUtil {
    /**
     * 建立与RabbitMQ的连接
     * @return
     * @throws Exception
     */
    public static Connection getConnection() throws Exception {
        //定义连接工厂
        ConnectionFactory factory = new ConnectionFactory();
        //设置服务地址
        factory.setHost("192.168.56.101");
        //端口
        factory.setPort(5672);
        //设置账号信息，用户名、密码、vhost
        factory.setVirtualHost("/leyou");
        factory.setUsername("leyou");
        factory.setPassword("leyou");
        // 通过工程获取连接
        Connection connection = factory.newConnection();
        return connection;
    }
}
```



生产者发送消息：

```java
public class Send {

    private final static String QUEUE_NAME = "simple_queue";

    public static void main(String[] argv) throws Exception {
        // 获取到连接
        Connection connection = ConnectionUtil.getConnection();
        // 从连接中创建通道，使用通道才能完成消息相关的操作
        Channel channel = connection.createChannel();
        // 声明（创建）队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        // 消息内容
        String message = "Hello World!";
        // 向指定的队列中发送消息
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
        
        System.out.println(" [x] Sent '" + message + "'");

        //关闭通道和连接
        channel.close();
        connection.close();
    }
}
```

控制台：

 ![1527072505530](assets/1527072505530.png)

#### web控制台查看消息

进入队列页面，可以看到新建了一个队列：simple_queue

![1527072699034](assets/1527072699034.png)

点击队列名称，进入详情页，可以查看消息：

 ![1527072746634](assets/1527072746634.png)

在控制台查看消息并不会将消息消费，所以消息还在。



#### 消费者获取消息

```java
public class Recv {
    private final static String QUEUE_NAME = "simple_queue";

    public static void main(String[] argv) throws Exception {
        // 获取到连接
        Connection connection = ConnectionUtil.getConnection();
        // 创建通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        // 定义队列的消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            // 获取消息，并且处理，这个方法类似事件监听，如果有消息的时候，会被自动调用
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                    byte[] body) throws IOException {
                // body 即消息体
                String msg = new String(body);
                System.out.println(" [x] received : " + msg + "!");
            }
        };
        // 监听队列，第二个参数：是否自动进行消息确认。
        channel.basicConsume(QUEUE_NAME, true, consumer);
    }
}
```

控制台：

 ![1527072874080](assets/1527072874080.png)

这个时候，队列中的消息就没了：

![1527072947108](assets/1527072947108.png)



#### 消费者的消息确认机制

通过刚才的案例可以看出，消息一旦被消费者接收，队列中的消息就会被删除。

那么问题来了：RabbitMQ怎么知道消息被接收了呢？

这就要通过消息确认机制（Acknowlege）来实现了。当消费者获取消息后，会向RabbitMQ发送回执ACK，告知消息已经被接收。不过这种回执ACK分两种情况：

- 自动ACK：消息一旦被接收，消费者自动发送ACK
- 手动ACK：消息接收后，不会发送ACK，需要手动调用

大家觉得哪种更好呢？

这需要看消息的重要性：

- 如果消息不太重要，丢失也没有影响，那么自动ACK会比较方便
- 如果消息非常重要，不容丢失。那么最好在消费完成后手动ACK，否则接收消息后就自动ACK，RabbitMQ就会把消息从队列中删除。如果此时消费者宕机，那么消息就丢失了。

我们之前的测试都是自动ACK的，如果要手动ACK，需要改动我们的代码：

```java
public class Recv2 {
    private final static String QUEUE_NAME = "simple_queue";

    public static void main(String[] argv) throws Exception {
        // 获取到连接
        Connection connection = ConnectionUtil.getConnection();
        // 创建通道
        final Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        // 定义队列的消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            // 获取消息，并且处理，这个方法类似事件监听，如果有消息的时候，会被自动调用
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                    byte[] body) throws IOException {
                // body 即消息体
                String msg = new String(body);
                System.out.println(" [x] received : " + msg + "!");
                // 手动进行ACK
                channel.basicAck(envelope.getDeliveryTag(), false);
            }
        };
        // 监听队列，第二个参数false，手动进行ACK
        channel.basicConsume(QUEUE_NAME, false, consumer);
    }
}
```

注意到最后一行代码：

```java
channel.basicConsume(QUEUE_NAME, false, consumer);
```

如果第二个参数为true，则会自动进行ACK；如果为false，则需要手动ACK。方法的声明：

![1527073500352](assets/1527073500352.png)



### 3.4.3.work消息模型

#### 说明

在刚才的基本模型中，一个生产者，一个消费者，生产的消息直接被消费者消费。比较简单。

Work queues，也被称为（Task queues），任务模型。

当消息处理比较耗时的时候，可能生产消息的速度会远远大于消息的消费速度。长此以往，消息就会堆积越来越多，无法及时处理。此时就可以使用work 模型：**让多个消费者绑定到一个队列，共同消费队列中的消息**。队列中的消息一旦消费，就会消失，因此任务是不会被重复执行的。

 ![1527078437166](assets/1527078437166.png)

角色：

- P：生产者：任务的发布者
- C1：消费者，领取任务并且完成任务，假设完成速度较慢
- C2：消费者2：领取任务并完成任务，假设完成速度快



#### 生产者

生产者与案例1中的几乎一样：

```java
public class Send {
    private final static String QUEUE_NAME = "test_work_queue";

    public static void main(String[] argv) throws Exception {
        // 获取到连接
        Connection connection = ConnectionUtil.getConnection();
        // 获取通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        // 循环发布任务
        for (int i = 0; i < 50; i++) {
            // 消息内容
            String message = "task .. " + i;
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            System.out.println(" [x] Sent '" + message + "'");

            Thread.sleep(i * 2);
        }
        // 关闭通道和连接
        channel.close();
        connection.close();
    }
}
```

不过这里我们是循环发送50条消息。

#### 消费者1

![1527085386747](assets/1527085386747.png)

#### 消费者2

![1527085448377](assets/1527085448377.png)

与消费者1基本类似，就是没有设置消费耗时时间。

这里是模拟有些消费者快，有些比较慢。



接下来，两个消费者一同启动，然后发送50条消息：

![1527085826462](assets/1527085826462.png)

可以发现，两个消费者各自消费了25条消息，而且各不相同，这就实现了任务的分发。



#### 能者多劳

刚才的实现有问题吗？

- 消费者1比消费者2的效率要低，一次任务的耗时较长
- 然而两人最终消费的消息数量是一样的
- 消费者2大量时间处于空闲状态，消费者1一直忙碌

现在的状态属于是把任务平均分配，正确的做法应该是消费越快的人，消费的越多。

怎么实现呢？

我们可以修改设置，让消费者同一时间只接收一条消息，这样处理完成之前，就不会接收更多消息，就可以让处理快的人，接收更多消息 ：

![1527086103576](assets/1527086103576.png)

再次测试：

![1527086159534](assets/1527086159534.png)



### 3.4.4.订阅模型分类

订阅模型示意图：

 ![1527086284940](assets/1527086284940.png)

前面2个案例中，只有3个角色：

- P：生产者，也就是要发送消息的程序
- C：消费者：消息的接受者，会一直等待消息到来。
- queue：消息队列，图中红色部分。类似一个邮箱，可以缓存消息；生产者向其中投递消息，消费者从其中取出消息。

而在订阅模型中，多了一个exchange角色，而且过程略有变化：

- P：生产者，也就是要发送消息的程序，但是不再发送到队列中，而是发给X（交换机）
- C：消费者，消息的接受者，会一直等待消息到来。
- Queue：消息队列，接收消息、缓存消息。
-  Exchange：交换机，图中的X。一方面，接收生产者发送的消息。另一方面，知道如何处理消息，例如递交给某个特别队列、递交给所有队列、或是将消息丢弃。到底如何操作，取决于Exchange的类型。Exchange有以下3种类型：
  - Fanout：广播，将消息交给所有绑定到交换机的队列
  - Direct：定向，把消息交给符合指定routing key 的队列
  - Topic：通配符，把消息交给符合routing pattern（路由模式） 的队列

**Exchange（交换机）只负责转发消息，不具备存储消息的能力**，因此如果没有任何队列与Exchange绑定，或者没有符合路由规则的队列，那么消息会丢失！



### 3.4.5.订阅模型-Fanout

Fanout，也称为广播。

#### 流程说明

流程图：

 ![1527086564505](assets/1527086564505.png)

在广播模式下，消息发送流程是这样的：

- 1）  可以有多个消费者
- 2）  每个**消费者有自己的queue**（队列）
- 3）  每个**队列都要绑定到Exchange**（交换机）
- 4）  **生产者发送的消息，只能发送到交换机**，交换机来决定要发给哪个队列，生产者无法决定。
- 5）  交换机把消息发送给绑定过的所有队列
- 6）  队列的消费者都能拿到消息。实现一条消息被多个消费者消费



#### 生产者

两个变化：

- 1）  声明Exchange，不再声明Queue
- 2）  发送消息到Exchange，不再发送到Queue

```java
public class Send {

    private final static String EXCHANGE_NAME = "fanout_exchange_test";

    public static void main(String[] argv) throws Exception {
        // 获取到连接
        Connection connection = ConnectionUtil.getConnection();
        // 获取通道
        Channel channel = connection.createChannel();
        
        // 声明exchange，指定类型为fanout
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
        
        // 消息内容
        String message = "Hello everyone";
        // 发布消息到Exchange
        channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
        System.out.println(" [生产者] Sent '" + message + "'");

        channel.close();
        connection.close();
    }
}
```

#### 消费者1

```java
public class Recv {
    private final static String QUEUE_NAME = "fanout_exchange_queue_1";

    private final static String EXCHANGE_NAME = "fanout_exchange_test";

    public static void main(String[] argv) throws Exception {
        // 获取到连接
        Connection connection = ConnectionUtil.getConnection();
        // 获取通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 绑定队列到交换机
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");

        // 定义队列的消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            // 获取消息，并且处理，这个方法类似事件监听，如果有消息的时候，会被自动调用
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                    byte[] body) throws IOException {
                // body 即消息体
                String msg = new String(body);
                System.out.println(" [消费者1] received : " + msg + "!");
            }
        };
        // 监听队列，自动返回完成
        channel.basicConsume(QUEUE_NAME, true, consumer);
    }
}
```

要注意代码中：**队列需要和交换机绑定**

#### 消费者2

```java
public class Recv2 {
    private final static String QUEUE_NAME = "fanout_exchange_queue_2";

    private final static String EXCHANGE_NAME = "fanout_exchange_test";

    public static void main(String[] argv) throws Exception {
        // 获取到连接
        Connection connection = ConnectionUtil.getConnection();
        // 获取通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        // 绑定队列到交换机
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");
        
        // 定义队列的消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            // 获取消息，并且处理，这个方法类似事件监听，如果有消息的时候，会被自动调用
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                    byte[] body) throws IOException {
                // body 即消息体
                String msg = new String(body);
                System.out.println(" [消费者2] received : " + msg + "!");
            }
        };
        // 监听队列，手动返回完成
        channel.basicConsume(QUEUE_NAME, true, consumer);
    }
}
```



#### 测试

我们运行两个消费者，然后发送1条消息：

 ![1527087071693](assets/1527087071693.png)

### 3.4.6.订阅模型-Direct

#### 说明

在Fanout模式中，一条消息，会被所有订阅的队列都消费。但是，在某些场景下，我们希望不同的消息被不同的队列消费。这时就要用到Direct类型的Exchange。

 在Direct模型下：

- 队列与交换机的绑定，不能是任意绑定了，而是要指定一个`RoutingKey`（路由key）
- 消息的发送方在 向 Exchange发送消息时，也必须指定消息的 `RoutingKey`。
- Exchange不再把消息交给每一个绑定的队列，而是根据消息的`Routing Key`进行判断，只有队列的`Routingkey`与消息的 `Routing key`完全一致，才会接收到消息



流程图：

 ![1527087677192](assets/1527087677192.png)

图解：

- P：生产者，向Exchange发送消息，发送消息时，会指定一个routing key。
- X：Exchange（交换机），接收生产者的消息，然后把消息递交给 与routing key完全匹配的队列
- C1：消费者，其所在队列指定了需要routing key 为 error 的消息
- C2：消费者，其所在队列指定了需要routing key 为 info、error、warning 的消息

#### 生产者

此处我们模拟商品的增删改，发送消息的RoutingKey分别是：insert、update、delete

```java
public class Send {
    private final static String EXCHANGE_NAME = "direct_exchange_test";

    public static void main(String[] argv) throws Exception {
        // 获取到连接
        Connection connection = ConnectionUtil.getConnection();
        // 获取通道
        Channel channel = connection.createChannel();
        // 声明exchange，指定类型为direct
        channel.exchangeDeclare(EXCHANGE_NAME, "direct");
        // 消息内容
        String message = "商品新增了， id = 1001";
        // 发送消息，并且指定routing key 为：insert ,代表新增商品
        channel.basicPublish(EXCHANGE_NAME, "insert", null, message.getBytes());
        System.out.println(" [商品服务：] Sent '" + message + "'");

        channel.close();
        connection.close();
    }
}
```

#### 消费者1

我们此处假设消费者1只接收两种类型的消息：更新商品和删除商品。

```java
public class Recv {
    private final static String QUEUE_NAME = "direct_exchange_queue_1";
    private final static String EXCHANGE_NAME = "direct_exchange_test";

    public static void main(String[] argv) throws Exception {
        // 获取到连接
        Connection connection = ConnectionUtil.getConnection();
        // 获取通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        
        // 绑定队列到交换机，同时指定需要订阅的routing key。假设此处需要update和delete消息
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "update");
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "delete");

        // 定义队列的消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            // 获取消息，并且处理，这个方法类似事件监听，如果有消息的时候，会被自动调用
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                    byte[] body) throws IOException {
                // body 即消息体
                String msg = new String(body);
                System.out.println(" [消费者1] received : " + msg + "!");
            }
        };
        // 监听队列，自动ACK
        channel.basicConsume(QUEUE_NAME, true, consumer);
    }
}
```



#### 消费者2

我们此处假设消费者2接收所有类型的消息：新增商品，更新商品和删除商品。

```java
public class Recv2 {
    private final static String QUEUE_NAME = "direct_exchange_queue_2";
    private final static String EXCHANGE_NAME = "direct_exchange_test";

    public static void main(String[] argv) throws Exception {
        // 获取到连接
        Connection connection = ConnectionUtil.getConnection();
        // 获取通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        
        // 绑定队列到交换机，同时指定需要订阅的routing key。订阅 insert、update、delete
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "insert");
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "update");
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "delete");

        // 定义队列的消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            // 获取消息，并且处理，这个方法类似事件监听，如果有消息的时候，会被自动调用
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                    byte[] body) throws IOException {
                // body 即消息体
                String msg = new String(body);
                System.out.println(" [消费者2] received : " + msg + "!");
            }
        };
        // 监听队列，自动ACK
        channel.basicConsume(QUEUE_NAME, true, consumer);
    }
}
```



#### 测试

我们分别发送增、删、改的RoutingKey，发现结果：

 ![1527088296131](assets/1527088296131.png)



### 3.4.7.订阅模型-Topic

#### 说明

`Topic`类型的`Exchange`与`Direct`相比，都是可以根据`RoutingKey`把消息路由到不同的队列。只不过`Topic`类型`Exchange`可以让队列在绑定`Routing key` 的时候使用通配符！



`Routingkey` 一般都是有一个或多个单词组成，多个单词之间以”.”分割，例如： `item.insert`

 通配符规则：

`#`：匹配一个或多个词

`*`：匹配不多不少恰好1个词



举例：

`item.#`：能够匹配`item.spu.insert` 或者 `item.spu`

`item.*`：只能匹配`item.spu`

​     

图示：

 ![1527088518574](assets/1527088518574.png)

解释：

- 红色Queue：绑定的是`usa.#` ，因此凡是以 `usa.`开头的`routing key` 都会被匹配到
- 黄色Queue：绑定的是`#.news` ，因此凡是以 `.news`结尾的 `routing key` 都会被匹配

#### 生产者

使用topic类型的Exchange，发送消息的routing key有3种： `item.isnert`、`item.update`、`item.delete`：

```java
public class Send {
    private final static String EXCHANGE_NAME = "topic_exchange_test";

    public static void main(String[] argv) throws Exception {
        // 获取到连接
        Connection connection = ConnectionUtil.getConnection();
        // 获取通道
        Channel channel = connection.createChannel();
        // 声明exchange，指定类型为topic
        channel.exchangeDeclare(EXCHANGE_NAME, "topic");
        // 消息内容
        String message = "新增商品 : id = 1001";
        // 发送消息，并且指定routing key 为：insert ,代表新增商品
        channel.basicPublish(EXCHANGE_NAME, "item.insert", null, message.getBytes());
        System.out.println(" [商品服务：] Sent '" + message + "'");

        channel.close();
        connection.close();
    }
}
```

#### 消费者1

我们此处假设消费者1只接收两种类型的消息：更新商品和删除商品

```java
public class Recv {
    private final static String QUEUE_NAME = "topic_exchange_queue_1";
    private final static String EXCHANGE_NAME = "topic_exchange_test";

    public static void main(String[] argv) throws Exception {
        // 获取到连接
        Connection connection = ConnectionUtil.getConnection();
        // 获取通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        
        // 绑定队列到交换机，同时指定需要订阅的routing key。需要 update、delete
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "item.update");
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "item.delete");

        // 定义队列的消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            // 获取消息，并且处理，这个方法类似事件监听，如果有消息的时候，会被自动调用
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                    byte[] body) throws IOException {
                // body 即消息体
                String msg = new String(body);
                System.out.println(" [消费者1] received : " + msg + "!");
            }
        };
        // 监听队列，自动ACK
        channel.basicConsume(QUEUE_NAME, true, consumer);
    }
}
```



#### 消费者2

我们此处假设消费者2接收所有类型的消息：新增商品，更新商品和删除商品。

```java
/**
 * 消费者2
 */
public class Recv2 {
    private final static String QUEUE_NAME = "topic_exchange_queue_2";
    private final static String EXCHANGE_NAME = "topic_exchange_test";

    public static void main(String[] argv) throws Exception {
        // 获取到连接
        Connection connection = ConnectionUtil.getConnection();
        // 获取通道
        Channel channel = connection.createChannel();
        // 声明队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        
        // 绑定队列到交换机，同时指定需要订阅的routing key。订阅 insert、update、delete
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "item.*");

        // 定义队列的消费者
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            // 获取消息，并且处理，这个方法类似事件监听，如果有消息的时候，会被自动调用
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                    byte[] body) throws IOException {
                // body 即消息体
                String msg = new String(body);
                System.out.println(" [消费者2] received : " + msg + "!");
            }
        };
        // 监听队列，自动ACK
        channel.basicConsume(QUEUE_NAME, true, consumer);
    }
}
```

## 3.5.持久化

如何避免消息丢失？

1）  消费者的ACK机制。可以防止消费者丢失消息。

2）  但是，如果在消费者消费之前，MQ就宕机了，消息就没了。



所以我们需要将消息持久化到硬盘，以防服务宕机。

 要将消息持久化，前提是：队列、Exchange都持久化

交换机持久化

 ![1527088933255](assets/1527088933255.png)

队列持久化

 ![1527088960059](assets/1527088960059.png)

消息持久化

![1527088984029](assets/1527088984029.png)



## 3.6.面试总结

总结：

面试题1：如何解决消息丢失？

- ack（消费者确认）
- 持久化
- 生产者确认（publisher confirm）：生产者发送消息后，等待mq的ACK，如果没有收到或者收到失败信息，则重试。如果收到成功消息则业务结束。
- 可靠消息服务（可选）：对于部分不支持生产者确认的消息队列，可以发送消息前，将消息持久化到数据库，并记录消息状态，后续消息发送、消费等过程都依赖于数据库中消息状态的判断和修改。

面试题2：如何避免消息堆积

- 通过同一个队列多消费者监听，实现消息的争抢，加快消息消费速度。

面试题3：如何保证消息的有序性？

答：大部分业务对消息的有序性要求不高，如果遇到对时序要求较高的业务，分两种情况来处理：

- 业务同时对并发要求不高：
  - 保证消息发送时有序同步发送
  - 保证消息发送被同一个队列接收
  - 保证一个队列只有一个消费者，可以有从机（待机状态），实现高可用。
  - 实现主从（Zookeeper集群选主）
- 业务同时对并发要求较高：
  - 满足上述第一个场景的条件
  - 可以有多个队列
  - 有时序要求的一组消息，通过hash方式分派到一个固定队列

面试题4：如何避免消息重复消费？

- 保证接口幂等即可，那么如何保证接口幂等呢？
  - 某些接口天生幂等，例如查询请求
  - 某些接口天生不幂等，比如新增，还有某些接口的修改功能
    - 能根据具体的业务或状态来确定的，在消费端通过业务判断是否执行过



# 4.Spring AMQP

## 4.1.简介

Sprin有很多不同的项目，其中就有对AMQP的支持：

![1527089338661](assets/1527089338661.png)

Spring AMQP的页面：<http://projects.spring.io/spring-amqp/> 

![1527089365281](assets/1527089365281.png)



注意这里一段描述：


         Spring-amqp是对AMQP协议的抽象实现，而spring-rabbit 是对协议的具体实现，也是目前的唯一实现。底层使用的就是RabbitMQ。



## 4.2.依赖和配置

添加AMQP的启动器：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

在`application.yml`中添加RabbitMQ地址：

```yaml
spring:
  rabbitmq:
    host: 192.168.56.101
    username: leyou
    password: leyou
    virtual-host: /leyou
```

## 4.3.监听者

在SpringAmqp中，对消息的消费者进行了封装和抽象，一个普通的JavaBean中的普通方法，只要通过简单的注解，就可以成为一个消费者。

```java
@Component
public class Listener {

    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(value = "spring.test.queue", durable = "true"),
            exchange = @Exchange(
                    value = "spring.test.exchange",
                    ignoreDeclarationExceptions = "true",
                    type = ExchangeTypes.TOPIC
            ),
            key = {"#.#"}))
    public void listen(String msg){
        System.out.println("接收到消息：" + msg);
    }
}
```

- `@Componet`：类上的注解，注册到Spring容器
- `@RabbitListener`：方法上的注解，声明这个方法是一个消费者方法，需要指定下面的属性：
  - `bindings`：指定绑定关系，可以有多个。值是`@QueueBinding`的数组。`@QueueBinding`包含下面属性：
    - `value`：这个消费者关联的队列。值是`@Queue`，代表一个队列
    - `exchange`：队列所绑定的交换机，值是`@Exchange`类型
    - `key`：队列和交换机绑定的`RoutingKey`

类似listen这样的方法在一个类中可以写多个，就代表多个消费者。



## 4.4.AmqpTemplate

Spring最擅长的事情就是封装，把他人的框架进行封装和整合。

Spring为AMQP提供了统一的消息处理模板：AmqpTemplate，非常方便的发送消息，其发送方法：

![1527090258083](assets/1527090258083.png)

红框圈起来的是比较常用的3个方法，分别是：

- 指定交换机、RoutingKey和消息体
- 指定消息
- 指定RoutingKey和消息，会向默认的交换机发送消息



## 4.5.测试代码

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
public class MqDemo {

    @Autowired
    private AmqpTemplate amqpTemplate;

    @Test
    public void testSend() throws InterruptedException {
        String msg = "hello, Spring boot amqp";
        this.amqpTemplate.convertAndSend("spring.test.exchange","a.b", msg);
        // 等待10秒后再结束
        Thread.sleep(10000);
    }
}
```

运行后查看日志：

![1527090414110](assets/1527090414110.png)

