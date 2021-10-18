# 0.学习目标

- 了解过滤功能的基本思路
- 独立实现分类和品牌展示
- 了解规格参数展示
- 实现过滤条件筛选
- 实现已选过滤项回显
- 实现取消选择过滤项



# 1.过滤功能分析

## 1.1.功能模块

首先看下页面要实现的效果：

![1526725119663](assets/1526725119663.png)

整个过滤部分有3块：

- 顶部的导航，已经选择的过滤条件展示：
  - 商品分类面包屑，根据用户选择的商品分类变化
  - 其它已选择过滤参数
- 过滤条件展示，又包含3部分
  - 商品分类展示
  - 品牌展示
  - 其它规格参数
- 展开或收起的过滤条件的按钮



顶部导航要展示的内容跟用户选择的过滤条件有关。

- 比如用户选择了某个商品分类，则面包屑中才会展示具体的分类
- 比如用户选择了某个品牌，列表中才会有品牌信息。

所以，这部分需要依赖第二部分：过滤条件的展示和选择。因此我们先不着急去做。



展开或收起的按钮是否显示，取决于过滤条件现在有多少，如果有很多，那么就没必要展示。所以也是跟第二部分的过滤条件有关。

这样分析来看，我们必须先做第二部分：过滤条件展示。



## 1.2.问题分析

过滤条件包括：分类过滤、品牌过滤、规格过滤项等。我们必须弄清楚几个问题：

- 什么时候查询这些过滤项？
- 这些过滤项的数据从何而来？

我们先以分类和品牌来讨论一下：

> 问题1，什么时候查询这些过滤项？

现在，页面加载后就会调用`loadData`方法，向服务端发起请求，查询商品数据。我们有两种选择：

- 方式1：在查询商品数据的同时，顺便把分类和品牌的过滤数据一起查出来
  - 优点：只有一次请求，逻辑简单
  - 缺点：该请求处理业务较多，业务复杂，效率较差
- 方式2：在查询商品后，再发一个ajax请求，专门查询分类和品牌的过滤数据
  - 优点：每个请求做个业务，耦合度低，每次请求处理效率较高
  - 缺点：需要发多次请求

这里考虑使用方式2，让每次请求做自己的事情，减少业务耦合。

> 问题2，过滤项的数据从何而来？

在我们的数据库中已经有所有的分类和品牌信息。在这个位置，是不是把所有的分类和品牌信息都展示出来呢？

显然不是，用户搜索的条件会对商品进行过滤，而在搜索结果中，不一定包含所有的分类和品牌，直接展示出所有商品分类，让用户选择显然是不合适的。

比如，用户搜索：`小米手机`，结果中肯定只有手机，而且还得是小米的手机，就把分类和品牌限定死了，此时显示出其它品牌和分类过滤项显然是不合适的。

因此，只有**`在搜索过滤的结果中存在的分类和品牌才能作为过滤项让用户选择`**。

那么问题来了：我们怎么知道搜索结果中有哪些分类和品牌呢？

答案是：利用elasticsearch提供的聚合功能，**`在搜索条件基础上，对搜索结果聚合`**，就能知道结果中包含哪些分类和品牌了。当然，规格参数也是一样的。；





# 2.分类和品牌过滤项

首先，我们先完成分类和品牌的过滤项的查询。

## 2.1.发起查询请求

首先，定义一个函数，在函数内部发起ajax请求，查询各种过滤项：

```js
loadFilterList(){
    // 发起请求，查询过滤项
    ly.http.post("/search/filter", this.search)
        .then(resp => {

    })
}
```

注意：请求的参数与搜索商品时的请求参数是一致的，因为我们需要**`在搜索条件基础上，对搜索结果聚合`**。



在created钩子函数中，在查询商品数据的之后，调用这个方法：

![1553738446247](assets/1553738446247.png)

## 2.2.请求分析

上面的请求发出了，我们就知道了下面的信息：

- 请求方式：Post
- 请求路径：/search/filter
- 请求参数：与商品搜索一样，是SearchRequest对象

那么问题来了：以什么格式返回呢？

来看下页面的展示效果：

 ![1526742664217](assets/1526742664217.png)

虽然分类、品牌等过滤内容都不太一样，但是结构相似，都是key和value的结构。

- key是过滤参数名称，如：分类

- value是过滤的待选项，如：手机，儿童手表。

类似这样的键值对结构，是不是可以用一个Map来表示呢？

而这样的过滤条件很多，所以可以用一个数组表示，其基本结构是这样的：

```js
{
    "过滤字段名称1":['过滤项1','过滤项2',...],
    "过滤字段名称2":['过滤项1','过滤项2',...]
}
```

类似于java中的：`Map<String,List<?>>`

注意，这里的分类和品牌过滤项，不仅仅是分类和品牌的名称，还有品牌的图片，id等。所以待选项应该是一个分类和品牌的对象。

## 2.3.聚合商品分类和品牌

大家不要忘了，索引库中存储的分类和品牌只有id，因此聚合出来的结果也只有id。

而页面中需要的是分类和品牌的对象，所以



我们对分类和品牌聚合，先获取得到的分类和品牌的id，然后再根据id去查询分类和品牌数据。

所以，商品微服务需要提供接口：

- 根据品牌id集合，批量查询品牌；
- 根据分类id集合查询分类（之前已经 写过）

### 2.2.1.提供查询品牌接口

我们在`ly-item-interface`中的`ItemClient`中定义一个新的API：

```java
/**
     * 根据品牌id批量查询品牌
     * @param idList 品牌id的集合
     * @return 品牌的集合
     */
@GetMapping("/brand/list")
List<BrandDTO> queryBrandByIds(@RequestParam("ids") List<Long> idList);
```

然后再`ly-item-service`中实现：

BrandController：

```java
/**
     * 根据id的集合查询品牌
     * @param idList 品牌id的集合
     * @return 品牌的集合
     */
@GetMapping("/list")
public ResponseEntity<List<BrandDTO>> queryBrandByIds(@RequestParam("ids") List<Long> idList){
    return ResponseEntity.ok(brandService.queryByIds(idList));
}
```

BrandService

```java
public List<BrandDTO> queryByIds(List<Long> ids) {
    List<Brand> list = brandMapper.selectByIdList(ids);
    // 判断是否为空
    if(CollectionUtils.isEmpty(list)){
        throw new LyException(ExceptionEnum.BRAND_NOT_FOUND);
    }
    return BeanHelper.copyWithCollection(list, BrandDTO.class);
}
```

BrandMapper，需要基础BaseMapper，以拓展更多功能。

```java
package com.leyou.item.mapper;

import com.leyou.common.mapper.BaseMapper;
// ... 略

public interface BrandMapper extends BaseMapper<Brand> {
	// ... 略
}
```



### 2.2.2.查询过滤项的接口

> handler

在`SearchController`中新增一个Handler：

```java
/**
     * 查询过滤项
     * @param request
     * @return
     */
@PostMapping("filter")
public ResponseEntity<Map<String, List<?>>> queryFilters(@RequestBody SearchRequest request) {
    return ResponseEntity.ok(searchService.queryFilters(request));
}
```



> searchService

因为在业务中，与商品搜索一样，都需要构建查询条件，我们把构建查询条件的代码封装成一个方法：

```java
private QueryBuilder buildBasicQuery(SearchRequest request) {
    // 构建基本的match查询
    return QueryBuilders.matchQuery("all", request.getKey()).operator(Operator.AND);
}
```

然后将原来的搜索逻辑修改一下，调用这个方法：

![1553741152844](assets/1553741152844.png)



新增searchService业务逻辑：

```java
@Autowired
private ElasticsearchTemplate esTemplate;

public Map<String, List<?>> queryFilters(SearchRequest request) {
    // 1.创建过滤项集合
    Map<String, List<?>> filterList = new LinkedHashMap<>();

    // 2.查询条件
    NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
    // 2.1.获取查询条件
    QueryBuilder basicQuery = buildBasicQuery(request);
    queryBuilder.withQuery(basicQuery);
    // 2.2.减少查询结果(这里只需要聚合结果)
    // 每页显示1个
    queryBuilder.withPageable(PageRequest.of(0, 1));
    // 显示空的source
    queryBuilder.withSourceFilter(new FetchSourceFilterBuilder().build());

    // 3.聚合条件
    // 3.1.分类聚合
    String categoryAgg = "categoryAgg";
    queryBuilder.addAggregation(AggregationBuilders.terms(categoryAgg).field("categoryId"));
    // 3.2.品牌聚合
    String brandAgg = "brandAgg";
    queryBuilder.addAggregation(AggregationBuilders.terms(brandAgg).field("brandId"));

    // 4.查询并解析结果
    AggregatedPage<Goods> result = esTemplate.queryForPage(queryBuilder.build(), Goods.class);
    Aggregations aggregations = result.getAggregations();
    // 4.1.获取分类聚合
    LongTerms cTerms = aggregations.get(categoryAgg);
    handleCategoryAgg(cTerms, filterList);
    // 4.2.获取分类聚合
    LongTerms bTerms = aggregations.get(brandAgg);
    handleBrandAgg(bTerms, filterList);

    return filterList;
}

private void handleBrandAgg(LongTerms terms, Map<String, List<?>> filterList) {
    // 解析bucket，得到id集合
    List<Long> idList = terms.getBuckets().stream()
        .map(LongTerms.Bucket::getKeyAsNumber)
        .map(Number::longValue)
        .collect(Collectors.toList());
    // 根据id集合查询品牌
    List<BrandDTO> brandList = itemClient.queryBrandByIds(idList);
    // 存入map
    filterList.put("品牌", brandList);
}

private void handleCategoryAgg(LongTerms terms, Map<String, List<?>> filterList) {
    // 解析bucket，得到id集合
    List<Long> idList = terms.getBuckets().stream()
        .map(LongTerms.Bucket::getKeyAsNumber)
        .map(Number::longValue)
        .collect(Collectors.toList());
    // 根据id集合查询分类
    List<CategoryDTO> categoryList = itemClient.queryCategoryByIds(idList);
    // 存入map
    filterList.put("分类", categoryList);
}
```



测试：

 ![1553741396967](assets/1553741396967.png)



## 2.4.页面渲染数据

### 2.4.1.过滤参数数据结构

来看下页面的展示效果：

 ![1526742664217](assets/1526742664217.png)

虽然分类、品牌内容都不太一样，但是结构相似，都是key和value的结构。

而且页面结构也极为类似：

 ![1526742817804](assets/1526742817804.png)



所以，利用`v-for`遍历一次生成。

后台返回的数据其基本结构是这样的：

```js
{
    "过滤字段名称1":['过滤项1','过滤项2',...],
    "过滤字段名称2":['过滤项1','过滤项2',...]
}
```

我们先在data中定义变量，接收这个结果：

 ![1553741496632](assets/1553741496632.png)

然后在查询过滤项的回调函数中，对过滤参数进行保存：

```js
loadFilterList(){
    // 发起请求，查询过滤项
    ly.http.post("/search/filter", this.search)
        .then(resp => {
        this.filterList = resp.data;
    })
}
```



然后刷新页面，通过浏览器工具，查看封装的结果：

 ![1553741590843](assets/1553741590843.png)



### 2.4.2.页面渲染数据

首先看页面原来的代码：

 ![1526803362517](assets/1526803362517.png)

我们注意到，虽然页面元素是一样的，但是品牌会比其它搜搜条件多出一些样式，因为品牌是以图片展示。需要进行特殊处理。数据展示是一致的，我们采用v-for处理，然后通过v-if判断分为两种情况：

- 如果不是品牌，则按照第一个div样式处理
- 是品牌，则按照第二个div样式处理

```html
<div class="type-wrap" v-for="(v,k,i) in filterList" :key="k" v-if="k!=='品牌'">
    <div class="fl key">{{k}}</div>
    <div class="fl value">
        <ul class="type-list">
            <li v-for="(o,j) in v" :key="j">
                <a>{{o.name}}</a>
            </li>
        </ul>
    </div>
    <div class="fl ext"></div>
</div>
<div class="type-wrap logo" v-else>
    <div class="fl key brand">{{k}}</div>
    <div class="value logos">
        <ul class="logo-list">
            <li v-for="(o,j) in v" :key="j" v-if="o.image"><img :src="o.image"/></li>
            <li style="text-align: center" v-else>
                <a style="line-height: 30px; font-size: 12px" href="#">{{o.name}}</a>
            </li>
        </ul>
    </div>
    <div class="fl ext">
        <a href="javascript:void(0);" class="sui-btn">多选</a>
    </div>
</div>
```

结果：

![1526804398051](assets/1526804398051.png)



# 3.生成规格参数过滤

## 3.1.谋而后动

有3个问题需要先思考清楚：

- 什么时候显示规格参数过滤？
- 如何知道哪些规格需要过滤？
- 要过滤的参数，其可选值是如何获取的？



> 什么情况下显示有关规格参数的过滤？

可能有同学会想，这还要思考吗？查询商品分类和品牌过滤项的同时，把规格参数过滤项一起返回啊！



但是，如果用户尚未选择商品分类，或者聚合得到的分类数大于1，那么就没必要进行规格参数的聚合。因为不同分类的商品，其规格是不同的，我们无法确定到底有多少规格需要聚合，代码无法进行。

因此，我们在后台**需要对聚合得到的商品分类数量进行判断，如果等于1，我们才继续进行规格参数的聚合**。

此时，只需要聚合当前分类下的规格参数即可，数量可以确定。

> 如何知道哪些规格需要过滤？

那么，我们是不是把数据库中的所有规格参数都拿来过滤呢？

显然不是！

因为并不是所有的规格参数都可以用来过滤。

庆幸的是，我们在设计规格参数时，已经标记了某些规格可搜索，某些不可搜索，还记得SpecParam中的searching字段吗？

因此，一旦商品分类确定，我们就可以根据商品分类查询到其对应的规格参数，并过滤哪些searching值为true的规格参数，然后对这些参数聚合即可。



> 要过滤的参数，其可选值是如何获取的？

虽然数据库中有所有的规格参数的可能值，但是不能把一切数据都用来供用户选择。

与商品分类和品牌一样，应该是结果中有哪些规格参数值，就显示哪些。

即：**`在搜索条件基础上，对搜索结果聚合`**，得到规格参数的待选项。

比如：用户搜索了OPPO 手机，那么过滤项中只应该有OPPO可能存在的屏幕尺寸，比如5.5以上的，不会存在5.5以下的尺寸让你选择。



## 3.3.动手实践

接下来，我们就用代码实现刚才的思路。

总结一下，应该是以下几步：

- 1）用户搜索得到商品，并聚合出商品分类（已完成）
- 2）判断分类数量是否等于1，如果是则进行规格参数聚合
- 3）先根据分类，查找可以用来搜索的规格
- 4）在用户搜索结果的基础上，对规格参数进行聚合
- 5）将规格参数聚合结果整理后返回

### 3.3.1. 返回聚合得到的分类id

我们修改处理分类聚合的方法，使其返回得到的分类id集合，方便下面判断分类的数量：

```java
private List<Long> handleCategoryAgg(LongTerms terms, Map<String, List<?>> filterList) {
    // 解析bucket，得到id集合
    List<Long> idList = terms.getBuckets().stream()
        .map(LongTerms.Bucket::getKeyAsNumber)
        .map(Number::longValue)
        .collect(Collectors.toList());
    // 根据id集合查询分类
    List<CategoryDTO> categoryList = itemClient.queryCategoryByIds(idList);
    // 存入map
    filterList.put("分类", categoryList);
    return idList;
}
```

### 3.3.2.判断是否需要聚合

在`queryFilter`方法中，聚合得到商品分类后，判断分类的个数，如果是1个则进行规格聚合：

![1553743054708](assets/1553743054708.png) 

此处调用了一个`handleSpecAgg`方法，处理规格参数聚合。



### 3.3.3.获取需要聚合的规格参数

然后，在`handleSpecAgg`中我们需要根据商品分类，查询所有可用于搜索的规格参数：

![1553743703220](assets/1553743703220.png)

### 3.3.4.聚合规格参数

在`handleSpecAgg`中，添加聚合条件。

因为规格参数保存时是specs的属性，因此所有的规格参数都会有一个`specs.`的前缀

![1553743727154](assets/1553743727154.png)

这里把规格参数的名称作为了聚合名称，因此取出结果时，也要以参数名来取

### 3.3.5.解析聚合结果

在`handleSpecAgg`中，解析聚合得到的结果，并封装到map中

![1553743997242](assets/1553743997242.png)

### 3.3.6.完整代码

```java
private void handleSpecAgg(Long cid, QueryBuilder basicQuery, Map<String, List<?>> filterList) {
    // 1.查询分类下需要搜索过滤的规格参数名称
    List<SpecParamDTO> specParams = itemClient.querySpecParams(null, cid, true);

    // 2.查询条件
    NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
    // 2.1.获取查询条件
    queryBuilder.withQuery(basicQuery);
    // 2.2.减少查询结果(这里只需要聚合结果)
    // 每页显示1个
    queryBuilder.withPageable(PageRequest.of(0, 1));
    // 显示空的source
    queryBuilder.withSourceFilter(new FetchSourceFilterBuilder().build());

    // 3.聚合条件
    for (SpecParamDTO param : specParams) {
        // 获取param的name，作为聚合名称
        String name = param.getName();
        queryBuilder.addAggregation(AggregationBuilders.terms(name).field("specs." + name));
    }

    // 4.查询并获取结果
    AggregatedPage<Goods> result = esTemplate.queryForPage(queryBuilder.build(), Goods.class);
    Aggregations aggregations = result.getAggregations();

    // 5.解析聚合结果
    for (SpecParamDTO param : specParams) {
        // 获取param的name，作为聚合名称
        String name = param.getName();
        StringTerms terms = aggregations.get(name);
        // 获取聚合结果，注意，规格聚合的结果 直接是字符串，不用做特殊处理
        List<String> paramValues = terms.getBuckets()
            .stream()
            .map(StringTerms.Bucket::getKeyAsString)
            // 过滤掉空的待选项
            .filter(StringUtils::isNotEmpty)
            .collect(Collectors.toList());
        // 存入map
        filterList.put(name, paramValues);
    }
}
```



### 3.3.7.测试结果：

![1553744260832](assets/1553744260832.png) 



### 3.3.8.整个SearchService的完整代码

```java
package com.leyou.search.service;

import com.fasterxml.jackson.core.type.TypeReference;
import com.leyou.common.enums.ExceptionEnum;
import com.leyou.common.exceptions.LyException;
import com.leyou.common.utils.JsonUtils;
import com.leyou.common.utils.NumberUtils;
import com.leyou.common.vo.PageResult;
import com.leyou.item.client.ItemClient;
import com.leyou.item.dto.*;
import com.leyou.search.pojo.Goods;
import com.leyou.search.pojo.SearchRequest;
import com.leyou.search.repository.GoodsRepository;
import org.apache.commons.lang3.StringUtils;
import org.elasticsearch.index.query.Operator;
import org.elasticsearch.index.query.QueryBuilder;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.search.aggregations.AggregationBuilders;
import org.elasticsearch.search.aggregations.Aggregations;
import org.elasticsearch.search.aggregations.bucket.terms.LongTerms;
import org.elasticsearch.search.aggregations.bucket.terms.StringTerms;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.elasticsearch.core.ElasticsearchTemplate;
import org.springframework.data.elasticsearch.core.aggregation.AggregatedPage;
import org.springframework.data.elasticsearch.core.query.FetchSourceFilter;
import org.springframework.data.elasticsearch.core.query.FetchSourceFilterBuilder;
import org.springframework.data.elasticsearch.core.query.NativeSearchQueryBuilder;
import org.springframework.stereotype.Service;

import java.util.*;
import java.util.stream.Collectors;

/**
 * @author 黑马程序员
 */
@Service
public class SearchService {

    @Autowired
    private ItemClient itemClient;

    /**
     * 把一个Spu转为一个Goods对象
     *
     * @param spu
     * @return
     */
    public Goods buildGoods(SpuDTO spu) {
        // 1 商品相关搜索信息的拼接：名称、分类、品牌、规格信息等
        // 1.1 分类
        String categoryNames = itemClient.queryCategoryByIds(spu.getCategoryIds())
                .stream().map(CategoryDTO::getName).collect(Collectors.joining(","));
        // 1.2 品牌
        BrandDTO brand = itemClient.queryBrandById(spu.getBrandId());
        // 1.3 名称等，完成拼接
        String all = spu.getName() + categoryNames + brand.getName();

        // 2 spu下的所有sku的JSON数组
        List<SkuDTO> skuList = itemClient.querySkuBySpuId(spu.getId());
        // 准备一个集合，用map来代替sku，只需要sku中的部分数据
        List<Map<String, Object>> skuMap = new ArrayList<>();
        for (SkuDTO sku : skuList) {
            Map<String, Object> map = new HashMap<>();
            map.put("id", sku.getId());
            map.put("price", sku.getPrice());
            map.put("title", sku.getTitle());
            map.put("image", StringUtils.substringBefore(sku.getImages(), ","));
            skuMap.add(map);
        }

        // 3 当前spu下所有sku的价格的集合
        Set<Long> price = skuList.stream().map(SkuDTO::getPrice).collect(Collectors.toSet());

        // 4 当前spu的规格参数
        Map<String, Object> specs = new HashMap<>();

        // 4.1 获取规格参数key，来自于SpecParam中当前分类下的需要搜索的规格
        List<SpecParamDTO> specParams = itemClient.querySpecParams(null, spu.getCid3(), true);
        // 4.2 获取规格参数的值，来自于spuDetail
        SpuDetailDTO spuDetail = itemClient.querySpuDetailById(spu.getId());
        // 4.2.1 通用规格参数值
        Map<Long, Object> genericSpec = JsonUtils.toMap(spuDetail.getGenericSpec(), Long.class, Object.class);
        // 4.2.2 特有规格参数值
        Map<Long, List<String>> specialSpec = JsonUtils.nativeRead(spuDetail.getSpecialSpec(), new TypeReference<Map<Long, List<String>>>() {
        });

        for (SpecParamDTO specParam : specParams) {
            // 获取规格参数的名称
            String key = specParam.getName();
            // 获取规格参数值
            Object value = null;
            // 判断是否是通用规格
            if (specParam.getGeneric()) {
                // 通用规格
                value = genericSpec.get(specParam.getId());
            } else {
                // 特有规格
                value = specialSpec.get(specParam.getId());
            }
            // 判断是否是数字类型
            if(specParam.getNumeric()){
                // 是数字类型，分段
                value = chooseSegment(value, specParam);
            }
            // 添加到specs
            specs.put(key, value);
        }

        Goods goods = new Goods();
        // 从spu对象中拷贝与goods对象中属性名一致的属性
        goods.setBrandId(spu.getBrandId());
        goods.setCategoryId(spu.getCid3());
        goods.setId(spu.getId());
        goods.setSubTitle(spu.getSubTitle());
        goods.setCreateTime(spu.getCreateTime().getTime());
        goods.setSkus(JsonUtils.toString(skuMap));// spu下的所有sku的JSON数组
        goods.setSpecs(specs); // 当前spu的规格参数
        goods.setPrice(price); // 当前spu下所有sku的价格的集合
        goods.setAll(all);// 商品相关搜索信息的拼接：标题、分类、品牌、规格信息等
        return goods;
    }

    private String chooseSegment(Object value, SpecParamDTO p) {
        if(value == null || StringUtils.isBlank(value.toString())){
            return "其它";
        }
        double val = NumberUtils.toDouble(value.toString());
        String result = "其它";
        // 保存数值段
        for (String segment : p.getSegments().split(",")) {
            String[] segs = segment.split("-");
            // 获取数值范围
            double begin = NumberUtils.toDouble(segs[0]);
            double end = Double.MAX_VALUE;
            if(segs.length == 2){
                end = NumberUtils.toDouble(segs[1]);
            }
            // 判断是否在范围内
            if(val >= begin && val < end){
                if(segs.length == 1){
                    result = segs[0] + p.getUnit() + "以上";
                }else if(begin == 0){
                    result = segs[1] + p.getUnit() + "以下";
                }else{
                    result = segment + p.getUnit();
                }
                break;
            }
        }
        return result;
    }

    @Autowired
    private GoodsRepository goodsRepository;

    public PageResult<Goods> search(SearchRequest request) {
        // 1.获取请求参数，完成健壮性校验
        String key = request.getKey();
        // 判断是否有搜索条件，如果没有，直接返回null。不允许搜索全部商品
        if (StringUtils.isBlank(key)) {
            throw new LyException(ExceptionEnum.GOODS_NOT_FOUND);
        }

        // 2.构建查询条件
        NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
        // 2.1.通过sourceFilter设置返回的结果字段,我们只需要id、skus、subTitle
        queryBuilder.withSourceFilter(new FetchSourceFilter(
                new String[]{"id","skus","subTitle"}, null));
        // 2.2.关键字的match匹配查询
        QueryBuilder basicQuery = buildBasicQuery(request);
        queryBuilder.withQuery(basicQuery);
        // 2.3.分页
        int page = request.getPage() - 1;
        int size = request.getSize();
        queryBuilder.withPageable(PageRequest.of(page, size));

        // 3.查询，获取结果
        Page<Goods> result = goodsRepository.search(queryBuilder.build());
        int totalPages = result.getTotalPages();
        long total = result.getTotalElements();
        List<Goods> list = result.getContent();
        // 4.封装结果并返回
        return new PageResult<>(total, totalPages, list);
    }

    private QueryBuilder buildBasicQuery(SearchRequest request) {
        // 构建基本的match查询
        return QueryBuilders.matchQuery("all", request.getKey()).operator(Operator.AND);
    }

    @Autowired
    private ElasticsearchTemplate esTemplate;

    public Map<String, List<?>> queryFilters(SearchRequest request) {
        // 1.创建过滤项集合
        Map<String, List<?>> filterList = new LinkedHashMap<>();

        // 2.查询条件
        NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
        // 2.1.获取查询条件
        QueryBuilder basicQuery = buildBasicQuery(request);
        queryBuilder.withQuery(basicQuery);
        // 2.2.减少查询结果(这里只需要聚合结果)
        // 每页显示1个
        queryBuilder.withPageable(PageRequest.of(0, 1));
        // 显示空的source
        queryBuilder.withSourceFilter(new FetchSourceFilterBuilder().build());

        // 3.聚合条件
        // 3.1.分类聚合
        String categoryAgg = "categoryAgg";
        queryBuilder.addAggregation(AggregationBuilders.terms(categoryAgg).field("categoryId"));
        // 3.2.品牌聚合
        String brandAgg = "brandAgg";
        queryBuilder.addAggregation(AggregationBuilders.terms(brandAgg).field("brandId"));

        // 4.查询并解析结果
        AggregatedPage<Goods> result = esTemplate.queryForPage(queryBuilder.build(), Goods.class);
        Aggregations aggregations = result.getAggregations();
        // 4.1.获取分类聚合
        LongTerms cTerms = aggregations.get(categoryAgg);
        List<Long> idList = handleCategoryAgg(cTerms, filterList);
        // 4.2.获取分类聚合
        LongTerms bTerms = aggregations.get(brandAgg);
        handleBrandAgg(bTerms, filterList);

        // 5.规格参数处理
        if (idList != null && idList.size() == 1) {
            // 处理规格，需要参数：分类的id，查询条件，过滤项集合
            handleSpecAgg(idList.get(0), basicQuery, filterList);
        }
        return filterList;
    }

    private void handleSpecAgg(Long cid, QueryBuilder basicQuery, Map<String, List<?>> filterList) {
        // 1.查询分类下需要搜索过滤的规格参数名称
        List<SpecParamDTO> specParams = itemClient.querySpecParams(null, cid, true);

        // 2.查询条件
        NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
        // 2.1.获取查询条件
        queryBuilder.withQuery(basicQuery);
        // 2.2.减少查询结果(这里只需要聚合结果)
        // 每页显示1个
        queryBuilder.withPageable(PageRequest.of(0, 1));
        // 显示空的source
        queryBuilder.withSourceFilter(new FetchSourceFilterBuilder().build());

        // 3.聚合条件
        for (SpecParamDTO param : specParams) {
            // 获取param的name，作为聚合名称
            String name = param.getName();
            queryBuilder.addAggregation(AggregationBuilders.terms(name).field("specs." + name));
        }

        // 4.查询并获取结果
        AggregatedPage<Goods> result = esTemplate.queryForPage(queryBuilder.build(), Goods.class);
        Aggregations aggregations = result.getAggregations();

        // 5.解析聚合结果
        for (SpecParamDTO param : specParams) {
            // 获取param的name，作为聚合名称
            String name = param.getName();
            StringTerms terms = aggregations.get(name);
            // 获取聚合结果，注意，规格聚合的结果 直接是字符串，不用做特殊处理
            List<String> paramValues = terms.getBuckets()
                    .stream()
                    .map(StringTerms.Bucket::getKeyAsString)
                    // 过滤掉空的待选项
                    .filter(StringUtils::isNotEmpty)
                    .collect(Collectors.toList());
            // 存入map
            filterList.put(name, paramValues);
        }
    }

    private void handleBrandAgg(LongTerms terms, Map<String, List<?>> filterList) {
        // 解析bucket，得到id集合
        List<Long> idList = terms.getBuckets().stream()
                .map(LongTerms.Bucket::getKeyAsNumber)
                .map(Number::longValue)
                .collect(Collectors.toList());
        // 根据id集合查询品牌
        List<BrandDTO> brandList = itemClient.queryBrandByIds(idList);
        // 存入map
        filterList.put("品牌", brandList);
    }

    private List<Long> handleCategoryAgg(LongTerms terms, Map<String, List<?>> filterList) {
        // 解析bucket，得到id集合
        List<Long> idList = terms.getBuckets().stream()
                .map(LongTerms.Bucket::getKeyAsNumber)
                .map(Number::longValue)
                .collect(Collectors.toList());
        // 根据id集合查询分类
        List<CategoryDTO> categoryList = itemClient.queryCategoryByIds(idList);
        // 存入map
        filterList.put("分类", categoryList);
        return idList;
    }
}
```



## 3.4.页面渲染

### 3.4.1.渲染规格过滤条件

刷新页面，发现出事了：

![1553744583157](assets/1553744583157.png)

 除了分类和品牌外，其它的规格过滤项没有正常显示出数据，为什么呢？



原因是待选项的格式不同：

![1553744714332](assets/1553744714332.png)

我们需要略做处理：

![1553744769448](assets/1553744769448.png)

最后的结果：

![1526836508277](assets/1526836508277.png)



### 3.4.2.展示或收起过滤条件

是不是感觉显示的太多了，我们可以通过按钮点击来展开和隐藏部分内容：

 ![1526836575516](assets/1526836575516.png)

我们在data中定义变量，记录展开或隐藏的状态：

 ![1553744949776](assets/1553744949776.png)

然后在按钮绑定点击事件，以改变showMore的取值：

![1553746647101](assets/1553746647101.png)



在展示规格时，对showMore进行判断：

![1553746730541](assets/1553746730541.png)

OK！



# 4.过滤条件的筛选

当我们点击页面的过滤项，要做哪些事情？

- 把过滤条件保存在search对象中
- 监控search属性变化，如果有新的值，则发起请求，重新查询商品及过滤项
- 在页面顶部展示已选择的过滤项

## 4.1.保存过滤项

### 4.1.1.定义属性

我们把已选择的过滤项保存在search中，因为不确定用户会选中几个，会选中什么，所以我们用一个对象（Map）来保存可能被选中的键值对：

![1553747483932](assets/1553747483932.png)

 

要注意，在created构造函数中会对search进行初始化，可能会覆盖filter的值，所以我们在created函数中对filter做初始化判断：

![1553747609474](assets/1553747609474.png)



### 4.1.2.绑定点击事件

给所有的过滤项绑定点击事件：

![1553747747188](assets/1553747747188.png)

要注意，点击事件传2个参数：

- k：过滤项的名称
- option或option.id：当前过滤项的值或者id（因为分类和品牌要拿id去索引库过滤）

在点击事件中，保存过滤项到`selectedFilter`：

```js
selectFilter(key,val){
    // 复制原来的search
    const {...obj} = this.search.filter;
    // 添加新的属性
    obj[key] = val;
    // 赋值给search
    this.search.filter = obj;
}
```

然后通过watch监控`search.filter`的变化：

```js
watch: {
    "search.filter":{
        handler(val) {
            // 把search对象中除了key以外的属性变成请求参数，
            const {key, ...obj} = this.saerch;
            // 拼接在url路径的hash中
            window.location.hash = "#" + ly.stringify(obj);
            // 因为hash变化不引起刷新，需要手动调用loadData
            this.loadData();
            // 还要加载新的过滤项
            this.loadFilterList();
        }
    }
}
```



我们刷新页面，点击后通过浏览器功能查看`search.filter`的属性变化：

 ![1526904752818](assets/1526904752818.png)

并且，此时浏览器地址也发生了变化：

```
http://www.leyou.com/search.html?key=%E6%89%8B%E6%9C%BA#%E5%93%81%E7%89%8C=15127&CPU%E5%93%81%E7%89%8C=%E8%81%94%E5%8F%91%E7%A7%91%EF%BC%88MTK%EF%BC%89&CPU%E6%A0%B8%E6%95%B0=%E5%9B%9B%E6%A0%B8
```

网络请求也正常发出：

![1553750383017](assets/1553750383017.png)



## 4.2.后台添加过滤条件

既然请求已经发送到了后台，那接下来我们就在后台去添加这些条件：

### 4.2.1.拓展请求对象

我们需要在请求类：`SearchRequest`中添加属性，接收过滤属性。过滤属性都是键值对格式，但是key不确定，所以用一个map来接收即可。

 ![1553750432400](assets/1553750432400.png)

### 4.2.2.添加过滤条件

目前，我们的基本查询是这样的：

```java
private QueryBuilder buildBasicQuery(SearchRequest request) {
    // 构建基本的match查询
    return QueryBuilders.matchQuery("all", request.getKey()).operator(Operator.AND);
}
```



现在，我们要把页面传递的过滤条件也进入进去。

因此不能在使用普通的查询，而是要用到BooleanQuery，基本结构是这样的：

```json
GET /heima/_search
{
    "query":{
        "bool":{
        	"must":{ "match": { "title": "小米手机",operator:"and"}},
        	"filter":{
                "terms":{"field": "specs.屏幕尺寸", "value": "5.0~5.5英寸"}
        	}
        }
    }
}
```

所以，我们对原来的基本查询进行改造：

```java
private QueryBuilder buildBasicQuery(SearchRequest request) {
    // 构建布尔查询
    BoolQueryBuilder queryBuilder = QueryBuilders.boolQuery();
    // 构建基本的match查询
    queryBuilder.must(QueryBuilders.matchQuery("all", request.getKey()).operator(Operator.AND));
    // 构建过滤条件
    Map<String, String> filters = request.getFilters();
    if(!CollectionUtils.isEmpty(filters)) {
        for (Map.Entry<String, String> entry : filters.entrySet()) {
            // 获取过滤条件的key
            String key = entry.getKey();
            // 规格参数的key要做前缀specs.
            if ("分类".equals(key)) {
                key = "categoryId";
            } else if ("品牌".equals(key)) {
                key = "brandId";
            } else {
                key = "specs." + key;
            }
            // value
            String value = entry.getValue();
            // 添加过滤条件
            queryBuilder.filter(QueryBuilders.termQuery(key, value));
        }
    }

    return queryBuilder;
}
```

其它不变。

## 4.3.页面测试

我们先不点击过滤条件，直接搜索手机：

 ![1526910966728](assets/1526910966728.png)

总共184条



接下来，我们点击一个过滤条件：

 ![1526911057839](assets/1526911057839.png)

得到的结果：

 ![1526911090064](assets/1526911090064.png)

# 5.页面展示选择的过滤项(作业)

## 5.1.商品分类面包屑

当用户选择一个商品分类以后，我们应该在过滤模块的上方展示一个面包屑，把三级商品分类都显示出来。

 ![1526912181355](assets/1526912181355.png)

用户选择的商品分类就存放在`search.filter`中，但是里面只有第三级分类的id：cid3

我们需要根据它查询出所有三级分类的id及名称

### 5.1.1.提供查询分类接口

我们在商品微服务`ly-item-service`的`CategoryController`中提供一个根据三级分类id查询1~3级分类集合的方法：

> Controller

```java
/**
     * 根据3级分类id，查询1~3级的分类
     * @param id
     * @return
     */
@GetMapping("/levels/")
public ResponseEntity<List<CategoryDTO>> queryAllByCid3(@RequestParam("id") Long id){
    return ResponseEntity.ok(categoryService.queryAllByCid3(id));
}
```

> Service

```java
public List<CategoryDTO> queryAllByCid3(Long id) {
    Category c3 = categoryMapper.selectByPrimaryKey(id);
    if (c3 == null) {
        throw new LyException(ExceptionEnum.CATEGORY_NOT_FOUND);
    }
    Category c2 = categoryMapper.selectByPrimaryKey(c3.getParentId());
    if (c2 == null) {
        throw new LyException(ExceptionEnum.CATEGORY_NOT_FOUND);
    }
    Category c1 = categoryMapper.selectByPrimaryKey(c2.getParentId());
    if (c1 == null) {
        throw new LyException(ExceptionEnum.CATEGORY_NOT_FOUND);
    }
    List<Category> list = Arrays.asList(c1, c2, c3);
    return BeanHelper.copyWithCollection(list, CategoryDTO.class);
}
```

测试：

![1553751953227](assets/1553751953227.png)

### 5.1.2.页面展示面包屑

我们先在data中定义变量，记录商品分类的信息：

![1553752113672](assets/1553752113672.png)

后台提供了查询接口，下面的问题是，我们在哪里去查询接口？

大家首先想到的肯定是当用户点击以后。但是用户点击的可能不是分类，所以我们还要判断是否是分类选项，如果是，则查询分类即可。

但是，有没有考虑过这种情况：

- 如果在搜索条件基础上，本身就剩下了一种分类，此时不再需要对分类进行过滤 了，用户无需点击，那么查询也不会触发了。

所以，我们可以再过滤项加载完毕后，判断分类的数量是否只剩下一个，如果是，说明有两种情况：

- 用户点击了分类，过滤后只剩下一个
- 用户每点击，但是搜索结果本来就只有一个分类

无论是哪一种，我们都可以去查询分类信息，渲染面包屑了：

![1553753352829](assets/1553753352829.png)



渲染：

 ![1528416823546](assets/1528416823546.png)



刷新页面：

 ![1526914954839](assets/1526914954839.png)



## 5.2.其它过滤项

接下来，我们需要在页面展示用户已选择的过滤项，如图：

![1526911364625](assets/1526911364625.png)

我们知道，所有已选择过滤项都保存在`search.filter`中，因此在页面遍历并展示即可。

但这里有个问题，filter中数据的格式：



基本有3类数据：

- 商品分类：这个不需要展示，分类展示在面包屑位置
- 品牌：这个要展示，但是其key和值不合适，我们不能显示一个id在页面。需要找到其name值
- 规格：直接展示

因此，我们在页面上这样处理：

```html
<!--已选择过滤项-->
<ul class="tags-choose">
    <li class="tag" v-for="(v,k) in search.filter" v-if="k !== '分类'">
        {{k}}:<span style="color: red">{{getFilterValue(k,v)}}</span>
        <i class="sui-icon icon-tb-close"></i>
    </li>
</ul>
```

- 判断如果 `k !== '分类'`说明是不商品分类，可以显示
- 值的处理比较复杂，我们用一个方法`getFilterValue(k,v)`来处理，调用时把`k`和`v`都传递

方法内部：

```js
getFilterValue(key,value){
    if(!this.filterList[key]){
        return "";
    }
    if(key === '品牌'){
        return this.filterList[key][0].name;
    }
    return value;
}
```

然后刷新页面，即可看到效果：

![1526911811998](assets/1526911811998.png)



## 5.3.隐藏已经选择的过滤项

现在，我们已经实现了已选择过滤项的展示，但是你会发现一个问题：

已经选择的过滤项，在过滤列表中依然存在：

![1526915075037](assets/1526915075037.png)



这些已经选择的过滤项，应该从列表中移除。

怎么做呢？

你必须先知道用户选择了什么。用户选择的项保存在`search.filter`中：

 ![1526915191753](assets/1526915191753.png)

我们可以编写一个计算属性，把filters中的 已经被选择的key过滤掉：

```js
computed:{
    remainFilters(){
        // 获取已经选中的过滤项的key
        const keys = Object.keys(this.search.filter);
        // 定义空对象，记录符合条件的过滤项
        const obj = {};
        // 遍历全部过滤项
        Object.keys(this.filterList).forEach(key => {
            // 过滤掉已经包含在已选中的过滤项中的，还有待选项不超过1个的
            if(!keys.includes(key) && this.filterList[key].length > 1){
                obj[key] = this.filterList[key];
            }
        });
        return obj;
    }
}
```

然后页面不再直接遍历`filters`，而是遍历`remainFilters`

![1526916315470](assets/1526916315470.png)

刷新页面：

![1526916838222](assets/1526916838222.png)





# 6.取消过滤项（作业）

我们能够看到，每个过滤项后面都有一个小叉，当点击后，应该取消对应条件的过滤。

思路非常简单：

- 给小叉绑定点击事件
- 点击后把过滤项从`search.filter`中移除，页面会自动刷新，OK



> 绑定点击事件：

![1526955150293](assets/1526955150293.png)

绑定点击事件时，把k传递过去，方便删除

> 删除过滤项

```js
removeFilter(k){
    // 复制原来的search的filter
    const {...obj} = this.search.filter;
    // 删除属性
    delete obj[k]
    // 赋值给search
    this.search.filter=obj;
}
```





# 7.优化

搜索系统需要优化的点：

- 查询规格参数部分可以添加缓存
- elasticsearch本身有查询缓存，可以不进行优化
- 商品图片应该采用缩略图，减少流量，提高页面加载速度
- 图片采用延迟加载
- 图片还可以采用CDN服务器
- sku信息在页面异步加载，而不是放到索引库





