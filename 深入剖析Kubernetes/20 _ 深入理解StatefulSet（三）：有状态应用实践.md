<audio title="20 _ 深入理解StatefulSet（三）：有状态应用实践" src="https://static001.geekbang.org/resource/audio/2e/dd/2e127711ab3309ee42f1c47254612add.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：深入理解StatefulSet之有状态应用实践。</p><p>在前面的两篇文章中，我详细讲解了StatefulSet的工作原理，以及处理拓扑状态和存储状态的方法。而在今天这篇文章中，我将通过一个实际的例子，再次为你深入解读一下部署一个StatefulSet的完整流程。</p><p>今天我选择的实例是部署一个MySQL集群，这也是Kubernetes官方文档里的一个经典案例。但是，很多工程师都曾向我吐槽说这个例子“完全看不懂”。</p><p>其实，这样的吐槽也可以理解：相比于Etcd、Cassandra等“原生”就考虑了分布式需求的项目，MySQL以及很多其他的数据库项目，在分布式集群的搭建上并不友好，甚至有点“原始”。</p><p>所以，这次我就直接选择了这个具有挑战性的例子，和你分享如何使用StatefulSet将它的集群搭建过程“容器化”。</p><blockquote>
<p>备注：在开始实践之前，请确保我们之前一起部署的那个Kubernetes集群还是可用的，并且网络插件和存储插件都能正常运行。具体的做法，请参考第11篇文章<a href="https://time.geekbang.org/column/article/39724">《从0到1：搭建一个完整的Kubernetes集群》</a>的内容。</p>
</blockquote><p>首先，用自然语言来描述一下我们想要部署的“有状态应用”。</p><!-- [[[read_end]]] --><ol>
<li>
<p>是一个“主从复制”（Maser-Slave Replication）的MySQL集群；</p>
</li>
<li>
<p>有1个主节点（Master）；</p>
</li>
<li>
<p>有多个从节点（Slave）；</p>
</li>
<li>
<p>从节点需要能水平扩展；</p>
</li>
<li>
<p>所有的写操作，只能在主节点上执行；</p>
</li>
<li>
<p>读操作可以在所有节点上执行。</p>
</li>
</ol><p>这就是一个非常典型的主从模式的MySQL集群了。我们可以把上面描述的“有状态应用”的需求，通过一张图来表示。</p><p><img src="https://static001.geekbang.org/resource/image/bb/02/bb2d7f03443392ca40ecde6b1a91c002.png?wh=785*735" alt=""><br>
在常规环境里，部署这样一个主从模式的MySQL集群的主要难点在于：如何让从节点能够拥有主节点的数据，即：如何配置主（Master）从（Slave）节点的复制与同步。</p><p>所以，在安装好MySQL的Master节点之后，你需要做的第一步工作，就是<strong>通过XtraBackup将Master节点的数据备份到指定目录。</strong></p><blockquote>
<p>备注：XtraBackup是业界主要使用的开源MySQL备份和恢复工具。</p>
</blockquote><p>这一步会自动在目标目录里生成一个备份信息文件，名叫：xtrabackup_binlog_info。这个文件一般会包含如下两个信息：</p><pre><code>$ cat xtrabackup_binlog_info
TheMaster-bin.000001     481
</code></pre><p>这两个信息会在接下来配置Slave节点的时候用到。</p><p><strong>第二步：配置Slave节点</strong>。Slave节点在第一次启动前，需要先把Master节点的备份数据，连同备份信息文件，一起拷贝到自己的数据目录（/var/lib/mysql）下。然后，我们执行这样一句SQL：</p><pre><code>TheSlave|mysql&gt; CHANGE MASTER TO
                MASTER_HOST='$masterip',
                MASTER_USER='xxx',
                MASTER_PASSWORD='xxx',
                MASTER_LOG_FILE='TheMaster-bin.000001',
                MASTER_LOG_POS=481;
</code></pre><p>其中，MASTER_LOG_FILE和MASTER_LOG_POS，就是该备份对应的二进制日志（Binary Log）文件的名称和开始的位置（偏移量），也正是xtrabackup_binlog_info文件里的那两部分内容（即：TheMaster-bin.000001和481）。</p><p><strong>第三步，启动Slave节点</strong>。在这一步，我们需要执行这样一句SQL：</p><pre><code>TheSlave|mysql&gt; START SLAVE;
</code></pre><p>这样，Slave节点就启动了。它会使用备份信息文件中的二进制日志文件和偏移量，与主节点进行数据同步。</p><p><strong>第四步，在这个集群中添加更多的Slave节点</strong>。</p><p>需要注意的是，新添加的Slave节点的备份数据，来自于已经存在的Slave节点。</p><p>所以，在这一步，我们需要将Slave节点的数据备份在指定目录。而这个备份操作会自动生成另一种备份信息文件，名叫：xtrabackup_slave_info。同样地，这个文件也包含了MASTER_LOG_FILE和MASTER_LOG_POS两个字段。</p><p>然后，我们就可以执行跟前面一样的“CHANGE MASTER TO”和“START SLAVE” 指令，来初始化并启动这个新的Slave节点了。</p><p>通过上面的叙述，我们不难看到，<strong>将部署MySQL集群的流程迁移到Kubernetes项目上，需要能够“容器化”地解决下面的“三座大山”：</strong></p><ol>
<li>
<p>Master节点和Slave节点需要有不同的配置文件（即：不同的my.cnf）；</p>
</li>
<li>
<p>Master节点和Slave节点需要能够传输备份信息文件；</p>
</li>
<li>
<p>在Slave节点第一次启动之前，需要执行一些初始化SQL操作；</p>
</li>
</ol><p>而由于MySQL本身同时拥有拓扑状态（主从节点的区别）和存储状态（MySQL保存在本地的数据），我们自然要通过StatefulSet来解决这“三座大山”的问题。</p><p><strong>其中，“第一座大山：Master节点和Slave节点需要有不同的配置文件”，很容易处理</strong>：我们只需要给主从节点分别准备两份不同的MySQL配置文件，然后根据Pod的序号（Index）挂载进去即可。</p><p>正如我在前面文章中介绍过的，这样的配置文件信息，应该保存在ConfigMap里供Pod使用。它的定义如下所示：</p><pre><code>apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
data:
  master.cnf: |
    # 主节点MySQL的配置文件
    [mysqld]
    log-bin
  slave.cnf: |
    # 从节点MySQL的配置文件
    [mysqld]
    super-read-only
</code></pre><p>在这里，我们定义了master.cnf和slave.cnf两个MySQL的配置文件。</p><ul>
<li>master.cnf开启了log-bin，即：使用二进制日志文件的方式进行主从复制，这是一个标准的设置。</li>
<li>slave.cnf的开启了super-read-only，代表的是从节点会拒绝除了主节点的数据同步操作之外的所有写操作，即：它对用户是只读的。</li>
</ul><p>而上述ConfigMap定义里的data部分，是Key-Value格式的。比如，master.cnf就是这份配置数据的Key，而“|”后面的内容，就是这份配置数据的Value。这份数据将来挂载进Master节点对应的Pod后，就会在Volume目录里生成一个叫作master.cnf的文件。</p><blockquote>
<p>备注：如果你对ConfigMap的用法感到陌生的话，可以稍微复习一下第15篇文章<a href="https://time.geekbang.org/column/article/40466">《深入解析Pod对象（二）：使用进阶》</a>中，我讲解Secret对象部分的内容。因为，ConfigMap跟Secret，无论是使用方法还是实现原理，几乎都是一样的。</p>
</blockquote><p>接下来，我们需要创建两个Service来供StatefulSet以及用户使用。这两个Service的定义如下所示：</p><pre><code>apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
</code></pre><p>可以看到，这两个Service都代理了所有携带app=mysql标签的Pod，也就是所有的MySQL Pod。端口映射都是用Service的3306端口对应Pod的3306端口。</p><p>不同的是，第一个名叫“mysql”的Service是一个Headless Service（即：clusterIP= None）。所以它的作用，是通过为Pod分配DNS记录来固定它的拓扑状态，比如“mysql-0.mysql”和“mysql-1.mysql”这样的DNS名字。其中，编号为0的节点就是我们的主节点。</p><p>而第二个名叫“mysql-read”的Service，则是一个常规的Service。</p><p>并且我们规定，所有用户的读请求，都必须访问第二个Service被自动分配的DNS记录，即：“mysql-read”（当然，也可以访问这个Service的VIP）。这样，读请求就可以被转发到任意一个MySQL的主节点或者从节点上。</p><blockquote>
<p>备注：Kubernetes中的所有Service、Pod对象，都会被自动分配同名的DNS记录。具体细节，我会在后面Service部分做重点讲解。</p>
</blockquote><p>而所有用户的写请求，则必须直接以DNS记录的方式访问到MySQL的主节点，也就是：“mysql-0.mysql“这条DNS记录。</p><p>接下来，我们再一起解决“第二座大山：Master节点和Slave节点需要能够传输备份文件”的问题。</p><p><strong>翻越这座大山的思路，我比较推荐的做法是：先搭建框架，再完善细节。其中，Pod部分如何定义，是完善细节时的重点。</strong></p><p><strong>所以首先，我们先为StatefulSet对象规划一个大致的框架，如下图所示：</strong></p><p><img src="https://static001.geekbang.org/resource/image/16/09/16aa2e42034830a0e64120ecc330a509.png?wh=612*1058" alt=""></p><p>在这一步，我们可以先为StatefulSet定义一些通用的字段。</p><p>比如：selector表示，这个StatefulSet要管理的Pod必须携带app=mysql标签；它声明要使用的Headless Service的名字是：mysql。</p><p>这个StatefulSet的replicas值是3，表示它定义的MySQL集群有三个节点：一个Master节点，两个Slave节点。</p><p>可以看到，StatefulSet管理的“有状态应用”的多个实例，也都是通过同一份Pod模板创建出来的，使用的是同一个Docker镜像。这也就意味着：如果你的应用要求不同节点的镜像不一样，那就不能再使用StatefulSet了。对于这种情况，应该考虑我后面会讲解到的Operator。</p><p>除了这些基本的字段外，作为一个有存储状态的MySQL集群，StatefulSet还需要管理存储状态。所以，我们需要通过volumeClaimTemplate（PVC模板）来为每个Pod定义PVC。比如，这个PVC模板的resources.requests.strorage指定了存储的大小为10 GiB；ReadWriteOnce指定了该存储的属性为可读写，并且一个PV只允许挂载在一个宿主机上。将来，这个PV对应的的Volume就会充当MySQL Pod的存储数据目录。</p><p><strong>然后，我们来重点设计一下这个StatefulSet的Pod模板，也就是template字段。</strong></p><p>由于StatefulSet管理的Pod都来自于同一个镜像，这就要求我们在编写Pod时，一定要保持清醒，用“人格分裂”的方式进行思考：</p><ol>
<li>
<p>如果这个Pod是Master节点，我们要怎么做；</p>
</li>
<li>
<p>如果这个Pod是Slave节点，我们又要怎么做。</p>
</li>
</ol><p>想清楚这两个问题，我们就可以按照Pod的启动过程来一步步定义它们了。</p><p><strong>第一步：从ConfigMap中，获取MySQL的Pod对应的配置文件。</strong></p><p>为此，我们需要进行一个初始化操作，根据节点的角色是Master还是Slave节点，为Pod分配对应的配置文件。此外，MySQL还要求集群里的每个节点都有一个唯一的ID文件，名叫server-id.cnf。</p><p>而根据我们已经掌握的Pod知识，这些初始化操作显然适合通过InitContainer来完成。所以，我们首先定义了一个InitContainer，如下所示：</p><pre><code>      ...
      # template.spec
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - &quot;-c&quot;
        - |
          set -ex
          # 从Pod的序号，生成server-id
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] &gt; /mnt/conf.d/server-id.cnf
          # 由于server-id=0有特殊含义，我们给ID加一个100来避开它
          echo server-id=$((100 + $ordinal)) &gt;&gt; /mnt/conf.d/server-id.cnf
          # 如果Pod序号是0，说明它是Master节点，从ConfigMap里把Master的配置文件拷贝到/mnt/conf.d/目录；
          # 否则，拷贝Slave的配置文件
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
</code></pre><p>在这个名叫init-mysql的InitContainer的配置中，它从Pod的hostname里，读取到了Pod的序号，以此作为MySQL节点的server-id。</p><p>然后，init-mysql通过这个序号，判断当前Pod到底是Master节点（即：序号为0）还是Slave节点（即：序号不为0），从而把对应的配置文件从/mnt/config-map目录拷贝到/mnt/conf.d/目录下。</p><p>其中，文件拷贝的源目录/mnt/config-map，正是ConfigMap在这个Pod的Volume，如下所示：</p><pre><code>      ...
      # template.spec
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
</code></pre><p>通过这个定义，init-mysql在声明了挂载config-map这个Volume之后，ConfigMap里保存的内容，就会以文件的方式出现在它的/mnt/config-map目录当中。</p><p>而文件拷贝的目标目录，即容器里的/mnt/conf.d/目录，对应的则是一个名叫conf的、emptyDir类型的Volume。基于Pod Volume共享的原理，当InitContainer复制完配置文件退出后，后面启动的MySQL容器只需要直接声明挂载这个名叫conf的Volume，它所需要的.cnf配置文件已经出现在里面了。这跟我们之前介绍的Tomcat和WAR包的处理方法是完全一样的。</p><p><strong>第二步：在Slave Pod启动前，从Master或者其他Slave Pod里拷贝数据库数据到自己的目录下。</strong></p><p>为了实现这个操作，我们就需要再定义第二个InitContainer，如下所示：</p><pre><code>      ...
      # template.spec.initContainers
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - &quot;-c&quot;
        - |
          set -ex
          # 拷贝操作只需要在第一次启动时进行，所以如果数据已经存在，跳过
          [[ -d /var/lib/mysql/mysql ]] &amp;&amp; exit 0
          # Master节点(序号为0)不需要做这个操作
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] &amp;&amp; exit 0
          # 使用ncat指令，远程地从前一个节点拷贝数据到本地
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # 执行--prepare，这样拷贝来的数据就可以用作恢复了
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
</code></pre><p>在这个名叫clone-mysql的InitContainer里，我们使用的是xtrabackup镜像（它里面安装了xtrabackup工具）。</p><p>而在它的启动命令里，我们首先做了一个判断。即：当初始化所需的数据（/var/lib/mysql/mysql 目录）已经存在，或者当前Pod是Master节点的时候，不需要做拷贝操作。</p><p>接下来，clone-mysql会使用Linux自带的ncat指令，向DNS记录为“mysql-&lt;当前序号减一&gt;.mysql”的Pod，也就是当前Pod的前一个Pod，发起数据传输请求，并且直接用xbstream指令将收到的备份数据保存在/var/lib/mysql目录下。</p><blockquote>
<p>备注：3307是一个特殊端口，运行着一个专门负责备份MySQL数据的辅助进程。我们后面马上会讲到它。</p>
</blockquote><p>当然，这一步你可以随意选择用自己喜欢的方法来传输数据。比如，用scp或者rsync，都没问题。</p><p>你可能已经注意到，这个容器里的/var/lib/mysql目录，<strong>实际上正是一个名为data的PVC</strong>，即：我们在前面声明的持久化存储。</p><p>这就可以保证，哪怕宿主机宕机了，我们数据库的数据也不会丢失。更重要的是，由于Pod Volume是被Pod里的容器共享的，所以后面启动的MySQL容器，就可以把这个Volume挂载到自己的/var/lib/mysql目录下，直接使用里面的备份数据进行恢复操作。</p><p>不过，clone-mysql容器还要对/var/lib/mysql目录，执行一句xtrabackup --prepare操作，目的是让拷贝来的数据进入一致性状态，这样，这些数据才能被用作数据恢复。</p><p>至此，我们就通过InitContainer完成了对“主、从节点间备份文件传输”操作的处理过程，也就是翻越了“第二座大山”。</p><p>接下来，我们可以开始定义MySQL容器,启动MySQL服务了。由于StatefulSet里的所有Pod都来自用同一个Pod模板，所以我们还要“人格分裂”地去思考：这个MySQL容器的启动命令，在Master和Slave两种情况下有什么不同。</p><p>有了Docker镜像，在Pod里声明一个Master角色的MySQL容器并不是什么困难的事情：直接执行MySQL启动命令即可。</p><p>但是，如果这个Pod是一个第一次启动的Slave节点，在执行MySQL启动命令之前，它就需要使用前面InitContainer拷贝来的备份数据进行初始化。</p><p>可是，别忘了，<strong>容器是一个单进程模型。</strong></p><p>所以，一个Slave角色的MySQL容器启动之前，谁能负责给它执行初始化的SQL语句呢？</p><p>这就是我们需要解决的“第三座大山”的问题，即：如何在Slave节点的MySQL容器第一次启动之前，执行初始化SQL。</p><p>你可能已经想到了，我们可以为这个MySQL容器额外定义一个sidecar容器，来完成这个操作，它的定义如下所示：</p><pre><code>      ...
      # template.spec.containers
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - &quot;-c&quot;
        - |
          set -ex
          cd /var/lib/mysql
          
          # 从备份信息文件里读取MASTER_LOG_FILEM和MASTER_LOG_POS这两个字段的值，用来拼装集群初始化SQL
          if [[ -f xtrabackup_slave_info ]]; then
            # 如果xtrabackup_slave_info文件存在，说明这个备份数据来自于另一个Slave节点。这种情况下，XtraBackup工具在备份的时候，就已经在这个文件里自动生成了&quot;CHANGE MASTER TO&quot; SQL语句。所以，我们只需要把这个文件重命名为change_master_to.sql.in，后面直接使用即可
            mv xtrabackup_slave_info change_master_to.sql.in
            # 所以，也就用不着xtrabackup_binlog_info了
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # 如果只存在xtrabackup_binlog_inf文件，那说明备份来自于Master节点，我们就需要解析这个备份信息文件，读取所需的两个字段的值
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            # 把两个字段的值拼装成SQL，写入change_master_to.sql.in文件
            echo &quot;CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}&quot; &gt; change_master_to.sql.in
          fi
          
          # 如果change_master_to.sql.in，就意味着需要做集群初始化工作
          if [[ -f change_master_to.sql.in ]]; then
            # 但一定要先等MySQL容器启动之后才能进行下一步连接MySQL的操作
            echo &quot;Waiting for mysqld to be ready (accepting connections)&quot;
            until mysql -h 127.0.0.1 -e &quot;SELECT 1&quot;; do sleep 1; done
            
            echo &quot;Initializing replication from clone position&quot;
            # 将文件change_master_to.sql.in改个名字，防止这个Container重启的时候，因为又找到了change_master_to.sql.in，从而重复执行一遍这个初始化流程
            mv change_master_to.sql.in change_master_to.sql.orig
            # 使用change_master_to.sql.orig的内容，也是就是前面拼装的SQL，组成一个完整的初始化和启动Slave的SQL语句
            mysql -h 127.0.0.1 &lt;&lt;EOF
          $(&lt;change_master_to.sql.orig),
            MASTER_HOST='mysql-0.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi
          
          # 使用ncat监听3307端口。它的作用是，在收到传输请求的时候，直接执行&quot;xtrabackup --backup&quot;命令，备份MySQL的数据并发送给请求者
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            &quot;xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root&quot;
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
</code></pre><p>可以看到，<strong>在这个名叫xtrabackup的sidecar容器的启动命令里，其实实现了两部分工作。</strong></p><p><strong>第一部分工作，当然是MySQL节点的初始化工作</strong>。这个初始化需要使用的SQL，是sidecar容器拼装出来、保存在一个名为change_master_to.sql.in的文件里的，具体过程如下所示：</p><p>sidecar容器首先会判断当前Pod的/var/lib/mysql目录下，是否有xtrabackup_slave_info这个备份信息文件。</p><ul>
<li>如果有，则说明这个目录下的备份数据是由一个Slave节点生成的。这种情况下，XtraBackup工具在备份的时候，就已经在这个文件里自动生成了"CHANGE MASTER TO" SQL语句。所以，我们只需要把这个文件重命名为change_master_to.sql.in，后面直接使用即可。</li>
<li>如果没有xtrabackup_slave_info文件、但是存在xtrabackup_binlog_info文件，那就说明备份数据来自于Master节点。这种情况下，sidecar容器就需要解析这个备份信息文件，读取MASTER_LOG_FILE和MASTER_LOG_POS这两个字段的值，用它们拼装出初始化SQL语句，然后把这句SQL写入到change_master_to.sql.in文件中。</li>
</ul><p>接下来，sidecar容器就可以执行初始化了。从上面的叙述中可以看到，只要这个change_master_to.sql.in文件存在，那就说明接下来需要进行集群初始化操作。</p><p>所以，这时候，sidecar容器只需要读取并执行change_master_to.sql.in里面的“CHANGE MASTER TO”指令，再执行一句START SLAVE命令，一个Slave节点就被成功启动了。</p><blockquote>
<p>需要注意的是：Pod里的容器并没有先后顺序，所以在执行初始化SQL之前，必须先执行一句SQL（select 1）来检查一下MySQL服务是否已经可用。</p>
</blockquote><p>当然，上述这些初始化操作完成后，我们还要删除掉前面用到的这些备份信息文件。否则，下次这个容器重启时，就会发现这些文件存在，所以又会重新执行一次数据恢复和集群初始化的操作，这是不对的。</p><p>同理，change_master_to.sql.in在使用后也要被重命名，以免容器重启时因为发现这个文件存在又执行一遍初始化。</p><p><strong>在完成MySQL节点的初始化后，这个sidecar容器的第二个工作，则是启动一个数据传输服务。</strong></p><p>具体做法是：sidecar容器会使用ncat命令启动一个工作在3307端口上的网络发送服务。一旦收到数据传输请求时，sidecar容器就会调用xtrabackup --backup指令备份当前MySQL的数据，然后把这些备份数据返回给请求者。这就是为什么我们在InitContainer里定义数据拷贝的时候，访问的是“上一个MySQL节点”的3307端口。</p><p>值得一提的是，由于sidecar容器和MySQL容器同处于一个Pod里，所以它是直接通过Localhost来访问和备份MySQL容器里的数据的，非常方便。</p><p>同样地，我在这里举例用的只是一种备份方法而已，你完全可以选择其他自己喜欢的方案。比如，你可以使用innobackupex命令做数据备份和准备，它的使用方法几乎与本文的备份方法一样。</p><p>至此，我们也就翻越了“第三座大山”，完成了Slave节点第一次启动前的初始化工作。</p><p>扳倒了这“三座大山”后，我们终于可以定义Pod里的主角，MySQL容器了。有了前面这些定义和初始化工作，MySQL容器本身的定义就非常简单了，如下所示：</p><pre><code>      ...
      # template.spec
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: &quot;1&quot;
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: [&quot;mysqladmin&quot;, &quot;ping&quot;]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # 通过TCP连接的方式进行健康检查
            command: [&quot;mysql&quot;, &quot;-h&quot;, &quot;127.0.0.1&quot;, &quot;-e&quot;, &quot;SELECT 1&quot;]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
</code></pre><p>在这个容器的定义里，我们使用了一个标准的MySQL 5.7 的官方镜像。它的数据目录是/var/lib/mysql，配置文件目录是/etc/mysql/conf.d。</p><p>这时候，你应该能够明白，如果MySQL容器是Slave节点的话，它的数据目录里的数据，就来自于InitContainer从其他节点里拷贝而来的备份。它的配置文件目录/etc/mysql/conf.d里的内容，则来自于ConfigMap对应的Volume。而它的初始化工作，则是由同一个Pod里的sidecar容器完成的。这些操作，正是我刚刚为你讲述的大部分内容。</p><p>另外，我们为它定义了一个livenessProbe，通过mysqladmin ping命令来检查它是否健康；还定义了一个readinessProbe，通过查询SQL（select 1）来检查MySQL服务是否可用。当然，凡是readinessProbe检查失败的MySQL Pod，都会从Service里被摘除掉。</p><p>至此，一个完整的主从复制模式的MySQL集群就定义完了。</p><p>现在，我们就可以使用kubectl命令，尝试运行一下这个StatefulSet了。</p><p><strong>首先，我们需要在Kubernetes集群里创建满足条件的PV</strong>。如果你使用的是我们在第11篇文章《<a href="https://time.geekbang.org/column/article/39724">从0到1：搭建一个完整的Kubernetes集群》</a>里部署的Kubernetes集群的话，你可以按照如下方式使用存储插件Rook：</p><pre><code>$ kubectl create -f rook-storage.yaml
$ cat rook-storage.yaml
apiVersion: ceph.rook.io/v1beta1
kind: Pool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: ceph.rook.io/block
parameters:
  pool: replicapool
  clusterNamespace: rook-ceph
</code></pre><p>在这里，我用到了StorageClass来完成这个操作。它的作用，是自动地为集群里存在的每一个PVC，调用存储插件（Rook）创建对应的PV，从而省去了我们手动创建PV的机械劳动。我在后续讲解容器存储的时候，会再详细介绍这个机制。</p><blockquote>
<p>备注：在使用Rook的情况下，mysql-statefulset.yaml里的volumeClaimTemplates字段需要加上声明storageClassName=rook-ceph-block，才能使用到这个Rook提供的持久化存储。</p>
</blockquote><p><strong>然后，我们就可以创建这个StatefulSet了</strong>，如下所示：</p><pre><code>$ kubectl create -f mysql-statefulset.yaml
$ kubectl get pod -l app=mysql
NAME      READY     STATUS    RESTARTS   AGE
mysql-0   2/2       Running   0          2m
mysql-1   2/2       Running   0          1m
mysql-2   2/2       Running   0          1m
</code></pre><p>可以看到，StatefulSet启动成功后，会有三个Pod运行。</p><p><strong>接下来，我们可以尝试向这个MySQL集群发起请求，执行一些SQL操作来验证它是否正常：</strong></p><pre><code>$ kubectl run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql &lt;&lt;EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('hello');
EOF
</code></pre><p>如上所示，我们通过启动一个容器，使用MySQL client执行了创建数据库和表、以及插入数据的操作。需要注意的是，我们连接的MySQL的地址必须是mysql-0.mysql（即：Master节点的DNS记录）。因为，只有Master节点才能处理写操作。</p><p>而通过连接mysql-read这个Service，我们就可以用SQL进行读操作，如下所示：</p><pre><code>$ kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
 mysql -h mysql-read -e &quot;SELECT * FROM test.messages&quot;
Waiting for pod default/mysql-client to be running, status is Pending, pod ready: false
+---------+
| message |
+---------+
| hello   |
+---------+
pod &quot;mysql-client&quot; deleted
</code></pre><p>在有了StatefulSet以后，你就可以像Deployment那样，非常方便地扩展这个MySQL集群，比如：</p><pre><code>$ kubectl scale statefulset mysql  --replicas=5
</code></pre><p>这时候，你就会发现新的Slave Pod mysql-3和mysql-4被自动创建了出来。</p><p>而如果你像如下所示的这样，直接连接mysql-3.mysql，即mysql-3这个Pod的DNS名字来进行查询操作：</p><pre><code>$ kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-3.mysql -e &quot;SELECT * FROM test.messages&quot;
Waiting for pod default/mysql-client to be running, status is Pending, pod ready: false
+---------+
| message |
+---------+
| hello   |
+---------+
pod &quot;mysql-client&quot; deleted
</code></pre><p>就会看到，从StatefulSet为我们新创建的mysql-3上，同样可以读取到之前插入的记录。也就是说，我们的数据备份和恢复，都是有效的。</p><h2>总结</h2><p>在今天这篇文章中，我以MySQL集群为例，和你详细分享了一个实际的StatefulSet的编写过程。这个YAML文件的<a href="https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/#statefulset">链接在这里</a>，希望你能多花一些时间认真消化。</p><p>在这个过程中，有以下几个关键点（坑）特别值得你注意和体会。</p><ol>
<li>
<p>“人格分裂”：在解决需求的过程中，一定要记得思考，该Pod在扮演不同角色时的不同操作。</p>
</li>
<li>
<p>“阅后即焚”：很多“有状态应用”的节点，只是在第一次启动的时候才需要做额外处理。所以，在编写YAML文件时，你一定要考虑“容器重启”的情况，不要让这一次的操作干扰到下一次的容器启动。</p>
</li>
<li>
<p>“容器之间平等无序”：除非是InitContainer，否则一个Pod里的多个容器之间，是完全平等的。所以，你精心设计的sidecar，绝不能对容器的顺序做出假设，否则就需要进行前置检查。</p>
</li>
</ol><p>最后，相信你也已经能够理解，StatefulSet其实是一种特殊的Deployment，只不过这个“Deployment”的每个Pod实例的名字里，都携带了一个唯一并且固定的编号。这个编号的顺序，固定了Pod的拓扑关系；这个编号对应的DNS记录，固定了Pod的访问方式；这个编号对应的PV，绑定了Pod与持久化存储的关系。所以，当Pod被删除重建时，这些“状态”都会保持不变。</p><p>而一旦你的应用没办法通过上述方式进行状态的管理，那就代表了StatefulSet已经不能解决它的部署问题了。这时候，我后面讲到的Operator，可能才是一个更好的选择。</p><h2>思考题</h2><p>如果我们现在的需求是：所有的读请求，只由Slave节点处理；所有的写请求，只由Master节点处理。那么，你需要在今天这篇文章的基础上再做哪些改动呢？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/96/89/9312b3a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vincen</span>
  </div>
  <div class="_2_QraFYR_0">因为是通过一个statefulset来实现的 感觉太复杂了，master和slave做成两个statefulset就非常简单了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没错，这是一个很好的解决思路</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-08 12:34:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/13/8c/c86340ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>巴西</span>
  </div>
  <div class="_2_QraFYR_0">信息量太大,容我冷静一下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-02 20:46:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2d/24/28acca15.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DJH</span>
  </div>
  <div class="_2_QraFYR_0">有一点我还是没想通，为啥数据复制操作要用sidecar容器来处理，而不用mysql主容器来一并解决。如果把数据复制操作和启动mysql服务一并写到同一个sh -c后面，这样算不算一个单一进程呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: sidecar是常驻进程，要监听3370端口呢。一个容器里显然没办法管理两个常驻进程，这就是容器是单进程的意思啊。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-10 00:13:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/50/4d/db91e218.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sam700000</span>
  </div>
  <div class="_2_QraFYR_0">通过这节我才更清晰理解了，kubernetes最小的管理单位是pod的概念，把思维从容器这层抽出来。然后也更清楚的理解了statefulset实现有状态，就是把特定的状态配置绑定在某个对象名称上面，然后通过之前说的控制循环不断的维护这个对象的特性，只要这写特定对象和它们各自绑定的状态配置不变就实现了有状态的应用部署和维护。果然实践出真知👍</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的对！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 21:05:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/00/15/6e399ec7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>彭锐</span>
  </div>
  <div class="_2_QraFYR_0">这样的有状态应用有个约束，pod标识一旦确定，角色就确定了。但实际应用中，不能有这样的约束，主挂了，要有一个从升主。这就要求至少有选主机制，所有的配置文件也不能依赖id确定角色。<br>这怎么玩呢？operater吗？operater可以看成一个特定应用的sre保姆吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没错。你说的这种情况得用operator，后面会讲。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-18 15:58:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b8/07/ef28a5f7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黄俊</span>
  </div>
  <div class="_2_QraFYR_0">error: unable to recognize &quot;rook-storage.yaml&quot;: no matches for kind &quot;Pool&quot; in version &quot;ceph.rook.io&#47;v1&quot;<br><br>应该是rook新版本的问题，rook-storage.yaml 文件中有两个位置需要修改。<br><br>apiVersion: ceph.rook.io&#47;v1<br>kind: CephBlockPool<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-29 03:46:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/df/3e/718d6f1b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wenxuan</span>
  </div>
  <div class="_2_QraFYR_0">关于思考提的解决方案，有回答提到可以通过创建mysql-read和mysql-write两个service，用不同的label来分别筛选主从。我觉得这里面忽略了一点，即使是StatefulSet，各个副本的metadata也是完全一致的，不可能做到给主从节点打上不同的标签。主从分两个StatefulSet部署可以解决这个问题，但又引入了需要人为控制两个StagefulSet启动顺序的问题。<br>其实我觉得应该可以修改mysql的启动参数，让master容器和slave容器监听在不同的端口，然后分别创建read和write两个svc，绑定不同的端口。由于svc会把readiness探针失败的pod剔出去，所以就达到read svc只有slave节点，而write svc只有master节点的目的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-15 12:24:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/0c/3a/d2161d37.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>侯操宇</span>
  </div>
  <div class="_2_QraFYR_0">老师，我发现在initContainers 的ini-mysql容器中，执行的只是简单的文件拷贝工作，但镜像却用的是mysql的镜像，可以用其他镜像取代吗？镜像的选择有什么讲究吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没啥讲究，能完成任务即可</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-11 17:32:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/18/34/c082419c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风轨</span>
  </div>
  <div class="_2_QraFYR_0">集群中的master&#47;slave节点关系不对等。Service是通过label来绑定pod的，一个statefulSet中所有pod的label都是一样的，无法区分，因此service也是无法区别绑定slave pod的。所以答案应该是老师在文中多次提及的Operator，一次编排多个角色不同甚至镜像不同的服务。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其实也可以写成两个statefulset </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-08 13:43:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/69/d8/112f69f8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wtcctw</span>
  </div>
  <div class="_2_QraFYR_0">张大大你好，<br>我之前自己玩过docker，在看这个专栏之前，特意简单过了一下《Docker容器与容器云 第二版》，之前一些章节看得非常过瘾，讲解得非常透彻。<br>但是看到StateFulSet的时候感觉有点晕。。。特别是里面的Pod YAML定义，感觉要记的东西非常多，学习曲线也比较陡峭，一段时间不用肯定就忘了。<br>所以想问一下，作为非专业的运维人员，对于k8s的掌握大致要到何种程度，多谢多谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 遇到问题能迅速定位到源码即可</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-08 12:00:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/da/48/0d3cd6fa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小水</span>
  </div>
  <div class="_2_QraFYR_0">用了很大功夫，调通了老师的示例，代码都是照着老师代码敲的，也是不可以避免的有拼写的错误，通过不断的调试，在那成功的一瞬间，喜悦不可言语，老师的示例中使用国外的镜像，我是更换为国内的源，yaml如下链接，希望对后来的同学有所帮助<br>https:&#47;&#47;github.com&#47;Alexwalt&#47;kubernetes-yamls&#47;blob&#47;master&#47;mysql.yaml<br>调试工具<br>kubectl describe pod pod名<br>kubectl logs -p pod名 -c docker名</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-01 13:15:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/63/47/e8164679.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>正z</span>
  </div>
  <div class="_2_QraFYR_0">mysql感觉有点复杂，自己试了下mongoDB，复杂度感觉低一点更容易成功，也同样能深刻理解上面的内容，伙计们也可以试试</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-13 16:33:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9f/26/89eda2c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>V V</span>
  </div>
  <div class="_2_QraFYR_0">这种部署方式，主节点pod挂了的时候，能自动重新创建一个出来吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 15:36:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4c/49/d21c134f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>swordholder</span>
  </div>
  <div class="_2_QraFYR_0">总结里给的&#39;mysql-statefulset.yaml&#39;好像有点问题，在声明&#39;volumeClaimTemplates&#39;时，是不是应该通过&#39;storageClassName&#39;属性引用前面定义的&#39;StorageClass&#39;:<br><br>storageClassName: rook-ceph-block<br><br>我开始一直遇到&quot;pod has unbound immediate PersistentVolumeClaims&quot; 错误，增加了这个属性就好了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，用rook的环境应该加上这个，我加一句注释上去。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-08 19:06:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIZtZz0LgYRdP1FKvmPfQUv7vQ04ibTvNTx3wcG0WEblMHQ8dORKqBeMlath93yO5a129tKhFPkA4w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小刚</span>
  </div>
  <div class="_2_QraFYR_0">K8S 部署的Mysql集群，是否有HA的风险问题？ 考虑两种场景：<br>1 如果mysq-0 POD发生意外重启，无法再次调度成功（资源不足等情况），其他的节点POD无法切换微主用，集群整体就不可用了； <br>2 即使能再次调度成功，也依赖K8S集群的在异常情况下，重新调度一个POD时间； 比如节点下电后，K8S可能需要5分钟，才会在其他节点调度；<br>所以是否数据库等高可用性的应用，在生产环境，还是不要部署在K8S为妙？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-23 10:04:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/78/92/ae2c1b12.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>D</span>
  </div>
  <div class="_2_QraFYR_0">这里创建MySQL的时候，由于设定了readnessProbe在启动后30秒执行，但是当你没办法在30秒之内启动成功过一会执行logs就会看到一个—init……的错误，这是因为容器检测到不可用就重新创建了一个新的，但是新的容器MySQL执行初始化的时候发现所在的数据目录(&#47;var&#47;lib&#47;MySQL)不为空，这时候只要响应的调高readnessProbe和livenessProbe的initialDelaySeconds值就好了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-28 13:20:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/11/4b/fa64f061.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xfan</span>
  </div>
  <div class="_2_QraFYR_0">感觉实现起来很复杂，不优美</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 所以才要讲operator的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-26 07:18:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/16/a2/26ed8f44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>加菲老猫</span>
  </div>
  <div class="_2_QraFYR_0">老师方便提供一个官方的mysql-statefulset.yaml样本嘛，我storageClassName: rook-ceph-block参数怎么加都跑不出来，多谢多谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 17:16:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/25/23/06a37753.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jesse</span>
  </div>
  <div class="_2_QraFYR_0">哈哈，看着大佬的这篇文章，跟着搭建了redis的主备集群，比mysql简洁的多</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-15 16:02:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a0/30/5a7247eb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>eden</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问mysql persistent volume需要扩容怎么做？通过kubectl scale不能修改volume大小。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要存储插件支持</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-11 18:48:49</div>
  </div>
</div>
</div>
</li>
</ul>