# SpringBoot Data整合ElasticSearch

## pom依赖

```xml
<!-- spring data和es的start依赖，会引入关联的elasticsearch-rest-high-level-clinet等依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

## RestHighLevelClient方式整合

`RestHighLevelClient`是ElasticSearch为java语言提供的客户端。我们可以通过配置类的方式引入到spring boot项目中，代码如下：

```java
@Configuration
public class RestClientConfig extends AbstractElasticsearchConfiguration {

    private static final int PORT = 9200;

    private static final String scheme = "http";

    @Value("${spring.elasticsearch.host}")
    private String hostAndPort;

    @Value("${spring.elasticsearch.username}")
    private String username;

    @Value("${spring.elasticsearch.password}")
    private String password;

    @Override
    @Bean
    public RestHighLevelClient elasticsearchClient() {
        final CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        credentialsProvider.setCredentials(AuthScope.ANY, new UsernamePasswordCredentials(username, password));

        // 通过builder创建rest client，配置http client的HttpClientConfigCallback。
        RestClientBuilder builder = RestClient.builder(new HttpHost(hostAndPort, PORT, scheme))
                .setHttpClientConfigCallback(httpClientBuilder -> httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider));

        // RestHighLevelClient实例通过REST low-level client builder进行构造。
        return new RestHighLevelClient(builder);
    }
}
```

配置好`RestHighLevelClient`之后，就可以通过他操作es，下面列出了常见的几个方法的Api。关于更多的细节参考：[elasticsearch.clients.rest](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#elasticsearch.clients.rest)

```java
@Service
public class EsServiceDemo {

    @Resource
    RestHighLevelClient highLevelClient;

    public void add(Log log) throws IOException {
        IndexRequest request = new IndexRequest(log.getIndex());
        request.source(JSONObject.toJSONString(log), XContentType.JSON);
        request.setRefreshPolicy(WriteRequest.RefreshPolicy.IMMEDIATE);
        highLevelClient.index(request, RequestOptions.DEFAULT);
    }

    public void update(Log log) throws IOException {
        GetRequest getRequest = new GetRequest(log.getIndex(), log.getId());
        boolean exists = highLevelClient.exists(getRequest, RequestOptions.DEFAULT);
        if (exists) {
            UpdateRequest updateRequest = new UpdateRequest(log.getIndex(), log.getId());
            updateRequest.doc(JSONObject.toJSONString(log), XContentType.JSON);
            highLevelClient.update(updateRequest, RequestOptions.DEFAULT);
        }
    }

    public void delete(String index, String id) throws IOException {
        DeleteRequest request = new DeleteRequest(index, id);
        highLevelClient.delete(request, RequestOptions.DEFAULT);
    }

    public Log searchById(String index, String id) throws IOException {
        GetRequest request = new GetRequest(index, id);
        GetResponse response = highLevelClient.get(request, RequestOptions.DEFAULT);
        String source = response.getSourceAsString();
        return JSONObject.parseObject(source, Log.class);
    }

    public List<Log> searchByKeyword(String index, String keyword) throws IOException {
        SearchRequest request = new SearchRequest(index);

        // 构建查询条件
        SearchSourceBuilder builder = new SearchSourceBuilder();
        MultiMatchQueryBuilder multiMatchQueryBuilder = QueryBuilders.multiMatchQuery(keyword, "clientIp", "message");
        builder.query(multiMatchQueryBuilder);

        HighlightBuilder highlightBuilder = new HighlightBuilder();
        highlightBuilder.field("message");
        highlightBuilder.preTags("<font color='red'>");
        highlightBuilder.postTags("</font>");

        builder.highlighter(highlightBuilder);

        // 将查询条件设置到请求中
        request.source(builder);

        SearchResponse response = highLevelClient.search(request, RequestOptions.DEFAULT);
        SearchHit[] hits = response.getHits().getHits();

        List<Log> logs = new LinkedList<>();
        for (SearchHit hit : hits) {
            String json = hit.getSourceAsString();
            Log log = JSONObject.parseObject(json, Log.class);

            // 设置高亮的文本到实体类中
            Map<String, HighlightField> highlightFields = hit.getHighlightFields();
            HighlightField message = highlightFields.get("message");
            if (null != message) {
                // 获取的高亮片段
                Text[] fragments = message.getFragments();
                StringBuilder sb = new StringBuilder();
                for (Text text : fragments) {
                    sb.append(text);
                }
                log.setMessage(sb.toString());
            }
            logs.add(log);
        }
        return logs;
    }
}
```

## ElasticsearchRepository方式整合

上面的方式我们在配置Bean中手动注册了客户端，spring boot也可以为我们自动装配客户端，当然这个需要我们将es的一些基本配置写在配置文件中。

```properties
spring.elasticsearch.uris=
spring.elasticsearch.username=
spring.elasticsearch.password=
```

上面用到的是es为我们提供的原生客户端，其实还是有一些麻烦的，和业务无关的代码确实有点多了。那spring data致力于为所有的数据库提供一整套统一的增删改查接口，当然也支持es。上面我们的依赖就会引入`spring-data-elasticsearch`的相关依赖。这里有一个很重要的接口，就是`ElasticsearchRepository`这个接口，他继承自`CrudRepository`，默认提供了增删改查的基本方法，并基于es扩展了一些方法。我们只需要新加一个接口，指定泛型，他就会为我们自动提供增删改查的能力。

```java
public interface ProductRepository extends ElasticsearchRepository<Product, Long> {
}
```

上面的接口我们指定泛型为Product，就是说要对Product做一些CRUD操作，但是他ES在查询的时候怎么知道你这个数据是在那个索引呢，es中的Field和你的Bean中的属性怎么对应呢。这里我们就需要在这个Product对象中进行一些配置了。

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Document(indexName = "products")  // 表示是ES中的一个文档，对应的索引是products
public class Product {

    @Id
    private Long id;

    @Field(type = FieldType.Text)  // Field注解将bean中的属性和es中的field进行映射
    private String title;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Double)
    private Double price;

    @Field(type = FieldType.Text)
    private String brand;

    @Field(type = FieldType.Float)
    private Float discount;
}
```

在上面的准备工作做好以后，es就会为我们自动装配es客户端，然后我们只要注入我们定义的这个接口`ProductRepository`，就可以使用他的基础的CRUD方法，下面是通过单元测试的方法进行验证的，亲测可行。

```java
@SpringBootTest
@RunWith(SpringRunner.class)
public class ProductTest {

    @Autowired
    private ProductRepository productRepository;

    @Test
    public void testAdd() {
        Product product = new Product();
        product.setId(1L);
        product.setTitle("Hongxing ERKE (ERKE) official flagship sportswear men's winter winter warm casual windproof sportswear men's coat black L(weight 120-140 kg)");
        product.setBrand("ERKE");
        product.setPrice(100.0);
        product.setDiscount(0.7F);
        product.setCategory("clothes");
        Product save = productRepository.save(product);
        Assert.assertEquals(1L, save.getId().longValue());
    }

    @Test
    public void testBatchAdd() {
        List<Product> products = new LinkedList<>();
        products.add(new Product(2L, "qiaodan trousers", "clothes", 97.5, "qiaodan", 0.9F));
        products.add(new Product(3L, "qiaodan Casual shoes", "shoes", 87.5, "qiaodan", 0.85F));
        products.add(new Product(4L, "nike fleece", "clothes", 197.5, "nike", 0.8F));
        products.add(new Product(5L, "nike trousers", "shoes", 97.5, "nike", 0.8F));
        Iterable<Product> result = productRepository.saveAll(products);
    }

    @Test
    public void testFindById() {

        Optional<Product> optional = productRepository.findById(1L);
        if (optional.isPresent()) {
            Product product = optional.get();
            Assert.assertEquals(1L, product.getId().longValue());
        }
    }
}
```

这还不是最牛逼的，最牛逼的是，他除了上面的功能之外，还支持自定义方法，并且将自定义的方法自动翻译成es对应的查询语言，比如我们定义这样一个方法。那么不需要多说什么，es就会知道这个方法的意思是查询价格介于lowPrice和highPrice之间的商品。

```java
public interface ProductRepository extends ElasticsearchRepository<Product, Long> {

    List<Product> findByPriceBetween(double lowPrice, double highPrice);
}

```

```java
@Test
public void testFindByPriceBetween() {
    List<Product> result = productRepository.findByPriceBetween(95.0, 150.0);
    Assert.assertEquals(3, result.size());
}
```

这里举一个简单的例子，说明如何使用，关于更多更详细的介绍参考：[Elasticsearch Repositories](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#elasticsearch.repositories)

## ElasticsearchRestTemplate方式整合

除了上面的方式，sping data还支持通过`ElasticsearchRestTemplate`的方式去访问es，这种方式我们可以灵活的构建原生的es查询语句，功能更加丰富。这个`ElasticsearchRestTemplate`对象也是spring boot为我们自动装配的。

```java
@SpringBootTest
@RunWith(SpringRunner.class)
public class ProductTest {

    @Autowired
    private ElasticsearchRestTemplate elasticsearchRestTemplate;

    @Test
    public void testSearch() {
        NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();

        // 通过QueryBuilders构建一个dis_max查询，这里可以构建所有的es的查询
        DisMaxQueryBuilder disMaxQueryBuilder = QueryBuilders.disMaxQuery();
        disMaxQueryBuilder.add(QueryBuilders.matchQuery("title", "qiaodan"));
        disMaxQueryBuilder.add(QueryBuilders.matchQuery("category", "qiaodan"));
        disMaxQueryBuilder.tieBreaker(0.7F);

        // 通过search方法发起请求，都有一样的套路
        SearchHits<Product> searchHits = elasticsearchRestTemplate.search(queryBuilder.withQuery(disMaxQueryBuilder).build(), Product.class);
        for (SearchHit<Product> searchHit : searchHits) {
            Product content = searchHit.getContent();
            System.out.println(JSONObject.toJSON(content));
        }
    }
}

```



