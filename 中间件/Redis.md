# Redis总结

## 生产常见问题

### 缓存击穿

描述：缓存击穿就是热点数据瞬间过期，在这个瞬间有大量请求过来，因为缓存失效，导致大量请求直接访问数据库，导致数据库宕机。

解决办法：

- 热点数据永不过期

- 使用Redis的互斥锁（**mutex**），缓存失效的时候不是立即去数据库查询，而是先使用Redis的SETNX去设置一个mutex key，当返回成功的时候，在进行数据库查询并设置到缓存；否则就重试整个get缓存的方法。SETNX，是「SET if Not eXists」的缩写，也就是只有不存在的时候才设置，可以利用它来实现锁的效果。

  

  ```java
  public String get(key) {
        String value = redis.get(key);
        if (value == null) { //代表缓存值过期
            //设置3min的超时，防止del操作失败的时候，下次缓存过期一直不能load db
        if (redis.setnx(key_mutex, 1, 3 * 60) == 1) {  //代表设置成功
                 value = db.get(key);
                        redis.set(key, value, expire_secs);
                        redis.del(key_mutex);
                } else {  //这个时候代表同时候的其他线程已经load db并回设到缓存了，这时候重试获取缓存值即可
                        sleep(50);
                        get(key);  //重试
                }
            } else {
                return value;      
            }
   }
  ```

- 使用同步锁synchronized或lock，当从缓存获取数据为空的时候，对从数据库获取的代码加锁，查询完数据后设置到缓存

  

  ```java
  private static volaite Object lockHelp=new Object();
  
     public String getValue(String key){ 
  					String value=redis.get(key,String.class);
  
           if(value=="null"||value==null||StringUtils.isBlank(value){
  
               synchronized(lockHelp){
                      value=redis.get(key,String.class);
                       if(value=="null"||value==null||StringUtils.isBlank(value){
                           value=db.query(key);
                            redis.set(key,value,1000);
                        }
                  }
  
              }else{
                   sleep(50);
                   value=redis.get(key,String.class);
               }    
  
              return value;
      }
  ```

- 使用布隆过滤器，布隆过滤器的巨大用处就是，能够迅速判断一个元素是否在一个集合中，有如下三个使用场景：

  - 网页爬虫对URL的去重，避免爬取相同的URL地址

  - 反垃圾邮件，从数十亿个垃圾邮件列表中判断某邮箱是否垃圾邮箱（同理，垃圾短信）

  - 缓存击穿，将已存在的缓存放到布隆过滤器中，当黑客访问不存在的缓存时迅速返回避免缓存及DB挂掉，缺点：需要单独维护一个集合，不支持删值操作

    

    ```java
         String get(String key) {  
       String value = redis.get(key);  
       if (value  == null) {  
    				//判断是否存在
            if(!bloomfilter.mightContain(key)){
                return null;
            }else{
               value = db.get(key);  
               redis.set(key, value);  
            }
        }
        return value；
    }
    ```

### 缓存穿透

一个一定不存在缓存及查询不到的数据，由于缓存是不命中时被动写的，并且出于容错考虑，如果从存储层查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到存储层去查询，失去了缓存的意义。

**有很多种方法可以有效地解决缓存穿透问题**，**最常见**的则是采用布隆过滤器，将所有可能存在的数据哈希到一个足够大的bitmap中，一个一定不存在的数据会被 这个bitmap拦截掉，从而避免了对底层存储系统的查询压力。**另外也有一个**更为简单粗暴的方法（我们采用的就是这种），如果一个查询返回的数据为空（不管是数据不存在，还是系统故障），我们仍然把这个空结果进行缓存，但它的过期时间会很短，最长不超过五分钟。



```java
//伪代码
public object GetProductListNew() {
    int cacheTime = 30;
    String cacheKey = "product_list";

    String cacheValue = CacheHelper.Get(cacheKey);
    if (cacheValue != null) {
        return cacheValue;
    }

    cacheValue = CacheHelper.Get(cacheKey);
    if (cacheValue != null) {
        return cacheValue;
    } else {
        //数据库查询不到，为空
        cacheValue = GetProductListFromDB();
        if (cacheValue == null) {
            //如果发现为空，设置个默认值，也缓存起来
            cacheValue = string.Empty;
        }
        CacheHelper.Add(cacheKey, cacheValue, cacheTime);
        return cacheValue;
    }
}
```

### 缓存雪崩

与缓存击穿的区别在于这里针对很多key缓存，前者则是某一个key。

缓存失效时的雪崩效应对底层系统的冲击非常可怕！大多数系统设计者考虑用加锁或者队列的方式保证来保证不会有大量的线程对数据库一次性进行读写，从而避免失效时大量的并发请求落到底层存储系统上。还有一个简单方案就时讲缓存失效时间分散开，比如我们可以在原有的失效时间基础上增加一个随机值，比如1-5分钟随机，这样每一个缓存的过期时间的重复率就会降低，就很难引发集体失效的事件。

``

```java
//伪代码
public object GetProductListNew() {
    int cacheTime = 30;
    String cacheKey = "product_list";
    String lockKey = cacheKey;

    String cacheValue = CacheHelper.get(cacheKey);
    if (cacheValue != null) {
        return cacheValue;
    } else {
        synchronized(lockKey) {
            cacheValue = CacheHelper.get(cacheKey);
            if (cacheValue != null) {
                return cacheValue;
            } else {
              //这里一般是sql查询数据
                cacheValue = GetProductListFromDB(); 
                CacheHelper.Add(cacheKey, cacheValue, cacheTime);
            }
        }
        return cacheValue;
    }
}
```

加锁排队只是为了减轻数据库的压力，并没有提高系统吞吐量。假设在高并发下，缓存重建期间key是锁着的，这是过来1000个请求999个都在阻塞的。同样会导致用户等待超时，这是个治标不治本的方法！

注意：加锁排队的解决方式分布式环境的并发问题，有可能还要解决分布式锁的问题；线程还会被阻塞，用户体验很差！因此，在真正的高并发场景下很少使用！

``

```java
//伪代码
public object GetProductListNew() {
    int cacheTime = 30;
    String cacheKey = "product_list";
    //缓存标记
    String cacheSign = cacheKey + "_sign";

    String sign = CacheHelper.Get(cacheSign);
    //获取缓存值
    String cacheValue = CacheHelper.Get(cacheKey);
    if (sign != null) {
        return cacheValue; //未过期，直接返回
    } else {
        CacheHelper.Add(cacheSign, "1", cacheTime);
        ThreadPool.QueueUserWorkItem((arg) -> {
      //这里一般是 sql查询数据
            cacheValue = GetProductListFromDB(); 
          //日期设缓存时间的2倍，用于脏读
          CacheHelper.Add(cacheKey, cacheValue, cacheTime * 2);                 
        });
        return cacheValue;
    }
} 
```

**解释说明：**

- 缓存标记：记录缓存数据是否过期，如果过期会触发通知另外的线程在后台去更新实际key的缓存；
- 缓存数据：它的过期时间比缓存标记的时间延长1倍，例：标记缓存时间30分钟，数据缓存设置为60分钟。这样，当缓存标记key过期后，实际缓存还能把旧数据返回给调用端，直到另外的线程在后台更新完成后，才会返回新缓存