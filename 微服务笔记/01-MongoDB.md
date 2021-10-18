# Education项目总结 01
# 1.项目架构
## 1.1.功能架构
![45E44BBD-79AC-4C80-A5F0-11CAB7A8D581](Education项目总结01.assets/45E44BBD-79AC-4C80-A5F0-11CAB7A8D581.png)

## 1.2.技术架构

![EDDED58D-DFAD-44D0-B5FE-EDF81510E452](Education项目总结01.assets/EDDED58D-DFAD-44D0-B5FE-EDF81510E452.png)

# 2.CMS
## 2.1.CMS是什么？
CMS(Content Management System)即内容管理系统，不同的项目对CMS的定位不同，比如：一个在线教育网站，有些公司认为CMS系统是对所有课程资源进行管理，而在早期网站刚开始盛行时很多公司的业务是网站制作，当时CMS的定位是创建网站，即对网站的页面，图片等静态资源进行管理。
## 2.2.CMS有哪些类型？
上边也谈到每个公司对每个项目的CMS定位不同，CMS基本上分为：针对后台数据内容的管理，针对前端页面的管理，针对样式风格的管理等。比如：一个给企业做网站的公司，其CMS系统主要是网站页面管理及样式风格管理。
## 2.3.本项目CMS的定位是什么？
本项目作为一个大型在线教育平台，对CMS系统的定位是对各各网站（子网站）页面的管理，主要管理由于运营需要而经常变动的页面，从而实现根据运营需要而快速进行页面开发，上线的需求。
## 2.4.SSI服务端包含技术
SSI(Server SIde Include)，SSI包含类似于JSP页面的include指令，SSI是在web服务端将include指令的页面包含在网页中，渲染html网页响应给客户端。nginx,apache等多数web容器都支持SSI指令
SSI指令如下：（注意：`<!---->`不是注释）

```html
<!‐‐#include virtual="/../....html"‐‐>
```
我们需要在nginx虚拟主机中开通SSI
![5A2DFB67-9927-44B7-9939-CC9C7D58B902](Education项目总结01.assets/5A2DFB67-9927-44B7-9939-CC9C7D58B902.png)
SSI的配置参数如下：

* `ssi on` :开启ssi支持
* `ssi_silent_errors on`:默认为off，设置为on则在处理SSI文件出错时不输出错误信息。
* `ssi_types text/html`:默认为text/html，如果需要支持shtml（服务器执行脚本，类似于jsp）则需要设置为`ssi_type text/shtml`。

# 3.MongoDB
MongoDB是一个分布式文件存储的数据库。MongoDB是介于关系数据库和非关系数据库之间的产品，是非关系数据库中功能最丰富，最像关系数据库的。它支持的数据结构非常松散，类似JSON格式，因此可以存储比较复杂的数据结构。MongoDB最大的特点是它支持的查询语言非常强大，其语法有点类似于面向对象的查询语言，几乎可以实现类似关系数据库单表查询的绝大部份功能，而且还支持对数据建立索引。
## 3.1.MongoDB的基本概念
下面是mongodb与关系数据库的一些概念对比：
![20323E84-E5E8-4B44-89D6-5CD32642DE72](Education项目总结01.assets/20323E84-E5E8-4B44-89D6-5CD32642DE72.png)

## 3.2.Java程序进行连接
1. 添加依赖
```xml
<dependency>
	<groupId>org.mongodb</groupId>
	<artifactId>mongo‐java‐driver</artifactId>
	<version>3.4.3</version>
</dependency>
```
2. 测试程序
```java
@Test
public void testConnection(){
	//创建mongodb 客户端
	MongoClient mongoClient = new MongoClient( "localhost" ，27017 );
	//或者采用连接字符串（注释以上的代码，解开下面的两句代码）
	//MongoClientURI connectionString = new MongoClientURI("mongodb://root:root@localhost:27017");
	//MongoClient mongoClient = new MongoClient(connectionString);
	//连接数据库
	MongoDatabase database = mongoClient.getDatabase("test");
	// 连接collection
	MongoCollection<Document> collection = database.getCollection("student");
	//查询第一个文档
	Document myDoc = collection.find().first();
	//得到文件内容 json串
	String json = myDoc.toJson();
	System.out.println(json);
}
```
## 3.3.MongoDB数据库操作
* 查询所有数据库`show dbs`
* 显示当前数据库`db`
* 创建数据库`use DATABASE_NAME`如有有该数据库则切换到该数据库，如果没有该数据库，那么则创建该数据库。（注意：新创建的数据库不显示，需要至少包括一个集合）
* 删除数据库`db.dropDatabase()`想要删除数据库，先切换到想要删除的数据库，然后执行删除。
## 3.4.MongoDB集合操作
集合相当于关系数据库中的表，一个数据库可以创建多个集合，一个集合是将相同类型的文档管理起来。
* 创建集合`db.createCollection(name,options)`
name:新创建集合的名称
options:创建参数（可以省略）
* 删除集合`db.集合名称.drop()`
## 3.5.MongoDB文档操作
每个文档默认以_id作为主键，主键默认类型为ObjectId（对象类型），mongodb会自动生成主键值。
* 插入文档
```json
db.COLLECTION_NAME.insert({"name":"fechin","age":18})
```
* 更新文档
```json
db.COLLECTION_NAME.update(
	<query>,
	<update>,
	<options>
)
```
`<query>`:查询条件，相当于SQL语句的where
`<update>`:更新文档的内容
`<options>`:选项
案例：

 	1. 将符合条件“name”:”fechin”的第一个文档进行更新。**这里的操作实际上是将文档删除，然后用新的文档进行替换**

```
db.fechinchu.update({"name":"fechin"},{"name":"fechin","age":19})
```
2. $set修改其指定要更新的key，key不存在则创建，存在则更新。将符合条件“name”：“fechin”的所有文档更新。**这里的操作实际上是更新什么字段就更新什么**

```
db.fechinchu.update({"name":"fechin"},{$set:{"name":"fechin","age":19}},{multi:true})
```
multi:false表示更新第一个匹配的文档，true表示更新所有匹配的文档。
* 删除文档
```
db.fechinchu.remove(<query>)
```
query:删除条件，相当于SQL语句中的where
	1. 删除所有文档`db.fechinchu.remove({})`
	2. 删除符合条件的文档`db.fechinchu.remove({"name":"fechinchu"})`
* 查询文档
```
db.fechinchu.find(<query>,<projection>) 
```
<query>:查询条件，可不填。
<projection>:投影查询key，可不填。
	1. 查询全部：`db.fechinchu.find()`
	2. 查询name为“fechin”的文档：`db.fechinchu.find({"name":"fechin"})`
	3. 投影查询：只显示name和age两个key,_id主键不显示：`db.fechinchu.find({"name":"fechin"},{name:1,age:1,_id:0})`

## 3.6.MongoDB用户操作
* 创建用户
```json
MongoDB>db.createUser(
{ user: "<name>",
	pwd: "<cleartext password>",
	customData: { <any information> },
	roles: [
		{ role: "<role>", db: "<database>" } | "<role>",
		...
	]}
)
```
例如：创建root用户，角色为root
```json
use admin
db.createUser(
	{
	user:"root",
	pwd:"320512",
	roles:[{role:"root",db:"admin"}]
	}
)
```
# 4.接口开发
为了严格按照接口进行开发，提高效率，对请求及相应格式进行规范化。
1. **get请求时，采用key/value格式请求，SpringMVC可采用基本类型的变量接收，也可以采用对象接收。**
2. **Post请求时，可以提交form表单数据（application/x-www-form-urlencoded）和Json数据（Content-Type=application/json）,文件等多部件类型（multipart/form-data）三种数据格式，SpringMVC接收json数据使用@RequestBody注释解析请求等json数据。**
3. **响应结果统一信息为：是否成功，操作代码，提示信息及自定义数据。
4. 响应结果统一格式为Json。

## 4.1.Optional

Optional是jdk1.8引入的类型，Optional是一个容器对象，它包括了我们需要的对象，使用isPresent方法判断是否为空，isPresent方法返回false则表示Optional包含的对象为空，否则可以使用get()取出对象进行操作。

~~~java
//修改
@Test
public void testUpdate() {
        //查询对象
        Optional<CmsPage> optional = cmsPageRepository.findById("5b4b1d8bf73c6623b03f8cec");
        if(optional.isPresent()){
            CmsPage cmsPage = optional.get();
            //设置要修改值
            cmsPage.setPageAliase("test01");
            //...
            //修改
            CmsPage save = cmsPageRepository.save(cmsPage);
            //修改完成之后会返回对象，save（）方法如果主键存在，那么就是修改.springdata mongodb中没有update这个方法。
            System.out.println(save);
        }
    }
~~~

## 4.2.自定义Dao方法

同Spring Data JPA一样Spring Data mongodb也提供自定义方法的规则。如下

~~~java
public interface CmsPageRepository extends MongoRepository<CmsPage,String> { 
  //根据页面名称查询
	CmsPage findByPageName(String pageName);
	//根据页面名称和类型查询
	CmsPage findByPageNameAndPageType(String pageName,String pageType); 
  //根据站点和页面类型查询记录数
	int countBySiteIdAndPageType(String siteId,String pageType); 
  //根据站点和页面类型分页查询
  Page<CmsPage> findBySiteIdAndPageType(String siteId,String pageType, Pageable pageable);
}
~~~

# 5.接口测试
## 5.1.Swagger
### 5.1.1.Swagger介绍
OpenAPI规范（OpenAPI Specification 简称OAS）是Linux基金会的一个项目，试图通过定义一种用来描述API格式或API定义的语言，来规范RESTful服务开发过程，目前版本是V3.0，并且已经发布并开源在github上。
Swagger是全球最大的OpenAPI规范API开发工具框架，支持从设计和文档到测试和部署到整个API生命周期的开发。
Spring Boot可以集成Swagger，生成Swagger接口。
### 5.1.2.Swagger常用注解
* @Api：修饰整个类，描述Controller的作用。
* @ApiOperation：描述一个类的一个方法，或者说一个接口。
* @ApiParam：单个参数描述。
* @ApiModel：用对象来接收参数
* @ApiModelProperty：用对象接收参数时，描述对象的一个字段
* @ApiResponse:HTTP响应其中1个描述
* @ApiResponses:HTTP响应整个描述
* @ApiIgnore:使用该注解忽略这个API
* @ApiError：发生错误返回的信息
* @ApiImplicitParam:一个请求参数
* @ApiImplicitParams:多个请求参数

@ApilmplicitParam属性：
![D6B4768F-74EE-4FE8-BFCD-0C4C814E7A2A](Education项目总结01.assets/D6B4768F-74EE-4FE8-BFCD-0C4C814E7A2A.png)

### .1.3.Swagger接口定义
修改接口工程中页面查询接口，添加Swagger注解。
```java
@Api(value="cms页面管理接口",description = "cms页面管理接口，提供页面的增、删、改、查") 
public interface CmsPageControllerApi {
	@ApiOperation("分页查询页面列表") 
  @ApiImplicitParams({
		@ApiImplicitParam(name="page",value = "页 码",required=true,paramType="path",dataType="int"),
		@ApiImplicitParam(name="size",value = "每页记录 数",required=true,paramType="path",dataType="int")
  	})
  public QueryResponseResult findList(int page, int size, QueryPageRequest queryPageRequest) ;
}
```
```java
@Data
public class QueryPageRequest extends RequestData {
	//站点id
	@ApiModelProperty("站点id")
	private String siteId;
	//页面ID
	@ApiModelProperty("页面ID")
	private String pageId;
	//页面名称
	@ApiModelProperty("页面名称")
	private String pageName;
	//页面别名
	@ApiModelProperty("页面别名")
	private String pageAliase;
	//模版id
	@ApiModelProperty("模版id")
	private String templateId;
}
```



Swagger接口生成工作原理：

1. 系统启动，扫描到api工程中的Swagger2Configuration类。
2. 在此类中指定了包路径com.xuecheng,找到在此包下及子标记有@RestController注解到controller类。
3. 根据controller类中的Swagger注解生成接口文档。
![81B09B45-8D52-43BA-9DFC-02FDAC63EBFB](Education项目总结01.assets/81B09B45-8D52-43BA-9DFC-02FDAC63EBFB.png)
启动cms服务工程，查看接口文档，请求[Swagger UI](http://localhost:31001/swagger-ui.html)
## 5.2.Postman
略

## 6.MongoDB与MySQL对比

**MongoDB**:

* 内存数据库,数据都是放在内存里面;
* 对数据的操作大部分都在内存中,但MongoDB并不是单纯的内存数据库.MongoDB由C++语言进行编写,是一个基于分布式文件存储的开源数据库系统;
* 在高负债的情况下,添加更多的节点,可以保证服务器的性能;
* MongoDB目的为WEB应用提供可扩展的高性能数据存储解决方案;

MongoDB将数据存储为一个文档,将数据机构由键值(key=>value)对组成,MongoDB文档类似于JSON对象.字段值可以包含其他文档,整数及文档数据;

**MongoDB的存储特点:**

在传统的关系型数据库中,数据是以表单为媒介进行存储的,每个表单均拥有纵向的列和横向的列;

由此可见,相比较MySQL,MongoDB以一种直观文档的方式来完成数据的存储.它很像JSON格式,不过数据在存储的时候MongoDB数据库为文档增加了序列化操作,最终存进磁盘的其实叫做BSON的格式,即Binary-JSON;

在数据库存放的数据中,有一种特殊的键值叫做主键,它用于唯一的标识表中的有一条记录.也就是说.一个表不能有多个主键,而且主键不能为空值;

**无论是MongoDB还是MySQL,都存在着主键的定义**

对于MongoDB来说,其主键名叫`_id`,如果用户不主动为其分配一个主键的话,MongoDB会自动为其生成一个随机分配的值;

* 数据库的平均插入速率:MongoDB不指定_id插入>MySQL不指定主键插入>MySQL指定主键插入>MongoDB指定id插入;
* 在MongoDB中,指定索引插入比不指定慢很多,这因那个味MongoDB里每一条数据的_id值都是唯一的.当不指定id插入数据时候,其id是系统进行自动计算生成.MongoDB通过计算机特征值,时间,进程ID与随机数来确保生成的id是唯一的.而在指定id插入时,MongoDB每插一条数据,都需要检查此id可不可用,当数据库中数据条数太多的时候,这一步的查询开销会拖慢整个数据的插入速度;
* MongoDB会充分使用系统内存作为缓存,这是一种非常优秀的特征.在插入时,MongoDB会尽可能地在内存快写不进去数据之后,再将数据持久化保存到硬盘上.这也是在不指定_id插入的时候,MongoDB的效率遥遥领先的原因.但在指定id插入时,当数据量一大内存装不下时候,MongoDB就需要将磁盘中的信息读取到内存中来查重,这样一来效率反而慢了.

**MongoDB的应用场景**

在另一方面,对于开发者来说,如果是因为业务需求或者是项目初始化阶段,而是导致数据具体格式无法明确定义的话.MongoDB的这一鲜明特性就脱颖而出了.相比传统的关系型数据库,它非常容易被扩展.

不过MongoDB对数据之间事务关系支持比较弱.如果业务这一方面要求比较高的话,MongoDB还不是不适合此类的应用;

非关系型数据库(NoSQL),属于文档型数据库,文档型数据库可以存放xml,json,bson类型系那个的数据.这些数据具备自述性(self-describing),层现分层的树状数据结构.数据结构由键值(key-value)对组成.








