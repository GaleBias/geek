<audio title="04｜如何快速搭建Prometheus系统？" src="https://static001.geekbang.org/resource/audio/b7/b5/b77101f206183f161dfb0a9cdd9e56b5.mp3" controls="controls"></audio> 
<p>你好，我是秦晓辉。</p><p>前面三讲我们介绍了监控系统的一些基本概念，这一讲我们开始进入实操环节，部署监控系统。业界可选的开源方案很多，随着云原生的流行，越来越多的公司开始拥抱云原生，而云原生标配的监控系统显然就是 Prometheus，而且Prometheus的部署非常简单，所以这一讲我们就先来自己动手搭建Prometheus。</p><h2>通用架构回顾</h2><p>还记得我们上一讲介绍的监控系统通用架构吗？我们可以回顾一下。</p><p><img src="https://static001.geekbang.org/resource/image/9e/f5/9edcfef623ea9583134533c3b4c477f5.png?wh=1920x781" alt="图片"></p><p>之所以说 Prometheus 比较容易搭建，是因为它把服务端组件，包括时序库、告警引擎、数据展示三大块，整合成了一个进程，组件的数量大幅减少。Prometheus生态的采集器就是各种Exporter，告警发送靠的是 AlertManager 组件，下面我们先来部署 Prometheus 模块。</p><h2>部署 Prometheus</h2><p>因为生产环境大概率是Linux的，所以我们选择Linux下的发布包，把Prometheus 和 Alertmanager 两个包都下载下来。</p><p><span class="reference">注：Prometheus 的下载地址：<a href="https://prometheus.io/download/">https://prometheus.io/download/</a></span></p><p><img src="https://static001.geekbang.org/resource/image/cf/f7/cf6af0ffc5f3be2867f8a18d0cd254f7.png?wh=1920x1259" alt="图片"></p><p>下载之后解压缩，使用 systemd 托管启动，你可以参考下面的命令。</p><pre><code class="language-bash">mkdir -p /opt/prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.37.1/prometheus-2.37.1.linux-amd64.tar.gz
tar xf prometheus-2.37.1.linux-amd64.tar.gz
cp -far prometheus-2.37.1.linux-amd64/*&nbsp; /opt/prometheus/

# service&nbsp;
cat &lt;&lt;EOF &gt;/etc/systemd/system/prometheus.service
[Unit]
Description="prometheus"
Documentation=https://prometheus.io/
After=network.target

[Service]
Type=simple

ExecStart=/opt/prometheus/prometheus&nbsp; --config.file=/opt/prometheus/prometheus.yml --storage.tsdb.path=/opt/prometheus/data --web.enable-lifecycle --enable-feature=remote-write-receiver --query.lookback-delta=2m --web.enable-admin-api

Restart=on-failure
SuccessExitStatus=0
LimitNOFILE=65536
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=prometheus



[Install]
WantedBy=multi-user.target
EOF

systemctl enable prometheus
systemctl start prometheus
systemctl status prometheus
</code></pre><!-- [[[read_end]]] --><p>这里需要重点关注的是 Prometheus 进程的启动参数，我在每个参数下面都做出了解释，你可以看一下。</p><pre><code class="language-bash">--config.file=/opt/prometheus/prometheus.yml
指定 Prometheus 的配置文件路径

--storage.tsdb.path=/opt/prometheus/data
指定 Prometheus 时序数据的硬盘存储路径

--web.enable-lifecycle
启用生命周期管理相关的 API，比如调用 /-/reload 接口就需要启用该项

--enable-feature=remote-write-receiver
启用 remote write 接收数据的接口，启用该项之后，categraf、grafana-agent 等 agent 就可以通过 /api/v1/write 接口推送数据给 Prometheus

--query.lookback-delta=2m
即时查询在查询当前最新值的时候，只要发现这个参数指定的时间段内有数据，就取最新的那个点返回，这个时间段内没数据，就不返回了

--web.enable-admin-api
启用管理性 API，比如删除时间序列数据的 /api/v1/admin/tsdb/delete_series 接口
</code></pre><p>如果正常启动，Prometheus 默认会在 9090 端口监听，访问这个端口就可以看到 Prometheus 的 Web 页面，输入下面的 PromQL 可以查到一些监控数据。</p><p><img src="https://static001.geekbang.org/resource/image/91/1a/91652cc17442bcb51df6230624d0a21a.png?wh=1920x843" alt="图片"></p><p>这个数据是从哪里来的呢？其实是 Prometheus 自己抓取自己的，Prometheus 会在 <code>/metrics</code> 接口暴露监控数据，你可以访问这个接口看一下输出。同时Prometheus在配置文件里配置了抓取规则，打开 prometheus.yml 就可以看到了。</p><pre><code class="language-bash">scrape_configs:
&nbsp; - job_name: 'prometheus'
&nbsp; &nbsp; static_configs:
&nbsp; &nbsp; - targets: ['localhost:9090']
</code></pre><p>localhost:9090是暴露监控数据的地址，没有指定接口路径，默认使用 <code>/metrics</code>，没有指定 scheme，默认使用 HTTP，所以实际请求的是 <code>http://localhost:9090/metrics</code>。了解了 Prometheus 自监控的方式，下面我们来看一下机器监控。</p><h2>部署 Node-Exporter</h2><p>Prometheus生态的机器监控比较简单，就是在所有的目标机器上部署 Node-Exporter，然后在抓取规则中给出所有 Node-Exporter 的地址就可以了。</p><p>首先，下载 <a href="https://prometheus.io/download/#node_exporter">Node-Exporter</a>。你可以选择当下比较稳定的版本 1.3.1，下载之后解压就可以直接运行了，比如使用 nohup（生产环境建议使用 systemd 托管） 简单启动的话，可以输入下面这一行命令。</p><pre><code class="language-bash">nohup ./node_exporter &amp;&gt; output.log &amp;
</code></pre><p>Node-Exporter 默认的监听端口是9100，我们可以通过下面的命令看到 Node-Exporter 采集的指标。</p><pre><code class="language-bash">curl -s localhost:9100/metrics
</code></pre><p>然后把 Node-Exporter 的地址配置到 prometheus.yml 中即可。修改了配置之后，记得给 Prometheus 发个 HUP 信号，让 Prometheus 重新读取配置：<code>kill -HUP &lt;prometheus pid&gt;</code>。最终 scrape_configs 部分变成下面这段内容。</p><pre><code class="language-bash">scrape_configs:
&nbsp; - job_name: 'prometheus'
&nbsp; &nbsp; static_configs:
&nbsp; &nbsp; - targets: ['localhost:9090']

&nbsp; - job_name: 'node_exporter'
&nbsp; &nbsp; static_configs:
&nbsp; &nbsp; - targets: ['localhost:9100']
</code></pre><p>其中 targets 是个数组，如果要监控更多机器，就在 targets 中写上多个 Node-Exporter 的地址，用逗号隔开。之后在 Prometheus 的 Web 上（菜单位置Status -&gt; Targets），就可以看到相关的 Targets 信息了。</p><p><img src="https://static001.geekbang.org/resource/image/f5/7c/f572c8cf62d7b52668c6fd71cdb7887c.png?wh=1920x529" alt="图片"></p><p>在查询监控数据的框里输入 node，就会自动提示很多 node 打头的指标。这些指标都是 Node-Exporter 采集的，选择其中某一个就可以查到对应的监控数据，比如查看不同硬盘分区的余量大小。</p><p><img src="https://static001.geekbang.org/resource/image/44/fd/44d2bc07de0a62299bb38e910010fbfd.png?wh=1920x590" alt="图片"></p><p>Node-Exporter 默认内置了很多 collector，比如 cpu、loadavg、filesystem 等，可以通过命令行启动参数来控制这些 collector，比如要关掉某个 collector，使用 <code>--no-collector.&lt;name&gt;</code>，如果要开启某个 collector，使用 <code>--collector.&lt;name&gt;</code>。具体可以参考 Node-Exporter 的 <a href="https://github.com/prometheus/node_exporter#collectors">README</a>。Node-Exporter 默认采集几百个指标，有了这些数据，我们就可以演示告警规则的配置了。</p><h2>配置告警规则</h2><p>Prometheus 进程内置了告警判断引擎，prometheus.yml 中可以指定告警规则配置文件，默认配置中有个例子。</p><pre><code class="language-yaml">rule_files:
&nbsp; # - "first_rules.yml"
&nbsp; # - "second_rules.yml"
</code></pre><p>我们可以把不同类型的告警规则拆分到不同的配置文件中，然后在 prometheus.yml 中引用。比如 Node-Exporter 相关的规则，我们命名为 node_exporter.yml，最终这个rule_files就变成了如下配置。</p><pre><code class="language-yaml">rule_files:
&nbsp; - "node_exporter.yml"
</code></pre><p>我设计了一个例子，监控 Node-Exporter 挂掉以及内存使用率超过1%这两种情况。这里我故意设置了一个很小的阈值，确保能够触发告警。</p><pre><code class="language-yaml">groups:
- name: node_exporter
&nbsp; rules:
&nbsp; - alert: HostDown
&nbsp; &nbsp; expr: up{job="node_exporter"} == 0
&nbsp; &nbsp; for: 1m
&nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; severity: critical
&nbsp; &nbsp; annotations:
&nbsp; &nbsp; &nbsp; summary: Host down {{ $labels.instance }}
&nbsp; - alert: MemUtil
&nbsp; &nbsp; expr: 100 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 &gt; 1
&nbsp; &nbsp; for: 1m
&nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; severity: warn
&nbsp; &nbsp; annotations:
&nbsp; &nbsp; &nbsp; summary: Mem usage larger than 1%, instance:{{ $labels.instance }}
</code></pre><p>最后，给 Prometheus 进程发个 HUP 信号，让它重新加载配置文件。</p><pre><code class="language-yaml">kill -HUP `pidof prometheus`
</code></pre><p>之后，我们就可以去 Prometheus 的 Web 上（Alerts菜单）查看告警规则的判定结果了。</p><p><img src="https://static001.geekbang.org/resource/image/bf/b3/bfb533fd9a65c0df2872cea67817f0b3.png?wh=4482x2170" alt="图片"></p><p>我们从图中可以看出，告警分成3个状态，Inactive、Pending、Firing。HostDown这个规则当前是Inactive状态，表示没有触发相关的告警事件，MemUtil这个规则触发了一个事件，处于Firing状态。那什么是Pending状态呢？触发过阈值但是还没有满足持续时长（ for 关键字后面指定的时间段）的要求，就是Pending状态。比如 for 1m，就表示触发阈值的时间持续1分钟才算满足条件，如果规则判定执行频率是10秒，就相当于连续6次都触发阈值才可以。</p><p>在页面上我们看到告警了，就是一个巨大的进步，如果我们还希望在告警的时候收到消息通知，比如邮件、短信等，就需要引入 AlertManager 组件了。</p><h2>部署 Alertmanager</h2><p>部署 Prometheus 的时候，我们已经顺便把 Alertmanager 的包下载下来了，下面我们就安装一下。安装过程很简单，把上面的 prometheus.service 拿过来改一下给 Alertmanager 使用即可，下面是我改好的 alertmanager.service。</p><pre><code class="language-yaml">[Unit]
Description="alertmanager"
After=network.target

[Service]
Type=simple

ExecStart=/usr/local/alertmanager/alertmanager
WorkingDirectory=/usr/local/alertmanager

Restart=on-failure
SuccessExitStatus=0
LimitNOFILE=65536
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=alertmanager



[Install]
WantedBy=multi-user.target
</code></pre><p>我把 Alertmanager 解压到 /usr/local/alertmanager 目录，通过ExecStart可以看出，直接执行二进制就可以，实际Alertmanager会读取二进制同级目录下的 alertmanager.yml 配置文件。我使用163邮箱作为SMTP发件服务器，下面我们来看下具体的配置。</p><pre><code class="language-yaml">global:
  smtp_from: 'username@163.com'
  smtp_smarthost: 'smtp.163.com:465'
  smtp_auth_username: 'username@163.com'
  smtp_auth_password: '这里填写授权码'
  smtp_require_tls: false
  
route:
&nbsp; group_by: ['alertname']
&nbsp; group_wait: 30s
&nbsp; group_interval: 1m
&nbsp; repeat_interval: 1h
&nbsp; receiver: 'email'

receivers:
&nbsp; - name: 'web.hook'
&nbsp; &nbsp; webhook_configs:
&nbsp; &nbsp; &nbsp; - url: 'http://127.0.0.1:5001/'

&nbsp; - name: 'email'
&nbsp; &nbsp; email_configs:
&nbsp; &nbsp; - to: 'ulricqin@163.com'

inhibit_rules:
&nbsp; - source_match:
&nbsp; &nbsp; &nbsp; severity: 'critical'
&nbsp; &nbsp; target_match:
&nbsp; &nbsp; &nbsp; severity: 'warning'
&nbsp; &nbsp; equal: ['alertname', 'dev', 'instance']
</code></pre><p>首先配置一个全局SMTP，然后修改 receivers。receivers 是个数组，默认例子里有个 web.hook，我又加了一个 email 的 receiver，然后配置 route.receiver 字段的值为 email。email_configs中的 to 表示收件人，多个人用逗号分隔，比如 <code>to: 'user1@163.com, user2@163.com'</code>，最后收到的邮件内容大概是这样的，你可以看一下我给出的样例。</p><p><img src="https://static001.geekbang.org/resource/image/bf/d2/bf8a77d0640d97415205c662f50069d2.png?wh=1920x1560" alt="图片"></p><p>收到告警邮件，就说明这整个告警链路走通了。最后我们再看一下数据可视化的问题。Prometheus 自带的看图工具，是给专家用的，需要对指标体系非常了解，经验没法沉淀，而且绘图工具单一，只有折线图。如果你希望有一个更好用的UI工具，可以试试 Grafana。</p><h2>部署 Grafana</h2><p>Grafana 是一个数据可视化工具，有丰富的图表类型，视觉效果很棒，插件式架构，支持各种数据源，是开源监控数据可视化的标杆之作。Grafana可以直接对接Prometheus，大部分使用Prometheus的用户，也都会使用Grafana，下面我们就来部署一下。</p><p>我们可以先把 <a href="https://grafana.com/grafana/download?pg=get&plcmt=selfmanaged-box1-cta1&edition=oss">Grafana</a> 下载下来，它分为两个版本，企业版和开源版，开源版本遵照AGPLV3协议，只要不做二次开发商业化分发，是可以直接使用的。我这里就下载了开源版本，选择 <a href="https://dl.grafana.com/oss/release/grafana-9.1.5.linux-amd64.tar.gz">tar.gz 包</a>，下载之后解压缩，执行 <code>./bin/grafana-server</code> 即可一键启动，Grafana默认的监听端口是3000，访问后就可以看到登录页面了，默认的用户名和密码都是 admin。</p><p>要看图首先要配置数据源，在菜单位置：Configuration -&gt; Data sources，点击 <strong>Add data source</strong> 就能进入数据源类型选择页面，选择Prometheus，填写Prometheus的链接信息，主要是URL，点击 <strong>Save & test</strong> 完成数据源配置。</p><p>Grafana提供了和Prometheus看图页面类似的功能，叫做Explore，我们可以在这个页面点选指标看图。</p><p><img src="https://static001.geekbang.org/resource/image/7d/72/7dd10e1295567613329a7c06eb873872.png?wh=1313x1120" alt="图片"></p><p>但Explore功能不是最核心的，我们使用Grafana，主要是使用Dashboard看图。Grafana社区有很多人制作了各式各样的大盘，以JSON格式上传保存在了 <a href="https://grafana.com/grafana/dashboards/">grafana.com</a>，我们想要某个Dashboard，可以先去这个网站搜索一下，看看是否有人分享过，特别方便。因为我们已经部署了Node-Exporter，那这里就可以直接导入Node-Exporter的大盘，大盘ID是1860，写到图中对应的位置，点击Load，然后选择数据源点击Import即可。</p><p><img src="https://static001.geekbang.org/resource/image/ed/5e/ed4598ac72020b58e03b84152ea2185e.png?wh=1920x1191" alt="图片"></p><p>导入成功的话会自动打开Dashboard，Node-Exporter的大盘长这个样子。</p><p><img src="https://static001.geekbang.org/resource/image/24/b7/24b12129c46b572f84c2f6550cf394b7.png?wh=1920x998" alt="图片"></p><p>走到这个监控看图的部分，我们也走完了整个流程。下面我们对这节课的内容做一个简单总结。</p><h2>小结</h2><p>本讲的核心内容就是演示Prometheus生态相关组件的部署。如果你在课程中是一步一步跟我操作下来的，相信你对Prometheus这套生态就有了入门级的认识。学完这些内容我们再来看一下 Prometheus 的架构图，和监控系统通用架构图相互做一个印证，加深理解。</p><p><img src="https://static001.geekbang.org/resource/image/8e/d6/8e7bcb19da502cbe4cc811f60be871d6.png?wh=1920x1159" alt="图片" title="图片来自官网"></p><p>图上有两个部分我们没有讲到，一个是Pushgateway组件，另一个是Service discovery部分。这里我再做一个简单的补充。</p><ul>
<li>Pushgateway：用于接收短生命周期任务的指标上报，是PUSH的接收方式。因为Prometheus主要是PULL的方式拉取监控数据，这就要求在拉取的时刻，监控对象得活着，但是很多短周期任务，比如cronjob，可能半秒就运行结束了，就没法拉取了。为了应对这种情况，才单独做了Pushgateway组件作为整个生态的补充。</li>
<li>Service discovery：我们演示抓取数据时，是直接在 prometheus.yml 中配置的多个 Targets。这种方式虽然简单直观，但是也有弊端，典型的问题就是如果 Targets 是动态变化的，而且变化得比较频繁，那就会造成管理上的灾难。所以 Prometheus 提供了多种服务发现机制，可以动态获取要监控的目标，比如 Kubernetes 的服务发现，可以通过调用 kube-apiserver 动态获取到需要监控的目标对象，大幅降低了抓取目标的管理成本。</li>
</ul><p>最后，我把这一讲的内容整理了一张脑图，供你理解和记忆。</p><p><img src="https://static001.geekbang.org/resource/image/57/35/57d84b93f63dbc1dc5779ba257c48235.jpg?wh=2379x2174" alt=""></p><h2>互动时刻</h2><p>监控数据的获取，有推（PUSH）拉（PULL）两种模式，不是非黑即白的，不同的场景选择不同的方式，本讲我们在介绍Pushgateway模块时提到了一些选型的依据，你知道推拉两种模式的其他优缺点和适用场景吗？欢迎留言分享，也欢迎你把今天的内容分享给你身边的朋友，邀他一起学习。我们下节课再见！</p><p>点击加入<a href="https://jinshuju.net/f/Ql3qlz">课程交流群</a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4f/b0/ab179368.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hshopeful</span>
  </div>
  <div class="_2_QraFYR_0">关于 pull 和 push 模式，个人的一些理解，有错误或遗漏的，希望老师指正：<br><br>pull 模式的优点：<br>1、pull 模式很容易判断监控对象的存活性，push 模式很难<br>2、pull 模式的监控配置在服务端，比较容易统一控制，push 模式的监控配置在客户端，需要引入类似配置中心的组件，由客户端去拉取监控配置，从配置变更到配置生效的难易程度和时效性，pull 模式占优<br>3、pull 模式可以按需获取监控指标，push 模式只能被动接收监控指标，当然 push 模式也可以在服务端增加指标过滤规则<br>4、pull 模式下，应用与监控系统的耦合比较低，push 模式下，应用于监控系统的耦合性较高<br><br>push 模式的优点：<br>1、针对移动端的监控，只能使用 push 模式，不能使用 pull 模式<br>2、push 模式支持天然的水平扩展，pull 模式水平扩展比较复杂<br>3、push 模式适合短生命周期任务，pull 模式不适合<br>4、pull 模式依赖服务发现模块，push 模式不依赖<br>5、push 模式只用保证网络的正向联通（能够发送数据到服务端），比较简单，而 pull 模式需要保证网络的反向联通（服务端能够抓取多种多样的客户端暴露的接口），相对复杂<br>6、pull 模式需要暴露端口，安全性存在隐患，而 push 模式不存在<br>7、在聚合场景和告警场景下，push 模式的时效性更好</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 挺全面的 👍 <br><br>pull模式优点2，不是太准确，有的时候，即便是pull模式，很多采集配置也是需要放在采集侧的，比如mysqld_exporter，监控系统这边(prometheus.yml)中只需要配置要拉取的mysqld_exporter的地址，但是mysqld_exporter要采集mysql的指标，其实mysqld_exporter里也有不少相关的配置</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-16 15:40:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/3c/88/ff81f846.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凡人</span>
  </div>
  <div class="_2_QraFYR_0">秦老师，你好，文章中 有两个小细节问题<br>1. 文章中没有提到 需要修改 prometheus.yml 中的  # - alertmanager:9093。<br>2. 163邮箱的smtp协议 非ssl端口号是25 (https:&#47;&#47;note.youdao.com&#47;ynoteshare&#47;index.html?id=f9fef46114fb922b45460f4f55d96853) </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍🏻 prometheus.yml中确实需要配置alertmanager的地址，根据自己的环境的情况来配置。smtp确实也是支持ssl和非ssl。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-29 17:01:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/5d/52/21275675.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>隆哥</span>
  </div>
  <div class="_2_QraFYR_0">①部署了prometheus，无法通过IP+9090访问，有可能是防火墙没有关闭，我使用的是CentOS系列<br>systemctl stop firewalld.service<br>systemctl disable firewalld.service<br>②添加node_exporter在Status -&gt; Targets无法查看，prometheus修改配置需要每次重新加载读取<br>systemctl restart prometheus<br>③如何添加报警规则<br>请在应用目录prometheus下创建一个rules文件夹，然后在文件夹中添加node_exporter.yml<br>[root@bogon prometheus]#mkdir -p rules &amp;&amp; cd rules &amp;&amp; vim node_exporter.yml<br>添加规则内容之后，配置prometheus.yml,这样子目录更美观，新重启服务systemctl restart prometheus<br>rule_files: <br>  - &quot;rules&#47;node_exporter.yml&quot;<br>④为什么我grafana控制面板有时候图标会出现断层呢？<br><br>如果觉得教程麻烦，提供一键docker-compose剧本<br>version: &#39;3&#39;<br><br>networks:<br>  monitor:<br>    driver: bridge<br><br>services:<br>  prometheus:<br>    image: prom&#47;prometheus<br>    container_name: prometheus<br>    hostname: prometheus<br>    restart: always<br>    volumes:<br>      - &#47;data&#47;prometheus&#47;data:&#47;prometheus<br>      - &#47;data&#47;prometheus&#47;rules:&#47;etc&#47;prometheus&#47;rules<br>      - &#47;data&#47;prometheus&#47;conf&#47;prometheus.yml:&#47;etc&#47;prometheus&#47;prometheus.yml<br>    ports:<br>      - &quot;9090:9090&quot;<br>    networks:<br>      - monitor<br><br>  alertmanager:<br>    image: prom&#47;alertmanager<br>    container_name: alertmanager<br>    hostname: alertmanager<br>    restart: always<br>    volumes:<br>      - &#47;data&#47;alertmanager&#47;config&#47;alertmanager.yml:&#47;etc&#47;alertmanager&#47;alertmanager.yml<br>    depends_on:<br>      - prometheus<br>    ports:<br>      - &quot;9093:9093&quot;<br>    networks:<br>      - monitor<br><br>  node-exporter:<br>    image: quay.io&#47;prometheus&#47;node-exporter<br>    container_name: node-exporter<br>    hostname: node-exporter<br>    restart: always<br>    environment:<br>      TZ: Asia&#47;Shanghai<br>    volumes:<br>    depends_on:<br>      - prometheus<br>    ports:<br>      - &quot;9100:9100&quot;<br>    networks:<br>      - monitor<br><br>  grafana:<br>    image: grafana&#47;grafana<br>    container_name: grafana<br>    hostname: grafana<br>    restart: always<br>    volumes:<br>      - &#47;data&#47;grafana&#47;data:&#47;var&#47;lib&#47;grafana<br>    depends_on:<br>      - prometheus<br>    ports:<br>      - &quot;3000:3000&quot;<br>    networks:<br>      - monitor</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍<br><br>数据断了，可能是没有采集到，使用range query，查一下具体的指标，看看数据是否连续，比如 cpu_usage_idle[10m]，使用Table视图查看，就能看到10分钟内具体的监控数据是什么，根据时间戳可以看出数据是否连续</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-16 16:42:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/e9/26/afc08398.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Amosヾ</span>
  </div>
  <div class="_2_QraFYR_0">关于告警有一些建议和疑问：<br>1、告警分级(提示、严重、致命)全部邮件推送很难体现优先级，尤其是致命告警应该电话处理<br>2、目前社区是否有成熟的告警推送工具，比如企业微信机器人、钉钉机器人，这些需要自行编码实现？电话告警该如何实现呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第8讲介绍的Nightingale内置钉钉，飞书，企微机器人的推送方式，好像还有个prometheus-alert工具可以去github搜一下支持挺多告警媒介。电话告警的话，一般有个电话通道API，但是不像邮件有smtp协议级别的规范，所以一般需要自定义一小段代码做电话告警</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-22 18:46:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/e3/12/fd02db2e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DBRE</span>
  </div>
  <div class="_2_QraFYR_0">Prometheus没有提供很多管理上的API,又不想引入Service Discovery, 在targets变化后直接操作prometheus.conf有什么更为简单的方式吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 使用基于文件的发现方式，然后通过配置管理工具，或者自己手撸，管理targets配置文件，targets配置文件变化之后，prometheus会自动重新加载，这种方式最为简单</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-16 08:42:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/32/45/b2/701f5ad7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mark.Q</span>
  </div>
  <div class="_2_QraFYR_0">yaml文件 建议老老实实敲空格，尤其是到了一个新公司不熟悉机器配置的时候。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-28 15:06:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/5d/52/21275675.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>隆哥</span>
  </div>
  <div class="_2_QraFYR_0">为啥不用docker形式来做实践呢，我觉得systemctl这种托管还是对主机入侵比较大。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这都是落地细节了，其实没有那么关键，二进制的方式能够让大家了解更多细节，容器的话，很多公司其实没有在用。<br><br>通常来讲，懂得如何使用二进制部署，一般也能自行搞定容器；懂得容器部署，未必能搞定二进制方式部署。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-16 09:17:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_df0d4d</span>
  </div>
  <div class="_2_QraFYR_0">请问生产上有什么prometheus的高可用方案吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 国内来看，用VictoriaMetrics和Thanos作为Prometheus增强方案的居多</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-06 21:38:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/ac/aa/2f117918.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>孙宇翔</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，我想监控像摄像机这类设备的在线、离线，还有流量等数据，普罗米修斯是否合适，配置里面该怎么设置，完全没头绪。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要看看这些设备是怎么暴露监控数据的，摄像机我不知道，比如打印机是走snmp，就通过snmp的采集插件去拉数据就可以了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-02 14:54:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEKSVuNarJuDhBSvHY0giaq6yriceEBKiaKuc04wCYWOuso50noqDexaPJJibJN7PHwvcQppnzsDia1icZkw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Matthew</span>
  </div>
  <div class="_2_QraFYR_0">老师，文章中提到的几个软件安装包，都需要从 github 上下载，但是下载速度太慢。能否提供下国内镜像源的地址呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 国内估计没有</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-06 01:42:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEKSVuNarJuDhBSvHY0giaq6yriceEBKiaKuc04wCYWOuso50noqDexaPJJibJN7PHwvcQppnzsDia1icZkw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Matthew</span>
  </div>
  <div class="_2_QraFYR_0">github 下载太慢，有啥好办法吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 找找🪜</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-03 23:47:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/13/67/910fb1dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leeeo</span>
  </div>
  <div class="_2_QraFYR_0">源码编译prometheus可以去掉ui部分吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理论上可以，这块我没研究</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-03 11:13:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/a1/69/0af5e082.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>顶级心理学家</span>
  </div>
  <div class="_2_QraFYR_0">秦老师，有个问题是关于exporter多实例的问题。<br>我们环境是prometheus operator，例如 redis采用redis-exporter采集。这样就造成多个redis实例，实现成了多个exporter 来采集。redis exporter有案例支持一个exporter采集多个实例，但是没能实现。<br>目前就是一个exporter 启动多个container，用不同端口采集多个redis实例，这样container会随着redis实例增多而庞大。<br>想听听您的建议和想法。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果是资源这块的考虑，我觉得还好，从原理上来看，redis-exporter采集数据的时候无非就是连上redis然后执行info命令，不需要太多资源消耗，可以给exporter容器分配少点资源</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-18 09:18:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/41/23/26f8f45a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>俺木饭 三克油😂😂</span>
  </div>
  <div class="_2_QraFYR_0">老师问下如何做prom 和各类exporter之间的认证？比如公司安全需求所有接口需要做鉴权</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: prometheus拉取目标&#47;metrics数据的时候可以开启basic auth认证，当然，这需要exporter本身也支持basic auth认证，很遗憾貌似大部分exporter都不支持呢。更多相关知识可以Google关键词：prometheus pull basic auth。或者更简单的，使用后文介绍的categraf做采集器，更容易达成你们的安全要求</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-17 18:59:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ce/d7/8168e1bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小飞同学</span>
  </div>
  <div class="_2_QraFYR_0">咨询下grafana端口外网通过ip无法访问可能是什么原因呢？3000端口在安全组已经放开了，防火墙也方阿奎了，另外Prometheus9090端口是正常用的。http:&#47;&#47;ip:3000&#47;login   <br>Request URL: http:&#47;&#47;39.107.88.69:3000&#47;login<br>Referrer Policy: strict-origin-when-cross-origin<br>Provisional headers are shown<br>Learn more<br>Upgrade-Insecure-Requests: 1</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你去部署grafana的机器上，用curl -L localhost:3000看看是否能正常返回，如果正常，说明grafana本身没问题，一定是网路问题，如果curl都不正常，一定是grafana自身就有问题</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-17 17:52:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/32/4b/33/48b278a4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>SEVEN</span>
  </div>
  <div class="_2_QraFYR_0">老师。如果想把启动用户改成prometheus，那systemd文件该怎么写？<br>[Unit]<br>Description=&quot;prometheus&quot;<br>Documentation=https:&#47;&#47;prometheus.io&#47;<br>After=network.target<br>[Service]<br>Type=simple<br>User=prometheus<br>Group=prometheus<br>WorkingDirectory=&#47;app&#47;prometheus&#47;prometheus&#47;<br>ExecStart=&#47;app&#47;prometheus&#47;prometheus&#47;prometheus  --config.file=&#47;app&#47;prometheus&#47;prometheus&#47;prometheus.yml --storage.tsdb.path=&#47;app&#47;prometheus&#47;prometheus&#47;data --web.enable-lifecycle --enable-feature=remote-write-receiver --query.lookback-delta=2m --web.enable-admin-api<br>Restart=on-failure<br>SuccessExitStatus=0<br>LimitNOFILE=65536<br>StandardOutput=syslog<br>StandardError=syslog<br>SyslogIdentifier=prometheus<br>[Install]<br>WantedBy=multi-user.target<br>我这么写但是启动失败<br>我们生产中对root权限的管理很严格。建议老师后面的课程能尽量以prometheus普通用户的权限讲。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个检查方法很简单，你就手工模仿systemd的逻辑就行了，先切到prometheus用户，然后手工执行ExecStart指定的那行命令，看看报什么错就知道了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-17 16:49:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/5e/45/50424a7a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>叶夏</span>
  </div>
  <div class="_2_QraFYR_0">请问会讲Prometheus的高可用吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-17 10:48:30</div>
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
  <div class="_2_QraFYR_0">请问老师，在k8s里部署prometheus，有哪些比较好的方式呢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有这么几个方式可以看看选择一下：<br>* https:&#47;&#47;prometheus-operator.dev&#47;<br>* https:&#47;&#47;devapo.io&#47;blog&#47;technology&#47;how-to-set-up-prometheus-on-kubernetes-with-helm-charts&#47;<br>* https:&#47;&#47;phoenixnap.com&#47;kb&#47;prometheus-kubernetes</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-17 09:50:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/39/22/8437fd56.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>embeder</span>
  </div>
  <div class="_2_QraFYR_0">老师我想问问，这个课程展示大盘使用夜莺来完成的吗？业务需要收集整个机房所有机器的运行日志，公司现在用的是zabbix监控，我想把收集日志和监控系统一起搞一下。我觉得夜莺就挺适合我们的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，大盘使用夜莺的格式搞的，每个图的核心其实是那个promql，只要了解了promql，用其他的绘图工具也是可以的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-17 09:16:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4f/b0/ab179368.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hshopeful</span>
  </div>
  <div class="_2_QraFYR_0">秦老师，另外还有一个问题想请教下：<br>a、关于配置 SMTP 发件服务器的相关信息，使用 163 或者 qq 邮箱，网上能够找到 smtp_smarthost 的配置 smtp.163.com:465 和 smtp.qq.com:465，另外也能找到怎么去获取授权码。<br>b、使用这些外部邮箱作为发件服务器，可以向公司的邮箱点对点发送告警邮件，但是不能向公司的邮件组发送告警邮件（公司的邮件组默认设置的规则是拒收外部邮箱的邮件）<br>c、怎么去获取公司邮件服务器的 smtp_smarthost 的配置和相关授权码，老师能给下建议吗？公司内部的邮件服务器不知道谁维护的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 公司内部的smtp一般不需要授权码，就直接配置用户名密码就行了，就类似你的outlook里的配置</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-16 15:53:53</div>
  </div>
</div>
</div>
</li>
</ul>