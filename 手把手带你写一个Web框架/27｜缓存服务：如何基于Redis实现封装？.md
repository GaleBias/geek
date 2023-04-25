<audio title="27｜缓存服务：如何基于Redis实现封装？" src="https://static001.geekbang.org/resource/audio/92/8f/92303d32ed479fd6f02a009e791ec98f.mp3" controls="controls"></audio> 
<p>你好，我是轩脉刃。</p><p>上面两节课把数据库操作接入到hade框架中了，现在我们能使用容器中的ORM服务来操作数据库了。在实际工作中，一旦数据库出现性能瓶颈，除了优化数据库本身之外，另外一个常用的方法是使用缓存来优化业务请求。所以这节课，我们来讨论一下，hade框架如何提供缓存支持。</p><p>现在的Web业务，大部分都是使用Redis来做缓存实现。但是，缓存的实现方式远不止Redis一种，比如在Redis出现之前，Memcached一般是缓存首选；在单机上，还可以使用文件来存储数据，又或者直接使用进程的内存也可以进行缓存实现。</p><p>缓存服务的底层使用哪个存储方式，和具体的业务架构原型相关。我个人在不同业务场景中用过不少的缓存存储方案，不过业界用的最多的Redis，还是优点比较突出。相比文件存储，它能集中分布式管理；而相比Memcached，优势在于多维度的存储数据结构。所以，顺应潮流，我们hade框架主要也针对使用Redis来实现缓存服务。</p><p>我们这节课会创建两个服务，一个是Redis服务，提供对Redis的封装，另外一个是缓存服务，提供一系列对“缓存”的统一操作。而这些统一操作，具体底层是由Redis还是内存进行驱动的，这个可以根据配置决定。</p><!-- [[[read_end]]] --><p>下面我们一个个来讨论吧。</p><h2>Redis服务</h2><p>首先封装一个可以对Redis进行操作的服务。和封装ORM一样，我们自己并不实现Redis的底层传输协议和操作封装，只将Redis“创建连接”的过程封装在hade中就行了。</p><p>这里我们就选择<a href="https://github.com/go-redis/redis">go-redis</a>这个库来实现对Redis的连接。这个库目前也是Golang开源社区最常用的Redis库，有12.8k的star数，使用的是BSD 协议，可以引用，可以修改，但是修改的同时要保留版权声明，这里我们并不需要修改，所以BSD已经足够了。这个库目前是v8版本，可以使用 <code>go get github.com/go-redis/redis/v8</code> 来引入它。</p><p>go-redis的连接非常简单，我们看官网的例子，就看创建连接部分：</p><pre><code class="language-go">import (
&nbsp; &nbsp; "context"
&nbsp; &nbsp; "github.com/go-redis/redis/v8"
)

var ctx = context.Background()

func ExampleClient() {
    // 创建连接
&nbsp; &nbsp; rdb := redis.NewClient(&amp;redis.Options{
&nbsp; &nbsp; &nbsp; &nbsp; Addr:&nbsp; &nbsp; &nbsp;"localhost:6379",
&nbsp; &nbsp; &nbsp; &nbsp; Password: "", // no password set
&nbsp; &nbsp; &nbsp; &nbsp; DB:&nbsp; &nbsp; &nbsp; &nbsp;0,&nbsp; // use default DB
&nbsp; &nbsp; })

&nbsp; &nbsp; ...
}
</code></pre><p><strong>核心的redis.NewClient方法，返回的是一个*redis.Client结构，它就相当于Gorm中的DB数据结构，就是我们要实例化Redis的实例</strong>。这个结构是一个封装了300+个Redis操作的数据结构，你可以使用 <code>go doc github.com/go-redis/redis/v8.Client</code> 来观察它封装的Redis操作。</p><h3>配置</h3><p>redis.NewClient方法还有一个参数：*redis.Options 数据结构。这个数据结构就相当于Gorm中的gorm.Config，里面封装了实例化redis.Client的各种配置信息，来看一些重要的配置，都做了注释：</p><pre><code class="language-go">// redis的连接配置
type Options struct {
   // 网络情况
   // Default is tcp.
   Network string
   // host:port 格式的地址
   Addr string
   
   // redis的用户名
   Username string
   // redis密码
   Password string
   // redis的database
   DB int
   
   // 连接超时
   // Default is 5 seconds.
   DialTimeout time.Duration
   // 读超时
   // Default is 3 seconds.
   ReadTimeout time.Duration
   // 写超时
   // Default is ReadTimeout.
   WriteTimeout time.Duration
   
   // 最小空闲连接数
   MinIdleConns int
   // 最大连接时长
   MaxConnAge time.Duration
   
   // 空闲连接时长
   // Default is 5 minutes. -1 disables idle timeout check.
   IdleTimeout time.Duration
   ...
}
</code></pre><p>这些配置项相信你也非常熟悉了，既有连接请求的配置项，也有连接池的配置项。</p><p>和Gorm的配置封装一样，我们想要给用户提供一个配置即用的缓存服务，需要做如下三个事情：</p><ul>
<li>自定义一个数据结构，封装redis.Options结构</li>
<li>让刚才自定义的结构能生成一个唯一标识（类似Gorm的DSN）</li>
<li>支持通过配置文件加载这个结构，同时，支持通过Option可变参数来修改它</li>
</ul><p>在framework/contract/redis.go中，我们首先定义<strong>RedisConfig数据结构</strong>，这个结构单纯封装redis.Options就行了，没有其他额外的参数需要设置：</p><pre><code class="language-go">// RedisConfig 为hade定义的Redis配置结构
type RedisConfig struct {
   *redis.Options
</code></pre><p>同时为这个RedisConfig定义一个唯一标识，来标识一个redis.Client。这里我们选用了Addr、DB、UserName、Network 四个字段值来标识。<strong>基本上这四个字段加起来能标识“用什么账号登录哪个Redis地址的哪个database”了</strong>：</p><pre><code class="language-go">// UniqKey 用来唯一标识一个RedisConfig配置
func (config *RedisConfig) UniqKey() string {
   return fmt.Sprintf("%v_%v_%v_%v", config.Addr, config.DB, config.Username, config.Network)
}
</code></pre><p>RedisConfig结构定义完成，下面想要把它加载并支持可修改，我们要结合实例化redis.Client对象来说。</p><h3>初始化连接</h3><p>如何封装Redis的连接实例，这个同Gorm的封装一样，使用Option可变参数的方式。还是在 framework/contract/redis.go中继续写入：</p><pre><code class="language-go">package contract

...

const RedisKey = "hade:redis"

// RedisOption 代表初始化的时候的选项
type RedisOption func(container framework.Container, config *RedisConfig) error

// RedisService 表示一个redis服务
type RedisService interface {
   // GetClient 获取redis连接实例
   GetClient(option ...RedisOption) (*redis.Client, error)
}
</code></pre><p>定义了一个RedisService，表示Redis服务对外提供的协议，它只有一个GetClient方法，通过这个方法能获取到Redis的一个连接实例redis.Client。</p><p>你能看到GetClient方法有一个可变参数RedisOption，这个可变参数是一个函数结构，参数中带有传递进入了的RedisConfig指针，所以<strong>这个RedisOption是有修改RedisConfig结构的能力的</strong>。</p><p>那具体提供哪些RedisOption函数呢？和ORM一样，我们要提供多层次的修改方案，包括默认配置、按照配置项进行配置，以及手动配置：</p><ul>
<li>GetBaseConfig获取redis.yaml根目录下的Redis配置，作为默认配置</li>
<li>GetConfigPath 根据指定配置路径获取Redis配置</li>
<li>WithRedisConfig 可以直接修改RedisConfig中的redis.Options配置信息</li>
</ul><p>在实现这三个函数之前，有必要先看一下我们的Redis配置文件cofig/testing/redis.yaml：</p><pre><code class="language-yaml">timeout: 10s # 连接超时
read_timeout: 2s # 读超时
write_timeout: 2s # 写超时

write:
    host: localhost # ip地址
    port: 3306 # 端口
    db: 0 #db
    username: jianfengye # 用户名
    password: "123456789" # 密码
    timeout: 10s # 连接超时
    read_timeout: 2s # 读超时
    write_timeout: 2s # 写超时
    conn_min_idle: 10 # 连接池最小空闲连接数
    conn_max_open: 20 # 连接池最大连接数
    conn_max_lifetime: 1h # 连接数最大生命周期
    conn_max_idletime: 1h # 连接数空闲时长
</code></pre><p>和database.yaml的配置一样，根级别的作为默认配置，二级配置作为单个Redis的配置，并且二级配置会覆盖默认配置。这里还有一个小心思，特意将这些配置项都和database.yaml保持一致了，这样使用者在配置的时候能减少学习成本。</p><p>这三个方法的具体实现和Gorm没有什么太大的区别。基本方法就是使用容器中的配置服务、读取配置信息，然后修改参数中的RedisConfig指针，你可以参考分支中的代码文件<a href="https://github.com/gohade/coredemo/blob/geekbang/27/framework/provider/redis/config.go">framework/provider/redis/config.go</a>。</p><p>我们重点把注意力放在<strong>GetClient方法的实现</strong>上，写在framework/provider/redis/service.go中。类似gorm的GetDB方法，它是一个单例模式，就是一个RedisConfig，只产生一个redis.Client，用一个map加上一个lock来初始化Redis实例：</p><pre><code class="language-go">// HadeRedis 代表hade框架的redis实现
type HadeRedis struct {
    container framework.Container      // 服务容器
    clients   map[string]*redis.Client // key为uniqKey, value为redis.Client (连接池）

    lock *sync.RWMutex
}
</code></pre><p>在GetClient函数中，首先还是获取基本Redis配置 redisConfig，使用参数opts对redisConfig进行修改，最后判断当前redisConfig是否已经实例化了：</p><ul>
<li>如果已经实例化，返回实例化redis.Client；</li>
<li>如果未实例化，实例化redis.Client，返回实例化的redis.Client。</li>
</ul><pre><code class="language-go">// GetClient 获取Client实例
func (app *HadeRedis) GetClient(option ...contract.RedisOption) (*redis.Client, error) {
    // 读取默认配置
    config := GetBaseConfig(app.container)

    // option对opt进行修改
    for _, opt := range option {
        if err := opt(app.container, config); err != nil {
            return nil, err
        }
    }

    // 如果最终的config没有设置dsn,就生成dsn
    key := config.UniqKey()

    // 判断是否已经实例化了redis.Client
    app.lock.RLock()
    if db, ok := app.clients[key]; ok {
        app.lock.RUnlock()
        return db, nil
    }
    app.lock.RUnlock()

    // 没有实例化gorm.DB，那么就要进行实例化操作
    app.lock.Lock()
    defer app.lock.Unlock()

    // 实例化gorm.DB
    client := redis.NewClient(config.Options)

    // 挂载到map中，结束配置
    app.clients[key] = client

    return client, nil
}
</code></pre><p>这里只讲了Redis服务的接口和服务实现的关键函数，其中provider的实现基本上和ORM的一致，没有什么特别，就不在这里重复列出代码了。</p><p>到这里我们就将Redis的服务融合进入hade框架了。但Redis只是缓存服务的一种实现，我们这节课最终目标是想实现一个缓存服务。</p><h2>缓存服务</h2><p>缓存服务的使用方式其实非常多，我们可以设置有超时/无超时的缓存，也可以使用计数器缓存，一份好的缓存接口的设计，能对应用的缓存使用帮助很大。</p><p>所以这一部分，相比缓存服务的具体实现，<strong>缓存服务的协议设计直接影响了这个服务的可用性</strong>，我们要重点理解对缓存协议的设计。</p><h3>协议</h3><p>实现一个服务的三步骤，服务协议、服务提供者、服务实例。就先从协议开始，我们希望这个缓存服务提供哪些能力呢？</p><p>首先，缓存协议一定是有两个方法，一个设置缓存、一个获取缓存。设定为Get方法为获取缓存，Set方法为设置缓存。</p><pre><code class="language-go">// Get 获取某个key对应的值
Get(ctx context.Context, key string) (string, error)
// Set 设置某个key和值到缓存，带超时时间
Set(ctx context.Context, key string, val string, timeout time.Duration) error
</code></pre><p>同时，注意设置缓存的时候，又区分出两种需求，我们需要设置带超时时间的缓存，也需要设置不带超时时间的、永久的缓存。所以，Set方法衍生出Set和SetForever两种。</p><pre><code class="language-go">// SetForever 设置某个key和值到缓存，不带超时时间
SetForever(ctx context.Context, key string, val string) error
</code></pre><p>在设置了某个key之后，会不会需要修改这个缓存key的缓存时长呢？完全是有可能的，比如将某个key的缓存时长加大，或者想要获取某个key的缓存时长，所以我们再把注意力放在缓存时长的操作上，提供对缓存时长的操作函数SetTTL和GetTTL：</p><pre><code class="language-go">// SetTTL 设置某个key的超时时间
SetTTL(ctx context.Context, key string, timeout time.Duration) error
// GetTTL 获取某个key的超时时间
GetTTL(ctx context.Context, key string) (time.Duration, error)
</code></pre><p>再来，Get和Set目前对应的value值为string，但是我们希望value值能不仅仅是一个字符串，它还可以直接是一个对象，这样缓存服务就能存储和获取一个对象出来，能大大方便缓存需求。</p><p>所以我们定义两个GetObj和SetObj方法，来实现对象的缓存存储和获取，但是这个对象在实际存储的时候，又势必要进行序列化和反序列的过程，所以我们对存储和获取的对象再增加一个要求，让它实现官方库的BinaryMarshaler和BinaryUnMarshaler接口：</p><pre><code class="language-go">// GetObj 获取某个key对应的对象, 对象必须实现 https://pkg.go.dev/encoding#BinaryUnMarshaler
GetObj(ctx context.Context, key string, model interface{}) error
// SetObj 设置某个key和对象到缓存, 对象必须实现 https://pkg.go.dev/encoding#BinaryMarshaler
SetObj(ctx context.Context, key string, val interface{}, timeout time.Duration) error
// SetForeverObj 设置某个key和对象到缓存，不带超时时间，对象必须实现 https://pkg.go.dev/encoding#BinaryMarshaler
SetForeverObj(ctx context.Context, key string, val interface{}) error
</code></pre><p>现在，我们已经可以一个key进行缓存获取和设置了，但是有时候要同时对多个key做缓存的获取和设置，来设置对多个key进行操作的方法GetMany和SetMany：</p><pre><code class="language-go">// GetMany 获取某些key对应的值
GetMany(ctx context.Context, keys []string) (map[string]string, error)
// SetMany 设置多个key和值到缓存
SetMany(ctx context.Context, data map[string]string, timeout time.Duration) error
</code></pre><p>在实际业务中，我们还会有一些计数器的需求，需要将计数器存储到缓存，同时也要能对这个计数器缓存进行增加和减少的操作。可以为计数器缓存设计Calc、Increment、Decrement的接口：</p><pre><code class="language-go">// Calc 往key对应的值中增加step计数
Calc(ctx context.Context, key string, step int64) (int64, error)
// Increment 往key对应的值中增加1
Increment(ctx context.Context, key string) (int64, error)
// Decrement 往key对应的值中减去1
Decrement(ctx context.Context, key string) (int64, error)
</code></pre><p>缓存的使用有一种Cache-Aside模式，可以提升“获取数据”的性能。可能你没有听过这个名字，但其实我们都用过，这个模式描述的就是在实际操作之前，先去缓存中查看有没有对应的数据，如果有的话，不进行操作，如果没有的话才进行实际操作生成数据，并且把数据存储在缓存中。</p><p>我们希望缓存服务也能支持这种Cache-Aside模式。如何支持呢？</p><p>首先，要有一个生成数据的通用方法结构，我们定义为RememberFunc，让这个函数将服务容器传递进去，这样在具体的实现中，使用者就可以从服务容器中获取各种各样的具体注册服务了，能大大增强这个RemeberFunc的实现能力：</p><pre><code class="language-go">// RememberFunc 缓存的Remember方法使用，Cache-Aside模式对应的对象生成方法
type RememberFunc func(ctx context.Context, container framework.Container) (interface{}, error)
</code></pre><p>然后，我们为缓存服务定义一个Remember方法，来实现这个Cache-Aside模式。</p><pre><code class="language-go">// Remember 实现缓存的Cache-Aside模式, 先去缓存中根据key获取对象，如果有的话，返回，如果没有，调用RememberFunc 生成
Remember(ctx context.Context, key string, timeout time.Duration, rememberFunc RememberFunc, model interface{}) error
</code></pre><p>它的参数来仔细看下。除了context之外，有一个key，代表这个缓存使用的key，其次是timeout 代表缓存时长，接着是前面定义的 RememberFunc了，代表如果缓存中没有这个key，就调用RememberFunc函数来生成数据对象。</p><p>这个数据对象从哪里输出呢？就是这里的最后一个参数model了，当然这个Obj必须实现BinaryMarshaler和BinaryUnmarshaler接口。这样定义之后，Remember的具体实现就简单了。</p><p>看这个我在单元测试代码provider/cache/services/redis_test.go中写的测试：</p><pre><code class="language-go">type Bar struct {
   Name string
}
func (b *Bar) MarshalBinary() ([]byte, error) {
   return json.Marshal(b)
}
func (b *Bar) UnmarshalBinary(bt []byte) error {
   return json.Unmarshal(bt, b)
}

Convey("remember op", func() {
   objNew := Bar{}
   objNewFunc := func(ctx context.Context, container framework.Container) (interface{}, error) {
      obj := &amp;Bar{
         Name: "bar",
      }
      return obj, nil
   }
   err = mc.Remember(ctx, "foo_remember", 1*time.Minute, objNewFunc, &amp;objNew)
   So(err, ShouldBeNil)
   So(objNew.Name, ShouldEqual, "bar")
})
</code></pre><p>我们定义了Bar结构，它实现了BinaryMarshaler和BinaryUnmarshaler接口，并且定义了一个objNewFunc方法实现了前面我们定义的RememberFunc。</p><p>之后可以使用Remember方法来为这个方法设置一个Cache-Aside缓存，它的key为foo_remember，缓存时长为1分钟。</p><p>最后回看一下我们对缓存的协议定义，各种缓存的设置和获取方法都有了，还差删除缓存的方法对吧。所以来定义删除单个key的缓存和删除多个key的缓存：</p><pre><code class="language-go">// Del 删除某个key
Del(ctx context.Context, key string) error
// DelMany 删除某些key
DelMany(ctx context.Context, keys []string) error
</code></pre><p>到这里缓存协议就定义完成了，一共16个方法，要好好理解下这些方法的定义，还是那句话，理解如何定义协议比实现更为重要。<br>
<img src="https://static001.geekbang.org/resource/image/da/e5/da9b83e6856e5fd523bc270981846fe5.jpg?wh=2364x2273" alt=""></p><h2>实现</h2><p>下面来实现这个缓存服务。前面一再强调了，Redis只是缓存的一种实现，Redis之外，我们可以用不同的存储来实现缓存，甚至，可以使用内存来实现。目前hade框架支持内存和Redis实现缓存，这里我们就先看看如何用Redis来实现缓存。</p><p>由于缓存有不同实现，所以和日志服务一样，<strong>要使用配置文件来cache.yaml中的driver字段，来区别使用哪个缓存</strong>。如果driver为redis，表示使用Redis来实现缓存，如果为memory，表示用内存来实现缓存。当然如果使用Redis的话，就需要同时带上Redis连接的各种参数，参数关键字都类似前面说的Redis服务的配置。</p><p>一个典型的cache.yaml的配置如下：</p><pre><code class="language-go">driver: redis # 连接驱动
host: 127.0.0.1 # ip地址
port: 6379 # 端口
db: 0 #db
timeout: 10s # 连接超时
read_timeout: 2s # 读超时
write_timeout: 2s # 写超时

#driver: memory # 连接驱动
</code></pre><p>那对应到具体实现上，区分使用哪个缓存驱动，我们会在服务提供者provider中来进行。在provider中，注意下Register方法，注册具体的服务实例方法时，要先读取配置中的cache.driver路径：</p><pre><code class="language-go">// Register 注册一个服务实例
func (l *HadeCacheProvider) Register(c framework.Container) framework.NewInstance {
   if l.Driver == "" {
      tcs, err := c.Make(contract.ConfigKey)
      if err != nil {
         // 默认使用console
         return services.NewMemoryCache
      }

      cs := tcs.(contract.Config)
      l.Driver = strings.ToLower(cs.GetString("cache.driver"))
   }

   // 根据driver的配置项确定
   switch l.Driver {
   case "redis":
      return services.NewRedisCache
   case "memory":
      return services.NewMemoryCache
   default:
      return services.NewMemoryCache
   }
}
</code></pre><p>如果是Redis驱动，我们使用service.NewRedisCache来初始化一个Redis连接，定义RedisCache结构来存储redis.Client。</p><p><strong>在初始化的时候，先确定下容器中是否已经绑定了Redis服务，如果没有的话，做一下绑定操作</strong>。这个行为能让我们的缓存容器更为安全。</p><p>接着使用cache.yaml中的配置，来初始化一个redis.Client，这里使用的redisService.GetClient和redis.WithConfigPath，都是上面设计Redis服务的时候刚设计实现的方法。最后将redis.Client 封装到RedisCache中，返回：</p><pre><code class="language-go">import (
   "context"
   "errors"
   redisv8 "github.com/go-redis/redis/v8"
   "github.com/gohade/hade/framework"
   "github.com/gohade/hade/framework/contract"
   "github.com/gohade/hade/framework/provider/redis"
   "sync"
   "time"
)

// RedisCache 代表Redis缓存
type RedisCache struct {
   container framework.Container
   client    *redisv8.Client
   lock      sync.RWMutex
}

// NewRedisCache 初始化redis服务
func NewRedisCache(params ...interface{}) (interface{}, error) {
   container := params[0].(framework.Container)
   if !container.IsBind(contract.RedisKey) {
      err := container.Bind(&amp;redis.RedisProvider{})
      if err != nil {
         return nil, err
      }
   }

   // 获取redis服务配置，并且实例化redis.Client
   redisService := container.MustMake(contract.RedisKey).(contract.RedisService)
   client, err := redisService.GetClient(redis.WithConfigPath("cache"))
   if err != nil {
      return nil, err
   }

   // 返回RedisCache实例
   obj := &amp;RedisCache{
      container: container,
      client:    client,
      lock:      sync.RWMutex{},
   }
   return obj, nil
}
</code></pre><p>好，有Redis缓存的实例了，下面来看16个方法的实现。</p><p>Set系列的方法一共有Set/SetObj/SetMany/SetForever/SetForeverObj/SetTTL 6个，其他5个相对简单一些，在生成的redis.Client结构中都有对应实现，我们直接使用redis.Client调用即可，就不赘述了。其中SetMany方法相对复杂些，我们着重说明下。</p><p>在Redis中，SetMany这种为多个key设置缓存的方法，一般可以遍历key，然后一个个调用Set方法，但是这样效率就低了。更好的实现方式是使用pipeline。</p><p>什么是Redis的pipeline呢？Redis的客户端和服务端的交互，采用的是客户端-服务端模式，就是每个客户端的请求发送到Redis服务端，都会有一个完整的响应。所以，向服务端发送n个请求，就对应有n次响应。那么<strong>对于这种n个请求且n个请求没有上下文逻辑关系，我们能不能批量发送，但是只发送一次请求，然后只获取一次响应呢</strong>？</p><p>Redis的pipeline就是这个原理，它将多个请求合成为一个请求，批量发送给Redis服务端，并且只从服务端获取一次数据，拿到这些请求的所有结果。</p><p>我们的SetMany就很符合这个场景。具体的代码如下：</p><pre><code class="language-go">// SetMany 设置多个key和值到缓存
func (r *RedisCache) SetMany(ctx context.Context, data map[string]string, timeout time.Duration) error {
   pipline := r.client.Pipeline()
   cmds := make([]*redisv8.StatusCmd, 0, len(data))
   for k, v := range data {
      cmds = append(cmds, pipline.Set(ctx, k, v, timeout))
   }
   _, err := pipline.Exec(ctx)
   return err
}
</code></pre><p>先用redis.Client.Pipeline() 来创建一个pipeline管道，然后用一个redis.StatusCmd数组来存储要发送的所有命令，最后调用一次pipeline.Exec来一次发送命令。</p><p>Set方法就讲到这里，Get系列的方法一共有4个，Get/GetObj/GetMany/GetTTL。</p><p>在实现Get系列方法的时候有地方需要注意下，因为<strong>Get是有可能Get一个不存在的key的</strong>，对于这种不存在的key是否返回error，是一个可以稍微思考的话题。</p><p>比如Get这个方法，返回的是string和error，如果对于一个不存在的key，返回了空字符串+空error的组合，而对于一个设置了空字符串的key，也返回空字符串+空error的组合，这里其实是丢失了“是否存在key”的信息的。</p><p>所以，对于这些不存在的key，我们设计返回一个 ErrKeyNotFound 的自定义error。像Get函数就实现为如下：</p><pre><code class="language-go">// Get 获取某个key对应的值
func (r *RedisCache) Get(ctx context.Context, key string) (string, error) {
   val, err := r.client.Get(ctx, key).Result()
   // 这里判断了key是否为空
   if errors.Is(err, redisv8.Nil) {
      return val, ErrKeyNotFound
   }
   return val, err
}
</code></pre><p>其他Get相关的实现没有什么难点。</p><p>除了Get系列和Set系列，其他的方法有Calc、Increment、Decrement、Del、DelMany 都没有什么太复杂的逻辑，都是redis.Client的具体封装。</p><p>最后看下Remember这个方法：</p><pre><code class="language-go">// Remember 实现缓存的Cache-Aside模式, 先去缓存中根据key获取对象，如果有的话，返回，如果没有，调用RememberFunc 生成
func (r *RedisCache) Remember(ctx context.Context, key string, timeout time.Duration, rememberFunc contract.RememberFunc, obj interface{}) error {
   err := r.GetObj(ctx, key, obj)
   // 如果返回为nil，说明有这个key，且有数据，obj已经注入了，返回nil
   if err == nil {
      return nil
   }

   // 有err，但是并不是key不存在，说明是有具体的error的，不能继续往下执行了，返回err
   if !errors.Is(err, ErrKeyNotFound) {
      return err
   }

   // 以下是key不存在的情况, 调用rememberFunc
   objNew, err := rememberFunc(ctx, r.container)
   if err != nil {
      return err
   }

   // 设置key
   if err := r.SetObj(ctx, key, objNew, timeout); err != nil {
      return err
   }
   // 用GetObj将数据注入到obj中
   if err := r.GetObj(ctx, key, obj); err != nil {
      return err
   }
   return nil
}
</code></pre><p>前面说过Remember方法是Cache-Aside模式的实现，它的逻辑是先判断缓存中是否有这个key，如果有的话，直接返回对象，如果没有的话，就调用RememberFunc方法来实例化这个对象，并且返回这个实例化对象。</p><p>好了，这里的framework/provider/cache/redis.go我们实现差不多了。</p><h2>验证</h2><p>来做验证，我们为缓存服务写一个简单的路由，在这个路由中：</p><ul>
<li>获取缓存服务</li>
<li>设置foo为key的缓存，值为bar</li>
<li>获取foo为key的缓存，把值打印到控制台</li>
<li>删除foo为key的缓存</li>
</ul><pre><code class="language-go">// DemoCache cache的简单例子
func (api *DemoApi) DemoCache(c *gin.Context) {
   logger := c.MustMakeLog()
   logger.Info(c, "request start", nil)
   // 初始化cache服务
   cacheService := c.MustMake(contract.CacheKey).(contract.CacheService)
   // 设置key为foo
   err := cacheService.Set(c, "foo", "bar", 1*time.Hour)
   if err != nil {
      c.AbortWithError(500, err)
      return
   }
   // 获取key为foo
   val, err := cacheService.Get(c, "foo")
   if err != nil {
      c.AbortWithError(500, err)
      return
   }
   logger.Info(c, "cache get", map[string]interface{}{
      "val": val,
   })
   // 删除key为foo
   if err := cacheService.Del(c, "foo"); err != nil {
      c.AbortWithError(500, err)
      return
   }
   c.JSON(200, "ok")
}
</code></pre><p>增加对应的路由：</p><pre><code class="language-go">r.GET("/demo/cache/redis", api.DemoRedis)
</code></pre><p>在浏览器中请求地址： <a href="http://localhost:8888/demo/cache/redis">http://localhost:8888/demo/cache/redis</a>：<br>
<img src="https://static001.geekbang.org/resource/image/2a/50/2acd5f030ab6b4d5ce8e05ec0c961850.png?wh=583x174" alt=""></p><p>查看控制台输出的日志：<br>
<img src="https://static001.geekbang.org/resource/image/6c/14/6cf8ba131ba431303f5f77560015f814.png?wh=617x130" alt=""></p><p>可以明显看到cacheService.Get的数据为bar，打印了出来。验证正确！</p><p>本节课我们主要修改了framework目录下Redis和cache相关的代码。目录截图也放在这里供你对比查看，所有代码都已经上传到<a href="https://github.com/gohade/coredemo/tree/geekbang/27">geekbang/27</a>分支了。<br>
<img src="https://static001.geekbang.org/resource/image/04/03/04d8a0896596b62ab57848a882d82903.png?wh=355x1116" alt=""></p><h2>小结</h2><p>除DB之外，缓存是我们最常使用的一个存储了，今天我们先是实现了Redis的服务，再用Redis服务实现了一个缓存服务。</p><p>第一部分的Redis服务，同上一节课ORM的逻辑一样，我们只是将go-redis库进行了封装，具体怎么使用，还是依赖你在实际工作中多使用、多琢磨，网上也有很多go-redis库的相关资料。<br>
<img src="https://static001.geekbang.org/resource/image/da/e5/da9b83e6856e5fd523bc270981846fe5.jpg?wh=2364x2273" alt=""><br>
在第二部分实现的过程中，相信你现在能理解，<strong>一个服务的接口设计，就是一个“我们想要什么服务”的思考过程</strong>。比如在缓存服务接口设计中，我们定义了16个方法，囊括了Get/Set/Del/Remember等一系列方法，你可以对照思维导图复习一下。但这些方法并不是随便拍脑袋出来的，是因为有设置缓存、获取缓存、删除缓存等需求，才这样设计的。</p><h3>思考题</h3><p>目前hade框架支持内存和Redis实现缓存，我们今天展示了Redis的实现。缓存服务的内存缓存如何实现呢？可以先思考一下，如果是你来实现会如何设计呢？如果有兴趣，你可以自己动手操作一下。完成之后，你可以比对GitHub分支上我已经实现的版本，看看有没有更好的方案。</p><p>欢迎在留言区分享你的思考。感谢你的收听，如果你觉得今天的内容对你有所帮助，也欢迎分享给你身边的朋友，邀请他一起学习。我们下节课见～</p>
<style>
    ul {
      list-style: none;
      display: block;
      list-style-type: disc;
      margin-block-start: 1em;
      margin-block-end: 1em;
      margin-inline-start: 0px;
      margin-inline-end: 0px;
      padding-inline-start: 40px;
    }
    li {
      display: list-item;
      text-align: -webkit-match-parent;
    }
    ._2sjJGcOH_0 {
      list-style-position: inside;
      width: 100%;
      display: -webkit-box;
      display: -ms-flexbox;
      display: flex;
      -webkit-box-orient: horizontal;
      -webkit-box-direction: normal;
      -ms-flex-direction: row;
      flex-direction: row;
      margin-top: 26px;
      border-bottom: 1px solid rgba(233,233,233,0.6);
    }
    ._2sjJGcOH_0 ._3FLYR4bF_0 {
      width: 34px;
      height: 34px;
      -ms-flex-negative: 0;
      flex-shrink: 0;
      border-radius: 50%;
    }
    ._2sjJGcOH_0 ._36ChpWj4_0 {
      margin-left: 0.5rem;
      -webkit-box-flex: 1;
      -ms-flex-positive: 1;
      flex-grow: 1;
      padding-bottom: 20px;
    }
    ._2sjJGcOH_0 ._36ChpWj4_0 ._2zFoi7sd_0 {
      font-size: 16px;
      color: #3d464d;
      font-weight: 500;
      -webkit-font-smoothing: antialiased;
      line-height: 34px;
    }
    ._2sjJGcOH_0 ._36ChpWj4_0 ._2_QraFYR_0 {
      margin-top: 12px;
      color: #505050;
      -webkit-font-smoothing: antialiased;
      font-size: 14px;
      font-weight: 400;
      white-space: normal;
      word-break: break-all;
      line-height: 24px;
    }
    ._2sjJGcOH_0 ._10o3OAxT_0 {
      margin-top: 18px;
      border-radius: 4px;
      background-color: #f6f7fb;
    }
    ._2sjJGcOH_0 ._3klNVc4Z_0 {
      display: -webkit-box;
      display: -ms-flexbox;
      display: flex;
      -webkit-box-orient: horizontal;
      -webkit-box-direction: normal;
      -ms-flex-direction: row;
      flex-direction: row;
      -webkit-box-pack: justify;
      -ms-flex-pack: justify;
      justify-content: space-between;
      -webkit-box-align: center;
      -ms-flex-align: center;
      align-items: center;
      margin-top: 15px;
    }
    ._2sjJGcOH_0 ._10o3OAxT_0 ._3KxQPN3V_0 {
      color: #505050;
      -webkit-font-smoothing: antialiased;
      font-size: 14px;
      font-weight: 400;
      white-space: normal;
      word-break: break-word;
      padding: 20px 20px 20px 24px;
    }
    ._2sjJGcOH_0 ._3klNVc4Z_0 {
      display: -webkit-box;
      display: -ms-flexbox;
      display: flex;
      -webkit-box-orient: horizontal;
      -webkit-box-direction: normal;
      -ms-flex-direction: row;
      flex-direction: row;
      -webkit-box-pack: justify;
      -ms-flex-pack: justify;
      justify-content: space-between;
      -webkit-box-align: center;
      -ms-flex-align: center;
      align-items: center;
      margin-top: 15px;
    }
    ._2sjJGcOH_0 ._3Hkula0k_0 {
      color: #b2b2b2;
      font-size: 14px;
    }
</style><ul><li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0a/da/dcf8f2b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qiutian</span>
  </div>
  <div class="_2_QraFYR_0">这篇感觉有点乱，一会是Redis的代码，一会又插入了Cache的代码，能看懂，但不是很清晰</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-17 00:08:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7b/36/fd46331c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jussi Lee</span>
  </div>
  <div class="_2_QraFYR_0">redis 关闭链接的相关逻辑，好像没有看到</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-28 18:34:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/6b/23/ddad5282.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Aaron</span>
  </div>
  <div class="_2_QraFYR_0">SetObj方法的说明【GetObj 获取某个key对应的对象, 对象必须实现 https:&#47;&#47;pkg.go.dev&#47;encoding#BinaryUnMarshaler】， 我并没有这样实现， 我直接以json.Marshal，转成字符串的形式进行存储。get的时候，再json.Unmarshal进行转换，转换成结构体。另外，我实现的时候，设置过期时间的时候， 0代表永久存储， 在V8(v8.11.4, GO是1.17) 的源码里也能看到说明。【Zero expiration means the key has no expiration time.】</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: json.Marshal 的方式也是可以的, 本质都是序列化和反序列化。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-08 10:34:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e2/52/56dbb738.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>牙小木</span>
  </div>
  <div class="_2_QraFYR_0">层次描述清晰，感觉缺了点层次图。字不如图哦</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: 哎是考虑到定义协议的时候容易乱，加了个图。还有哪里你觉得加个图更合适的吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-02 10:33:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/70/67/0c1359c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qinsi</span>
  </div>
  <div class="_2_QraFYR_0">看上去封装的redis服务只会返回go-redis实现的redis client，那么定义redis服务的接口似乎就不是很必要，因为不会再有其他的redis服务的实现了。文中使用redis服务是为了实现缓存服务，那么直接用go-redis实现就好了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，但是服务层毕竟封装了实例化的方法，这样至少把配置&#47;日志的逻辑都封装了，还是有一定必要的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-21 21:41:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f1/ed/4e249c6b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vincent</span>
  </div>
  <div class="_2_QraFYR_0">以redis为基础，最大限度扩展cache的抽象，学习了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，cache不只是redis</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-20 22:23:14</div>
  </div>
</div>
</div>
</li>
</ul>