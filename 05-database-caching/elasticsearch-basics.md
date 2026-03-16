# Elasticsearch Basics for Java Engineers

---

## When to Use Elasticsearch

- Full-text search (product search, document search)
- Log aggregation and analysis (ELK stack)
- Complex filtering with many dimensions
- Geospatial search
- Analytics and aggregations

NOT a replacement for PostgreSQL — use both:
- PostgreSQL: Source of truth, transactions, relationships
- Elasticsearch: Search index, derived from PostgreSQL

---

## Spring Data Elasticsearch Setup

```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-elasticsearch</artifactId>
</dependency>
```

```java
@Document(indexName = "products")
public class ProductDocument {

    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "english")
    private String name;

    @Field(type = FieldType.Text, analyzer = "english")
    private String description;

    @Field(type = FieldType.Keyword)  // Exact match, not analyzed
    private String category;

    @Field(type = FieldType.Double)
    private BigDecimal price;

    @Field(type = FieldType.Boolean)
    private boolean inStock;
}

@Repository
public interface ProductSearchRepository
        extends ElasticsearchRepository<ProductDocument, String> {

    List<ProductDocument> findByNameContainingOrDescriptionContaining(
        String name, String description);
}
```

---

## Sync PostgreSQL to Elasticsearch

```java
@Component
public class ProductElasticsearchSync {

    @Autowired
    private ElasticsearchOperations elasticsearchOperations;

    @EventListener
    @Async
    public void onProductCreated(ProductCreatedEvent event) {
        ProductDocument doc = convertToDocument(event.getProduct());
        elasticsearchOperations.save(doc);
    }

    @EventListener
    @Async
    public void onProductUpdated(ProductUpdatedEvent event) {
        ProductDocument doc = convertToDocument(event.getProduct());
        elasticsearchOperations.save(doc);  // Upsert
    }

    @EventListener
    @Async
    public void onProductDeleted(ProductDeletedEvent event) {
        elasticsearchOperations.delete(event.getProductId(), ProductDocument.class);
    }
}
```
