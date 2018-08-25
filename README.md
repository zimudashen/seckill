高并发秒杀系统
----
                语言与主体框架支持<br>
                    SpringBoot1.5.8<br>
                    JDK1.8<br>
                其他技术<br>
                    RabbitMQ<br>
                    Redis<br>
                    Validation JSR303<br>
                    Mybatis<br>
                    Thymeleaf<br>
                    Bootstrap<br>
                    fastjson<br>
                    druid<br>
       
实现技术点
----
1.两次MD5加密<br>

    将用户输入的密码和固定Salt通过MD5加密生成第一次加密后的密码，再讲该密码和随机生成的Salt通过MD5进行第二次加密
    最后将第二次加密后的密码和第一次的固定Salt存数据库
      好处：
    第一次作用：防止用户明文密码在网络进行传输
    第二次作用：防止数据库被盗，避免通过MD5反推出密码，双重保险
  <br>
2.session共享<br>

    验证用户账号密码都正确情况下，通过UUID生成唯一id作为token，再将token作为key、用户信息作为value模拟session存储到redis
    同时将token存储到cookie，保存登录状态
    好处： 

    在分布式集群情况下，服务器间需要同步，定时同步各个服务器的session信息，会因为延迟到导致session不一致
    使用redis把session数据集中存储起来，解决session不一致问题。
 <br>
3.JSR303自定义参数验证<br>

    使用JSR303自定义校验器，实现对用户账号、密码的验证，使得验证逻辑从业务代码中脱离出来。
<br>
4.全局异常统一处理<br>

    通过拦截所有异常，对各种异常进行相应的处理，当遇到异常就逐层上抛，一直抛到最终由一个统一的、专门负责异常处理的地方处理
    这有利于对异常的维护。
<br>
5.页面缓存 + 对象缓存<br>

    页面缓存：通过在手动渲染得到的html页面缓存到redis
    对象缓存：包括对用户信息、商品信息、订单信息和token等数据进行缓存，利用缓存来减少对数据库的访问，大大加快查询速度。
<br>
6.页面静态化<br>

    对商品详情和订单详情进行页面静态化处理，页面是存在html，动态数据是通过接口从服务端获取，实现前后端分离
    静态页面无需连接数据库打开速度较动态页面会有明显提高
<br>
7.本地标记 + redis预处理 + RabbitMQ异步下单 + 客户端轮询<br>

    描述：通过三级缓冲保护
    1、本地标记
    2、redis预处理 
    3、RabbitMQ异步下单，最后才会访问数据库，这样做是为了最大力度减少对数据库的访问。

    实现：
    在秒杀阶段使用本地标记对用户秒杀过的商品做标记，若被标记过直接返回重复秒杀，未被标记才查询redis，
    通过本地标记来减少对redis的访问，抢购开始前，将商品和库存数据同步到redis中，所有的抢购操作都在redis中进行处理，
    通过Redis预减少库存减少数据库访问为了保护系统不受高流量的冲击而导致系统崩溃的问题，使用RabbitMQ用异步队列处理下单，
    实际做了一层缓冲保护，做了一个窗口模型，窗口模型会实时的刷新用户秒杀的状态。client端用js轮询一个接口，用来获取处理状态
 <br>
 8.解决超卖<br>
 
    描述：
    比如某商品的库存为1，此时用户1和用户2并发购买该商品，用户1提交订单后该商品的库存被修改为0
    而此时用户2并不知道的情况下提交订单，该商品的库存再次被修改为-1，这就是超卖现象

    实现：
    对库存更新时，先对库存判断，只有当库存大于0才能更新库存
    对用户id和商品id建立一个唯一索引，通过这种约束避免同一用户发同时两个请求秒杀到两件相同商品
    实现乐观锁，给商品信息表增加一个version字段，为每一条数据加上版本。每次更新的时候version+1，并且更新时候带上版本号
    当提交前版本号等于更新前版本号，说明此时没有被其他线程影响到，正常更新，如果冲突了则不会进行提交更新。
    当库存是足够的情况下发生乐观锁冲突就进行一定次数的重试。
<br>
9.使用数学公式验证码<br>

    描述：
    点击秒杀前，先让用户输入数学公式验证码，验证正确才能进行秒杀。
 
    好处：
    防止恶意的机器人和爬虫
    分散用户的请求
    
    实现：
    前端通过把商品id作为参数调用服务端创建验证码接口
    服务端根据前端传过来的商品id和用户id生成验证码，并将商品id+用户id作为key，生成的验证码作为value存入redis
    同时将生成的验证码输入图片写入imageIO让前端展示
    将用户输入的验证码与根据商品id+用户id从redis查询到的验证码对比，相同就返回验证成功，进入秒杀；
    不同或从redis查询的验证码为空都返回验证失败，刷新验证码重试
<br>
10.使用AccessLimit实现限流<br>

    描述：
    当我们去秒杀一些商品时，此时可能会因为访问量太大而导致系统崩溃，此时要使用限流来进行限制访问量，当达到限流阀值
    后续请求会被降级；降级后的处理方案可以是：返回排队页面（高峰期访问太频繁，等一会重试）、错误页等。

    实现：
    项目使用AccessLimit+Redis来实现限流，AccessLimit是一个注解，定义了时间、访问次数
