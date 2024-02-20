
# GUIA

## Configuración base

### Plugins

JPA, Web, devtools, h2, lombok, validation, Actuator

### Aditional Plugin

```xml
    <dependency>
        <groupId>org.modelmapper</groupId>
        <artifactId>modelmapper</artifactId>
        <version>2.4.4</version>
    </dependency>
```

### Properties

```properties
spring.application.name=service-product
server.port=8080
spring.profiles.active=development
logging.level.com.ibm.catalog=debug


#H2 Database
spring.datasource.name=product
spring.h2.console.enabled=true
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=update
spring.jpa.generate-ddl=true
spring.jpa.defer-datasource-initialization=true

## Actuator
management.endpoints.web.exposure.include=health,info
```


### data.sql

```sql
INSERT INTO product (id,name,price,create_at) VALUES('6966f770-64e1-11ee-8c99-0242ac120002','NINTENDO', 8000, NOW());
INSERT INTO product (id,name,price,create_at) VALUES('6966f630-64e1-11ee-8c99-0242ac120002','XBOX', 9000, NOW());
INSERT INTO product (id,name,price,create_at) VALUES('6966f4f0-64e1-11ee-8c99-0242ac120002','SAMSUNG', 5000, NOW());
INSERT INTO product (id,name,price,create_at) VALUES('6966f392-64e1-11ee-8c99-0242ac120002','XIOMI', 12000, NOW());
INSERT INTO product (id,name,price,create_at) VALUES('6966f130-64e1-11ee-8c99-0242ac120002','PLAYSTATION', 1000, NOW());
```
### Definir estructura

```txt
controller, domain, model, exception, repository, service, service, impl
```

## Codigo

### domain

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
@Entity
@Table(name = "product")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    private String name;

    private String currency;

    @Column(name = "create_at")
    @Temporal(TemporalType.TIMESTAMP)
    private LocalDateTime createAt;

}
```
### Exception

```java
@ControllerAdvice
public class HandleException {

    public ResponseEntity<Object> handleResponseStatusException(ResponseStatusException ex){
        Map<String, Object> body = new LinkedHashMap<>();
        body.put("timestamp", LocalDateTime.now());
        body.put("message", ex.getReason());
        body.put("error", HttpStatus.resolve(ex.getStatusCode().value()).getReasonPhrase());
        body.put("status", ex.getStatusCode().value());

        return new ResponseEntity<>(body, ex.getStatusCode());
    }


}
```

### Model

```java
public record ProductRequest(
        @NotNull String name,
        @NotNull BigDecimal price
) {
}

@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class ProductResponseModel {

   private UUID id;

    private String name;

    private BigDecimal price;

    private LocalDateTime createAt;
}

```

### Controller 

```java
@Slf4j
@Validated
@RestController
@RequestMapping("/product")
public class ProductController {

    //http://localhost:8080/product/6966f392-64e1-11ee-8c99-0242ac120002
    //http://localhost:8080/product?productId=6966f392-64e1-11ee-8c99-0242ac120002&name=user


    @Autowired
    ProductService service;

    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    @ResponseStatus(HttpStatus.OK)
    public List<ProductResponseModel> findAll(){
        return service.getAllProducts();
    }


    @GetMapping(value="/{productId}",produces = MediaType.APPLICATION_JSON_VALUE)
    @ResponseStatus(HttpStatus.OK)
    public ProductResponseModel findByProductId(@PathVariable("productId") UUID productId){
        log.debug("Finding product with id [{}]", productId);
        return service.findByProductId(productId);
    }

    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    @ResponseStatus(HttpStatus.CREATED)
    public ProductResponseModel saveProduct(@Valid @RequestBody ProductRequest request){
        return service.saveProduct(request);
    }


    @PutMapping(value="/{productId}", consumes = MediaType.APPLICATION_JSON_VALUE)
    @ResponseStatus(HttpStatus.OK)
    public ProductResponseModel updatedProduct(@PathVariable("productId") UUID productId,@Valid @RequestBody ProductRequest request){
        log.debug("Updating product with id [{}]", productId);
        return service.updatedProduct(productId,request);
    }

    @DeleteMapping(value="/{productId}",produces = MediaType.APPLICATION_JSON_VALUE)
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteProductById(@PathVariable("productId") UUID productId){
         log.debug("Deleting product with id [{}]", productId);
         service.deleteProductById(productId);
    }
}
```

### Services

```java
@Service
public interface ProductService {
    List<ProductResponseModel> getAllProducts();
    ProductResponseModel findByProductId(UUID productId);
    void deleteProductById(UUID productId);

    ProductResponseModel saveProduct(ProductRequest request);


    ProductResponseModel updatedProduct(UUID productId, ProductRequest request);
}

@Component
public class ProductServiceImpl implements ProductService {

    @Autowired
    ProductRepository productRepository;

    private final ModelMapper mapper = new ModelMapper();


    @Override
    public List<ProductResponseModel> getAllProducts() {
        return productRepository
                .findAll()
                .stream().map(product -> mapper.map(product,ProductResponseModel.class))
                .collect(Collectors.toList());
    }

    @Override
    public ProductResponseModel findByProductId(UUID productId) {
        log.debug("Finding product with id [{}]", productId);
        Product product = findProductById(productId);

        /*ProductResponseModel model = new ProductResponseModel()
        model.setName(product.getName());
        model.setPrice(product.getPrice());*/

        return mapper.map(product, ProductResponseModel.class);
    }

    @Override
    public void deleteProductById(UUID productId) {
        //Product product = findProductById(productId);
        //productRepository.delete(product);

        productRepository.deleteById(productId);
    }

    @Override
    public ProductResponseModel saveProduct(ProductRequest request) {
        Product product = Product
                .builder()
                .name(request.name())
                .price(request.price())
                .createAt(LocalDateTime.now())
                .build();


        //Product p = new Product
        //p.setName(request.name)

        try{
            //productRepository.save(product)
            //sout(product.getId() falla

            productRepository.saveAndFlush(product);
            System.out.println(product.getId());
        }
        catch (Exception ex){
            throw new ResponseStatusException(HttpStatus.INTERNAL_SERVER_ERROR, "internal server error");
        }


        return mapper.map(product,ProductResponseModel.class);
    }

    @Override
    public ProductResponseModel updatedProduct(UUID productId, ProductRequest request) {
        log.debug("Updating product [{}]", request);
        Product product = findProductById(productId);

        //product.setName(request.name !=null? request.name: product.getName())

        mapper.getConfiguration()
                .setFieldMatchingEnabled(true)
                .setFieldAccessLevel(Configuration.AccessLevel.PRIVATE)
                .setMatchingStrategy(MatchingStrategies.STRICT)
                .setSkipNullEnabled(true);

        mapper.map(request, product);

        productRepository.saveAndFlush(product);


        return mapper.map(product, ProductResponseModel.class);
    }


    private Product findProductById(UUID productId){
        return productRepository
                .findById(productId)
                .orElseThrow(()-> new ResponseStatusException(HttpStatus.NOT_FOUND, "Producto no fue encontrado [%s]".formatted(productId)));
    }
}
```

### Repositories

```java
public interface ProductRepository extends JpaRepository<Product, UUID> {

}
```

## Documentar el API

https://spring.io/projects/spring-restdocs#learn

https://springdoc.org/

```xml
   <dependency>
      <groupId>org.springdoc</groupId>
      <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
      <version>2.3.0</version>
   </dependency>
```

En el properties se agrega lo siguiente:

```txt
#To disable the springdoc-openapi endpoint (/v3/api-docs by default true
springdoc.api-docs.enabled = false 
#To disable the swagger-ui endpoint (/swagger-ui.html by default true
springdoc.swagger-ui.enabled=true

#For custom path of the swagger-ui HTML documentation.
springdoc.swagger-ui.path=/doc/swagger-ui.html

#The list of paths to match (comma separated)
#springdoc.packagesToScan=com.ibm.catalog.controller
springdoc.pathsToMatch=/product/**
```

Definicion del Bean en la clase principal

```java
@Bean
	public OpenAPI customDocOpenApi(){
		return new OpenAPI()
				.info(new Info()
						.title("Document for products")
						.version("1.0.0")
						.description("Demo Bootcamp Microservicios")
						.termsOfService("https://springdoc.org/")
						.contact(new Contact()
								.name("Gabriel Castillo")
								.email("gabcast@mx1.ibm.com")
								.url("https://github.ibm.com"))
						.license(new License()
								.name("Apache 2.0")
								.url("https://springdoc.org/")));

	}
```

http://localhost:8080/doc/swagger-ui/index.html

http://localhost:8080/v3/api-docs

https://www.ibm.com/docs/en/cognos-analytics/12.0.0?topic=api-rest-reference

## Levantar el server discovery-services

### Configuración base

### Plugin

```txt
eureka-server
```

### Clase principal 

```java
@SpringBootApplication
@EnableEurekaServer
public class RegistryApplication {

	public static void main(String[] args) {
		SpringApplication.run(RegistryApplication.class, args);
	}

}
```
### application.properties Definicion

```txt
spring.application.name=discovery-service
server.port=8761


#Register like Server
#eureka.instance.hostname=<name of server>

eureka.instance.hostname=localhost
eureka.client.service-url.default-zone= http://${eureka.instance.hostname}:${server.port}/eureka/ 
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false

hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 63000
ribbon.ConnectTimeout: 3000
ribbon.ReadTimeout: 60000
```
