<audio title="30 _ ORM：CURD 神器 GORM 包介绍及实战" src="https://static001.geekbang.org/resource/audio/45/6b/4516bc60yya708d0b2155cb61f12be6b.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>在用Go开发项目时，我们免不了要和数据库打交道。每种语言都有优秀的ORM可供选择，在Go中也不例外，比如<a href="https://github.com/go-gorm/gorm">gorm</a>、<a href="https://github.com/go-xorm/xorm">xorm</a>、<a href="https://github.com/gohouse/gorose">gorose</a>等。目前，GitHub上 star数最多的是GORM，它也是当前Go项目中使用最多的ORM。</p><p>IAM项目也使用了GORM。这一讲，我就来详细讲解下GORM的基础知识，并介绍iam-apiserver是如何使用GORM，对数据进行CURD操作的。</p><h2>GORM基础知识介绍</h2><p>GORM是Go语言的ORM包，功能强大，调用方便。像腾讯、华为、阿里这样的大厂，都在使用GORM来构建企业级的应用。GORM有很多特性，开发中常用的核心特性如下：</p><ul>
<li>功能全。使用ORM操作数据库的接口，GORM都有，可以满足我们开发中对数据库调用的各类需求。</li>
<li>支持钩子方法。这些钩子方法可以应用在Create、Save、Update、Delete、Find方法中。</li>
<li>开发者友好，调用方便。</li>
<li>支持Auto Migration。</li>
<li>支持关联查询。</li>
<li>支持多种关系数据库，例如MySQL、Postgres、SQLite、SQLServer等。</li>
</ul><p>GORM有两个版本，<a href="https://github.com/jinzhu/gorm">V1</a>和<a href="https://github.com/go-gorm/gorm">V2</a>。遵循用新不用旧的原则，IAM项目使用了最新的V2版本。</p><!-- [[[read_end]]] --><h2>通过示例学习GORM</h2><p>接下来，我们先快速看一个使用GORM的示例，通过该示例来学习GORM。示例代码存放在<a href="https://github.com/marmotedu/gopractise-demo/blob/main/gorm/main.go">marmotedu/gopractise-demo/gorm/main.go</a>文件中。因为代码比较长，你可以使用以下命令克隆到本地查看：</p><pre><code class="language-bash">$ mkdir -p $GOPATH/src/github.com/marmotedu
$ cd $GOPATH/src/github.com/marmotedu
$ git clone https://github.com/marmotedu/gopractise-demo
$ cd gopractise-demo/gorm/
</code></pre><p>假设我们有一个MySQL数据库，连接地址和端口为 <code>127.0.0.1:3306</code> ，用户名为 <code>iam</code> ，密码为 <code>iam1234</code> 。创建完main.go文件后，执行以下命令来运行：</p><pre><code class="language-bash">$ go run main.go -H 127.0.0.1:3306 -u iam -p iam1234 -d test
2020/10/17 15:15:50 totalcount: 1
2020/10/17 15:15:50 	code: D42, price: 100
2020/10/17 15:15:51 totalcount: 1
2020/10/17 15:15:51 	code: D42, price: 200
2020/10/17 15:15:51 totalcount: 0
</code></pre><p>在企业级Go项目开发中，使用GORM库主要用来完成以下数据库操作：</p><ul>
<li>连接和关闭数据库。连接数据库时，可能需要设置一些参数，比如最大连接数、最大空闲连接数、最大连接时长等。</li>
<li>插入表记录。可以插入一条记录，也可以批量插入记录。</li>
<li>更新表记录。可以更新某一个字段，也可以更新多个字段。</li>
<li>查看表记录。可以查看某一条记录，也可以查看符合条件的记录列表。</li>
<li>删除表记录。可以删除某一个记录，也可以批量删除。删除还支持永久删除和软删除。</li>
<li>在一些小型项目中，还会用到GORM的表结构自动迁移功能。</li>
</ul><p>GORM功能强大，上面的示例代码展示的是比较通用的一种操作方式。</p><p>上述代码中，首先定义了一个GORM模型（Models），Models是标准的Go struct，用来代表数据库中的一个表结构。我们可以给 Models 添加 TableName 方法，来告诉 GORM 该Models映射到数据库中的哪张表。Models定义如下：</p><pre><code class="language-go">type Product struct {
    gorm.Model
    Code  string `gorm:"column:code"`
    Price uint   `gorm:"column:price"`
}

// TableName maps to mysql table name.
func (p *Product) TableName() string {
    return "product"
}
</code></pre><p>如果没有指定表名，则GORM使用结构体名的蛇形复数作为表名。例如：结构体名为 <code>DockerInstance</code> ，则表名为 <code>dockerInstances</code> 。</p><p>在之后的代码中，使用Pflag来解析命令行的参数，通过命令行参数指定数据库的地址、用户名、密码和数据库名。之后，使用这些参数生成建立 MySQL 连接需要的配置文件，并调用 <code>gorm.Open</code> 建立数据库连接：</p><pre><code class="language-go">var (
    host     = pflag.StringP("host", "H", "127.0.0.1:3306", "MySQL service host address")
    username = pflag.StringP("username", "u", "root", "Username for access to mysql service")
    password = pflag.StringP("password", "p", "root", "Password for access to mysql, should be used pair with password")
    database = pflag.StringP("database", "d", "test", "Database name to use")
    help     = pflag.BoolP("help", "h", false, "Print this help message")
)

func main() {
    // Parse command line flags
    pflag.CommandLine.SortFlags = false
    pflag.Usage = func() {
        pflag.PrintDefaults()
    }
    pflag.Parse()
    if *help {
        pflag.Usage()
        return
    }

    dsn := fmt.Sprintf(`%s:%s@tcp(%s)/%s?charset=utf8&amp;parseTime=%t&amp;loc=%s`,
        *username,
        *password,
        *host,
        *database,
        true,
        "Local")
    db, err := gorm.Open(mysql.Open(dsn), &amp;gorm.Config{})
    if err != nil {
        panic("failed to connect database")
    }
}
</code></pre><p>创建完数据库连接之后，会返回数据库实例 <code>db</code> ，之后就可以调用db实例中的方法，完成数据库的CURD操作。具体操作如下，一共可以分为六个操作：</p><p>第一个操作，自动迁移表结构。</p><pre><code class="language-go">// 1. Auto migration for given models
db.AutoMigrate(&amp;Product{})
</code></pre><p><strong>我不建议你在正式的代码中自动迁移表结构。</strong>因为变更现网数据库是一个高危操作，现网数据库字段的添加、类型变更等，都需要经过严格的评估才能实施。这里将变更隐藏在代码中，在组件发布时很难被研发人员感知到，如果组件启动，就可能会自动修改现网表结构，也可能会因此引起重大的现网事故。</p><p>GORM的AutoMigrate方法，只对新增的字段或索引进行变更，理论上是没有风险的。在实际的Go项目开发中，也有很多人使用AutoMigrate方法自动同步表结构。但我更倾向于规范化、可感知的操作方式，所以我在实际开发中，都是手动变更表结构的。当然，具体使用哪种方法，你可以根据需要自行选择。</p><p>第二个操作，插入表记录。</p><pre><code class="language-go">// 2. Insert the value into database
if err := db.Create(&amp;Product{Code: "D42", Price: 100}).Error; err != nil {
    log.Fatalf("Create error: %v", err)
}
PrintProducts(db)
</code></pre><p>通过 <code>db.Create</code> 方法创建了一条记录。插入记录后，通过调用 <code>PrintProducts</code> 方法打印当前表中的所有数据记录，来测试是否成功插入。</p><p>第三个操作，获取符合条件的记录。</p><pre><code class="language-go">// 3. Find first record that match given conditions
product := &amp;Product{}
if err := db.Where("code= ?", "D42").First(&amp;product).Error; err != nil {
    log.Fatalf("Get product error: %v", err)
}
</code></pre><p>First方法只会返回符合条件的记录列表中的第一条，你可以使用First方法来获取某个资源的详细信息。</p><p>第四个操作，更新表记录。</p><pre><code class="language-go">// 4. Update value in database, if the value doesn't have primary key, will insert it
product.Price = 200
if err := db.Save(product).Error; err != nil {
    log.Fatalf("Update product error: %v", err)
}
PrintProducts(db)
</code></pre><p>通过Save方法，可以把 product 变量中所有跟数据库不一致的字段更新到数据库中。具体操作是：先获取某个资源的详细信息，再通过 <code>product.Price = 200</code> 这类赋值语句，对其中的一些字段重新赋值。最后，调用 <code>Save</code> 方法更新这些字段。你可以将这些操作看作一种更新数据库的更新模式。</p><p>第五个操作，删除表记录。</p><p>通过 <code>Delete</code> 方法删除表记录，代码如下：</p><pre><code class="language-go">// 5. Delete value match given conditions
if err := db.Where("code = ?", "D42").Delete(&amp;Product{}).Error; err != nil {
    log.Fatalf("Delete product error: %v", err)
}
PrintProducts(db)
</code></pre><p>这里需要注意，因为 <code>Product</code> 中有 <code>gorm.DeletedAt</code> 字段，所以，上述删除操作不会真正把记录从数据库表中删除掉，而是通过设置数据库 <code>product</code> 表 <code>deletedAt</code> 字段为当前时间的方法来删除。</p><p>第六个操作，获取表记录列表。</p><pre><code class="language-go">products := make([]*Product, 0)
var count int64
d := db.Where("code like ?", "%D%").Offset(0).Limit(2).Order("id desc").Find(&amp;products).Offset(-1).Limit(-1).Count(&amp;count)
if d.Error != nil {
    log.Fatalf("List products error: %v", d.Error)
}
</code></pre><p>在PrintProducts函数中，会打印当前的所有记录，你可以根据输出，判断数据库操作是否成功。</p><h2>GORM常用操作讲解</h2><p>看完上面的示例，我想你已经初步掌握了GORM的使用方法。接下来，我再来给你详细介绍下GORM所支持的数据库操作。</p><h3>模型定义</h3><p>GORM使用模型（Models）来映射一个数据库表。默认情况下，使用ID作为主键，使用结构体名的 <code>snake_cases</code> 作为表名，使用字段名的 <code>snake_case</code> 作为列名，并使用 CreatedAt、UpdatedAt、DeletedAt字段追踪创建、更新和删除时间。</p><p>使用GORM的默认规则，可以减少代码量，但我更喜欢的方式是<strong>直接指明字段名和表名</strong>。例如，有以下模型：</p><pre><code class="language-go">type Animal struct {
  AnimalID int64        // 列名 `animal_id`
  Birthday time.Time    // 列名 `birthday`
  Age      int64        // 列名 `age`
}
</code></pre><p>上述模型对应的表名为 <code>animals</code> ，列名分别为 <code>animal_id</code> 、 <code>birthday</code> 和 <code>age</code> 。我们可以通过以下方式来重命名表名和列名，并将 <code>AnimalID</code> 设置为表的主键：</p><pre><code class="language-go">type Animal struct {
    AnimalID int64     `gorm:"column:animalID;primarykey"` // 将列名设为 `animalID`
    Birthday time.Time `gorm:"column:birthday"`            // 将列名设为 `birthday`
    Age      int64     `gorm:"column:age"`                 // 将列名设为 `age`
}

func (a *Animal) TableName() string {
    return "animal"
}
</code></pre><p>上面的代码中，通过 <code>primaryKey</code> 标签指定主键，使用 <code>column</code> 标签指定列名，通过给Models添加 <code>TableName</code> 方法指定表名。</p><p>数据库表通常会包含4个字段。</p><ul>
<li>ID：自增字段，也作为主键。</li>
<li>CreatedAt：记录创建时间。</li>
<li>UpdatedAt：记录更新时间。</li>
<li>DeletedAt：记录删除时间（软删除时有用）。</li>
</ul><p>GORM也预定义了包含这4个字段的Models，在我们定义自己的Models时，可以直接内嵌到结构体内，例如：</p><pre><code class="language-go">type Animal struct {
    gorm.Model
    AnimalID int64     `gorm:"column:animalID"` // 将列名设为 `animalID`
    Birthday time.Time `gorm:"column:birthday"` // 将列名设为 `birthday`
    Age      int64     `gorm:"column:age"`      // 将列名设为 `age`
}
</code></pre><p>Models中的字段能支持很多GORM标签，但如果我们不使用GORM自动创建表和迁移表结构的功能，很多标签我们实际上是用不到的。在开发中，用得最多的是 <code>column</code> 标签。</p><h3>连接数据库</h3><p>在进行数据库的CURD操作之前，我们首先需要连接数据库。你可以通过以下代码连接MySQL数据库：</p><pre><code class="language-go">import (
  "gorm.io/driver/mysql"
  "gorm.io/gorm"
)

func main() {
  // 参考 https://github.com/go-sql-driver/mysql#dsn-data-source-name 获取详情
  dsn := "user:pass@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&amp;parseTime=True&amp;loc=Local"
  db, err := gorm.Open(mysql.Open(dsn), &amp;gorm.Config{})
}
</code></pre><p>如果需要GORM正确地处理 <code>time.Time</code> 类型，在连接数据库时需要带上 <code>parseTime</code> 参数。如果要支持完整的UTF-8编码，可将<code>charset=utf8</code>更改为<code>charset=utf8mb4</code>。</p><p>GORM支持连接池，底层是用 <code>database/sql</code> 包来维护连接池的，连接池设置如下：</p><pre><code class="language-go">sqlDB, err := db.DB()
sqlDB.SetMaxIdleConns(100)              // 设置MySQL的最大空闲连接数（推荐100）
sqlDB.SetMaxOpenConns(100)             // 设置MySQL的最大连接数（推荐100）
sqlDB.SetConnMaxLifetime(time.Hour)    // 设置MySQL的空闲连接最大存活时间（推荐10s）
</code></pre><p>上面这些设置，也可以应用在大型后端项目中。</p><h3>创建记录</h3><p>我们可以通过 <code>db.Create</code> 方法来创建一条记录：</p><pre><code class="language-go">type User struct {
  gorm.Model
  Name         string
  Age          uint8
  Birthday     *time.Time
}
user := User{Name: "Jinzhu", Age: 18, Birthday: time.Now()}
result := db.Create(&amp;user) // 通过数据的指针来创建
</code></pre><p>db.Create函数会返回如下3个值：</p><ul>
<li>user.ID：返回插入数据的主键，这个是直接赋值给user变量。</li>
<li>result.Error：返回error。</li>
<li>result.RowsAffected：返回插入记录的条数。</li>
</ul><p>当需要插入的数据量比较大时，可以批量插入，以提高插入性能：</p><pre><code class="language-go">var users = []User{{Name: "jinzhu1"}, {Name: "jinzhu2"}, {Name: "jinzhu3"}}
DB.Create(&amp;users)

for _, user := range users {
  user.ID // 1,2,3
}
</code></pre><h3>删除记录</h3><p>我们可以通过Delete方法删除记录：</p><pre><code class="language-go">// DELETE from users where id = 10 AND name = "jinzhu";
db.Where("name = ?", "jinzhu").Delete(&amp;user)
</code></pre><p>GORM也支持根据主键进行删除，例如：</p><pre><code class="language-go">// DELETE FROM users WHERE id = 10;
db.Delete(&amp;User{}, 10)
</code></pre><p>不过，我更喜欢使用db.Where的方式进行删除，这种方式有两个优点。</p><p>第一个优点是删除方式更通用。使用db.Where不仅可以根据主键删除，还能够随意组合条件进行删除。</p><p>第二个优点是删除方式更显式，这意味着更易读。如果使用<code>db.Delete(&amp;User{}, 10)</code>，你还需要确认User的主键，如果记错了主键，还可能会引入Bug。</p><p>此外，GORM也支持批量删除：</p><pre><code class="language-go">db.Where("name in (?)", []string{"jinzhu", "colin"}).Delete(&amp;User{})
</code></pre><p>GORM支持两种删除方法：软删除和永久删除。下面我来分别介绍下。</p><ol>
<li>软删除</li>
</ol><p>软删除是指执行Delete时，记录不会被从数据库中真正删除。GORM会将 <code>DeletedAt</code> 设置为当前时间，并且不能通过正常的方式查询到该记录。如果模型包含了一个 <code>gorm.DeletedAt</code> 字段，GORM在执行删除操作时，会软删除该记录。</p><p>下面的删除方法就是一个软删除：</p><pre><code class="language-go">// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE age = 20;
db.Where("age = ?", 20).Delete(&amp;User{})

// SELECT * FROM users WHERE age = 20 AND deleted_at IS NULL;
db.Where("age = 20").Find(&amp;user)
</code></pre><p>可以看到，GORM并没有真正把记录从数据库删除掉，而是只更新了 <code>deleted_at</code> 字段。在查询时，GORM查询条件中新增了<code>AND deleted_at IS NULL</code>条件，所以这些被设置过 <code>deleted_at</code> 字段的记录不会被查询到。对于一些比较重要的数据，我们可以通过软删除的方式删除记录，软删除可以使这些重要的数据后期能够被恢复，并且便于以后的排障。</p><p>我们可以通过下面的方式查找被软删除的记录：</p><pre><code class="language-go">// SELECT * FROM users WHERE age = 20;
db.Unscoped().Where("age = 20").Find(&amp;users)
</code></pre><ol start="2">
<li>永久删除</li>
</ol><p>如果想永久删除一条记录，可以使用Unscoped：</p><pre><code class="language-go">// DELETE FROM orders WHERE id=10;
db.Unscoped().Delete(&amp;order)
</code></pre><p>或者，你也可以在模型中去掉gorm.DeletedAt。</p><h3>更新记录</h3><p>GORM中，最常用的更新方法如下：</p><pre><code class="language-go">db.First(&amp;user)

user.Name = "jinzhu 2"
user.Age = 100
// UPDATE users SET name='jinzhu 2', age=100, birthday='2016-01-01', updated_at = '2013-11-17 21:34:10' WHERE id=111;
db.Save(&amp;user)
</code></pre><p>上述方法会保留所有字段，所以执行Save时，需要先执行First，获取某个记录的所有列的值，然后再对需要更新的字段设置值。</p><p>还可以指定更新单个列：</p><pre><code class="language-go">// UPDATE users SET age=200, updated_at='2013-11-17 21:34:10' WHERE name='colin';
db.Model(&amp;User{}).Where("name = ?", "colin").Update("age", 200)
</code></pre><p>也可以指定更新多个列：</p><pre><code class="language-go">// UPDATE users SET name='hello', age=18, updated_at = '2013-11-17 21:34:10' WHERE name = 'colin';
db.Model(&amp;user).Where("name", "colin").Updates(User{Name: "hello", Age: 18, Active: false})
</code></pre><p>这里要注意，这个方法只会更新非零值的字段。</p><h3>查询数据</h3><p>GORM支持不同的查询方法，下面我来讲解三种在开发中经常用到的查询方式，分别是检索单个记录、查询所有符合条件的记录和智能选择字段。</p><ol>
<li>检索单个记录</li>
</ol><p>下面是检索单个记录的示例代码：</p><pre><code class="language-go">// 获取第一条记录（主键升序）
// SELECT * FROM users ORDER BY id LIMIT 1;
db.First(&amp;user)

// 获取最后一条记录（主键降序）
// SELECT * FROM users ORDER BY id DESC LIMIT 1;
db.Last(&amp;user)
result := db.First(&amp;user)
result.RowsAffected // 返回找到的记录数
result.Error        // returns error

// 检查 ErrRecordNotFound 错误
errors.Is(result.Error, gorm.ErrRecordNotFound)
</code></pre><p>如果model类型没有定义主键，则按第一个字段排序。</p><ol start="2">
<li>查询所有符合条件的记录</li>
</ol><p>示例代码如下：</p><pre><code class="language-go">users := make([]*User, 0)

// SELECT * FROM users WHERE name &lt;&gt; 'jinzhu';
db.Where("name &lt;&gt; ?", "jinzhu").Find(&amp;users)
</code></pre><ol start="3">
<li>智能选择字段</li>
</ol><p>你可以通过Select方法，选择特定的字段。我们可以定义一个较小的结构体来接受选定的字段：</p><pre><code class="language-go">type APIUser struct {
  ID   uint
  Name string
}

// SELECT `id`, `name` FROM `users` LIMIT 10;
db.Model(&amp;User{}).Limit(10).Find(&amp;APIUser{})
</code></pre><p>除了上面讲的三种常用的基本查询方法，GORM还支持高级查询，下面我来介绍下。</p><h3>高级查询</h3><p>GORM支持很多高级查询功能，这里我主要介绍4种。</p><ol>
<li>指定检索记录时的排序方式</li>
</ol><p>示例代码如下：</p><pre><code class="language-go">// SELECT * FROM users ORDER BY age desc, name;
db.Order("age desc, name").Find(&amp;users)
</code></pre><ol start="2">
<li>Limit &amp; Offset</li>
</ol><p>Offset指定从第几条记录开始查询，Limit指定返回的最大记录数。Offset和Limit值为-1时，消除Offset和Limit条件。另外，Limit和Offset位置不同，效果也不同。</p><pre><code class="language-go">// SELECT * FROM users OFFSET 5 LIMIT 10;
db.Limit(10).Offset(5).Find(&amp;users)
</code></pre><ol start="3">
<li>Distinct</li>
</ol><p>Distinct可以从数据库记录中选择不同的值。</p><pre><code class="language-go">db.Distinct("name", "age").Order("name, age desc").Find(&amp;results)
</code></pre><ol start="4">
<li>Count</li>
</ol><p>Count可以获取匹配的条数。</p><pre><code class="language-go">var count int64
// SELECT count(1) FROM users WHERE name = 'jinzhu'; (count)
db.Model(&amp;User{}).Where("name = ?", "jinzhu").Count(&amp;count)
</code></pre><p>GORM还支持很多高级查询功能，比如内联条件、Not 条件、Or 条件、Group &amp; Having、Joins、Group、FirstOrInit、FirstOrCreate、迭代、FindInBatches等。因为IAM项目中没有用到这些高级特性，我在这里就不展开介绍了。你如果感兴趣，可以看下<a href="https://gorm.io/zh_CN/docs/index.html">GORM的官方文档</a>。</p><h3>原生SQL</h3><p>GORM支持原生查询SQL和执行SQL。原生查询SQL用法如下：</p><pre><code class="language-go">type Result struct {
  ID   int
  Name string
  Age  int
}

var result Result
db.Raw("SELECT id, name, age FROM users WHERE name = ?", 3).Scan(&amp;result)
</code></pre><p>原生执行SQL用法如下；</p><pre><code class="language-go">db.Exec("DROP TABLE users")
db.Exec("UPDATE orders SET shipped_at=? WHERE id IN ?", time.Now(), []int64{1,2,3})
</code></pre><h3>GORM钩子</h3><p>GORM支持钩子功能，例如下面这个在插入记录前执行的钩子：</p><pre><code class="language-go">func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
  u.UUID = uuid.New()

    if u.Name == "admin" {
        return errors.New("invalid name")
    }
    return
}
</code></pre><p>GORM支持的钩子见下表：</p><p><img src="https://static001.geekbang.org/resource/image/20/2c/20fb0b6a11dbcebd9ddf428517240d2c.jpg?wh=1920x1338" alt="图片"></p><h2>iam-apiserver中的CURD操作实战</h2><p>接下来，我来介绍下iam-apiserver是如何使用GORM，对数据进行CURD操作的。</p><p><strong>首先，</strong>我们需要配置连接MySQL的各类参数。iam-apiserver通过<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/pkg/options/mysql_options.go#L29">NewMySQLOptions</a>函数创建了一个带有默认值的<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/pkg/options/mysql_options.go#L17">MySQLOptions</a>类型的变量，将该变量传给<a href="https://github.com/marmotedu/iam/blob/v1.0.4/pkg/app/app.go#L157">NewApp</a>函数。在App框架中，最终会调用MySQLOptions提供的AddFlags方法，将MySQLOptions提供的命令行参数添加到Cobra命令行中。</p><p><strong>接着，</strong>在<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/server.go#L81">PrepareRun</a>函数中，调用<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/store/mysql/mysql.go#L55">GetMySQLFactoryOr</a>函数，初始化并获取仓库层的实例<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/store/mysql/mysql.go#L50">mysqlFactory</a>。实现了仓库层<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/store/store.go#L12">store.Factory</a>接口：</p><pre><code class="language-go">type Factory interface {
    Users() UserStore
    Secrets() SecretStore
    Policies() PolicyStore
    Close() error
}
</code></pre><p>GetMySQLFactoryOr函数采用了我们在 <a href="https://time.geekbang.org/column/article/386238">11讲</a> 中提过的单例模式，确保iam-apiserver进程中只有一个仓库层的实例，这样可以减少内存开支和系统的性能开销。</p><p>GetMySQLFactoryOr函数中，使用<a href="https://github.com/marmotedu/iam/blob/v1.0.4/pkg/db/mysql.go#L30">github.com/marmotedu/iam/pkg/db</a>包提供的New函数，创建了MySQL实例。New函数代码如下：</p><pre><code class="language-go">func New(opts *Options) (*gorm.DB, error) {&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; dsn := fmt.Sprintf(`%s:%s@tcp(%s)/%s?charset=utf8&amp;parseTime=%t&amp;loc=%s`,&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; opts.Username,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; opts.Password,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; opts.Host,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; opts.Database,&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; true,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; "Local")&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; db, err := gorm.Open(mysql.Open(dsn), &amp;gorm.Config{&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; Logger: logger.New(opts.LogLevel),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; })&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; if err != nil {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; return nil, err&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; sqlDB, err := db.DB()&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; if err != nil {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; return nil, err&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; // SetMaxOpenConns sets the maximum number of open connections to the database.
&nbsp; &nbsp; sqlDB.SetMaxOpenConns(opts.MaxOpenConnections)

&nbsp; &nbsp; // SetConnMaxLifetime sets the maximum amount of time a connection may be reused.
&nbsp; &nbsp; sqlDB.SetConnMaxLifetime(opts.MaxConnectionLifeTime)

&nbsp; &nbsp; // SetMaxIdleConns sets the maximum number of connections in the idle connection pool.
&nbsp; &nbsp; sqlDB.SetMaxIdleConns(opts.MaxIdleConnections)

&nbsp; &nbsp; return db, nil
}
</code></pre><p>上述代码中，我们先创建了一个 <code>*gorm.DB</code> 类型的实例，并对该实例进行了如下设置：</p><ul>
<li>通过SetMaxOpenConns方法，设置了MySQL的最大连接数（推荐100）。</li>
<li>通过SetConnMaxLifetime方法，设置了MySQL的空闲连接最大存活时间（推荐10s）。</li>
<li>通过SetMaxIdleConns方法，设置了MySQL的最大空闲连接数（推荐100）。</li>
</ul><p>GetMySQLFactoryOr函数最后创建了datastore类型的变量mysqlFactory，该变量是仓库层的变量。mysqlFactory变量中，又包含了 <code>*gorm.DB</code> 类型的字段 <code>db</code> 。</p><p><strong>最终</strong><strong>，</strong>我们通过仓库层的变量mysqlFactory，调用其 <code>db</code> 字段提供的方法来完成数据库的CURD操作。例如，创建密钥、更新密钥、删除密钥、获取密钥详情、查询密钥列表，具体代码如下（代码位于<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/store/mysql/secret.go">secret.go</a>文件中）：</p><pre><code class="language-go">// Create creates a new secret.
func (s *secrets) Create(ctx context.Context, secret *v1.Secret, opts metav1.CreateOptions) error {
	return s.db.Create(&amp;secret).Error
}

// Update updates an secret information by the secret identifier.
func (s *secrets) Update(ctx context.Context, secret *v1.Secret, opts metav1.UpdateOptions) error {
	return s.db.Save(secret).Error
}

// Delete deletes the secret by the secret identifier.
func (s *secrets) Delete(ctx context.Context, username, name string, opts metav1.DeleteOptions) error {
	if opts.Unscoped {
		s.db = s.db.Unscoped()
	}

	err := s.db.Where("username = ? and name = ?", username, name).Delete(&amp;v1.Secret{}).Error
	if err != nil &amp;&amp; !errors.Is(err, gorm.ErrRecordNotFound) {
		return errors.WithCode(code.ErrDatabase, err.Error())
	}

	return nil
}

// Get return an secret by the secret identifier.
func (s *secrets) Get(ctx context.Context, username, name string, opts metav1.GetOptions) (*v1.Secret, error) {
	secret := &amp;v1.Secret{}
	err := s.db.Where("username = ? and name= ?", username, name).First(&amp;secret).Error
	if err != nil {
		if errors.Is(err, gorm.ErrRecordNotFound) {
			return nil, errors.WithCode(code.ErrSecretNotFound, err.Error())
		}

		return nil, errors.WithCode(code.ErrDatabase, err.Error())
	}

	return secret, nil
}

// List return all secrets.
func (s *secrets) List(ctx context.Context, username string, opts metav1.ListOptions) (*v1.SecretList, error) {
	ret := &amp;v1.SecretList{}
	ol := gormutil.Unpointer(opts.Offset, opts.Limit)

	if username != "" {
		s.db = s.db.Where("username = ?", username)
	}

	selector, _ := fields.ParseSelector(opts.FieldSelector)
	name, _ := selector.RequiresExactMatch("name")

	d := s.db.Where(" name like ?", "%"+name+"%").
		Offset(ol.Offset).
		Limit(ol.Limit).
		Order("id desc").
		Find(&amp;ret.Items).
		Offset(-1).
		Limit(-1).
		Count(&amp;ret.TotalCount)

	return ret, d.Error
}
</code></pre><p>上面的代码中， <code>s.db</code> 就是 <code>*gorm.DB</code> 类型的字段。</p><p>上面的代码段执行了以下操作：</p><ul>
<li>通过 <code>s.db.Save</code> 来更新数据库表的各字段；</li>
<li>通过 <code>s.db.Unscoped</code> 来永久性从表中删除一行记录。对于支持软删除的资源，我们还可以通过 <code>opts.Unscoped</code> 选项来控制是否永久删除记录。 <code>true</code> 永久删除， <code>false</code> 软删除，默认软删除。</li>
<li>通过 <code>errors.Is(err, gorm.ErrRecordNotFound)</code> 来判断GORM返回的错误是否是没有找到记录的错误类型。</li>
<li>通过下面两行代码，来获取查询条件name的值：</li>
</ul><pre><code class="language-go">selector, _ := fields.ParseSelector(opts.FieldSelector)&nbsp; &nbsp;&nbsp;
name, _ := selector.RequiresExactMatch("name")
</code></pre><p>我们的整个调用链是：控制层 -&gt; 业务层 -&gt; 仓库层。这里你可能要问：<strong>我们<strong><strong>是</strong></strong>如何<strong><strong>调用</strong></strong>到<strong><strong>仓库层的</strong></strong>实例mysqlFactory<strong><strong>的</strong></strong>呢？</strong></p><p>这是因为我们的控制层实例包含了业务层的实例。在创建控制层实例时，我们传入了业务层的实例：</p><pre><code class="language-go">type UserController struct {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; srv srvv1.Service&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
}&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
// NewUserController creates a user handler.&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
func NewUserController(store store.Factory) *UserController {
&nbsp; &nbsp; return &amp;UserController{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; srv: srvv1.NewService(store),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
}&nbsp;
</code></pre><p>业务层的实例包含了仓库层的实例。在创建业务层实例时，传入了仓库层的实例：</p><pre><code class="language-go">type service struct {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; store store.Factory&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
}&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
// NewService returns Service interface.&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
func NewService(store store.Factory) Service {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; return &amp;service{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; store: store,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; }
}
</code></pre><p>通过这种包含关系，我们在控制层可以调用业务层的实例，在业务层又可以调用仓库层的实例。这样，我们最终通过仓库层实例的 <code>db</code> 字段（<code>*gorm.DB</code> 类型）完成数据库的CURD操作。</p><h2>总结</h2><p>在Go项目中，我们需要使用ORM来进行数据库的CURD操作。在Go生态中，当前最受欢迎的ORM是GORM，IAM项目也使用了GORM。GORM有很多功能，常用的功能有模型定义、连接数据库、创建记录、删除记录、更新记录和查询数据。这些常用功能的常见使用方式如下：</p><pre><code class="language-go">package main

import (
	"fmt"
	"log"

	"github.com/spf13/pflag"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

type Product struct {
	gorm.Model
	Code&nbsp; string `gorm:"column:code"`
	Price uint&nbsp; &nbsp;`gorm:"column:price"`
}

// TableName maps to mysql table name.
func (p *Product) TableName() string {
	return "product"
}

var (
	host&nbsp; &nbsp; &nbsp;= pflag.StringP("host", "H", "127.0.0.1:3306", "MySQL service host address")
	username = pflag.StringP("username", "u", "root", "Username for access to mysql service")
	password = pflag.StringP("password", "p", "root", "Password for access to mysql, should be used pair with password")
	database = pflag.StringP("database", "d", "test", "Database name to use")
	help&nbsp; &nbsp; &nbsp;= pflag.BoolP("help", "h", false, "Print this help message")
)

func main() {
	// Parse command line flags
	pflag.CommandLine.SortFlags = false
	pflag.Usage = func() {
		pflag.PrintDefaults()
	}
	pflag.Parse()
	if *help {
		pflag.Usage()
		return
	}

	dsn := fmt.Sprintf(`%s:%s@tcp(%s)/%s?charset=utf8&amp;parseTime=%t&amp;loc=%s`,
		*username,
		*password,
		*host,
		*database,
		true,
		"Local")
	db, err := gorm.Open(mysql.Open(dsn), &amp;gorm.Config{})
	if err != nil {
		panic("failed to connect database")
	}

	// 1. Auto migration for given models
	db.AutoMigrate(&amp;Product{})

	// 2. Insert the value into database
	if err := db.Create(&amp;Product{Code: "D42", Price: 100}).Error; err != nil {
		log.Fatalf("Create error: %v", err)
	}
	PrintProducts(db)

	// 3. Find first record that match given conditions
	product := &amp;Product{}
	if err := db.Where("code= ?", "D42").First(&amp;product).Error; err != nil {
		log.Fatalf("Get product error: %v", err)
	}

	// 4. Update value in database, if the value doesn't have primary key, will insert it
	product.Price = 200
	if err := db.Save(product).Error; err != nil {
		log.Fatalf("Update product error: %v", err)
	}
	PrintProducts(db)

	// 5. Delete value match given conditions
	if err := db.Where("code = ?", "D42").Delete(&amp;Product{}).Error; err != nil {
		log.Fatalf("Delete product error: %v", err)
	}
	PrintProducts(db)
}

// List products
func PrintProducts(db *gorm.DB) {
	products := make([]*Product, 0)
	var count int64
	d := db.Where("code like ?", "%D%").Offset(0).Limit(2).Order("id desc").Find(&amp;products).Offset(-1).Limit(-1).Count(&amp;count)
	if d.Error != nil {
		log.Fatalf("List products error: %v", d.Error)
	}

	log.Printf("totalcount: %d", count)
	for _, product := range products {
		log.Printf("\tcode: %s, price: %d\n", product.Code, product.Price)
	}
}
</code></pre><p>此外，GORM还支持原生查询SQL和原生执行SQL，可以满足更加复杂的SQL场景。</p><p>GORM中，还有一个非常有用的功能是支持Hooks。Hooks可以在执行某个CURD操作前被调用。在Hook中，可以添加一些非常有用的功能，例如生成唯一ID。目前，GORM支持 <code>BeforeXXX</code> 、 <code>AfterXXX</code> 和 <code>AfterFind</code> Hook，其中 <code>XXX</code> 可以是 Save、Create、Delete、Update。</p><p>最后，我还介绍了IAM项目的GORM实战，具体使用方式跟总结中的示例代码大体保持一致，你可以返回文稿查看。</p><h2>课后练习</h2><ol>
<li>GORM支持AutoMigrate功能，思考下，你的生产环境是否可以使用AutoMigrate功能，为什么？</li>
<li>查看<a href="https://gorm.io/zh_CN/docs/index.html">GORM官方文档</a>，看下如何用GORM实现事务回滚功能。</li>
</ol><p>欢迎你在留言区与我交流讨论，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7a/d2/4ba67c0c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sch0ng</span>
  </div>
  <div class="_2_QraFYR_0">go语言中，orm使用gorm包。<br>gorm功能全，操作界面符合直觉，不需要额外的理解负担。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是这样的，老哥学习课程很认真！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-16 22:49:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">测试发现项目中的软删除功能不起作用, 需要将源码<br>https:&#47;&#47;github.com&#47;marmotedu&#47;component-base&#47;blob&#47;master&#47;pkg&#47;meta&#47;v1&#47;types.go 中的<br>DeletedAt *time.Time `json:&quot;-&quot; gorm:&quot;column:deletedAt;index:idx_deletedAt&quot;` <br>改为: 	<br>DeletedAt gorm.DeletedAt `json:&quot;-&quot; gorm:&quot;index;column:deletedAt&quot;`<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢反馈，我修复下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-09 12:53:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83equY82MMjfvGtzlo8fhT9fdKO5LjWoy0P8pfCmiaFJS0v8Z4ibzrmwHjib9CnmgMiaYMhPyja7qS6KqiaQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_004fb2</span>
  </div>
  <div class="_2_QraFYR_0">老师,企业级的应用往往需要在业务层处理&quot;事物&quot;,这个模式应该如何设计比较优雅?我看了iam项目暂时没发现相关处理</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 事务可以在 service层，不过可能要重新设计下代码了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-08 13:41:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/64/3882d90d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yandongxiao</span>
  </div>
  <div class="_2_QraFYR_0">总结：<br>1. 首先定义一个 GORM 模型（Models），GORM模型就是一个普普通通的golang struct。使用结构体名的 snake_cases 作为表名，使用字段名的 snake_case 作为列名。<br>2. gorm.Model 为表增加了 ID、CreatedAt、UpdatedAt、DeletedAt 等字段。如果是资源表，建议统一资源的元数据。参见第29章节。<br>3. 在结构体中，通过 tag，指定数据库中字段的名称。<br>4. gorm 支持表CRUD操作，每种操作也会有各种变形，与 数据库操作相对应。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 6666</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-03 01:40:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/5f/e2/e6d3d9bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>XI</span>
  </div>
  <div class="_2_QraFYR_0">go的连接池网上教程有点少，go 连接redis,monggodb 应该都需要连接池，gorm官方文档连接池的讲解也是寥寥数语，老师能不能详细补充点连接池相关的，如何使用，如果能再加上redis ，mongdb 的连接池一类的就更好了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回头整理下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-12 14:48:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/8f/61/cf0d0a95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>huining</span>
  </div>
  <div class="_2_QraFYR_0">如果按照这样写，那用不了事务啊，比如老师的代码里删除用户时，是先调用polices.delete(),然后再delete用户，那假如中途出错，也无法回滚。我的做法是再加一个*DB参数，但是感觉写起来很丑，方法里要判断之前是否启动了事务。所以如果想要添加对事务的支持应该怎么做呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果要用事务，肯定是要传*db参数。<br><br>或者在失败时调用defer func() {s.srv.Delete()}()这样的语句，来回滚</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-15 11:43:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/1e/18/9d1f1439.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>liaomars</span>
  </div>
  <div class="_2_QraFYR_0">老师：<br>sqlDB.SetMaxIdleConns(10)              &#47;&#47; 设置MySQL的最大空闲连接数（推荐100）<br>这个最大空闲连接数是10还是100？代码写的是10，注释写的是100</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 100哈，代码我找编辑更正下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-03 16:51:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘世杰</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，<br>我在使用 github.com&#47;go-xorm&#47;xorm 这个包连接 mycat 的时候<br>当我执行带 ? 号的 sql 语句时出现 {&quot;Number&quot;:1047,&quot;Message&quot;:&quot;Prepare unsupported!&quot;} 报错<br>我应该怎么配置连接信息呢？<br>以下是我的数据库连接信息<br>user:pass@tcp(127.0.0.1:3306)&#47;db?charset=utf8&amp;interpolateparams=true<br>以下是我执行的代码<br>engine := model.Get()<br>uid := 1<br>var list []model.User<br>res1 := engine.SQL(&quot;select * from tr_user where id = ?&quot;, uid).Find(&amp;list)<br>res := engine.SQL(&quot;select * from tr_user where id = 1&quot;).Find(&amp;list)<br>data := map[string]interface{}{<br>	&quot;userid&quot;: con.Uid,<br>	&quot;res&quot;: res,<br>	&quot;res1&quot;: res1,<br>	&quot;list&quot;: list,<br>}<br>以上 res1 打印的信息是 &quot;res1&quot;:{&quot;Number&quot;:1047,&quot;Message&quot;:&quot;Prepare unsupported!&quot;}<br>res 运行的 sql 能够正常运行<br>直连数据库的时候，本地连接数据库，以上代码均正常<br>使用 mycat 时，无法正常执行 sql<br>网上看了一些文档，说是 interpolateparams 这个参数的问题，但是我试了都没有用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没用过 github.com&#47;go-xorm&#47;xorm这个包。<br><br>建议使用gorm包哈，有问题可以问我，这个包我熟悉</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-26 23:48:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/75/00/618b20da.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>徐海浪</span>
  </div>
  <div class="_2_QraFYR_0">生产环境不应该使用AutoMigrate，除非有靠谱的code review流程，否则随着协作的人数越多，越容易失控</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，生产环境最好别使用AutoMigrate</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-23 19:19:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">看了下iam启动过程的源码，发现依赖的mysql实例获取不到也不会退出程序，个人感觉这样设计是否更好：写两个函数，一个用来初始化mysql单例，一个用来获取mysql单例，在server的Preparerun函数中来初始化，初始化失败则退出程序提示用户，在router中来获取已经初始化过的mysql单例，这样感觉更清晰一些～</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: GetMySQLFactoryOr这个函数如果实例不存在会返回err，这个地方你可以根据需要，判断err是否为nil，如果!=nil，可以终止程序。<br><br>iam之所以没有退出，原因如下：<br>1. 后续在执行请求时，如果初始化数据库失败，请求会报错，也就是说，如果初始化数据库失败，一定能够被感知到，并被修复。<br>2. 当时是方便测试吧，数据库初始化失败，不会影响程序启动，不会影响其他功能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-04 20:15:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/80/67/4e381da5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Derek</span>
  </div>
  <div class="_2_QraFYR_0">老师，拿要是连接多个数据库，不同的struct使用不同的库要怎么搞，是不是要初始化好几个连接，然后指定一下struct用的库？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以这么来定义下：<br>&#47;&#47; Factory defines the iam platform storage interface.    <br>type DBIns2Store interface { <br>    Users() UserStore<br>    Secrets() SecretStore<br>    Policies() PolicyStore<br>    Close() error<br>}<br><br>type DBIns2Store interface { <br>}<br><br>type Factory interface {               <br>    DBIns1Store DBIns2Store                                                           <br>    DBIns2Store DBIns2Store                            <br>}</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-27 16:45:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/14/5b/a1656a72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>潘达</span>
  </div>
  <div class="_2_QraFYR_0">gorm下是否有从表中直接生成struct的工具呢，xorm下的反向生成工具还是挺实用的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 试试这个：https:&#47;&#47;github.com&#47;Shelnutt2&#47;db2struct</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-15 07:39:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">“如果没有指定表名，则 GORM 使用结构体名的蛇形复数作为表名。例如：结构体名为 DockerInstance ，则表名为 dockerInstances 。”，蛇形不是下划线连接吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里写的有问题，我找编辑更新下，感谢反馈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-12 22:01:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/a5/5b/164dc7d2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>二三闲语</span>
  </div>
  <div class="_2_QraFYR_0">怎么配置主从，多库</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主从，是mysql实例层保证的，应用层可以不用关注。<br><br>多库，可以参照现有的实现，在Store层添加一个新的db实例</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-09 00:57:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/b5/74/cd80b9f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>友</span>
  </div>
  <div class="_2_QraFYR_0">目前用过几款orm。有beego自带的  gorm ent</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-03 10:32:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8f/cf/890f82d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那时刻</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，在链接数据库New方法里，是否要调用下sqlDB.ping来确定链接是否成功呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不用哈，如果没连接成功功能测试时，你是能感觉到的，那时候可以修复下。<br><br>不过加上也可以</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-03 10:32:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/52/40/e57a736e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pedro</span>
  </div>
  <div class="_2_QraFYR_0">GORM 支持 AutoMigrate 功能，思考下，你的生产环境是否可以使用 AutoMigrate 功能，为什么？<br><br>肯定不能啊，生产环境严格的要死，提交的SQL都要一圈的审核，怎么能直接偏移，爆炸了年奖全都没了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的呀，最好别用AutoMigrate</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-03 07:39:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/02/e4/700e5bcd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Harris Han</span>
  </div>
  <div class="_2_QraFYR_0">十分想了解老师的事务实现方式，现在就只能传db *gorm.DB 来在service层开启事务了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-15 21:07:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/9a/6f/c4490cf2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>czy</span>
  </div>
  <div class="_2_QraFYR_0"><br>&#47;&#47; DELETE from users where id = 10 AND name = &quot;jinzhu&quot;;<br>db.Where(&quot;name = ?&quot;, &quot;jinzhu&quot;).Delete(&amp;user)<br>这段代码是不是有问题？没有添加id过滤条件呀？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个地方有问题，感谢反馈，我找编辑更正下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-26 09:22:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_c2083d</span>
  </div>
  <div class="_2_QraFYR_0">java用惯了mybatis-plus 感觉这个好难用呀，sql写在代码里面不是很难维护码，</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 个人感觉难以维护</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-07 16:36:17</div>
  </div>
</div>
</div>
</li>
</ul>