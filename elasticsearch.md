### 一、倒排索引

1. 什么是文档和词条？

   每一条数据就是一个文档

   对文档中的内容分词，得到的词语就是词条

2. 什么是正向索引？

   基于文档id创建索引。查询词条时必须先找到文档，而后判断是否包含词条

3. 什么是倒排索引？

   对于文档内容分词，对词条创建索引，并记录词条所在文档的信息。查询时先根据词条查询到文档id，而后获取到数据。

正向：一条条顺序遍历，每遍历一条判断是否包含该词条，得到查询数据

倒排：根据词条查询id，得到查询数据

### 二、ES与MySQL概念对比

MySQL        ES

table            index

row              document

column        field

schema        mapping

SQL               DSL

文档：一条数据就是一个文档，ES中是json格式

字段：json文档中的字段

索引：同类型文档的集合

映射：索引中文档的约束，比如字段名称、类型

**elasticsearch和数据库的区别：**

数据库负责事务类型的操作

elasticsearch负责海量数据的搜索、分析、计算

### 三、IK分词器

1. ik_smart和ik_max_word（分词器的两种模式）

2. 扩展词和停用词的配置

   在/var/lib/docker/volumes/es-plugins/_data/ik/config路径下的IKAnalyzer.cfg.xml文件中配置，停用词和扩展词文件名，扩展词文件和停用词文件需和IKAnalyzer.cfg.xml在同一路径下

分词器的作用：

1. 创建倒排索引时对文档分词
2. 用户搜索时对输入的内容分词

### 四、索引库操作

**mapping属性：**mapping是对索引库中文档的约束

1. type：字段数据类型
2. index：是否创建索引，默认为true
3. analyzer：使用哪种分词器
4. properties：该字段的子字段

**type属性：**

1. 字符串：text（可分词文本）、keyword（精确值，例如：品牌、国家）
2. 数字：long、integer、short、byte、double、float
3. 布尔：boolean
4. 日期：date
5. 对象：object

**创建索引库：**PUT /索引库名称

``` json
# 创建索引库
PUT /heima
{
  "mappings": {
    "properties": {
      "info":{
        "type": "text",
        "analyzer": "ik_smart"
      },
      "email":{
        "type": "keyword",
        "index": false
      },
      "name":{
        "type": "object",
        "properties": {
          "firstName":{
            "type":"keyword"
          },
          "lastName":{
            "type":"keyword"
          }
        }
      }
    }
  }
}
```

**查询索引库：**GET /索引库名

**删除索引库：**DELETE /索引库名

**修改索引库：**索引库和mapping一旦创建成功则**无法修改**，但是**可以添加**新的字段

PUT /索引库名/_mapping

```json
PUT /heima/_mapping
{
    "properties":{
        "新字段名":{
            "type":"integer"
        }
    }
}
```

### 五、文档操作

**添加文档**

POST /索引库名/_doc/文档id    （**如果没有id，会随机生成id**，不建议）

```json
POST /索引库名/_doc/文档id
{
    "字段1":"值1"，
    "字段2":{
    "子属性1":"值2"，
    "子属性2":"值3"
}
}
```

**查询文档：**GET /heima/_doc/文档id

**删除文档：**DELETE /heima/_doc/文档id

**修改文档：分为全量修改和局部修改**

全量修改：会先删除旧文档，在添加新文档（如果原文档id不存在则相当于新增操作）

```json
PUT /索引库名/_doc/文档id
{
    "字段1":"值1",
    "字段2":"值2"
}
```

局部修改：修改指定字段

```json
POST /索引库名/_update/文档id
{
    "doc":{
        "字段名":"新的值"
    }
}
```



### 六、RestClient操作索引库

RestClient：es提供的各种不同语言的客户端，用来操作es，本质就是组装DSL语句，通过http请求发送给ES

字段拷贝可以使用copy_to属性，将当前字段拷贝到指定字段

```json
"all":{
    "type":"text",
    "analyzer":"ik_max_word"
},
"name":{
    "type":"keyword",
    "copy_to":"all"
}
"age":{
    "type":"integer",
    "copy_to":"all"
}
```

当有多个字段需要参与搜索时，可以拷贝到一个字段"all"，在查询时不会出现"all"字段内容，但可以根据"all"字段内容进行搜索

**初始化JavaRestClient**

1. 引入依赖

   ```java
   <dependency>
               <groupId>org.elasticsearch.client</groupId>
               <artifactId>elasticsearch-rest-high-level-                   			client</artifactId>
   </dependency>
   ```

2. 因为SpringBoot默认的版本是7.6.2，所以需要覆盖默认的ES版本

   ```java
   <properties>
           <elasticsearch.version>7.12.1</elasticsearch.version>
   </properties>
   ```

3. 初始化RestHighLevelClient（在每个Test执行前会初始化，完成后会关闭client）

   ```java
   public class HotelIndexTest {
       private RestHighLevelClient client;
   
       @BeforeEach
       void detUp(){
           this.client = new RestHighLevelClient(RestClient.builder(
                   HttpHost.create("http://192.168.40.132:9200")
           ));
       }
   
       @AfterEach
       void tearDown() throws IOException {
           this.client.close();
       }
   }
   
   ```

   **创建索引库：**

   ```java
         public static final String MAPPING_TEMPLATE = "{\n" +
               "  \"mappings\": {\n" +
               "    \"properties\": {\n" +
               "      \"info\":{\n" +
               "        \"type\": \"text\",\n" +
               "        \"analyzer\": \"ik_smart\"\n" +
               "      },\n" +
               "      \"email\":{\n" +
               "        \"type\": \"keyword\",\n" +
               "        \"index\": false\n" +
               "      },\n" +
               "      \"name\":{\n" +
               "        \"type\": \"object\",\n" +
               "        \"properties\": {\n" +
               "          \"firstName\":{\n" +
               "            \"type\":\"keyword\"\n" +
               "          },\n" +
               "          \"lastName\":{\n" +
               "            \"type\":\"keyword\"\n" +
               "          }\n" +
               "        }\n" +
               "      }\n" +
               "    }\n" +
               "  }\n" +
               "}\n";
   
       @Test
       void createHotelIndex() throws IOException {
           // 1.创建Request对象
           CreateIndexRequest request = new CreateIndexRequest("hotel");
           // 2.准备请求的参数: DSL语句
           request.source(MAPPING_TEMPLATE, XContentType.JSON);
           // 3.发送请求
           client.indices().create(request, RequestOptions.DEFAULT);
       }
   
   ```

   **删除索引库：**
   
   ```java
       @Test
       void testDeleteHotelIndex() throws IOException {
           // 1.创建Request对象
           DeleteIndexRequest request = new DeleteIndexRequest("hotel");
           // 3.发送请求
           client.indices().delete(request, RequestOptions.DEFAULT);
       }
   ```
   
   **判断索引库是否存在**：
   
   ```java
       @Test
       void testExistsHotelIndex() throws IOException {
           // 1.创建Request对象
           GetIndexRequest request = new GetIndexRequest("hotel");
           // 3.发送请求
           boolean exists = client.indices().exists(request, 		   			RequestOptions.DEFAULT);
           System.out.println(exists);
       }
   ```
   
   ### 七、RestClient操作文档
   
   **首先初始化RestClient**和六一样
   
   **新增文档：**
   
   ```java
       @Test
       void testAddDocument() throws IOException {
           // DSL语句
           String DSLtest = "{\n" +
                   "  \"name\":\"Jack\",\n" +
                   "  \"age\":24\n" +
                   "}";
           // 1.创建Request对象
           IndexRequest request = new IndexRequest("IndexName").id("1");
           // 2.准备请求的参数: DSL语句
           request.source(DSLtest, XContentType.JSON);
           // 3.发送请求
           client.index(request,RequestOptions.DEFAULT);
       }
   ```
   
   **根据id查询文档：**
   
   ```java
       @Test
       void testGetDocument() throws IOException {
           
           // 1.创建Request对象
           GetRequest request = new GetRequest("IndexName", "1");
           // 2.发送请求
           GetResponse response = client.get(request, RequestOptions.DEFAULT);
           // 3.解析结果
           String json = response.getSourceAsString();
   
           System.out.println(json);
       }
   ```
   
   **根据id修改酒店数据的两种方式：**
   
   1.全量更新：和新增文档代码一样（这里不演示）
   
   2.局部更新：只更新部分字段
   
   ```java
       @Test
       void testUpdateDocument() throws IOException {
   
           // 1.创建Request对象
           UpdateRequest request = new UpdateRequest("IndexName", "1");
           // 2.准备参数，每两个参数为一对，都用逗号隔开
           request.doc(
                   "age", 18,
                   "name", "Tom"
           );
           // 3.更新文档
           client.update(request, RequestOptions.DEFAULT);
       }
   ```
   
   **删除文档：**
   
   ```java
       @Test
       void testDeleteDocument() throws IOException {
           // 1.创建Request对象
           DeleteRequest request = new DeleteRequest("hotel", "1");
           // 2.发送请求
           client.delete(request, RequestOptions.DEFAULT);
       }
   ```
   
   **批量导入酒店数据到ES**
   
   ```java
       @Test
       void testBulkRequest() throws IOException {
           // 批量查询酒店数据
           List<Hotel> hotels = hotelService.list();
   
           // 1.创建Request对象
           BulkRequest request = new BulkRequest();
           for(Hotel hotel:hotels){
               // 转换为文档类型
               HotelDoc hotelDoc = new HotelDoc(hotel);
               // 创建新增文档的Request对象
               request.add(new IndexRequest("hotel")
                       .id(hotelDoc.getId().toString())
                       .source(JSON.toJSONString(hotelDoc), XContentType.JSON)
               );
           }
   
           // 2.发送请求
           client.bulk(request, RequestOptions.DEFAULT);
       }
   ```
   

### 八、DSL查询语法

1. **查询所有**：查询出所有数据，一般测试用。例如：**match_all**
2. **全文检索查询**：利用分词器对用户输入内容分词，然后去倒排索引库中匹配常见的两种方式是（**match_query** 和 **multi_match_query**）
3. **精确查询**：根据精确词条值查询数据，一般查找keyword、数值、日期、boolean等类型字段。不会对搜索条件分词。例如（**ids**：根据id查询  **range**：根据值的范围查询  **term**：根据词条精确值查询）
4. **地理查询**：根据经纬度查询。例如：
   * geo_distance
   * geo_bounding_box
5. **符合查询**：符合查询可以将上述各种查询条件组合起来，合并查询条件。例如：
   * bool
   * function_score

查询语法基本如下：

```json
GET /indexName/_search
{
    "query":{
        "查询类型":{
            "查询条件":"条件值"
        }
    }
}
```



**查询所有：**

```json
GET /indexName/_search
{
    "query":{
        "match_all":{
        }
    }
}
```

#### 全文检索查询：

**match查询**：全文检索查询的一种，对用户输入内容分词，然后去倒排索引库检索

```json
GET /indexName/_search
{
    "query":{
        "match":{
            "FIELD":"TEXT"
        }
    }
}
```

**multi_match**：与match类似，只不过允许同时查询多个字段。（参与查询字段越多，查询性能越差）

```json
GET /indexName/_search
{
    "query":{
        "multi_match":{
            "query":"TEXT",
            "fields":["FIELD1","FIELD2"]
        }
    }
}
```

**推荐将多个要查询的字段copy_to到一个"all"字段中，和multi_match查询作用一样，但是查询性能更好**

**精确查询：常见的是"term" 和 "range"查询**

**term查询语法**:

```json
GET /indexName/_search
{
    "query":{
        "term":{
            "FIELD":{
                "value":"VALUE"
            }
        }
    }
}

// eg
GET /hotel/_search
{
    "query":{
        "term":{
            "city":{
                "value":"上海"
            }
        }
    }
}
```

**range查询**：根据数值范围查询，可以是数值、日期的范围（gte大于等于，gt大于）

```json
GET /indexName/_search
{
    "query":{
        "range":{
            "FILED":{
                "gte":10,
                "lte":20
            }
        }
    }
}
```

**地理查询：**根据经纬度查询，常见的有两种

* **geo_bounding_box**：查询geo_point值落在某个矩形范围的所有文档

```json
GET /indexName/_search
{
    "query":{
        "geo_bounding_box":{
            "FIELD":{
                "top_left":{
                    "lat":31.1,
                    "lon":121.5
                },
                "bottom_right":{
                    "lat":30.9,
                    "lon":121.7
                }
            }
        }
    }
}
```

* **geo_distance**：查询到指定中心点小于某个距离值的所有文档

```json
GET /indexName/_search
{
    "query":{
        "geo_distance":{
            "distance":"15km",
            "FIELD":"31.21,121.5"
        }
    }
}
```

**复合查询：**

* **fuction score**：算分函数查询，可以控制文档相关性算分，控制文档排名。

**相关性算分：当我们利用match查询时，文档结果会根据词条关联度打分**

常见的三种算分方式：BM25 > IDF >TF （可用性）

* TF（词条频率）
* IDF（逆文档频率）
* BM25

**Function Score Query：**可以修改文档的相关性算分，根据得到的新的算分排序

```json
GET /indexName/_search
{
    "query":{
        "function_score":{
            "query":{"match":{"all":"外滩"}}, //原始查询条件，进行相关性打分
            "function":[
                {
                    "filter":{"term":{"id":1}}, //过滤条件，符合条件的才重新算分
                    "weight":10  //算分函数
                }
            ],
            "boost_mode":"multiply" //加权模式
        }
    }
}
// 算分函数
// 1. weight: 给一个常量作为函数结果
// 2. field_value_factor: 用文档中的某个字段值作为函数结果
// 3. random_score: 随机生成一个值，作为函数结果
// 4. script_score: 自定义计算公式，公式结果作为函数结果

// 加权模式，定义function score 和 query score的运算方式
// multiply: 两者相乘，默认为这个
// replace: 用function score 代替 query score
// 常见的还有: sum, avg, max, min
```

**Boolean Query：**布尔查询是一个或多个查询子句的组合。子查询组合方式有：

* must：必须匹配每个子查询："与"
* should：选择性匹配子查询："或"
* must_not：必须不匹配，且**不参与算分**："非"
* filter：必须匹配，**不参与算分**

```json
GET /hotel/_search
{
    "query":{
        "bool":{
            "must":[
                {"term":{"city":"上海"}}
            ],
            "should":[
                {"term":{"brand":"皇冠假日"}},
                {"term":{"brand":"华美达"}}
            ],
            "must_not":[
                {"range":{"price":{"lte":500}}} // <=500取反
            ],
            "filter":[
                {"range":{"score":{"gte":45}}},
                {
                    "geo_distance":{
                        "distance":"10km",
                        "location":{
                            "lat":31.21,
                            "lon":121.5
                        }
                    }
                }
            ]
        }
    }
}
// 查询城市必须是上海的，品牌是"皇冠假日"或者"华美达"的，
// 价格大于500的，评分必须>=45的，与选取位置距离<=10km的酒店
```

#### 对搜索结果处理

**排序：**es默认根据相关度算分来排序。可以排序的字段类型有：keyword类型（字典排序），数值类型，地理坐标类型，日期类型等。

```json
GET /indexName/_search
{
    "query":{
        "match_all":{}
    },
    "sort":[
        {"FIELD1":"desc"}, //排序字段和排序方式ASC、DESC
        {"FIELD2":"asc"} //第一个字段相等则根据第二个字段排序
    ]
}
```

**分页：**ES默认只返回top10的数据。如果要查询更多数据就要修改分页参数了

ES通过修改from、size参数来控制要返回的分页结果：

```json
GET /hotel/_search
{
    "query":{
        "match_all":{}
    },
    "from":990, // 分页开始的位置
    "size":10, // 期望获取的文档总数
    "sort":[
        {"price":"asc"}
    ]
}
```

**深度分页问题：**ES是分布式的，所以面临深度分页问题。例如按price排序后，获取前1000条数据

1. 首先在每个数据分片上都排序并查询前1000条文档
2. 然后将所有节点的结果聚合，在内存中重新排序，选出前1000条文档

如果搜索页数过深，或者结果集(from+size)越大，对内存和CPU的消耗也越高。因此ES设定结果集查询的上限是10000

**深度分页的两种解决方案：**

* search after：分页时需要排序，原理是从上一次的排序值开始，查询下一页数据。**官方推荐使用**
* scroll：原理将排序数据形成**快照**，保存在内存。**官方不推荐使用**

**from+size：**

* 优点：支持随机分页
* 缺点：深度分页问题，默认查询上限是10000
* 场景：百度、京东、谷歌这样的随机翻页搜索

**after search:**

* 优点：没有查询上限（单次查询的size不超过10000）
* 缺点：只能向后逐页查询，不支持随机翻页
* 场景：没有随机翻页需求的搜索，例如手机向下滚动翻页

**scroll：**

* 优点：没有查询上限（单次查询的size不超过10000）
* 缺点：会有额外的内存消耗，并且搜索结果是非实时的
* 场景：海量数据的获取和迁移。（不推荐，建议用after search方案）

**高亮：**在搜索结果中把搜索关键字突出显示

其原理是：

* 将搜索结果中的关键字用标签标记出来
* 在页面中给标签调价css样式

```json
GET /hotel/_search
{
    "query":{
        "match":{
            "FIELD":"TEXT"
        }
    },
    "highlight":{
        "fields":{ //指定要高亮的字段
            "FIELD":{
                "pre_tags":"<em>", // 用来标记高亮字段的前置标签
                "post_tags":"</em>" // 用来标记高亮字段的后置标签
            },
            "name":{
                "require_field_match":"false" //高亮字段和搜索字段不需要匹配
            }
        }
    }
}
// 标记默认的标签就是<em>、</em>
// require_field_match默认为true，不匹配才能更好的高亮显示
```

### 九、使用RestClient实现文档操作

**首先创建好连接，实现请求后断开连接**

```java
    @BeforeEach
    void detUp(){
        this.client = new RestHighLevelClient(RestClient.builder(
                HttpHost.create("http://192.168.40.132:9200")
        ));
    }

    @AfterEach
    void tearDown() throws IOException {
        this.client.close();
    }
```



1. **使用RestClient实现查询文档**

```java
    @Test
    void testMatchAll() throws IOException {
        // 1. 准备request
        SearchRequest request = new SearchRequest("hotel");
        // 2. 组织DSL参数
        request.source()  //source代表请求的DSL，也就是DSL中的大json代码
                .query(QueryBuilders.matchAllQuery());
        // 3. 发送请求，得到响应结果
        SearchResponse response = client.search(request, 	     					RequestOptions.DEFAULT);
        // 4. 解析响应
        SearchHits searchHits = response.getHits();
        // 4.1 获取总条数
        long total = searchHits.getTotalHits().value;
        System.out.println("共搜索到: "+total+"条数据");
        // 4.2 文档数组
        SearchHit[] hits = searchHits.getHits();
        // 4.3 遍历
        for(SearchHit hit:hits){
            // 获取文档source
            String json = hit.getSourceAsString();
            // 反序列化
            HotelDoc hotelDoc = JSON.parseObject(json, HotelDoc.class);
            System.out.println("hotelDoc = "+hotelDoc);
        }
        System.out.println(response);
    }
```

2. **使用RestClient全文检索查询**

   全文检索的match和multi_match查询与match_all的API基本一致。差别是查询条件，也就是query的部分

```java
    @Test
    void testMatch() throws IOException {
        // 1. 准备request
        SearchRequest request = new SearchRequest("hotel");
        // 2. 组织DSL参数
        request.source()  //source代表请求的DSL，也就是DSL中的大json代码
                .query(QueryBuilders.matchQuery("all","如家"));
        // 3. 发送请求，得到响应结果
        SearchResponse response = client.search(request,   							RequestOptions.DEFAULT);
        handleResponse(response);
    }
```

精确查询常见的有**term查询和range查询**，也同样用QueryBuilders实现

```java
        // 词条查询
        QueryBuilders.termQuery("city", "杭州");
        // 范围查询
        QueryBuilders.rangeQuery("price").gte(100).lte(150);
```

**复合查询-boolean query**

```java
        // 创建bool查询
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
        // 添加must条件
        boolQuery.must(QueryBuilders.termQuery("city", "杭州"));
        // 添加filter条件
        boolQuery.filter(QueryBuilders.rangeQuery("price").lte(250));
```

**排序和分页，排序和分页与query是同级的参数**

```java
    @Test
    void testPageAndSort() throws IOException {
        // 页码和每页大小
        int page = 1, size = 5;
        // 1. 准备request
        SearchRequest request = new SearchRequest("hotel");
        // 2.1 组织DSL参数
        request.source().query(QueryBuilders.matchAllQuery());
        // 2.2 排序sort
        request.source().sort("price", SortOrder.ASC);
        // 2.3 分页 from、size
        request.source().from((page-1)*size).size(5);
        // 3. 发送请求，得到响应结果
        SearchResponse response = client.search(request, 			   				RequestOptions.DEFAULT);
        // 4. 解析响应
        handleResponse(response);
    }
```

**RestClient实现高亮**

1. 高亮DSL构建

```java
    @Test
    void testHighlight() throws IOException {
        // 1. 准备request
        SearchRequest request = new SearchRequest("hotel");
        // 2 组织DSL参数
        request.source().query(QueryBuilders.matchQuery("all", "如家"));
        // 高亮
        request.source().highlighter(new 			   								HighlightBuilder().field("name").requireFieldMatch(false));
        // 3. 发送请求，得到响应结果
        SearchResponse response = client.search(request, 							RequestOptions.DEFAULT);
        // 4. 解析响应
        handleHighlightResponse(response);
    }
```

2. 高亮结果解析

```java
    private void handleHighlightResponse(SearchResponse response) {
        // 4. 解析响应
        SearchHits searchHits = response.getHits();
        // 4.1 获取总条数
        long total = searchHits.getTotalHits().value;
        System.out.println("共搜索到: "+total+"条数据");
        // 4.2 文档数组
        SearchHit[] hits = searchHits.getHits();
        // 4.3 遍历
        for(SearchHit hit:hits){
            // 获取文档source
            String json = hit.getSourceAsString();
            // 反序列化
            HotelDoc hotelDoc = JSON.parseObject(json, HotelDoc.class);
            // 获取高亮结果
            Map<String, HighlightField> highlightFields = 								hit.getHighlightFields();
            if(!CollectionUtils.isEmpty(highlightFields)){
                // 根据字段名获取高亮结果
                HighlightField highlightField = highlightFields.get("name");
                if(highlightField!=null){
                    // 获取高量值
                    String name = highlightField.getFragments()[0].string();
                    // 覆盖非高亮结果
                    hotelDoc.setName(name);
                }
            }
        }

    }
```

### 十、数据聚合

聚合可以实现对文档数据的统计、分析、运算。

常见的聚合有三类：

* **桶(Bucket)聚合：**用来对文档做分组
  * TermAggregation：按照文档字段分组
  * Date Histogram：按照日期阶梯分组，例如一周或者一月为一组
* **度量(Metric)聚合：**用以计算一些值，比如：最大值、最小值、平均值等
  * Avg：求平均值
  * Max：求最大值
  * Min：求最小值
  * Stats：同时求max、min、avg、sum等
* **管道(pipeline)聚合：**以其他聚合的结果为基础做聚合

参与聚合的字段类型必须是：keyword、数值、日期、布尔

#### **DSL实现Bucket聚合：**

eg：根据酒店品牌名称做聚合（酒店名称为term类型）

```json
GET /hotel/_search
{
    "size":0, // 设置size为0，结果中不包含文档，只包含聚合结果
    "aggs":{ // 定义聚合
        "brandAgg":{ // 给聚合起名字
            "terms":{ // 聚合的类型，按品牌聚合，所以选择term
                "field":"brand", // 参与聚合的字段
                "size":20 // 希望获取的聚合结果数量
            }
        }
    }
}
```

默认情况下，Bucket聚合会统计Bucket内的文档数量，记为_count，并且按照_count降序排序

我们可以修改排序方式：

``` json
GET /hotel/_search
{
    "size":0,
    "aggs":{
        "brandAgg":{
            "terms":{ 
                "field":"brand", 
                "order":{
                    "_count":"asc" // 按照_count升序排序
                },
                "size":20 
            }
        }
    }
}
```

Bucket默认对索引库的所有文档做聚合，我们可以**限定聚合的文档范围**，只要添加**query条件**即可

```json
GET /hotel/_search
{
    "query":{
        "range":{
            "price":{
                "lte":200 // 只对200元以下的文档聚合
            }
        }
    },
    "size":0,
    "aggs":{
        "brandAgg":{
            "term":{
                "field":"brand",
                "size":20
            }
        }
    }
}
```

总结：aggs代表聚合，与query同级（query限定聚合的文档范围）

**聚合必须得三要素：**

* 聚合名称
* 聚合类型
* 聚合字段

**聚合可配置的属性：**

* size：指定聚合结果的数量
* order：指定聚合结果的排序方式
* field：指定聚合字段

#### DSL实现Metrics聚合

例如，我们要获取每个品牌的用户评分的min、max、avg等值

可以用stats聚合：

```json
GET /hotel/_search{
    "size":0,
    "aggs":{
        "brandAgg":{
            "terms":{
                "field":"brand",
                "size":20
            },
            "aggs":{ // 是brands聚合的子聚合，也就是分组后对每组分别计算
                "score_stats":{ // 聚合名称
                    "stats":{ // 聚合类型。这里stats可以计算min、max、avg等
                        "field":"score" // 聚合字段
                    }
                }
            }
        }
    }
}
```

也可以按照score_stats中的某个值排序，以平均值为例

**排序条件写在terms属性里**

```json
GET /hotel/_search{
    "size":0,
    "aggs":{
        "brandAgg":{
            "terms":{
                "field":"brand",
                "size":20,
                "order":{
                    "scoreAgg.avg":"desc" // 按照scoreAgg属性中的avg降序
                }
            },
            "aggs":{ // 是brands聚合的子聚合，也就是分组后对每组分别计算
                "scoreAgg":{ // 聚合名称
                    "stats":{ // 聚合类型。这里stats可以计算min、max、avg等
                        "field":"score" // 聚合字段
                    }
                }
            }
        }
    }
}
```

**RestAPI实现聚合**

RestClient与DSL对比

```java
        request.source().size(0);
        request.source().aggregation(
                AggregationBuilders
                        .terms("brand_agg")
                        .field("brand")
                        .size(20)
        );
```

```json
GET /hotel/_search
{
    "size":0,
    "aggs":{
        "brand_agg":{
            "terms":{
                "field":"brand",
                "size":20
            }
        }
    }
}
```

使用RestAPI解析聚合结果

```java
        // 解析聚合结果
        Aggregations aggregations = response.getAggregations();
        // 根据名称获取聚合结果
        Terms brandTerms = aggregations.get("brand_agg");
        // 获取桶
        List<? extends Terms.Bucket> buckets = brandTerms.getBuckets();
        // 遍历
        for(Terms.Bucket bucket:buckets){
            // 获取key, 也就是品牌信息
            String brandName = bucket.getKeyAsString();
            System.out.println(brandName);
        }
```

#### **拼音分词器**

**自定义拼音分词器**

ES中分词器(analyzer)的组成包含三部分：

* character filter：在tokenizer之前对文本进行处理。例如删除字符、替换字符
* tokenizer：将文本按照一定规则切割成词条(term)。例如keyword，就是不分词；还有ik_smart
* tokenizer filter：将tokenizer输出的词条做进一步处理。例如大小写转换、同义词处理、拼音处理等

eg：(  :) -> 开心，:( -> 伤心   )

"四级考试通过了:)"  --> (character filter) --> "四级考试通过了开心" --> (tokenizer：ik_smart) --> 

("四级"、"考试"、"通过"、"开心") --> (tokenizer filter) --> ("siji"、"kaoshi"、"tongguo"、"kaixin")

```json
PUT /test
{
    "settings":{
        "analysis":{
            "analyzer":{ // 自定义分词器
                "my_analyzer":{ // 分词器名称
                    "tokenizer":"ik_max_word",
                    "filter":"pinyin"
                }
            }
        }
    }
}
```

进一步对pinyin filter做配置(在setting中配置)

```json
PUT /test  // 索引库名，自定义的filter只能用于该索引库
{
    "settings":{
        "analysis":{
            "analyzer":{ // 自定义分词器
                "my_analyzer":{ // 分词器名称
                    "tokenizer":"ik_max_word",
                    "filter":"py"
                }
            },
            "filter":{ // 自定义 tokenizer filter
                "py":{ // 过滤器名称
                    "type":"pinyin", // 过滤器类型
                    "keep_full_pinyin":false,
                    "keep_joined_full_pinyin":true,
                    "keep_original":true,
                    "limit_first_letter_length":16,
                    "remove_duplicated_term":true,
                    "none_chinese_pinyin_tokenize":false
                }
            }
        }
    },
    "mappings":{ // 必须指定mapping才可以用
        "properties":{
            "name":{
                "type":"text",
                "analyzer":"my_analyzer"
            }
        }
    }
}
```

**拼音分词器适合在创建倒排索引的时候使用，但不能再搜索时使用**

eg：

创建倒排索引时：

```json
{
    "id":1,
    "name":"狮子"
}
// 分词：狮子、shizi、sz
{
    "id":2,
    "name":"虱子"
}
// 分词：虱子、shizi、sz


//  词条    文档编号
//  狮子    1
//  shizi   1,2
//  sz      1,2
//  虱子    2
```

搜索时：用户搜索"狮子"

```json
// 分词：狮子、shizi、sz
// 搜索文档：1,2 (2为虱子，显然不是想要的结果)
```

解决方法：在mapping映射中指定两个analyzer

```json
PUT /test  // 索引库名，自定义的filter只能用于该索引库
{
    "settings":{
        "analysis":{
            "analyzer":{ // 自定义分词器
                "my_analyzer":{ // 分词器名称
                    "tokenizer":"ik_max_word",
                    "filter":"py"
                }
            },
            "filter":{ // 自定义 tokenizer filter
                "py":{ // 过滤器名称
                    "type":"pinyin", // 过滤器类型
                    "keep_full_pinyin":false,
                    "keep_joined_full_pinyin":true,
                    "keep_original":true,
                    "limit_first_letter_length":16,
                    "remove_duplicated_term":true,
                    "none_chinese_pinyin_tokenize":false
                }
            }
        }
    },
    "mappings":{ // 必须指定mapping才可以用
        "properties":{
            "name":{
                "type":"text",
                "analyzer":"my_analyzer", // 创建索引时用的
                "search_analyzer":"ik_smart" // 搜索时用的
            }
        }
    }
}
```

**DSL实现自动补全：completion suggester查询**

ES提供了completion suggester查询实现自动补全功能。这个查询会匹配以用户输入内容开头的词条并返回。为了提高补全查询效率，对于文档中字段的类型有一些约束

* 参与补全查询的字段必须是completion类型
* 字段的内容一般是用来补全的多个词条形成的数组

```json
// 创建索引库
PUT test
{
   "mappings":{
        "properties":{
            "title":{
                "type":"completion"
            }
        }
    }
}

// 实例数据
POST test/_doc
{
    "title":["Sony","WH-1000XM3"]
}
POST test/_doc
{
    "title":["SK_II","PITERA"]
}
```

查询语法：

```json
// 自动补全查询
GET /test/_search
{
    "suggest":{
        "title_suggest":{ // 给查询起名
            "text":"s", // 查询关键字
            "completion":{
                "field":"title", // 补全查询的字段
                "skip_duplicates":true, // 跳过重复的
                "size":10 //获取前10条结果
            }
        }
    }
}
```

**RestAPI实现自动补全**（对比上面DSL）

```java
        // 1. 准备request
        SearchRequest request = new SearchRequest("hotel");
        // 2. 组织DSL参数
        request.source()
                .suggest(new SuggestBuilder().addSuggestion(
                        "mySuggestion",
                        SuggestBuilders
                                .completionSuggestion("title")
                                .prefix("h")
                                .skipDuplicates(true)
                                .size(10)
                ));
        // 3. 发送请求，得到响应结果
        SearchResponse response = client.search(request, 							RequestOptions.DEFAULT);
```

处理结果

```java
        // 4. 处理结果
        Suggest suggest = response.getSuggest();
        // 4.1 根据名称获取补全结果
        CompletionSuggestion suggestion = 											suggest.getSuggestion("hotelSuggestion");
        // 4.2 获取option并遍历
        for(CompletionSuggestion.Entry.Option option:suggestion.getOptions())		{
            // 4.3 获取一个option中的text，也就是补全的词条
            String text = option.getText().toString();
            System.out.println(text);
        }
```

#### ES集群

单机的ES做数据存储会面临两个问题

* 海量数据存储问题：将索引库从逻辑上拆分为N个分片（ shard ），存储到多个节点
* 单点故障问题：将分片数据在不同节点备份( replica )

#### ES集群状态监控

1. win安装cerebro (不推荐) cerebro与win不兼容，出现闪退问题
2. 使用虚拟机安装cerebro并在虚拟机里启动
   * 输入命令启动cerebro  /usr/share/cerebro/bin/cerebro (输入文件路径即可)
   * 外部不能访问虚拟机内部端口问题，设置虚拟机防火墙
   * firewall-cmd --zone=public --add-port=9000/tcp --permanent
   * firewall-cmd --add-port=9000/tcp  （添加端口外部访问权限）
   * firewall-cmd --reload  （重新载入，添加端口后重新载入才能起作用）

**ES集群中的节点角色**（ES默认每个节点同时肩负下面四种角色）

* **master eligible**：备选主节点，主节点可以管理和记录集群状态、决定分片在哪个节点、处理创建和删除索引库的请求
* **data**：数据节点，存储数据、搜索、聚合、crud
* **ingest**：数据存储前的预处理
* **coordinating**：路由请求到其他节点，合并其他节点处理的结果，返回给用户（路由、负载均衡）

