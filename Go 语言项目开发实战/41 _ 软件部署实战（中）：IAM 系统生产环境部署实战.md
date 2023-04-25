<audio title="41 _ 软件部署实战（中）：IAM 系统生产环境部署实战" src="https://static001.geekbang.org/resource/audio/d1/1a/d176cc8dce95c7f1104ee245e586ed1a.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>上一讲，我介绍了IAM部署用到的两个核心组件，Nginx和Keepalived。那么这一讲，我们就来看下，如何使用Nginx和Keepalived来部署一个高可用的IAM应用。下一讲，我再介绍下IAM应用安全和弹性伸缩能力的构建方式。</p><p>这一讲，我们会通过下面四个步骤来部署IAM应用：</p><ol>
<li>在服务器上部署IAM应用中的服务。</li>
<li>配置Nginx，实现反向代理功能。通过反向代理，我们可以通过Nginx来访问部署在内网的IAM服务。</li>
<li>配置Nginx，实现负载均衡功能。通过负载均衡，我们可以实现服务的水平扩缩容，使IAM应用具备高可用能力。</li>
<li>配置Keepalived，实现Nginx的高可用。通过Nginx + Keepalived的组合，可以实现整个应用架构的高可用。</li>
</ol><h2>部署IAM应用</h2><p>部署一个高可用的IAM应用，需要至少两个节点。所以，我们按照先后顺序，分别在<code>10.0.4.20</code>和<code>10.0.4.21</code>服务器上部署IAM应用。</p><h3>在<code>10.0.4.20</code>服务器上部署IAM应用</h3><p>首先，我来介绍下如何在<code>10.0.4.20</code>服务器上部署IAM应用。</p><p>我们要在这个服务器上部署如下组件：</p><ul>
<li>iam-apiserver</li>
<li>iam-authz-server</li>
<li>iam-pump</li>
<li>MariaDB</li>
<li>Redis</li>
<li>MongoDB</li>
</ul><!-- [[[read_end]]] --><p>这些组件的部署方式，<a href="https://time.geekbang.org/column/article/378082">03讲</a> 有介绍，这里就不再说明。</p><p>此外，我们还需要设置MariaDB，给来自于<code>10.0.4.21</code>服务器的数据库连接授权，授权命令如下：</p><pre><code class="language-bash">$ mysql -hlocalhost -P3306 -uroot -proot # 先以root用户登陆数据库
MariaDB [(none)]&gt; grant all on iam.* TO iam@10.0.4.21 identified by 'iam1234';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]&gt; flush privileges;
Query OK, 0 rows affected (0.000 sec)
</code></pre><h3>在<code>10.0.4.21</code>服务器上部署IAM应用</h3><p>然后，在<code>10.0.4.21</code>服务器上安装好iam-apiserver、iam-authz-server 和 iam-pump。这些组件通过<code>10.0.4.20</code> IP地址，连接<code>10.0.4.20</code>服务器上的MariaDB、Redis和MongoDB。</p><h2>配置Nginx作为反向代理</h2><p>假定要访问的API Server和IAM Authorization Server的域名分别为<code>iam.api.marmotedu.com</code>和<code>iam.authz.marmotedu.com</code>，我们需要分别为iam-apiserver和iam-authz-server配置Nginx反向代理。整个配置过程可以分为5步（在<code>10.0.4.20</code>服务器上操作）。</p><p><strong>第一步，配置iam-apiserver。</strong></p><p>新建Nginx配置文件<code>/etc/nginx/conf.d/iam-apiserver.conf</code>，内容如下：</p><pre><code class="language-plain">server {
&nbsp; &nbsp; listen&nbsp; &nbsp; &nbsp; &nbsp;80;
&nbsp; &nbsp; server_name&nbsp; iam.api.marmotedu.com;
&nbsp; &nbsp; root&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;/usr/share/nginx/html;
&nbsp; &nbsp; location / {
&nbsp; &nbsp; 	proxy_set_header X-Forwarded-Host $http_host;
&nbsp; &nbsp; 	proxy_set_header X-Real-IP $remote_addr;
&nbsp; &nbsp; 	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
&nbsp; &nbsp; 	proxy_pass&nbsp; http://127.0.0.1:8080/;
&nbsp; &nbsp; 	client_max_body_size 5m;
&nbsp; &nbsp; }

&nbsp; &nbsp; error_page 404 /404.html;
&nbsp; &nbsp; &nbsp; &nbsp; location = /40x.html {
&nbsp; &nbsp; }

&nbsp; &nbsp; error_page 500 502 503 504 /50x.html;
&nbsp; &nbsp; &nbsp; &nbsp; location = /50x.html {
&nbsp; &nbsp; }
}
</code></pre><p>有几点你在配置时需要注意，这里说明下。</p><ul>
<li><code>server_name</code>需要为<code>iam.api.marmotedu.com</code>，我们通过<code>iam.api.marmotedu.com</code>访问iam-apiserver。</li>
<li>iam-apiserver默认启动的端口为<code>8080</code>。</li>
<li>由于Nginx默认允许客户端请求的最大单文件字节数为<code>1MB</code>，实际生产环境中可能太小，所以这里将此限制改为5MB（<code>client_max_body_size 5m</code>）。如果需要上传图片之类的，可能需要设置成更大的值，比如<code>50m</code>。</li>
<li>server_name用来说明访问Nginx服务器的域名，例如<code>curl -H 'Host: iam.api.marmotedu.com' http://x.x.x.x:80/healthz</code>，<code>x.x.x.x</code>为Nginx服务器的IP地址。</li>
<li>proxy_pass表示反向代理的路径。因为这里是本机的iam-apiserver服务，所以IP为<code>127.0.0.1</code>。端口要和API服务端口一致，为<code>8080</code>。</li>
</ul><p>最后还要提醒下，因为 Nginx 配置选项比较多，跟实际需求和环境有关，所以这里的配置是基础的、未经优化的配置，在实际生产环境中需要你再做调节。</p><p><strong>第二步，配置iam-authz-server。</strong></p><p>新建Nginx配置文件<code>/etc/nginx/conf.d/iam-authz-server.conf</code>，内容如下：</p><pre><code class="language-plain">server {
&nbsp; &nbsp; listen&nbsp; &nbsp; &nbsp; &nbsp;80;
&nbsp; &nbsp; server_name&nbsp; iam.authz.marmotedu.com;
&nbsp; &nbsp; root&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;/usr/share/nginx/html;
&nbsp; &nbsp; location / {
&nbsp; &nbsp; 	proxy_set_header X-Forwarded-Host $http_host;
&nbsp; &nbsp; 	proxy_set_header X-Real-IP $remote_addr;
&nbsp; &nbsp; 	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
&nbsp; &nbsp; 	proxy_pass&nbsp; http://127.0.0.1:9090/;
&nbsp; &nbsp; 	client_max_body_size 5m;
&nbsp; &nbsp; }

&nbsp; &nbsp; error_page 404 /404.html;
&nbsp; &nbsp; &nbsp; &nbsp; location = /40x.html {
&nbsp; &nbsp; }

&nbsp; &nbsp; error_page 500 502 503 504 /50x.html;
&nbsp; &nbsp; &nbsp; &nbsp; location = /50x.html {
&nbsp; &nbsp; }
}
</code></pre><p>下面是一些配置说明。</p><ul>
<li>server_name需要为<code>iam.authz.marmotedu.com</code>，我们通过<code>iam.authz.marmotedu.com</code>访问iam-authz-server。</li>
<li>iam-authz-server默认启动的端口为<code>9090</code>。</li>
<li>其他配置跟<code>/etc/nginx/conf.d/iam-apiserver.conf</code>一致。</li>
</ul><p><strong>第三步，配置完Nginx后，重启Nginx：</strong></p><pre><code class="language-bash">$ sudo systemctl restart nginx
</code></pre><p><strong>第四步，在/etc/hosts中追加下面两行：</strong></p><pre><code class="language-bash">127.0.0.1 iam.api.marmotedu.com
127.0.0.1 iam.authz.marmotedu.com
</code></pre><p><strong>第五步，发送HTTP请求：</strong></p><pre><code class="language-bash">$ curl http://iam.api.marmotedu.com/healthz
{"status":"ok"}
$ curl http://iam.authz.marmotedu.com/healthz
{"status":"ok"}
</code></pre><p>我们分别请求iam-apiserver和iam-authz-server的健康检查接口，输出了<code>{"status":"ok"}</code>，说明我们可以成功通过代理访问后端的API服务。</p><p>在用curl请求<code>http://iam.api.marmotedu.com/healthz</code>后，后端的请求流程实际上是这样的：</p><ol>
<li>因为在<code>/etc/hosts</code>中配置了<code>127.0.0.1 iam.api.marmotedu.com</code>，所以请求<code>http://iam.api.marmotedu.com/healthz</code>实际上是请求本机的Nginx端口（<code>127.0.0.1:80</code>）。</li>
<li>Nginx在收到请求后，会解析请求，得到请求域名为<code>iam.api.marmotedu.com</code>。根据请求域名去匹配 Nginx的server配置，匹配到<code>server_name  iam.api.marmotedu.com;</code>配置。</li>
<li>匹配到server后，把请求转发到该server的<code>proxy_pass</code>路径。</li>
<li>等待API服务器返回结果，并返回客户端。</li>
</ol><h2>配置Nginx作为负载均衡</h2><p>这门课采用Nginx轮询的负载均衡策略转发请求。负载均衡需要至少两台服务器，所以会分别在<code>10.0.4.20</code>和<code>10.0.4.21</code>服务器上执行相同的操作。下面我分别来介绍下如何配置这两台服务器，并验证配置是否成功。</p><h3><code>10.0.4.20</code>服务器配置</h3><p>登陆<code>10.0.4.20</code>服务器，在<code>/etc/nginx/nginx.conf</code>中添加upstream配置，配置过程可以分为3步。</p><p><strong>第一步，在</strong><code>/etc/nginx/nginx.conf</code><strong>中添加upstream：</strong></p><pre><code class="language-plain">http {
&nbsp; &nbsp; log_format&nbsp; main&nbsp; '$remote_addr - $remote_user [$time_local] "$request" '
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; '$status $body_bytes_sent "$http_referer" '
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; '"$http_user_agent" "$http_x_forwarded_for"';

&nbsp; &nbsp; access_log&nbsp; /var/log/nginx/access.log&nbsp; main;

&nbsp; &nbsp; sendfile&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; on;
&nbsp; &nbsp; tcp_nopush&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; on;
&nbsp; &nbsp; tcp_nodelay&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;on;
&nbsp; &nbsp; keepalive_timeout&nbsp; &nbsp;65;
&nbsp; &nbsp; types_hash_max_size 2048;

&nbsp; &nbsp; include&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;/etc/nginx/mime.types;
&nbsp; &nbsp; default_type&nbsp; &nbsp; &nbsp; &nbsp; application/octet-stream;

&nbsp; &nbsp; # Load modular configuration files from the /etc/nginx/conf.d directory.
&nbsp; &nbsp; # See http://nginx.org/en/docs/ngx_core_module.html#include
&nbsp; &nbsp; # for more information.
&nbsp; &nbsp; include /etc/nginx/conf.d/*.conf;
&nbsp; &nbsp; upstream iam.api.marmotedu.com {
&nbsp; &nbsp; &nbsp; &nbsp; server 127.0.0.1:8080
&nbsp; &nbsp; &nbsp; &nbsp; server 10.0.4.21:8080
&nbsp; &nbsp; }
&nbsp; &nbsp; upstream iam.authz.marmotedu.com {
&nbsp; &nbsp; &nbsp; &nbsp; server 127.0.0.1:9090
&nbsp; &nbsp; &nbsp; &nbsp; server 10.0.4.21:9090
&nbsp; &nbsp; }
}
</code></pre><p>配置说明：</p><ul>
<li>upstream是配置在<code>/etc/nginx/nginx.conf</code>文件中的<code>http{ … }</code>部分的。</li>
<li>因为我们要分别为iam-apiserver和iam-authz-server配置负载均衡，所以我们创建了两个upstream，分别是<code>iam.api.marmotedu.com</code>和<code>iam.authz.marmotedu.com</code>。为了便于识别，upstream名称和域名最好保持一致。</li>
<li>在upstream中，我们需要分别添加所有的iam-apiserver和iam-authz-server的后端（<code>ip:port</code>），本机的后端为了访问更快，可以使用<code>127.0.0.1:&lt;port&gt;</code>，其他机器的后端，需要使用<code>&lt;内网&gt;:port</code>，例如<code>10.0.4.21:8080</code>、<code>10.0.4.21:9090</code>。</li>
</ul><p><strong>第二步，修改proxy_pass。</strong></p><p>修改<code>/etc/nginx/conf.d/iam-apiserver.conf</code>文件，将<code>proxy_pass</code>修改为：</p><pre><code class="language-plain">proxy_pass http://iam.api.marmotedu.com/;
</code></pre><p>修改<code>/etc/nginx/conf.d/iam-authz-server.conf</code>文件，将<code>proxy_pass</code>修改为：</p><pre><code class="language-plain">proxy_pass http://iam.authz.marmotedu.com/;
</code></pre><p>当Nginx转发到<code>http://iam.api.marmotedu.com/</code>域名时，会从<code>iam.api.marmotedu.com</code> upstream配置的后端列表中，根据负载均衡策略选取一个后端，并将请求转发过去。转发<code>http://iam.authz.marmotedu.com/</code>域名的逻辑也一样。</p><p><strong>第三步，配置完Nginx后，重启Nginx：</strong></p><pre><code class="language-bash">$ sudo systemctl restart nginx
</code></pre><p>最终配置好的配置文件，你可以参考下面这些（保存在<a href="https://github.com/marmotedu/iam/tree/v1.0.8/configs/ha/10.0.4.20">configs/ha/10.0.4.20</a>目录下）：</p><ul>
<li>nginx.conf：<a href="https://github.com/marmotedu/iam/blob/v1.0.8/configs/ha/10.0.4.20/nginx.conf">configs/ha/10.0.4.20/nginx.conf</a>。</li>
<li>iam-apiserver.conf：<a href="https://github.com/marmotedu/iam/blob/v1.0.8/configs/ha/10.0.4.20/iam-apiserver.conf">configs/ha/10.0.4.20/iam-apiserver.conf</a>。</li>
<li>iam-authz-server.conf：<a href="https://github.com/marmotedu/iam/blob/v1.0.8/configs/ha/10.0.4.20/iam-authz-server.conf">configs/ha/10.0.4.20/iam-authz-server.conf</a>。</li>
</ul><h3><code>10.0.4.21</code>服务器配置</h3><p>登陆<code>10.0.4.21</code>服务器，在<code>/etc/nginx/nginx.conf</code>中添加upstream配置。配置过程可以分为下面4步。</p><p><strong>第一步，在</strong><code>/etc/nginx/nginx.conf</code><strong>中添加upstream：</strong></p><pre><code class="language-plain">http {
&nbsp; &nbsp; log_format&nbsp; main&nbsp; '$remote_addr - $remote_user [$time_local] "$request" '
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; '$status $body_bytes_sent "$http_referer" '
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; '"$http_user_agent" "$http_x_forwarded_for"';

&nbsp; &nbsp; access_log&nbsp; /var/log/nginx/access.log&nbsp; main;

&nbsp; &nbsp; sendfile&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; on;
&nbsp; &nbsp; tcp_nopush&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; on;
&nbsp; &nbsp; tcp_nodelay&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;on;
&nbsp; &nbsp; keepalive_timeout&nbsp; &nbsp;65;
&nbsp; &nbsp; types_hash_max_size 2048;

&nbsp; &nbsp; include&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;/etc/nginx/mime.types;
&nbsp; &nbsp; default_type&nbsp; &nbsp; &nbsp; &nbsp; application/octet-stream;

&nbsp; &nbsp; # Load modular configuration files from the /etc/nginx/conf.d directory.
&nbsp; &nbsp; # See http://nginx.org/en/docs/ngx_core_module.html#include
&nbsp; &nbsp; # for more information.
&nbsp; &nbsp; include /etc/nginx/conf.d/*.conf;
&nbsp; &nbsp; upstream iam.api.marmotedu.com {
&nbsp; &nbsp; &nbsp; &nbsp; server 127.0.0.1:8080
&nbsp; &nbsp; &nbsp; &nbsp; server 10.0.4.20:8080
&nbsp; &nbsp; }
&nbsp; &nbsp; upstream iam.authz.marmotedu.com {
&nbsp; &nbsp; &nbsp; &nbsp; server 127.0.0.1:9090
&nbsp; &nbsp; &nbsp; &nbsp; server 10.0.4.20:9090
&nbsp; &nbsp; }
}
</code></pre><p>upstream中，需要配置<code>10.0.4.20</code>服务器上的iam-apiserver和iam-authz-server的后端，例如<code>10.0.4.20:8080</code>、<code>10.0.4.20:9090</code>。</p><p><strong>第二步，创建</strong><code>/etc/nginx/conf.d/iam-apiserver.conf</code><strong>文件</strong>（iam-apiserver的反向代理+负载均衡配置），内容如下：</p><pre><code class="language-plain">server {
&nbsp; &nbsp; listen&nbsp; &nbsp; &nbsp; &nbsp;80;
&nbsp; &nbsp; server_name&nbsp; iam.api.marmotedu.com;
&nbsp; &nbsp; root&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;/usr/share/nginx/html;
&nbsp; &nbsp; location / {
&nbsp; &nbsp; 	proxy_set_header X-Forwarded-Host $http_host;
&nbsp; &nbsp; 	proxy_set_header X-Real-IP $remote_addr;
&nbsp; &nbsp; 	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
&nbsp; &nbsp; 	proxy_pass&nbsp; http://iam.api.marmotedu.com/;
&nbsp; &nbsp; 	client_max_body_size 5m;
&nbsp; &nbsp; }

&nbsp; &nbsp; error_page 404 /404.html;
&nbsp; &nbsp; &nbsp; &nbsp; location = /40x.html {
&nbsp; &nbsp; }

&nbsp; &nbsp; error_page 500 502 503 504 /50x.html;
&nbsp; &nbsp; &nbsp; &nbsp; location = /50x.html {
&nbsp; &nbsp; }
}
</code></pre><p><strong>第三步，创建</strong><code>/etc/nginx/conf.d/iam-authz-server</code><strong>文件</strong>（iam-authz-server的反向代理+负载均衡配置），内容如下：</p><pre><code class="language-plain">server {
&nbsp; &nbsp; listen&nbsp; &nbsp; &nbsp; &nbsp;80;
&nbsp; &nbsp; server_name&nbsp; iam.authz.marmotedu.com;
&nbsp; &nbsp; root&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;/usr/share/nginx/html;
&nbsp; &nbsp; location / {
&nbsp; &nbsp; 	proxy_set_header X-Forwarded-Host $http_host;
&nbsp; &nbsp; 	proxy_set_header X-Real-IP $remote_addr;
&nbsp; &nbsp; 	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
&nbsp; &nbsp; 	proxy_pass&nbsp; http://iam.authz.marmotedu.com/;
&nbsp; &nbsp; 	client_max_body_size 5m;
&nbsp; &nbsp; }

&nbsp; &nbsp; error_page 404 /404.html;
&nbsp; &nbsp; &nbsp; &nbsp; location = /40x.html {
&nbsp; &nbsp; }

&nbsp; &nbsp; error_page 500 502 503 504 /50x.html;
&nbsp; &nbsp; &nbsp; &nbsp; location = /50x.html {
&nbsp; &nbsp; }
}

</code></pre><p><strong>第四步，配置完Nginx后，重启Nginx：</strong></p><pre><code class="language-bash">$ sudo systemctl restart nginx
</code></pre><p>最终配置好的配置文件，你可以参考下面这些（保存在<a href="https://github.com/marmotedu/iam/tree/v1.0.8/configs/ha/10.0.4.21">configs/ha/10.0.4.21</a>目录下）：</p><ul>
<li>nginx.conf：<a href="https://github.com/marmotedu/iam/blob/v1.0.8/configs/ha/10.0.4.21/nginx.conf">configs/ha/10.0.4.21/nginx.conf</a>。</li>
<li>iam-apiserver.conf：<a href="https://github.com/marmotedu/iam/blob/v1.0.8/configs/ha/10.0.4.21/iam-apiserver.conf">configs/ha/10.0.4.21/iam-apiserver.conf</a>。</li>
<li>iam-authz-server.conf：<a href="https://github.com/marmotedu/iam/blob/v1.0.8/configs/ha/10.0.4.21/iam-authz-server.conf">configs/ha/10.0.4.21/iam-authz-server.conf</a>。</li>
</ul><h3>测试负载均衡</h3><p>上面，我们配置了Nginx负载均衡器，这里我们还需要测试下是否配置成功。</p><p><strong>第一步，执行测试脚本（</strong><a href="https://github.com/marmotedu/iam/blob/v1.0.8/test/nginx/loadbalance.sh">test/nginx/loadbalance.sh</a><strong>）：</strong></p><pre><code class="language-bash">#!/usr/bin/env bash

for domain in iam.api.marmotedu.com iam.authz.marmotedu.com
do
  for n in $(seq 1 1 10)
  do
    echo $domain
    nohup curl http://${domain}/healthz &amp;&gt;/dev/null &amp;
  done
done
</code></pre><p><strong>第二步，分别查看iam-apiserver和iam-authz-server的日志。</strong></p><p>这里我展示下iam-apiserver的日志（iam-authz-server的日志你可自行查看）。</p><p><code>10.0.4.20</code>服务器的iam-apiserver日志如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/58/26/58d072c92552fa3068e3ef3acd0ed726.png?wh=1920x498" alt="图片"></p><p><code>10.0.4.21</code>服务器的iam-apiserver日志如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/19/85/199066c65ff60007f80f3c2dyy11c785.png?wh=1920x482" alt="图片"></p><p>通过上面两张图，你可以看到<code>10.0.4.20</code>和<code>10.0.4.21</code>各收到<code>5</code>个<code>/healthz</code>请求，说明负载均衡配置成功。</p><h2>配置Keepalived</h2><p>在 <a href="https://time.geekbang.org/column/article/411663">40讲</a>，我们分别在<code>10.0.4.20</code>和<code>10.0.4.21</code>服务器上安装了Keepalived。这里，我来介绍下如何配置Keepalived，实现Nginx的高可用。为了避免故障恢复时，VIP切换造成的服务延时，这一讲采用Keepalived的非抢占模式。</p><p>配置Keepalived的流程比较复杂，分为创建腾讯云HAVIP、主服务器配置、备服务器配置、测试Keepalived、VIP绑定公网IP和测试公网访问六大步，每一步中都有很多小步骤，下面我们来一步步地看下。</p><h3><strong>第一步：创建腾讯云HAVIP</strong></h3><p>公有云厂商的普通内网IP，出于安全考虑（如避免ARP欺骗等），不支持主机通过ARP宣告IP 。如果用户直接在<code>keepalived.conf</code>文件中指定一个普通内网IP为virtual IP，当Keepalived将virtual IP从MASTER机器切换到BACKUP机器时，将无法更新IP和MAC地址的映射，而需要调API来进行IP切换。所以，这里的VIP需要申请腾讯云的HAVIP。</p><p>申请的流程可以分为下面4步：</p><ol>
<li>登录私有网络控制台<strong>。</strong></li>
<li>在左侧导航栏中，选择【IP与网卡】&gt;【高可用虚拟IP】。</li>
<li>在HAVIP管理页面，选择所在地域，单击【申请】。</li>
<li>在弹出的【申请高可用虚拟IP】对话框中输入名称，选择HAVIP所在的私有网络和子网等信息，单击【确定】即可。</li>
</ol><p>这里选择的私有网络和子网，需要和<code>10.0.4.20</code>、<code>10.0.4.21</code>相同。HAVIP 的 IP 地址可以自动分配，也可以手动填写，这里我们手动填写为10.0.4.99。申请页面如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/a4/11/a49d6e7e080d658392dbb144a1560811.png?wh=827x1016" alt="图片"></p><h3>第二步：主服务器配置</h3><p>进行主服务器配置，可以分为两步。</p><p><strong>首先，修改Keepalived配置文件。</strong></p><p>登陆服务器<code>10.0.4.20</code>，编辑<code>/etc/keepalived/keepalived.conf</code>，修改配置，修改后配置内容如下（参考：<a href="https://github.com/marmotedu/iam/blob/v1.0.8/configs/ha/10.0.4.20/keepalived.conf">configs/ha/10.0.4.20/keepalived.conf</a>）：</p><pre><code class="language-bash"># 全局定义，定义全局的配置选项
global_defs {
# 指定keepalived在发生切换操作时发送email，发送给哪些email
# 建议在keepalived_notify.sh中发送邮件
  notification_email {
    acassen@firewall.loc
  }
  notification_email_from Alexandre.Cassen@firewall.loc # 发送email时邮件源地址
    smtp_server 192.168.200.1 # 发送email时smtp服务器地址
    smtp_connect_timeout 30 # 连接smtp的超时时间
    router_id VM-4-20-centos # 机器标识，通常可以设置为hostname
    vrrp_skip_check_adv_addr # 如果接收到的报文和上一个报文来自同一个路由器，则不执行检查。默认是跳过检查
    vrrp_garp_interval 0 # 单位秒，在一个网卡上每组gratuitous arp消息之间的延迟时间，默认为0
    vrrp_gna_interval 0 # 单位秒，在一个网卡上每组na消息之间的延迟时间，默认为0
}
# 检测脚本配置
vrrp_script checkhaproxy
{
  script "/etc/keepalived/check_nginx.sh" # 检测脚本路径
    interval 5 # 检测时间间隔（秒）
    weight 0 # 根据该权重改变priority，当值为0时，不改变实例的优先级
}
# VRRP实例配置
vrrp_instance VI_1 {
  state BACKUP  # 设置初始状态为'备份'
    interface eth0 # 设置绑定VIP的网卡，例如eth0
    virtual_router_id 51  # 配置集群VRID，互为主备的VRID需要是相同的值
    nopreempt               # 设置非抢占模式，只能设置在state为backup的节点上
    priority 100 # 设置优先级，值范围0～254，值越大优先级越高，最高的为master
    advert_int 1 # 组播信息发送时间间隔，两个节点必须设置一样，默认为1秒
# 验证信息，两个节点必须一致
    authentication {
      auth_type PASS # 认证方式，可以是PASS或AH两种认证方式
        auth_pass 1111 # 认证密码
    }
  unicast_src_ip 10.0.4.20  # 设置本机内网IP地址
    unicast_peer {
      10.0.4.21             # 对端设备的IP地址
    }
# VIP，当state为master时添加，当state为backup时删除
  virtual_ipaddress {
    10.0.4.99 # 设置高可用虚拟VIP，如果是腾讯云的CVM，需要填写控制台申请到的HAVIP地址。
  }
  notify_master "/etc/keepalived/keepalived_notify.sh MASTER" # 当切换到master状态时执行脚本
    notify_backup "/etc/keepalived/keepalived_notify.sh BACKUP" # 当切换到backup状态时执行脚本
    notify_fault "/etc/keepalived/keepalived_notify.sh FAULT" # 当切换到fault状态时执行脚本
    notify_stop "/etc/keepalived/keepalived_notify.sh STOP" # 当切换到stop状态时执行脚本
    garp_master_delay 1    # 设置当切为主状态后多久更新ARP缓存
    garp_master_refresh 5   # 设置主节点发送ARP报文的时间间隔
    # 跟踪接口，里面任意一块网卡出现问题，都会进入故障(FAULT)状态
    track_interface {
      eth0
    }
  # 要执行的检查脚本
  track_script {
    checkhaproxy
  }
}
</code></pre><p>这里有几个注意事项：</p><ul>
<li>确保已经配置了garp相关参数。因为Keepalived依赖ARP报文更新IP信息，如果缺少这些参数，会导致某些场景下主设备不发送ARP，进而导致通信异常。garp相关参数配置如下：</li>
</ul><pre><code class="language-plain">garp_master_delay 1
garp_master_refresh 5
</code></pre><ul>
<li>确定没有采用 strict 模式，即需要删除vrrp_strict配置。</li>
<li>配置中的<code>/etc/keepalived/check_nginx.sh</code>和<code>/etc/keepalived/keepalived_notify.sh</code>脚本文件，可分别拷贝自<a href="https://github.com/marmotedu/iam/blob/v1.0.8/scripts/check_nginx.sh">scripts/check_nginx.sh</a>和<a href="https://github.com/marmotedu/iam/blob/v1.0.8/scripts/keepalived_notify.sh">scripts/keepalived_notify.sh</a>。</li>
</ul><p><strong>然后，重启Keepalived：</strong></p><pre><code class="language-bash">$ sudo systemctl restart keepalived
</code></pre><h3>第三步：备服务器配置</h3><p>进行备服务器配置也分为两步。</p><p><strong>首先，修改Keepalived配置文件。</strong></p><p>登陆服务器<code>10.0.4.21</code>，编辑<code>/etc/keepalived/keepalived.conf</code>，修改配置，修改后配置内容如下（参考：<a href="https://github.com/marmotedu/iam/blob/v1.0.8/configs/ha/10.0.4.21/keepalived.conf">configs/ha/10.0.4.21/keepalived.conf</a>）：</p><pre><code class="language-plain"># 全局定义，定义全局的配置选项
global_defs {
# 指定keepalived在发生切换操作时发送email，发送给哪些email
# 建议在keepalived_notify.sh中发送邮件
&nbsp; notification_email {
&nbsp; &nbsp; acassen@firewall.loc
&nbsp; }
&nbsp; notification_email_from Alexandre.Cassen@firewall.loc # 发送email时邮件源地址
&nbsp; &nbsp; smtp_server 192.168.200.1 # 发送email时smtp服务器地址
&nbsp; &nbsp; smtp_connect_timeout 30 # 连接smtp的超时时间
&nbsp; &nbsp; router_id VM-4-21-centos # 机器标识，通常可以设置为hostname
&nbsp; &nbsp; vrrp_skip_check_adv_addr # 如果接收到的报文和上一个报文来自同一个路由器，则不执行检查。默认是跳过检查
&nbsp; &nbsp; vrrp_garp_interval 0 # 单位秒，在一个网卡上每组gratuitous arp消息之间的延迟时间，默认为0
&nbsp; &nbsp; vrrp_gna_interval 0 # 单位秒，在一个网卡上每组na消息之间的延迟时间，默认为0
}
# 检测脚本配置
vrrp_script checkhaproxy
{
&nbsp; script "/etc/keepalived/check_nginx.sh" # 检测脚本路径
&nbsp; &nbsp; interval 5 # 检测时间间隔（秒）
&nbsp; &nbsp; weight 0 # 根据该权重改变priority，当值为0时，不改变实例的优先级
}
# VRRP实例配置
vrrp_instance VI_1 {
&nbsp; state BACKUP&nbsp; # 设置初始状态为'备份'
&nbsp; &nbsp; interface eth0 # 设置绑定VIP的网卡，例如eth0
&nbsp; &nbsp; virtual_router_id 51&nbsp; # 配置集群VRID，互为主备的VRID需要是相同的值
&nbsp; &nbsp; nopreempt&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;# 设置非抢占模式，只能设置在state为backup的节点上
&nbsp; &nbsp; priority 50 # 设置优先级，值范围0～254，值越大优先级越高，最高的为master
&nbsp; &nbsp; advert_int 1 # 组播信息发送时间间隔，两个节点必须设置一样，默认为1秒
# 验证信息，两个节点必须一致
&nbsp; &nbsp; authentication {
&nbsp; &nbsp; &nbsp; auth_type PASS # 认证方式，可以是PASS或AH两种认证方式
&nbsp; &nbsp; &nbsp; &nbsp; auth_pass 1111 # 认证密码
&nbsp; &nbsp; }
&nbsp; unicast_src_ip 10.0.4.21&nbsp; # 设置本机内网IP地址
&nbsp; &nbsp; unicast_peer {
&nbsp; &nbsp; &nbsp; 10.0.4.20&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;# 对端设备的IP地址
&nbsp; &nbsp; }
# VIP，当state为master时添加，当state为backup时删除
&nbsp; virtual_ipaddress {
&nbsp; &nbsp; 10.0.4.99 # 设置高可用虚拟VIP，如果是腾讯云的CVM，需要填写控制台申请到的HAVIP地址。
&nbsp; }
&nbsp; notify_master "/etc/keepalived/keepalived_notify.sh MASTER" # 当切换到master状态时执行脚本
&nbsp; &nbsp; notify_backup "/etc/keepalived/keepalived_notify.sh BACKUP" # 当切换到backup状态时执行脚本
&nbsp; &nbsp; notify_fault "/etc/keepalived/keepalived_notify.sh FAULT" # 当切换到fault状态时执行脚本
&nbsp; &nbsp; notify_stop "/etc/keepalived/keepalived_notify.sh STOP" # 当切换到stop状态时执行脚本
&nbsp; &nbsp; garp_master_delay 1&nbsp; &nbsp; # 设置当切为主状态后多久更新ARP缓存
&nbsp; &nbsp; garp_master_refresh 5&nbsp; &nbsp;# 设置主节点发送ARP报文的时间间隔
&nbsp; &nbsp; # 跟踪接口，里面任意一块网卡出现问题，都会进入故障(FAULT)状态
&nbsp; &nbsp; track_interface {
&nbsp; &nbsp; &nbsp; eth0
&nbsp; &nbsp; }
&nbsp; # 要执行的检查脚本
&nbsp; track_script {
&nbsp; &nbsp; checkhaproxy
&nbsp; }
}
</code></pre><p><strong>然后，重启Keepalived：</strong></p><pre><code class="language-bash">$ sudo systemctl restart keepalived
</code></pre><h3>第四步：测试Keepalived</h3><p>上面的配置中，<code>10.0.4.20</code>的优先级更高，所以正常情况下<code>10.0.4.20</code>将被选择为主节点，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/54/79/54968f40707b779e2ab70d3ab5a53479.png?wh=1920x500" alt="图片"></p><p>接下来，我们分别模拟一些故障场景，来看下配置是否生效。</p><p><strong>场景1：Keepalived故障</strong></p><p>在<code>10.0.4.20</code>服务器上执行<code>sudo systemctl stop keepalived</code>模拟Keepalived故障，查看VIP，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/2a/ee/2a57e958bd9fce3b9c842c1cf09c48ee.png?wh=1920x544" alt="图片"></p><p>可以看到，VIP从<code>10.0.4.20</code>服务器上，漂移到了<code>10.0.4.21</code>服务器上。查看<code>/var/log/keepalived.log</code>，可以看到<code>10.0.4.20</code>服务器新增如下一行日志：</p><pre><code class="language-plain">[2020-10-14 14:01:51] notify_stop
</code></pre><p><code>10.0.4.21</code>服务器新增如下日志：</p><pre><code class="language-plain">[2020-10-14 14:01:52] notify_master
</code></pre><p><strong>场景2：Nginx故障</strong></p><p>在<code>10.0.4.20</code>和<code>10.0.4.21</code>服务器上分别执行<code>sudo systemctl restart keepalived</code>，让VIP漂移到<code>10.0.4.20</code>服务器上。</p><p>在<code>10.0.4.20</code>服务器上，执行 <code>sudo systemctl stop nginx</code> 模拟Nginx故障，查看VIP，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/2a/ee/2a57e958bd9fce3b9c842c1cf09c48ee.png?wh=1920x544" alt="图片"></p><p>可以看到，VIP从<code>10.0.4.20</code>服务器上，漂移到了<code>10.0.4.21</code>服务器上。查看<code>/var/log/keepalived.log</code>，可以看到<code>10.0.4.20</code>服务器新增如下一行日志：</p><pre><code class="language-plain">[2020-10-14 14:02:34] notify_fault
</code></pre><p><code>10.0.4.21</code> 服务器新增如下日志：</p><pre><code class="language-plain">[2020-10-14 14:02:35] notify_master
</code></pre><p><strong>场景3：Nginx恢复</strong></p><p>基于<strong>场景2</strong>，在<code>10.0.4.20</code>服务器上执行<code>sudo systemctl start nginx</code>恢复Nginx，查看VIP，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/2a/ee/2a57e958bd9fce3b9c842c1cf09c48ee.png?wh=1920x544" alt="图片"></p><p>可以看到，VIP仍然在<code>10.0.4.21</code>服务器上，没有被<code>10.0.4.20</code>抢占。查看<code>/var/log/keepalived.log</code>，可以看到<code>10.0.4.20</code>服务器新增如下一行日志：</p><pre><code class="language-plain">[2020-10-14 14:03:44] notify_backup
</code></pre><p><code>10.0.4.21</code>服务器没有新增日志。</p><h3>第五步：VIP绑定公网IP</h3><p>到这里，我们已经成功配置了Keepalived + Nginx的高可用方案。但是，我们的VIP是内网，还不能通过外网访问。这时候，我们需要将VIP绑定一个外网IP，供外网访问。在腾讯云上，可通过绑定弹性公网IP来实现外网访问，需要先申请公网IP，然后将VIP绑定弹性公网IP。下面我来讲讲具体步骤。</p><p>申请公网IP：</p><ol>
<li>登录<strong>私有网络控制台。</strong></li>
<li>在左侧导航栏中，选择【IP与网卡】&gt;【弹性公网IP】。</li>
<li>在弹性公网IP管理页面，选择所在地域，单击【申请】。</li>
</ol><p>将VIP绑定弹性公网IP：</p><ol>
<li>登录<strong>私有网络控制台</strong>。</li>
<li>在左侧导航栏中，选择【IP与网卡】&gt;【高可用虚拟】。</li>
<li>单击需要绑定的HAVIP所在行的【绑定】。</li>
<li>在弹出界面中，选择需要绑定的公网IP即可，如下图所示：</li>
</ol><p><img src="https://static001.geekbang.org/resource/image/83/62/83bc9f4595325e9d339e7c3269aa3462.png?wh=1388x666" alt="图片"></p><p>绑定的弹性公网IP是<code>106.52.252.139</code>。</p><p>这里提示下，腾讯云平台中，如果HAVIP没有绑定实例，绑定HAVIP的EIP会处于闲置状态，按<code>¥0.2/小时</code> 收取闲置费用。所以，你需要正确配置高可用应用，确保绑定成功。</p><h3>第六步：测试公网访问</h3><p>最后，你可以通过执行如下命令来测试：</p><pre><code class="language-bash">$ curl -H"Host: iam.api.marmotedu.com" http://106.52.252.139/healthz -H"iam.api.marmotedu.com"
{"status":"ok"}
</code></pre><p>可以看到，我们可以成功通过公网访问后端的高可用服务。到这里，我们成功部署了一个可用性很高的IAM应用。</p><h2>总结</h2><p>今天，我主要讲了如何使用Nginx和Keepalived，来部署一个高可用的IAM应用。</p><p>为了部署一个高可用的IAM应用，我们至少需要两台服务器，并且部署相同的服务iam-apiserver、iam-authz-server、iam-pump。而且，选择其中一台服务器部署数据库服务：MariaDB、Redis、MongoDB。</p><p>为了安全和性能，iam-apiserver、iam-authz-server、iam-pump服务都是通过内网来访问数据库服务的。这一讲，我还介绍了如何配置Nginx来实现负载均衡，如何配置Keepalived来实现Nginx的高可用。</p><h2>课后练习</h2><ol>
<li>思考下，当前部署架构下如果iam-apiserver需要扩容，可以怎么扩容？</li>
<li>思考下，当VIP切换时，如何实现告警功能，给系统运维人员告警？</li>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/52/40/e57a736e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pedro</span>
  </div>
  <div class="_2_QraFYR_0">iamctl 好用的不行，已经沉淀为了自己的 pctl 了，这不是抄袭，这是模仿～</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 优秀，哈哈哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-29 19:48:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d5/99/8c0d8f66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>㊣Coldstar</span>
  </div>
  <div class="_2_QraFYR_0">阿里云有免费的内网负载均衡可以使用 腾讯云 没看到可以创建 内网专用的负载均衡，这样的话，利用云基础设施可以简化部署</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 腾讯云支持内网负载均衡，腾讯云的内网负载均衡收费情况如下：内网负载均衡免收公网网络费，收取实例费。<br><br>创建负载均衡的时候，选择  【内网】即可创建内网负载均衡</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-06 16:55:57</div>
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
  <div class="_2_QraFYR_0">总结：<br>1. 在服务器上部署 IAM应用中的服务。20 机器上还会部署 Mysql, Redis, MongoDB<br>2. 配置 Nginx。主要是添加两个 server，在 http{} 中添加 upstream。<br>4. 配置 Keepalived</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-04 23:21:36</div>
  </div>
</div>
</div>
</li>
</ul>