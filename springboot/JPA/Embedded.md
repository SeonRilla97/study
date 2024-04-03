
하나의 Entity가 다른 객체의 필드를 포함해야 할 때 사용하는것으로

@Embedded @ElementCollection @Embedable

가 있다.


> Embedable

Entity가 사용할 객체에 정의함으로, 해당 객체는 Embeded의 대상이 된다.

```
@Embeddable // Element Collection 값 타입 객체임을 명시
@Getter
@ToString
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class ProductImage {

    private String fileName;
    private int ord;
```


> @Embedded의 경우. 하나의 타입을 의미하고 <br>
> @ElementCollection은 Collection으로 객체 타입을 다룰 때 사용된다.

```
    @ElementCollection
    private List<ProductImage> imageList = new ArrayList<>();


    @Embedded
    private ProductImage productImage;
```



### 실습 중 발생한 문제

다음 테스트 코드는 Product와 productImage의 Fetch Join을 테스트 하는 코드이다.
```
    @Test
    public void testUpdate() {
        Long pno = 10L;

        //조회
        Product product = productRepository.selectOne(pno).get();
        
        //수정
        product.changePname("10번 상품1");
        product.changedesc("10번 상품 설명입니다.");
        product.changePrice(5000);

        //삭제
        product.clearList();

        //추가
        product.addImageString(UUID.randomUUID().toString()+"_"+"NEWIMAGE1.jpg");
        productRepository.save(product);
    }

```

그리고 Repository는 다음과 같다. JPQL에서 Fetch Join을 Annotation으로 처리 하기 위해 @EntityGraph를 사용했다.

```
    @EntityGraph(attributePaths = "imageList")
    @Query("select p from Product p where p.pno = :pno")
    Optional<Product> selectOne(@Param("pno")Long pno);
```

조회는 Fetch Join을 통해 하나만 발생 할 것으로 예상 했다 하지만, Join 이후 개별 쿼리가 한번 씩 더 발생했다


```
Hibernate: 
    select
        p1_0.pno,
        p1_0.del_flag,
        il1_0.product_pno,
        il1_0.file_name,
        il1_0.ord,
        p1_0.pdesc,
        p1_0.pname,
        p1_0.price 
    from
        tbl_product p1_0 
    left join
        product_image_list il1_0 
            on p1_0.pno=il1_0.product_pno 
    where
        p1_0.pno=?


Hibernate: 
    select
        p1_0.pno,
        p1_0.del_flag,
        p1_0.pdesc,
        p1_0.pname,
        p1_0.price 
    from
        tbl_product p1_0 
    where
        p1_0.pno=?


Hibernate: 
    select
        il1_0.product_pno,
        il1_0.file_name,
        il1_0.ord 
    from
        product_image_list il1_0 
    where
        il1_0.product_pno=?
```

Fetch Join을 하지않아 영속성이 저장되지 않아 다시 개별 조회 쿼리를 발생시킨것으로 보인다.


(추가)조회 시 쿼리는 한번만 나가고, 각각을 호출 할 수 있지만, 수정/삭제를 위해서는각각을 다시 호출한다. (이게무슨...?)

