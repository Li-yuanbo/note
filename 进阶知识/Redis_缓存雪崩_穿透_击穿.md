### 1. 背景

Redis 是一个完全开源的、遵守 BSD 协议的、高性能的 key-value数据结构存储系统，它支持数据的持久化，可以将内存中的数据保存在磁盘中，而且不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储，功能十分强大，Redis还支持数据的备份，即master-slave模式的数据备份，从而提高可用性。当然最重要的还是读写速度快，作为我们平常开发中最常用的缓存方案被广泛应用。但在实际应用过程中，它会存在缓存雪崩、缓存击穿和缓存穿透等异常情况，如果忽视这些情况可能会带来灾难性的后果，下面主要对这些缓存异常和常见处理方案进行相应分析与总结。

### 2. 缓存雪崩

#### 2.1 是什么

一段时间内本应在redis缓存中处理的大量请求，都发送到了数据库进行处理，导致对数据库的压力迅速增大，严重时甚至可能导致数据库崩溃，从而导致整个系统崩溃，就像雪崩一样，引发连锁效应，所以叫缓存雪崩。

#### 2.2 为什么

出现上述情况的常见原因主要有以下两点：

- 大量缓存数据同时过期，导致本应请求到缓存的需重新从数据库中获取数据

- redis本身出现故障，无法处理请求，那自然会再请求到数据库那里

#### 2.3 怎么办

1. **针对大量缓存数据同时过期的情况：**

- 实际设置过期时间时，应当尽量避免大量 key 同时过期的场景，如果真的有，那就通过随机、微调、均匀设置等方式设置过期时间，从而避免同一时间过期；
- 添加互斥锁，使得构建缓存的操作不会在同一时间进行；
- 双key策略，主key是原始缓存，备key为拷贝缓存，主key失效时，可以访问备key，主key缓存失效时间设置为短期，备key设置为长期；
- 后台更新缓存策略，采用定时任务或者消息队列的方式进行redis缓存更新或移除等。

2. **针对redis本身出现故障的情况：**

- 在预防层面，可以通过主从节点的方式构建高可用的集群，也就是实现主 Redis 实例挂掉后，能有其他从库快速切换为主库，继续提供服务；
- 如果事情已经发生了，那就要为了防止数据库被大量的请求搞崩溃，可以采用服务熔断或者请求限流的方法。当然服务熔断相对粗暴一些，停止服务直到redis服务恢复，请求限流相对温和一些，保证一些请求可以处理，不是一刀切，不过还是看具体业务情况选择合适的处理方案。

### 3. 缓存击穿

#### 3.1 是什么

缓存击穿一般出现在高并发系统中，是大量并发用户同时请求到缓存中没有但数据库中有的数据，也就是同时读缓存没读到数据，又同时去数据库去取数据，引起数据库压力瞬间增大。和缓存雪崩不同的是，缓存击穿指并发查同一条数据，缓存雪崩是不同数据都过期了，很多数据都查不到从而查数据库

#### 3.2 为什么

这种情况其实一般出现的原因就是某个热点数据缓存过期，由于是热点数据，请求并发量又大，所以过期的时候还是会有大量请求同时过来，来不及更新缓存就全部打到数据库那边了

#### 3.3 怎么办

针对这种情况有两种常见的处理方案：

- 简单粗暴的对热点数据不设置过期时间，这样不会过期，自然也就不会出现上述情况了，如果后续想清理，可以通过后台进行清理
- 添加互斥锁，即当过期之后，除了请求过来的第一个查询的请求可以获取到锁请求到数据库，并再次更新到缓存中，其他的会被阻塞住，直到锁被释放，同时新的缓存也被更新上去了，后续请求又会请求到缓存上，这样就不会出现缓存击穿了

### 4. 缓存穿透

#### 4.1 是什么

缓存穿透是指数据既不在 redis 中，也不在数据库中，这样就导致每次请求过来的时候，在缓存中找不到对应key之后，每次都还要去数据库再查询一遍，发现数据库也没有，相当于进行了两次无用的查询。这样请求就可以绕过缓存直接查数据库，如果这个时候有人想恶意攻击系统，就可以故意使用空值或者其他不存在的值进行频繁请求，那么就会对数据库造成比较大的压力。

#### 4.2 为什么

这种现象的原因其实很好理解，业务逻辑里面如果用户对某些信息还没有进行相应的操作或者处理，那对应的存放信息的数据库或者缓存中自然也就没有相应的数据，也就容易出现上述问题。

#### 4.3 怎么办

针对缓存穿透，一般有以下三种处理方案：

- 非法请求的限制，主要是指参数校验、鉴权校验等，从而一开始就把大量的非法请求拦截在外，这在实际业务开发中是必要的手段；
- 缓存空值或者默认值，如果从缓存取不到的数据，在数据库中也没有取到，那我们仍然把这个空结果进行缓存，同时设置一个较短的过期时间。通过这个设置的默认值存放到缓存，这样第二次到缓存中获取就有值了，而不会继续访问数据库，可以防止有大量恶意请求是反复用同一个key进行攻击；
- 使用布隆过滤器快速判断数据是否存在。

### 5. 优化处理方式

#### 5.1 缓存预热

缓存预热就是系统上线前后，将相关的缓存数据直接加载到缓存系统中去，而不依赖用户。这样就可以避免在用户请求的时候，先查询数据库，然后再将数据缓存的问题。用户直接查询事先被预热的缓存数据，这样可以避免那么系统上线初期，对于高并发的流量，都会访问到数据库中， 对数据库造成流量的压力。根据数据不同量级，可以有以下几种做法：

- 数据量不大：项目启动的时候自动进行加载；
- 数据量较大：后台定时刷新缓存；
- 数据量极大：只针对热点数据进行预加载缓存操作。

#### 5.2 缓存降级

缓存降级是指当缓存失效或缓存服务出现问题时，为了防止缓存服务故障，导致数据库跟着一起发生雪崩问题，所以也不去访问数据库，但因为一些原因，仍然想要保证服务还是基本可用的，虽然肯定会是有损服务。因此，对于不重要的缓存数据，我们可以采取服务降级策略。一般做法有以下两种：

- 直接访问内存部分的数据缓存
- 直接返回系统设置的默认值