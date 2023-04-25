<audio title="26｜GORM（下）：数据库的使用必不可少" src="https://static001.geekbang.org/resource/audio/a9/d4/a9dbf1b9437f4c8bfe9140c622fba0d4.mp3" controls="controls"></audio> 
<p>你好，我是轩脉刃。</p><p>上一节课，我们梳理了Gorm的核心逻辑，也通过思维导图，详细分析了Gorm的源码搞清楚它是如何封装database/sql的。这节课我们就要思考和操作，如何将Gorm融合进入hade框架了。</p><p>Gorm的使用分为两个部分，数据库的连接和数据库的操作。</p><p>对于数据库操作接口的封装，Gorm已经做的非常好了，它在gorm.DB中定义了非常多的对数据库的操作接口，这些接口已经是非常易用了，而且每个操作接口在<a href="https://gorm.io/docs/">官方文档</a>中都有对应的说明和使用教程。比如在DB的操作接口列表中，我们可以看到常用的增删改查的逻辑：</p><pre><code class="language-go">func (db *DB) Create(value interface{}) (tx *DB)

func (db *DB) Delete(value interface{}, conds ...interface{}) (tx *DB)

func (db *DB) Get(key string) (interface{}, bool)

func (db *DB) Update(column string, value interface{}) (tx *DB)
</code></pre><p>同时，<a href="https://gorm.io/docs/">官方首页</a>的例子也把获取到DB后的增删改查操作显示很清楚了，建议你在浏览器收藏这个Gorm的说明文档，因为在具体的应用开发中，你会经常参考使用它的。</p><!-- [[[read_end]]] --><p>所以今天我们要做的事情，就是封装Gorm的数据库连接部分。</p><h2>ORM服务</h2><p>按照“一切皆服务”的思想，我们也计划将Gorm封装为一个服务。而服务三要素是服务接口、服务提供者、服务实例化。我们先来定义ORM服务接口。</p><h3>服务接口</h3><p>这个服务接口并不复杂，它的唯一任务就是能够初始化出gorm.DB 实例。回顾上节课说的Gorm初始化gorm.DB的方法：</p><pre><code class="language-go">  dsn := "xxxxxxx"
  db, err := gorm.Open(mysql.Open(dsn), &amp;gorm.Config{})
</code></pre><p>参数看起来就这两个部分DSN和gorm.Config。</p><p>不过我们希望设计一个hade框架自定义的配置结构，将所有创建连接需要的配置项整合起来。所以除了DSN和gorm.Config这两个配置项，其实还需要加上连接池的配置，就是上节课说的database/sql中提供的对连接池的配置信息。再回顾一下这四个影响底层创建连接池设置的配置信息：</p><pre><code class="language-go">// 设置连接的最大空闲时长
func (db *DB) SetConnMaxIdleTime(d time.Duration)
// 设置连接的最大生命时长
func (db *DB) SetConnMaxLifetime(d time.Duration)
// 设置最大空闲连接数
func (db *DB) SetMaxIdleConns(n int)
// 设置最大打开连接数
func (db *DB) SetMaxOpenConns(n int)
</code></pre><p><strong>所以可以定义这么一个DBConfig结构，将所有的创建DB相关的配置都放在这里面</strong>。代码在framework/contract/orm.go中：</p><pre><code class="language-go">// DBConfig 代表数据库连接的所有配置
type DBConfig struct {
   // 以下配置关于dsn
   WriteTimeout string `yaml:"write_timeout"` // 写超时时间
   Loc          string `yaml:"loc"`           // 时区
   Port         int    `yaml:"port"`          // 端口
   ReadTimeout  string `yaml:"read_timeout"`  // 读超时时间
   Charset      string `yaml:"charset"`       // 字符集
   ParseTime    bool   `yaml:"parse_time"`    // 是否解析时间
   Protocol     string `yaml:"protocol"`      // 传输协议
   Dsn          string `yaml:"dsn"`           // 直接传递dsn，如果传递了，其他关于dsn的配置均无效
   Database     string `yaml:"database"`      // 数据库
   Collation    string `yaml:"collation"`     // 字符序
   Timeout      string `yaml:"timeout"`       // 连接超时时间
   Username     string `yaml:"username"`      // 用户名
   Password     string `yaml:"password"`      // 密码
   Driver       string `yaml:"driver"`        // 驱动
   Host         string `yaml:"host"`          // 数据库地址

   // 以下配置关于连接池
   ConnMaxIdle     int    `yaml:"conn_max_idle"`     // 最大空闲连接数
   ConnMaxOpen     int    `yaml:"conn_max_open"`     // 最大连接数
   ConnMaxLifetime string `yaml:"conn_max_lifetime"` // 连接最大生命周期
   ConnMaxIdletime string `yaml:"conn_max_idletime"` // 空闲最大生命周期

   // 以下配置关于gorm
   *gorm.Config // 集成gorm的配置
}
</code></pre><p>其中DSN是一个复杂的字符串。但我们又不希望使用者直接设置这些复杂字符串来进行传递，所以这里设置了多个字段来生成这个DSN。</p><p>另外上节课也说过，DSN并没有一个标准的格式约定，不同的数据库可能有不同的解析，所以也同时保留直接设置DSN的权限，如果用户手动设置了Dsn字段，那么其他关于Dsn的字段设置均无效。</p><p>所以这里同时需要实现一个方法，使用DBConfig来生成最终使用的字符串Dsn，使用上节课介绍的 github.com/go-sql-driver/mysql 库，就能很方便地实现了。我们继续写：</p><pre><code class="language-go">import (
   "github.com/go-sql-driver/mysql"
   ...
)

// FormatDsn 生成dsn
func (conf *DBConfig) FormatDsn() (string, error) {
   port := strconv.Itoa(conf.Port)
   timeout, err := time.ParseDuration(conf.Timeout)
   if err != nil {
      return "", err
   }
   readTimeout, err := time.ParseDuration(conf.ReadTimeout)
   if err != nil {
      return "", err
   }
   writeTimeout, err := time.ParseDuration(conf.WriteTimeout)
   if err != nil {
      return "", err
   }
   location, err := time.LoadLocation(conf.Loc)
   if err != nil {
      return "", err
   }
   driverConf := &amp;mysql.Config{
      User:         conf.Username,
      Passwd:       conf.Password,
      Net:          conf.Protocol,
      Addr:         net.JoinHostPort(conf.Host, port),
      DBName:       conf.Database,
      Collation:    conf.Collation,
      Loc:          location,
      Timeout:      timeout,
      ReadTimeout:  readTimeout,
      WriteTimeout: writeTimeout,
      ParseTime:    conf.ParseTime,
   }
   return driverConf.FormatDSN(), nil
}
</code></pre><p>可以看到Gorm配置，我们使用结构嵌套的方式，将gorm.Config直接嵌套进入DBConfig中。你可以琢磨下这种写法，它有两个好处。</p><p><strong>一是可以直接设置DBConfig来设置gorm.Config</strong>。比如这个函数是可行的，它直接设置config.DryRun，就是直接设置gorm.Config：</p><pre><code class="language-go">func(container framework.Container, config *contract.DBConfig) error {
   config.DryRun = true
   return nil
}
</code></pre><p><strong>二是DBConfig继承了*gorm.Config的所有方法</strong>。比如这段代码，我们来理解一下：</p><pre><code class="language-go">config := &amp;contract.DBConfig{}
db, err = gorm.Open(mysql.Open(config.Dsn), config)
</code></pre><p>还记得gorm.Open的第二个参数是Option么，它是一个接口，需要实现Apply和AfterInitialize方法，而我们的DBConfig并没有显式实现这两个方法。但是它嵌套了实现了这两个方法的*gorm.Config，所以，默认DB.Config也就实现了这两个方法。</p><pre><code class="language-go">type Option interface {
   Apply(*Config) error
   AfterInitialize(*DB) error
}
</code></pre><p>现在，gorm.Open的两个参数DSN和gorm.Config都封装在DBConfig中，而修改DBConfig的方法，我们封装为DBOption。</p><p>如何让设置DBOption的方法更为优雅呢？这里就使用到上节课刚学到的Option可变参数的编程方法了。定义一个DBOption的结构，它代表一个可以对DBConfig进行设置的方法，这个结构作为获取ORM服务GetDB方法的参数。在framework/contract/orm.go中：</p><pre><code class="language-go">package contract

// ORMKey 代表 ORM的服务
const ORMKey = "hade:orm"

// ORMService 表示传入的参数
type ORMService interface {
   GetDB(option ...DBOption) (*gorm.DB, error)
}

// DBOption 代表初始化的时候的选项
type DBOption func(container framework.Container, config *DBConfig) error
</code></pre><p>这样就能通过设置不同的方法来对DBConfig进行配置。</p><p>比如要设置DBConfig中gorm.Config的DryRun空跑字段，设计了这么一个方法在framework/provider/orm/config.go中：</p><pre><code class="language-go">// WithDryRun 设置空跑模式
func WithDryRun() contract.DBOption {
   return func(container framework.Container, config *contract.DBConfig) error {
      config.DryRun = true
      return nil
   }
}
</code></pre><p>之后，在使用ORM服务的时候，我们就可以这样设置：</p><pre><code class="language-go">gormService := c.MustMake(contract.ORMKey).(contract.ORMService)
// 可变参数为WithDryRun()
db, err := gormService.GetDB(orm.WithDryRun())
</code></pre><h3>服务提供者</h3><p>下一步来完成服务提供者，我们也并不需要过于复杂的设计，只要注意一下两点：</p><ul>
<li>ORM服务一定是要延迟加载的，因为这个服务并不是一个基础服务。如果设置为非延迟加载，在框架启动的时候就会去建立这个服务，这并不是我们想要的。所以我们设计ORM的provider的时候，需要将IsDefer函数设置为true。</li>
<li>第二点考虑到我们后续会使用container中的配置服务，来创建具体的gorm.DB实例，传递一个container是必要的。</li>
</ul><p>所以具体的服务提供者代码如下，在framework/provider/orm/provider.go中：</p><pre><code class="language-go">package orm

import (
   "github.com/gohade/hade/framework"
   "github.com/gohade/hade/framework/contract"
)

// GormProvider 提供App的具体实现方法
type GormProvider struct {
}

// Register 注册方法
func (h *GormProvider) Register(container framework.Container) framework.NewInstance {
   return NewHadeGorm
}

// Boot 启动调用
func (h *GormProvider) Boot(container framework.Container) error {
   return nil
}

// IsDefer 是否延迟初始化
func (h *GormProvider) IsDefer() bool {
   return true
}

// Params 获取初始化参数
func (h *GormProvider) Params(container framework.Container) []interface{} {
   return []interface{}{container}
}

// Name 获取字符串凭证
func (h *GormProvider) Name() string {
   return contract.ORMKey
}
</code></pre><h2>服务实例化</h2><p>服务实例化是今天的重点内容，我们先把Gorm的配置结构和日志结构的准备工作完成，再写稍微复杂一点的具体ORM服务的实例 HadeGorm。</p><h3>配置</h3><p>前面定义了hade框架专属的DBConfig配置结构，如何设置它是一个需要讲究的问题。</p><p>虽然已经设计了一种修改配置文件的方式，就是通过GetDB中的Option参数来设置。但是每个字段都这么设置又非常麻烦，我们自然会想到使用配置文件来配置这个结构。另外如果要连接多个数据库，每个数据库都进行同样的配置，还是颇为麻烦，是不是可以有个默认配置呢？</p><p>于是我们的配置文件可以这样设计：在 database.yaml 中保存数据库的默认值，如果想对某个数据库连接有单独的配置，可以用内嵌yaml结构的方式来进行配置。看下面这个配置例子：</p><pre><code class="language-go">conn_max_idle: 10 # 通用配置，连接池最大空闲连接数
conn_max_open: 100 # 通用配置，连接池最大连接数
conn_max_lifetime: 1h # 通用配置，连接数最大生命周期
protocol: tcp # 通用配置，传输协议
loc: Local # 通用配置，时区

default:
    driver: mysql # 连接驱动
    dsn: "" # dsn，如果设置了dsn, 以下的所有设置都不生效
    host: localhost # ip地址
    port: 3306 # 端口
    database: coredemo # 数据库
    username: jianfengye # 用户名
    password: "123456789" # 密码
    charset: utf8mb4 # 字符集
    collation: utf8mb4_unicode_ci # 字符序
    timeout: 10s # 连接超时
    read_timeout: 2s # 读超时
    write_timeout: 2s # 写超时
    parse_time: true # 是否解析时间
    protocol: tcp # 传输协议
    loc: Local # 时区
    conn_max_idle: 10 # 连接池最大空闲连接数
    conn_max_open: 20 # 连接池最大连接数
    conn_max_lifetime: 1h # 连接数最大生命周期

read:
    driver: mysql # 连接驱动
    dsn: "" # dsn，如果设置了dsn, 以下的所有设置都不生效
    host: localhost # ip地址
    port: 3306 # 端口
    database: coredemo # 数据库
    username: jianfengye # 用户名
    password: "123456789" # 密码
    charset: utf8mb4 # 字符集
    collation: utf8mb4_unicode_ci # 字符序
</code></pre><p>在这个database.yaml中，我们配置了database.default和database.read两个数据源。database.read数据源，并没有设置诸如时区loc、连接池conn_max_open配置，这些缺省的配置要从databse.yaml的根结构中获取。</p><p>要实现这个也并不难，先在framework/provider/orm/service.go中实现一个GetBaseConfig方法，来读取database.yaml根目录的结构：</p><pre><code class="language-go">// GetBaseConfig 读取database.yaml根目录结构
func GetBaseConfig(c framework.Container) *contract.DBConfig {

   configService := c.MustMake(contract.ConfigKey).(contract.Config)
   logService := c.MustMake(contract.LogKey).(contract.Log)
   
   config := &amp;contract.DBConfig{}
   // 直接使用配置服务的load方法读取,yaml文件
   err := configService.Load("database", config)
   if err != nil {
      // 直接使用logService来打印错误信息
      logService.Error(context.Background(), "parse database config error", nil)
      return nil
   }
   return config
}

</code></pre><p>然后设计一个根据配置路径加载某个配置结构的方法。这里这个方法一定是在具体初始化某个DB实例的时候使用到，所以要封装为一个Option结构，写在framework/provider/orm/config.go中：</p><pre><code class="language-go">// WithConfigPath 加载配置文件地址
func WithConfigPath(configPath string) contract.DBOption {
   return func(container framework.Container, config *contract.DBConfig) error {
      configService := container.MustMake(contract.ConfigKey).(contract.Config)
        // 加载configPath配置路径
      if err := configService.Load(configPath, config); err != nil {
         return err
      }
      return nil
   }
}
</code></pre><p>现在，对于使用者来说，要初始化一个配置路径为database.default的数据库，就可以这么使用：</p><pre><code class="language-go">gormService := c.MustMake(contract.ORMKey).(contract.ORMService)
db, err := gormService.GetDB(orm.WithConfigPath("database.default"), orm.WithDryRun())
</code></pre><h3>日志</h3><p>配置项设计清楚了，我们再来思考下日志这块。上一章介绍过了，Gorm是有自己的输出规范的，在初始化参数 gorm.Config 中定义了一个日志输出接口Interface。我们来仔细看下这个接口的定义：</p><pre><code class="language-go">const (
   Silent LogLevel = iota + 1
   Error
   Warn
   Info
)

// Interface logger interface
type Interface interface {
   LogMode(LogLevel) Interface // 日志级别
   Info(context.Context, string, ...interface{})
   Warn(context.Context, string, ...interface{})
   Error(context.Context, string, ...interface{})
   Trace(ctx context.Context, begin time.Time, fc func() (sql string, rowsAffected int64), err error)
}
</code></pre><p>Gorm接口的日志级别分类比较简单：Info、Warn、Error、Trace。恰巧，这几个日志级别都在我们hade框架定义的7个日志级别中，所以完全可以将Gorm的这几个级别，映射到hade的日志级别中。也就是说，Gorm打印的Info级别日志输出到hade的Info日志中、error日志输出到hade的error日志中。</p><p>至于Gorm提供的一个LogMode来调整日志级别，由于我们的hade框架已经可以通过配置进行日志级别设置了，所以LogMode函数对我们来说是没有什么意义的。</p><p>好，了解Gorm的日志接口之后，我们明确了接下来要做的事情：<strong>实现一个Gorm的日志实现类，但是这个日志实现类中的每个方法都用 hade 的日志服务来实现</strong>。</p><p>我们在framework/provider/orm/logger.go中定义一个OrmLogger结构，它带有一个logger属性，这个logger属性存放的是hade容器中的log服务：</p><pre><code class="language-go">// OrmLogger orm的日志实现类, 实现了gorm.Logger.Interface
type OrmLogger struct {
   logger contract.Log // 有一个logger对象存放hade的log服务
}

// NewOrmLogger 初始化一个ormLogger,
func NewOrmLogger(logger contract.Log) *OrmLogger {
   return &amp;OrmLogger{logger: logger}
}
</code></pre><p>它实现了Gorm的Logger.Interface 接口。其中LogMode什么都不做，Info、Error、Warn、Trace 分别对应hade容器中log服务的Info、Error、Warn、Trace方法：</p><pre><code class="language-go">// Info 对接hade的info输出
func (o *OrmLogger) Info(ctx context.Context, s string, i ...interface{}) {
   fields := map[string]interface{}{
      "fields": i,
   }
   o.logger.Info(ctx, s, fields)
}

// Warn 对接hade的Warn输出
func (o *OrmLogger) Warn(ctx context.Context, s string, i ...interface{}) {
   fields := map[string]interface{}{
      "fields": i,
   }
   o.logger.Warn(ctx, s, fields)
}

// Error 对接hade的Error输出
func (o *OrmLogger) Error(ctx context.Context, s string, i ...interface{}) {
   fields := map[string]interface{}{
      "fields": i,
   }
   o.logger.Error(ctx, s, fields)
}

// Trace 对接hade的Trace输出
func (o *OrmLogger) Trace(ctx context.Context, begin time.Time, fc func() (sql string, rowsAffected int64), err error) {
   sql, rows := fc()
   elapsed := time.Since(begin)
   fields := map[string]interface{}{
      "begin": begin,
      "error": err,
      "sql":   sql,
      "rows":  rows,
      "time":  elapsed,
   }

   s := "orm trace sql"
   o.logger.Trace(ctx, s, fields)
}
</code></pre><p>这里稍微注意下Trace方法，Gorm的Trace方法的参数中有传递时间戳begin，这个时间戳代表SQL执行的开始时间，而在函数中使用time.Now获取到当前时间之后，两个相减，我们可以获取到这个SQL的实际执行时间，然后作为hade 日志服务的fields map的一个字段输出。除了Trace，其他几个基本上简单封装hade的日志服务方法就好了。</p><h3>服务实例</h3><p>好了，到现在Gorm的配置结构和日志结构也完成了。万事俱备，下面我们就开始写具体的ORM服务的实例 HadeGorm，在framework/provider/orm/service.go中。</p><p>首先，定义实现contract.ORMService的结构HadeGorm。要明确一点，我们会使用这个结构来生成不同数据库的gorm.DB结构，<strong>所以这个HadeGorm是一个与某个数据库设置无关的结构，而且它应该对单个数据库是一个单例模式</strong>，即在一个服务中，我从HadeGorm两次获取到的default数据库的gorm.DB是同一个。</p><p>设置HadeGrom结构如下：</p><pre><code class="language-go">// HadeGorm 代表hade框架的orm实现
type HadeGorm struct {
   container framework.Container // 服务容器
   dbs       map[string]*gorm.DB // key为dsn, value为gorm.DB（连接池）

   lock *sync.RWMutex
}
</code></pre><p>dbs就是为了单例存在，它的key直接设计为一个string，也就是连接数据库的DSN字符串，而value就是gorm.DB结构。</p><p>这样我们在拿到一个DSN的时候，从这个map中就能判断出是否已经实例化过这个数据库对应的gorm.DB了；如果没有实例化过，就实例化一个gorm.DB，并且将这个实例挂到这个map中。<strong>不过这个逻辑会对dbs有并发修改操作，所以这里要使用一个读写锁来锁住这个dbs的修改</strong>。</p><p>对应实例化HadeGorm的方法为NewHadeGorm，它的具体实现就是初始化HadeGorm中的每个字段。继续写入这段：</p><pre><code class="language-go">// NewHadeGorm 代表实例化Gorm
func NewHadeGorm(params ...interface{}) (interface{}, error) {
   container := params[0].(framework.Container)
   dbs := make(map[string]*gorm.DB)
   lock := &amp;sync.RWMutex{}
   return &amp;HadeGorm{
      container: container,
      dbs:       dbs,
      lock:      lock,
   }, nil
}
</code></pre><p>重头戏在GetDB方法的实现上。</p><p>首先初始化orm.Config，其中包括从配置中获取设置项，也包括初始化内部的Gorm；然后将GetDB的option参数作用于初始化的orm.Config，修改默认配置；通过orm.Config生成DSN字符串。</p><pre><code class="language-go">  // 读取默认配置
    config := GetBaseConfig(app.container)

    logService := app.container.MustMake(contract.LogKey).(contract.Log)

    // 设置Logger
    ormLogger := NewOrmLogger(logService)
    config.Config = &amp;gorm.Config{
        Logger: ormLogger,
    }

    // option对opt进行修改
    for _, opt := range option {
        if err := opt(app.container, config); err != nil {
            return nil, err
        }
    }
</code></pre><p><strong>之后根据dsn字符串判断数据库实例gorm.DB是否已经存在了</strong>。如果存在直接返回gorm.DB，如果不存在需要实例化gorm.DB，这一步逻辑稍微复杂一点：</p><ul>
<li>根据配置项orm.Config中的不同驱动，来实例化gorm.DB（支持MySQL/Postgres/SQLite/SQL Server/ClickHouse）</li>
<li>根据配置项orm.Config中的连接池配置，设置gorm.DB的连接池</li>
<li>将实例化后的gorm.DB和DSN放入map映射中</li>
<li>返回实例化后的gorm.DB</li>
</ul><p>代码如下：</p><pre><code class="language-go">// 如果最终的config没有设置dsn,就生成dsn
    if config.Dsn == "" {
        dsn, err := config.FormatDsn()
        if err != nil {
            return nil, err
        }
        config.Dsn = dsn
    }

    // 判断是否已经实例化了gorm.DB
    app.lock.RLock()
    if db, ok := app.dbs[config.Dsn]; ok {
        app.lock.RUnlock()
        return db, nil
    }
    app.lock.RUnlock()

    // 没有实例化gorm.DB，那么就要进行实例化操作
    app.lock.Lock()
    defer app.lock.Unlock()

    // 实例化gorm.DB
    var db *gorm.DB
    var err error
    switch config.Driver {
    case "mysql":
        db, err = gorm.Open(mysql.Open(config.Dsn), config)
    case "postgres":
        db, err = gorm.Open(postgres.Open(config.Dsn), config)
    case "sqlite":
        db, err = gorm.Open(sqlite.Open(config.Dsn), config)
    case "sqlserver":
        db, err = gorm.Open(sqlserver.Open(config.Dsn), config)
    case "clickhouse":
        db, err = gorm.Open(clickhouse.Open(config.Dsn), config)
    }

    // 设置对应的连接池配置
    sqlDB, err := db.DB()
    if err != nil {
        return db, err
    }

    if config.ConnMaxIdle &gt; 0 {
        sqlDB.SetMaxIdleConns(config.ConnMaxIdle)
    }
    if config.ConnMaxOpen &gt; 0 {
        sqlDB.SetMaxOpenConns(config.ConnMaxOpen)
    }
    if config.ConnMaxLifetime != "" {
        liftTime, err := time.ParseDuration(config.ConnMaxLifetime)
        if err != nil {
            logger.Error(context.Background(), "conn max lift time error", map[string]interface{}{
                "err": err,
            })
        } else {
            sqlDB.SetConnMaxLifetime(liftTime)
        }
    }

    if config.ConnMaxIdletime != "" {
        idleTime, err := time.ParseDuration(config.ConnMaxIdletime)
        if err != nil {
            logger.Error(context.Background(), "conn max idle time error", map[string]interface{}{
                "err": err,
            })
        } else {
            sqlDB.SetConnMaxIdleTime(idleTime)
        }
    }

    // 挂载到map中，结束配置
    if err != nil {
        app.dbs[config.Dsn] = db
    }

    return db, err
</code></pre><p>如果前面的内容都理解了，这段代码实现也没有什么难点了。唯一要注意的地方就是锁的使用，<strong>由于对存在gorm.DB的map是读多写少，所以这里也是使用读写锁</strong>，在读取的时候加了一个读锁，如果map中没有我们要的gorm.DB，先把读锁解开，再加一个写锁，初始化完gorm.DB、保存进入map映射后，再把写锁解开。这样能有效防止对map的并发读写。</p><p>完整的GetDB方法可以参考GitHub上的<a href="https://github.com/gohade/coredemo/blob/geekbang/26/framework/provider/orm/service.go">framework/provider/orm/service.go</a>。</p><p>最后记得去业务代码main.go中，把我们的GormProvider注入服务容器：</p><pre><code class="language-go">func main() {
   // 初始化服务容器
   container := framework.NewHadeContainer()
   ...
   container.Bind(&amp;orm.GormProvider{})

   ...

   // 运行root命令
   console.RunCommand(container)
}
</code></pre><p>整个Gorm就已经结合到hade框架中了。</p><h2>测试</h2><p>下面来做一下测试。我们用真实的MySQL进行测试。当然你需要在本机/远端/Docker搭建一个MySQL，至于怎么搭建，教程网上有很多了，这里就不详细描述。</p><p>我用的是Mac，使用homebrew 能很方便搭建一个MySQL服务。我的MySQL实例搭建在本机的3306端口，并且搭建完成之后，我创建了一个coredemo的database数据库：<br>
<img src="https://static001.geekbang.org/resource/image/e8/68/e81e9f92ea1fe6e27edd3d819f577268.png?wh=1836x1388" alt=""></p><p>所以我的配置文件config/development/database.yaml配置如下：</p><pre><code class="language-go">conn_max_idle: 10 # 通用配置，连接池最大空闲连接数
conn_max_open: 100 # 通用配置，连接池最大连接数
conn_max_lifetime: 1h # 通用配置，连接数最大生命周期
protocol: tcp # 通用配置，传输协议
loc: Local # 通用配置，时区

default:
    driver: mysql # 连接驱动
    dsn: "" # dsn，如果设置了dsn, 以下的所有设置都不生效
    host: localhost # ip地址
    port: 3306 # 端口
    database: coredemo # 数据库
    username: jianfengye # 用户名
    password: "123456789" # 密码
    charset: utf8mb4 # 字符集
    collation: utf8mb4_unicode_ci # 字符序
    timeout: 10s # 连接超时
    read_timeout: 2s # 读超时
    write_timeout: 2s # 写超时
    parse_time: true # 是否解析时间
    protocol: tcp # 传输协议
    loc: Local # 时区
</code></pre><p>我们想在coredemo数据库中增加一个user表，按照Gorm的规范，需要先定义一个数据结构User。在app/http/module/demo/model.go中：</p><pre><code class="language-go">// User is gorm model
type User struct {
   ID           uint
   Name         string
   Email        *string
   Age          uint8
   Birthday     *time.Time
   MemberNumber sql.NullString
   ActivatedAt  sql.NullTime
   CreatedAt    time.Time
   UpdatedAt    time.Time
}
</code></pre><p>然后在应用目录app/http/module/demo/api_orm.go中，定义了一个新的路由方法DemoOrm，在这个方法中，我们先从容器中获取到gorm.DB的实例，然后使用db.AutoMigrate 同步数据表user。</p><p>如果第一次执行的时候，数据库中没有表user，它会自动创建user表，然后分别调用db.Create、db.Save、db.First、db.Delete来对user表进行增删改查操作：</p><pre><code class="language-go">// DemoOrm Orm的路由方法
func (api *DemoApi) DemoOrm(c *gin.Context) {
    logger := c.MustMakeLog()
    logger.Info(c, "request start", nil)

    // 初始化一个orm.DB
    gormService := c.MustMake(contract.ORMKey).(contract.ORMService)
    db, err := gormService.GetDB(orm.WithConfigPath("database.default"))
    if err != nil {
        logger.Error(c, err.Error(), nil)
        c.AbortWithError(50001, err)
        return
    }
    db.WithContext(c)

    // 将User模型创建到数据库中
    err = db.AutoMigrate(&amp;User{})
    if err != nil {
        c.AbortWithError(500, err)
        return
    }
    logger.Info(c, "migrate ok", nil)

    // 插入一条数据
    email := "foo@gmail.com"
    name := "foo"
    age := uint8(25)
    birthday := time.Date(2001, 1, 1, 1, 1, 1, 1, time.Local)
    user := &amp;User{
        Name:         name,
        Email:        &amp;email,
        Age:          age,
        Birthday:     &amp;birthday,
        MemberNumber: sql.NullString{},
        ActivatedAt:  sql.NullTime{},
        CreatedAt:    time.Now(),
        UpdatedAt:    time.Now(),
    }
    err = db.Create(user).Error
    logger.Info(c, "insert user", map[string]interface{}{
        "id":  user.ID,
        "err": err,
    })

    // 更新一条数据
    user.Name = "bar"
    err = db.Save(user).Error
    logger.Info(c, "update user", map[string]interface{}{
        "err": err,
        "id":  user.ID,
    })

    // 查询一条数据
    queryUser := &amp;User{ID: user.ID}

    err = db.First(queryUser).Error
    logger.Info(c, "query user", map[string]interface{}{
        "err":  err,
        "name": queryUser.Name,
    })

    // 删除一条数据
    err = db.Delete(queryUser).Error
    logger.Info(c, "delete user", map[string]interface{}{
        "err": err,
        "id":  user.ID,
    })
    c.JSON(200, "ok")
}
</code></pre><p>记得修改app/http/module/demo/api.go中的路由注册：</p><pre><code class="language-go">func Register(r *gin.Engine) error {
   api := NewDemoApi()
   ...
   r.GET("/demo/orm", api.DemoOrm)
   return nil
}
</code></pre><p>现在，使用 <code>./hade build self</code> 来重新编译hade文件，使用 <code>./hade app start</code> 启动服务，并挂起在控制台，日志会输出到控制台。浏览器调用 <a href="http://localhost:8888/demo/orm">http://localhost:8888/demo/orm</a> ，控制台打印日志如下：<br>
<img src="https://static001.geekbang.org/resource/image/1f/a0/1f7ec4246a851faa3e4f6c9eebfbbda0.png?wh=1920x559" alt=""></p><p>可以清晰地通过trace日志看到底层的Insert/Update/Select/Delete的操作，并且可以通过time字段看到这个请求的具体耗时。到这里Gorm融合hade框架就验证完成了。</p><p>本节课我们主要修改了framework目录下的contract/orm.go 和 provider/orm 目录。目录截图如下，供对比查看，所有代码都已经上传到<a href="https://github.com/gohade/coredemo/tree/geekbang/25">geekbang/25</a>分支了。<br>
<img src="https://static001.geekbang.org/resource/image/07/00/071144ba671c077937f76a9627263000.png?wh=922x1618" alt=""></p><h3>小结</h3><p>对于Gorm这样比较庞大的库，要把Gorm完美集成到hade框架，更好地支持业务对数据库频繁的增删改查操作，我们并不是一开始就动手修改代码，而是先把Gorm的实例化部分的源码都理清楚了，再动手集成才不会出现问题。</p><p>现在我们可以在hade框架中方便获取到gorm.DB了。但是在具体开发业务的时候，如何使用好Gorm来为业务服务，也是一个非常值得花心思研究的课题。好在我们的技术选型是目前Golang业界最火的Gorm，网络上关于如何使用Gorm的课程有非常多了，在具体开发业务的时候，你可以自己参考和研究。</p><h3>思考题</h3><p>在ORM框架中，model层的存放位置一直是个很有争论的话题。比如geekbang/25 分支上model层的User结构，我存放在app/http/module/demo中，有同学会觉得model层放在 app/http/model目录比较好么？具体model是否应该单独作为一个文件夹出来呢？</p><p>欢迎在留言区分享你的思考。感谢你的收听，如果你觉得今天的内容对你有所帮助，也欢迎分享给你身边的朋友，邀请他一起学习。我们下节课见～</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2b/24/3d/0682fdb9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>宁建峰</span>
  </div>
  <div class="_2_QraFYR_0">api_orm.go中，以下代码gin1.8.1已经过过限制了，999&lt;code&gt;100，所以这里使用50001，会发生panic：invalid WriteHeader code 50001<br>if err != nil { <br>   logger.Error(c, err.Error(), nil) <br>   c.AbortWithError(50001, err) <br>   return <br>}<br>gin v1.8.1 源码如下：<br>func checkWriteHeaderCode(code int) {<br>	&#47;&#47; Issue 22880: require valid WriteHeader status codes.<br>	&#47;&#47; For now we only enforce that it&#39;s three digits.<br>	&#47;&#47; In the future we might block things over 599 (600 and above aren&#39;t defined<br>	&#47;&#47; at https:&#47;&#47;httpwg.org&#47;specs&#47;rfc7231.html#status.codes)<br>	&#47;&#47; and we might block under 200 (once we have more mature 1xx support).<br>	&#47;&#47; But for now any three digits.<br>	&#47;&#47;<br>	&#47;&#47; We used to send &quot;HTTP&#47;1.1 000 0&quot; on the wire in responses but there&#39;s<br>	&#47;&#47; no equivalent bogus thing we can realistically send in HTTP&#47;2,<br>	&#47;&#47; so we&#39;ll consistently panic instead and help people find their bugs<br>	&#47;&#47; early. (We can&#39;t return an error from WriteHeader even if we wanted to.)<br>	if code &lt; 100 || code &gt; 999 {<br>		panic(fmt.Sprintf(&quot;invalid WriteHeader code %v&quot;, code))<br>	}<br>}</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-24 16:53:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/72/85/c337e9a1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>老兵</span>
  </div>
  <div class="_2_QraFYR_0">不知道是不是我理解不对，感觉目前gorm在数据库字段的迁移的方案还是不行。比如数据库表加一个字段，删除一个字段，用auto-migrate还是无法做到精准像active_record那样的方便吧？<br>不知道叶老师是否有一些golang下orm+migration的经验？<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-12 21:12:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/23/50/1f5154fe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无笔秀才</span>
  </div>
  <div class="_2_QraFYR_0">我觉得除了orm 还应该支持直接sql,毕竟很多人不喜欢用orm</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-11 16:16:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/b0/d3/200e82ff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>功夫熊猫</span>
  </div>
  <div class="_2_QraFYR_0">我都是直接靠Raw直接写sql语句的。因为有时候不太好定义模型</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 还是orm之争，哈，看来老哥属于极客型</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-19 18:28:45</div>
  </div>
</div>
</div>
</li>
</ul>