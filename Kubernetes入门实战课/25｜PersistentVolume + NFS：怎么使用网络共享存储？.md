<audio title="25｜PersistentVolume + NFS：怎么使用网络共享存储？" src="https://static001.geekbang.org/resource/audio/a9/58/a91fabcaf4d6e61fyy30622585686858.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>在上节课里我们看到了Kubernetes里的持久化存储对象PersistentVolume、PersistentVolumeClaim、StorageClass，把它们联合起来就可以为Pod挂载一块“虚拟盘”，让Pod在其中任意读写数据。</p><p>不过当时我们使用的是HostPath，存储卷只能在本机使用，而Kubernetes里的Pod经常会在集群里“漂移”，所以这种方式不是特别实用。</p><p>要想让存储卷真正能被Pod任意挂载，我们需要变更存储的方式，不能限定在本地磁盘，而是要改成<strong>网络存储</strong>，这样Pod无论在哪里运行，只要知道IP地址或者域名，就可以通过网络通信访问存储设备。</p><p>网络存储是一个非常热门的应用领域，有很多知名的产品，比如AWS、Azure、Ceph，Kubernetes还专门定义了CSI（Container Storage Interface）规范，不过这些存储类型的安装、使用都比较复杂，在我们的实验环境里部署难度比较高。</p><p>所以今天的这次课里，我选择了相对来说比较简单的NFS系统（Network File System），以它为例讲解如何在Kubernetes里使用网络存储，以及静态存储卷和动态存储卷的概念。</p><!-- [[[read_end]]] --><h2>如何安装NFS服务器</h2><p>作为一个经典的网络存储系统，NFS有着近40年的发展历史，基本上已经成为了各种UNIX系统的标准配置，Linux自然也提供对它的支持。</p><p>NFS采用的是Client/Server架构，需要选定一台主机作为Server，安装NFS服务端；其他要使用存储的主机作为Client，安装NFS客户端工具。</p><p>所以接下来，我们在自己的Kubernetes集群里再增添一台名字叫Storage的服务器，在上面安装NFS，实现网络存储、共享网盘的功能。<strong>不过这台Storage也只是一个逻辑概念，我们在实际安装部署的时候完全可以把它合并到集群里的某台主机里</strong>，比如这里我就复用了<a href="https://time.geekbang.org/column/article/534762">第17讲</a>里的Console。</p><p>新的网络架构如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/78/07/786e13af0e2f62f9cd73f5ab555a4507.jpg?wh=1920x1235" alt="图片"></p><p>在Ubuntu系统里安装NFS服务端很容易，使用apt即可：</p><pre><code class="language-plain">sudo apt -y install nfs-kernel-server
</code></pre><p>安装好之后，你需要给NFS指定一个存储位置，也就是网络共享目录。一般来说，应该建立一个专门的 <code>/data</code> 目录，这里为了简单起见，我就使用了<strong>临时目录 <code>/tmp/nfs</code></strong>：</p><pre><code class="language-plain">mkdir -p /tmp/nfs
</code></pre><p>接下来你需要配置NFS访问共享目录，修改 <code>/etc/exports</code>，指定目录名、允许访问的网段，还有权限等参数。这些规则比较琐碎，和我们的Kubernetes课程关联不大，我就不详细解释了，你只要把下面这行加上就行，注意目录名和IP地址要改成和自己的环境一致：</p><pre><code class="language-plain">/tmp/nfs 192.168.10.0/24(rw,sync,no_subtree_check,no_root_squash,insecure)
</code></pre><p><strong>改好之后，需要用 <code>exportfs -ra</code> 通知NFS，让配置生效</strong>，再用 <code>exportfs -v</code> 验证效果：</p><pre><code class="language-plain">sudo exportfs -ra
sudo exportfs -v
</code></pre><p><img src="https://static001.geekbang.org/resource/image/0c/d1/0cd8889ee51c6d8a8947f6bd615d6bd1.png?wh=1920x116" alt="图片"></p><p>现在，你就可以使用 <code>systemctl</code> 来启动NFS服务器了：</p><pre><code class="language-plain">sudo systemctl start&nbsp; nfs-server
sudo systemctl enable nfs-server
sudo systemctl status nfs-server
</code></pre><p><img src="https://static001.geekbang.org/resource/image/29/5a/29fb58f93f0e764ca8309ed9eff5175a.png?wh=1832x486" alt="图片"></p><p>你还可以使用命令 <code>showmount</code> 来检查NFS的网络挂载情况：</p><pre><code class="language-plain">showmount -e 127.0.0.1
</code></pre><p><img src="https://static001.geekbang.org/resource/image/90/d2/905ea675a49daef860d21b41a668d6d2.png?wh=978x176" alt="图片"></p><h2>如何安装NFS客户端</h2><p>有了NFS服务器之后，为了让Kubernetes集群能够访问NFS存储服务，我们还需要在每个节点上都安装NFS客户端。</p><p>这项工作只需要一条apt命令，不需要额外的配置：</p><pre><code class="language-plain">sudo apt -y install nfs-common
</code></pre><p>同样，在节点上可以用 <code>showmount</code> 检查NFS能否正常挂载，注意IP地址要写成NFS服务器的地址，我在这里就是“192.168.10.208”：</p><p><img src="https://static001.geekbang.org/resource/image/7e/9c/7ed89f8468d6d4fa315a6d456f2eee9c.png?wh=1182x186" alt="图片"></p><p>现在让我们尝试手动挂载一下NFS网络存储，先创建一个目录 <code>/tmp/test</code> 作为挂载点：</p><pre><code class="language-plain">mkdir -p /tmp/test
</code></pre><p>然后用命令 <code>mount</code> 把NFS服务器的共享目录挂载到刚才创建的本地目录上：</p><pre><code class="language-plain">sudo mount -t nfs 192.168.10.208:/tmp/nfs /tmp/test
</code></pre><p>最后测试一下，我们在 <code>/tmp/test</code> 里随便创建一个文件，比如 <code>x.yml</code>：</p><pre><code class="language-plain">touch /tmp/test/x.yml
</code></pre><p>再回到NFS服务器，检查共享目录 <code>/tmp/nfs</code>，应该会看到也出现了一个同样的文件 <code>x.yml</code>，这就说明NFS安装成功了。之后集群里的任意节点，只要通过NFS客户端，就能把数据写入NFS服务器，实现网络存储。</p><h2>如何使用NFS存储卷</h2><p>现在我们已经为Kubernetes配置好了NFS存储系统，就可以使用它来创建新的PV存储对象了。</p><p>先来手工分配一个存储卷，需要指定 <code>storageClassName</code> 是 <code>nfs</code>，而 <code>accessModes</code> 可以设置成 <code>ReadWriteMany</code>，这是由NFS的特性决定的，它<strong>支持多个节点同时访问一个共享目录</strong>。</p><p>因为这个存储卷是NFS系统，所以我们还需要在YAML里添加 <code>nfs</code> 字段，指定NFS服务器的IP地址和共享目录名。</p><p>这里我在NFS服务器的 <code>/tmp/nfs</code> 目录里又创建了一个新的目录 <code>1g-pv</code>，表示分配了1GB的可用存储空间，相应的，PV里的 <code>capacity</code> 也要设置成同样的数值，也就是 <code>1Gi</code>。</p><p>把这些字段都整理好后，我们就得到了一个使用NFS网络存储的YAML描述文件：</p><pre><code class="language-yaml">apiVersion: v1
kind: PersistentVolume
metadata:
&nbsp; name: nfs-1g-pv

spec:
&nbsp; storageClassName: nfs
&nbsp; accessModes:
&nbsp; &nbsp; - ReadWriteMany
&nbsp; capacity:
&nbsp; &nbsp; storage: 1Gi

&nbsp; nfs:
&nbsp; &nbsp; path: /tmp/nfs/1g-pv
&nbsp; &nbsp; server: 192.168.10.208
</code></pre><p>现在就可以用命令 <code>kubectl apply</code> 来创建PV对象，再用 <code>kubectl get pv</code> 查看它的状态：</p><pre><code class="language-plain">kubectl apply -f nfs-static-pv.yml
kubectl get pv
</code></pre><p><img src="https://static001.geekbang.org/resource/image/3b/39/3bb0be2483e92467d3cac14fbc635739.png?wh=1688x300" alt="图片"></p><p><strong>再次提醒你注意，<code>spec.nfs</code> 里的IP地址一定要正确，路径一定要存在（事先创建好）</strong>，否则Kubernetes按照PV的描述会无法挂载NFS共享目录，PV就会处于“pending”状态无法使用。</p><p>有了PV，我们就可以定义申请存储的PVC对象了，它的内容和PV差不多，但不涉及NFS存储的细节，只需要用 <code>resources.request</code> 来表示希望要有多大的容量，这里我写成1GB，和PV的容量相同：</p><pre><code class="language-yaml">apiVersion: v1
kind: PersistentVolumeClaim
metadata:
&nbsp; name: nfs-static-pvc

spec:
&nbsp; storageClassName: nfs
&nbsp; accessModes:
&nbsp; &nbsp; - ReadWriteMany

&nbsp; resources:
&nbsp; &nbsp; requests:
&nbsp; &nbsp; &nbsp; storage: 1Gi
</code></pre><p>创建PVC对象之后，Kubernetes就会根据PVC的描述，找到最合适的PV，把它们“绑定”在一起，也就是存储分配成功：</p><p><img src="https://static001.geekbang.org/resource/image/a7/8c/a7bbcc5dce117f9872cee3f08e6a6c8c.png?wh=1920x309" alt="图片"></p><p>我们再创建一个Pod，把PVC挂载成它的一个volume，具体的做法和<a href="https://time.geekbang.org/column/article/542376">上节课</a>是一样的，用 <code>persistentVolumeClaim</code> 指定PVC的名字就可以了：</p><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
&nbsp; name: nfs-static-pod

spec:
&nbsp; volumes:
&nbsp; - name: nfs-pvc-vol
&nbsp; &nbsp; persistentVolumeClaim:
&nbsp; &nbsp; &nbsp; claimName: nfs-static-pvc

&nbsp; containers:
&nbsp; &nbsp; - name: nfs-pvc-test
&nbsp; &nbsp; &nbsp; image: nginx:alpine
&nbsp; &nbsp; &nbsp; ports:
&nbsp; &nbsp; &nbsp; - containerPort: 80

&nbsp; &nbsp; &nbsp; volumeMounts:
&nbsp; &nbsp; &nbsp; &nbsp; - name: nfs-pvc-vol
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; mountPath: /tmp
</code></pre><p>Pod、PVC、PV和NFS存储的关系可以用下图来形象地表示，你可以对比一下HostPath PV的用法，看看有什么不同：</p><p><img src="https://static001.geekbang.org/resource/image/2a/a7/2a21d16b028afdea4f525439bd8f06a7.jpg?wh=1920x1125" alt="图片"></p><p>因为我们在PV/PVC里指定了 <code>storageClassName</code> 是 <code>nfs</code>，节点上也安装了NFS客户端，所以Kubernetes就会自动执行NFS挂载动作，把NFS的共享目录 <code>/tmp/nfs/1g-pv</code> 挂载到Pod里的 <code>/tmp</code>，完全不需要我们去手动管理。</p><p>最后还是测试一下，用 <code>kubectl apply</code> 创建Pod之后，我们用 <code>kubectl exec</code> 进入Pod，再试着操作NFS共享目录：</p><p><img src="https://static001.geekbang.org/resource/image/bb/90/bbc244b6cd21b71f50807864718d8990.png?wh=1386x542" alt="图片"></p><p>退出Pod，再看一下NFS服务器的 <code>/tmp/nfs/1g-pv</code> 目录，你就会发现Pod里创建的文件确实写入了共享目录：</p><p><img src="https://static001.geekbang.org/resource/image/87/d0/87cdc722da478db6f938db4d424be0d0.png?wh=756x354" alt="图片"></p><p>而且更好的是，因为NFS是一个网络服务，不会受Pod调度位置的影响，所以只要网络通畅，这个PV对象就会一直可用，数据也就实现了真正的持久化存储。</p><h2>如何部署NFS Provisoner</h2><p>现在有了NFS这样的网络存储系统，你是不是认为Kubernetes里的数据持久化问题就已经解决了呢？</p><p>对于这个问题，我觉得可以套用一句现在的流行语：“解决了，但没有完全解决。”</p><p>说它“解决了”，是因为网络存储系统确实能够让集群里的Pod任意访问，数据在Pod销毁后仍然存在，新创建的Pod可以再次挂载，然后读取之前写入的数据，整个过程完全是自动化的。</p><p>说它“没有完全解决”，是因为<strong>PV还是需要人工管理</strong>，必须要由系统管理员手动维护各种存储设备，再根据开发需求逐个创建PV，而且PV的大小也很难精确控制，容易出现空间不足或者空间浪费的情况。</p><p>在我们的这个实验环境里，只有很少的PV需求，管理员可以很快分配PV存储卷，但是在一个大集群里，每天可能会有几百几千个应用需要PV存储，如果仍然用人力来管理分配存储，管理员很可能会忙得焦头烂额，导致分配存储的工作大量积压。</p><p>那么能不能让创建PV的工作也实现自动化呢？或者说，让计算机来代替人类来分配存储卷呢？</p><p>这个在Kubernetes里就是“<strong>动态存储卷</strong>”的概念，它可以用StorageClass绑定一个Provisioner对象，而这个Provisioner就是一个能够自动管理存储、创建PV的应用，代替了原来系统管理员的手工劳动。</p><p>有了“动态存储卷”的概念，前面我们讲的手工创建的PV就可以称为“静态存储卷”。</p><p>目前，Kubernetes里每类存储设备都有相应的Provisioner对象，对于NFS来说，它的Provisioner就是“NFS subdir external provisioner”，你可以在GitHub上找到这个项目（<a href="https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner">https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner</a>）。</p><p>NFS Provisioner也是以Pod的形式运行在Kubernetes里的，<strong>在GitHub的 <code>deploy</code> 目录里是部署它所需的YAML文件，一共有三个，分别是rbac.yaml、class.yaml和deployment.yaml</strong>。</p><p>不过这三个文件只是示例，想在我们的集群里真正运行起来还要修改其中的两个文件。</p><p>第一个要修改的是rbac.yaml，它使用的是默认的 <code>default</code> 名字空间，应该把它改成其他的名字空间，避免与普通应用混在一起，你可以用“查找替换”的方式把它统一改成 <code>kube-system</code>。</p><p>第二个要修改的是deployment.yaml，它要修改的地方比较多。首先要把名字空间改成和rbac.yaml一样，比如是 <code>kube-system</code>，然后重点要修改 <code>volumes</code> 和 <code>env</code> 里的IP地址和共享目录名，必须和集群里的NFS服务器配置一样。</p><p>按照我们当前的环境设置，就应该把IP地址改成 <code>192.168.10.208</code>，目录名改成 <code>/tmp/nfs</code>：</p><pre><code class="language-yaml">spec:
&nbsp; template:
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; serviceAccountName: nfs-client-provisioner
&nbsp; &nbsp; &nbsp; containers:
		&nbsp; ...
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; env:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; - name: PROVISIONER_NAME
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; value: k8s-sigs.io/nfs-subdir-external-provisioner
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; - name: NFS_SERVER
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; value: 192.168.10.208        #改IP地址
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; - name: NFS_PATH
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; value: /tmp/nfs              #改共享目录名
&nbsp; &nbsp; &nbsp; volumes:
&nbsp; &nbsp; &nbsp; &nbsp; - name: nfs-client-root
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; nfs:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; server: 192.168.10.208         #改IP地址
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; Path: /tmp/nfs                 #改共享目录名
</code></pre><p>还有一件麻烦事，deployment.yaml的镜像仓库用的是gcr.io，拉取很困难，而国内的镜像网站上偏偏还没有它，为了让实验能够顺利进行，我不得不“曲线救国”，把它的镜像转存到了Docker Hub上。</p><p>所以你还需要把镜像的名字由原来的“k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2”改成“chronolaw/nfs-subdir-external-provisioner:v4.0.2”，其实也就是变动一下镜像的用户名而已。</p><p>把这两个YAML修改好之后，我们就可以在Kubernetes里创建NFS Provisioner了：</p><pre><code class="language-plain">kubectl apply -f rbac.yaml
kubectl apply -f class.yaml
kubectl apply -f deployment.yaml
</code></pre><p>使用命令 <code>kubectl get</code>，再加上名字空间限定 <code>-n kube-system</code>，就可以看到NFS Provisioner在Kubernetes里运行起来了。</p><p><img src="https://static001.geekbang.org/resource/image/35/6d/35758cbe60ddf264bcf59d703fd4986d.png?wh=1920x407" alt="图片"></p><h2>如何使用NFS动态存储卷</h2><p>比起静态存储卷，动态存储卷的用法简单了很多。因为有了Provisioner，我们就不再需要手工定义PV对象了，只需要在PVC里指定StorageClass对象，它再关联到Provisioner。</p><p>我们来看一下NFS默认的StorageClass定义：</p><pre><code class="language-yaml">apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
&nbsp; name: nfs-client

provisioner: k8s-sigs.io/nfs-subdir-external-provisioner&nbsp;
parameters:
&nbsp; archiveOnDelete: "false"
</code></pre><p>YAML里的关键字段是 <code>provisioner</code>，它指定了应该使用哪个Provisioner。另一个字段 <code>parameters</code> 是调节Provisioner运行的参数，需要参考文档来确定具体值，在这里的 <code>archiveOnDelete: "false"</code> 就是自动回收存储空间。</p><p>理解了StorageClass的YAML之后，你也可以不使用默认的StorageClass，而是根据自己的需求，任意定制具有不同存储特性的StorageClass，比如添加字段 <code>onDelete: "retain"</code> 暂时保留分配的存储，之后再手动删除：</p><pre><code class="language-yaml">apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
&nbsp; name: nfs-client-retained

provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
&nbsp; onDelete: "retain"
</code></pre><p>接下来我们定义一个PVC，向系统申请10MB的存储空间，使用的StorageClass是默认的 <code>nfs-client</code>：</p><pre><code class="language-yaml">apiVersion: v1
kind: PersistentVolumeClaim
metadata:
&nbsp; name: nfs-dyn-10m-pvc

spec:
&nbsp; storageClassName: nfs-client
&nbsp; accessModes:
&nbsp; &nbsp; - ReadWriteMany

&nbsp; resources:
&nbsp; &nbsp; requests:
&nbsp; &nbsp; &nbsp; storage: 10Mi
</code></pre><p>写好了PVC，我们还是在Pod里用 <code>volumes</code> 和 <code>volumeMounts</code> 挂载，然后Kubernetes就会自动找到NFS Provisioner，在NFS的共享目录上创建出合适的PV对象：</p><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
&nbsp; name: nfs-dyn-pod

spec:
&nbsp; volumes:
&nbsp; - name: nfs-dyn-10m-vol
&nbsp; &nbsp; persistentVolumeClaim:
&nbsp; &nbsp; &nbsp; claimName: nfs-dyn-10m-pvc

&nbsp; containers:
&nbsp; &nbsp; - name: nfs-dyn-test
&nbsp; &nbsp; &nbsp; image: nginx:alpine
&nbsp; &nbsp; &nbsp; ports:
&nbsp; &nbsp; &nbsp; - containerPort: 80

&nbsp; &nbsp; &nbsp; volumeMounts:
&nbsp; &nbsp; &nbsp; &nbsp; - name: nfs-dyn-10m-vol
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; mountPath: /tmp
</code></pre><p>使用 <code>kubectl apply</code> 创建好PVC和Pod，让我们来查看一下集群里的PV状态：</p><p><img src="https://static001.geekbang.org/resource/image/57/bb/570d73409db1edc757yy10e6aba56ebb.png?wh=1920x271" alt="图片"></p><p>从截图你可以看到，虽然我们没有直接定义PV对象，但由于有NFS Provisioner，它就自动创建一个PV，大小刚好是在PVC里申请的10MB。</p><p>如果你这个时候再去NFS服务器上查看共享目录，也会发现多出了一个目录，名字与这个自动创建的PV一样，但加上了名字空间和PVC的前缀：</p><p><img src="https://static001.geekbang.org/resource/image/a9/ea/a9b6942b824bc9f7841850ee15yy68ea.png?wh=1714x126" alt="图片"></p><p>我还是把Pod、PVC、StorageClass和Provisioner的关系画成了一张图，你可以清楚地看出来这些对象的关联关系，还有Pod是如何最终找到存储设备的：</p><p><img src="https://static001.geekbang.org/resource/image/e3/1e/e3905990be6fb8739fb51a4ab9856f1e.jpg?wh=1920x856" alt="图片"></p><h2>小结</h2><p>好了，今天的这节课里我们继续学习PV/PVC，引入了网络存储系统，以NFS为例研究了静态存储卷和动态存储卷的用法，其中的核心对象是StorageClass和Provisioner。</p><p>我再小结一下今天的要点：</p><ol>
<li>在Kubernetes集群里，网络存储系统更适合数据持久化，NFS是最容易使用的一种网络存储系统，要事先安装好服务端和客户端。</li>
<li>可以编写PV手工定义NFS静态存储卷，要指定NFS服务器的IP地址和共享目录名。</li>
<li>使用NFS动态存储卷必须要部署相应的Provisioner，在YAML里正确配置NFS服务器。</li>
<li>动态存储卷不需要手工定义PV，而是要定义StorageClass，由关联的Provisioner自动创建PV完成绑定。</li>
</ol><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>动态存储卷相比静态存储卷有什么好处？有没有缺点？</li>
<li>StorageClass在动态存储卷的分配过程中起到了什么作用？</li>
</ol><p>期待你的思考。如果觉得有收获，也欢迎你分享给朋友一起讨论。我们下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/20/ff/2022f24dcc6a3d76214bbc59c3bd2aff.jpg?wh=1920x2122" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/83/af/1cb42cd3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>马以</span>
  </div>
  <div class="_2_QraFYR_0">思考题：<br>1: 首先，静态存储卷PV这个动作是要由运维手动处理的，如果是处在大规模集群的成千上万个PVC的场景下，这个工作量是难以想象的；<br>   再者，业务的迭代变更是动态的，这也就意味着随时会有新的PVC被创建，或者就的PVC被删除，这也就要求运维每碰到PVC的变更，就要跟着去手动维护一个新的<br>   PV。来满足业务的新需求，这个情况如果解决不了，我相信运维这个职业马上就会消失。<br>   最后，动态存储卷的好处还在于分层和解耦，对于简单的localPath或者NFS这种存储卷或许相对来说还比较简单一些，但是像类似于远程存储磁盘这种就相对来说比较<br>   复杂了，动态存储可以让我们只关注于需求点，至于怎么把这些东西创建出来，就交由各个类型的provisioner去处理就行了。<br>   <br>  缺点：缺点的话就是在于资源的管控方面，比如原本我可能只需要2Gi的空间，但是业务人员对容量把握不够申请了10Gi，就会有8Gi空间的浪费。<br>2:StorageClass 作用是帮助指定特定类型的provisioner，这决定了你要使用的具体某种类型的存储插件；另外它还限定了PV和PVC的绑定关系，只有从属于同一StorageClass的PV和PVC才能做绑定动作，比如指定GlusterFS类型的PVC对象不能绑定到另外一个PVC定义的NFS类型的StorageClass 模版创建出的Volume的PV对象上面去。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: very nice！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-20 16:06:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b3/d9/cf061262.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>新时代农民工</span>
  </div>
  <div class="_2_QraFYR_0">不妨试一试docker版的nfs-server，简单又方便 https:&#47;&#47;hub.docker.com&#47;r&#47;fuzzle&#47;docker-nfs-server</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-20 22:43:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/1b/f9/018197f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小江爱学术</span>
  </div>
  <div class="_2_QraFYR_0">老师，不知道这么理解对不对，因为存储它涉及到了对物理机文件系统绑定的操作，因此K8S做了一系列抽象。PV在这个抽象里，其实就指代了主机文件系统的路径，当然至于再往实现层面走，是网络文件系统还是主机文件系统，这就全由PV的绑定类型决定。而往抽象层走，作为K8S的核心系统，K8S想尽可能屏蔽掉底层，也就是主机文件系统的概念，所以它抽象了StorageClass，用来统一指代&#47;管理PV。至此，K8S持久化存储就可以分两个部分，第一部分是由 主机文件系统+PV+StorageClass组成的，用来将抽象对象绑定到真实文件系统的生产者部分；第二部分就是 Volume+PVC+StorageClass，完全被抽象为K8S核心业务的消费者部分，而StorageClass，可以看作是两部分连接的桥梁。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解的非常好，这个就是Kubernetes关键的抽象思路，把底层的实现细节屏蔽了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-28 09:19:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/83/af/1cb42cd3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>马以</span>
  </div>
  <div class="_2_QraFYR_0">云服务器环境搭建：主要在nfs权限那边比较麻烦<br><br>-- 服务端<br>1: 查看是否安装了必要的软件<br>	$ dpkg -l | grep rpcbind<br>	...<br><br>	$ sudo apt -y install rpcbind<br>	$ sudo apt -y install nfs-kernel-server<br>	$ sudo apt -y install nfs-common<br>2.修改 &#47;etc&#47;services 追加<br><br>	# Local services<br>	mountd 4011&#47;udp<br>	mountd 4011&#47;tcp<br><br>	内容，固定mountd 的端口号<br><br><br><br>	打开端口号访问限制：<br>	 TCP: 2049、111、4011<br>	 UDP: 111、4046、4011<br><br><br>3. 创建目录 追加配置（和文档中的一样）<br>  $ mkdir -p &#47;tmp&#47;nfs<br><br>  &#47;etc&#47;exports 内容增加以下内容（我的云服务器外网ip是175.179网段，这里要根据自身的情况修改）：<br>    &#47;tmp&#47;nfs 175.178.0.0&#47;16(rw,sync,no_subtree_check,no_root_squash,insecure)<br><br><br>   $ sudo exportfs -v<br>   $ sudo exportfs -v<br><br>4.开启服务<br><br>	$ sudo systemctl start  nfs-server<br>	$ sudo systemctl enable nfs-server<br>	$ sudo systemctl status nfs-server<br><br>	$ sudo systemctl start  rpcbind<br>	$ sudo systemctl enable rpcbind<br>	$ sudo systemctl status rpcbind<br><br><br>-- 客户端<br>1.安装客户端软件<br><br>   $ sudo apt -y install nfs-common<br>   $ sudo apt -y install rpcbind<br><br>   其它步骤和老师基本一样<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-19 21:12:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/8f/48/c7fb8067.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zzz</span>
  </div>
  <div class="_2_QraFYR_0">nfs provisoner 在生产建议使用HELM包方式进行，而且已经在生产实践过，<br>nfs-subdir-external-provisioner<br>优势2个：<br>1. 无需复杂的配置<br>2. 可支持离线下载，方便传输到生产环境。<br>自己可以玩可以采用教程的方法，了解一下RBAC的流程<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-08 17:25:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cd/e0/c85bb948.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱雯</span>
  </div>
  <div class="_2_QraFYR_0">看了这节课和上节课，终于把storageclass的使用方式搞清楚了，如果都是静态存储和使用，那么只需要在pv和pvc中加上这个name这个属性就可以，并不需要storageClass这个对象，而关键是，这个东西是可以引用provisioner的，让这个和deployment挂钩，其实iwo有一个问题啊，为啥不讲provisioner单独设置为一个对象，而是使用deployment进行部署呢，又是设置环境变量，又是配置参数，另外rabc到底是干嘛的，我真不理解。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这些都是Kubernetes比较深层次的知识点了，感兴趣可以再找其他资料研究，如果平常用不到也就不必浪费精力。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-14 17:34:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/65/da/29fe3dde.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小宝</span>
  </div>
  <div class="_2_QraFYR_0">思考题:<br>1. 动态存储卷PV需要手动运维，而且受限于分配的大小问题，如果分配的PV都比较小，需求又比较大，极易造成PV使用不当，造成浪费。反之，动态存储则按需分配。权利的无限放大也会带来问题，就是预先申请与实际使用之间的矛盾，动态申请大了，也是一种浪费。<br>2. StorageClass应证了“所有问题都可以通过增加一层来解决”。作用是解决了特定底层存储与K8S上资源的解耦问题，通过SC统一接口，具体厂商负责具体的存储实现。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-23 16:10:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/94/fe/5fbf1bdc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Layne</span>
  </div>
  <div class="_2_QraFYR_0">已实操<br>遇到的问题：<br>搭建nfs的时候，目录挂载成功，但是创建文件，没有同步到服务端<br>原因：<br>在挂载的时候，客户端目录和服务端共享目录一样导致的，挂载目录和共享目录不能根目录相同<br>修改前：<br>sudo mount -t nfs 192.168.56.103:&#47;home&#47;layne&#47;data&#47;nfs &#47;home&#47;layne&#47;data&#47;nfs<br>修改后：<br>sudo mount -t nfs 192.168.56.103:&#47;home&#47;layne&#47;data&#47;nfs &#47;data&#47;nfs</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-17 11:00:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0c/5a/3b01789e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Quintos</span>
  </div>
  <div class="_2_QraFYR_0">由于用的是azure国际版的aks服务，无法直接搭建nfsserver。老师能否提供基于azure的nfs sample</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没用过这些云服务器，抱歉了，可以考虑换用其他的存储Provisioner，记得有aws的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-24 11:37:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/ce/58/71ed845f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dexter</span>
  </div>
  <div class="_2_QraFYR_0">这个link是官方的吗？https:&#47;&#47;github.com&#47;kubernetes-sigs&#47;nfs-subdir-external-provisioner</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，看用户名kubernetes-sigs。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-19 08:12:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/9b/64/dadf0ca5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>至尊猴</span>
  </div>
  <div class="_2_QraFYR_0">找到pvc不能调整大小的答案了：nfs provision 不能有空间限制，并且也不能resize<br>------<br>The provisioned storage limit is not enforced. The application can expand to use all the available storage regardless of the provisioned size.<br>Storage resize&#47;expansion operations are not presently supported in any form. You will end up in an error state: Ignoring the PVC: didn&#39;t find a plugin capable of expanding the volume; waiting for an external controller to process this PVC.<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-19 17:43:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/9b/64/dadf0ca5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>至尊猴</span>
  </div>
  <div class="_2_QraFYR_0">请问Chrono老师，对于存储扩容的需求，nfs能满足么？试了一下直接改pvc的配置是报错不能动态调整，<br>通过改storageclass的属性可以实现么？<br>---------------------<br>[root@master nfs]# kubectl apply -f pvc.yml <br>Error from server (Forbidden): error when applying patch:<br>{&quot;metadata&quot;:{&quot;annotations&quot;:{&quot;kubectl.kubernetes.io&#47;last-applied-configuration&quot;:&quot;{\&quot;apiVersion\&quot;:\&quot;v1\&quot;,\&quot;kind\&quot;:\&quot;PersistentVolumeClaim\&quot;,\&quot;metadata\&quot;:{\&quot;annotations\&quot;:{},\&quot;name\&quot;:\&quot;nfs-dyn-10m-pvc\&quot;,\&quot;namespace\&quot;:\&quot;default\&quot;},\&quot;spec\&quot;:{\&quot;accessModes\&quot;:[\&quot;ReadWriteMany\&quot;],\&quot;resources\&quot;:{\&quot;requests\&quot;:{\&quot;storage\&quot;:\&quot;20Mi\&quot;}},\&quot;storageClassName\&quot;:\&quot;nfs-client\&quot;}}\n&quot;}},&quot;spec&quot;:{&quot;resources&quot;:{&quot;requests&quot;:{&quot;storage&quot;:&quot;20Mi&quot;}}}}<br>to:<br>Resource: &quot;&#47;v1, Resource=persistentvolumeclaims&quot;, GroupVersionKind: &quot;&#47;v1, Kind=PersistentVolumeClaim&quot;<br>Name: &quot;nfs-dyn-10m-pvc&quot;, Namespace: &quot;default&quot;<br>for: &quot;pvc.yml&quot;: persistentvolumeclaims &quot;nfs-dyn-10m-pvc&quot; is forbidden: only dynamically provisioned pvc can be resized and the storageclass that provisions the pvc must support resize<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-19 17:21:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/g4os8I4iaB6jn06PsvyqI1BooV5XbOC0vI3niaJ4I3SlAhkbBKG2eewlPHHJ4ROcDia18bbPFSZPDXXmgHXtrBlLg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f2f06e</span>
  </div>
  <div class="_2_QraFYR_0">&#47;tmp&#47;nfs 192.168.10.0&#47;24(rw,sync,no_subtree_check,no_root_squash,insecure)<br><br>执行这句话的时候报了这个错误：<br>bash: syntax error near unexpected token `(&#39;<br><br>加了转译符之后报了这个错误：<br>bash: &#47;tmp&#47;nfs: Is a directory<br><br>这个应该怎么解决啊<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 网上查一下资料，看看在自己的系统上该怎么正确配置nfs。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-11 12:32:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ca/07/22dd76bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kobe</span>
  </div>
  <div class="_2_QraFYR_0">我偶然发现 这个动态数据卷创建的StorageClass和pv好像在每个namespace都会有一个 这个是k8s的设计就是如此么</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-07 10:55:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/3b/29/0f86235e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明月夜</span>
  </div>
  <div class="_2_QraFYR_0">nfs-static-pvc的例子，不需要创建nfs的storageclass吗，直接在pvc里指定storageClassName：nfs 就可以？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kubernetes直接支持nfs。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-02 16:36:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7e/25/3932dafd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GeekNEO</span>
  </div>
  <div class="_2_QraFYR_0">yaml方式，安装github操作，做了下修改：在本地创建个目录，如nfs，放入自定义的yaml。<br>---<br># namespace.yaml<br>apiVersion: v1<br>kind: Namespace<br>metadata:<br>  name: nfs-provisioner<br><br>---<br># patch_nfs_details.yaml<br>apiVersion: apps&#47;v1<br>kind: Deployment<br>metadata:<br>  labels:<br>    app: nfs-client-provisioner<br>  name: nfs-client-provisioner<br>spec:<br>  template:<br>    spec:<br>      containers:<br>        - name: nfs-client-provisioner<br>          image: chronolaw&#47;nfs-subdir-external-provisioner:v4.0.2<br>          env:<br>            - name: NFS_SERVER<br>              value: 172.17.40.171<br>            - name: NFS_PATH<br>              value: &#47;tmp&#47;nfs<br>      volumes:<br>        - name: nfs-client-root<br>          nfs:<br>            server: 172.17.40.171<br>            path: &#47;tmp&#47;nfs<br><br>---<br># kustomization.yaml<br>namespace: nfs-provisioner<br>bases:<br>  - github.com&#47;kubernetes-sigs&#47;nfs-subdir-external-provisioner&#47;&#47;deploy<br>resources:<br>  - namespace.yaml<br>patchesStrategicMerge:<br>  - patch_nfs_details.yaml<br><br>在nfs目录下执行：<br>kubectl apply -k .<br>很简单的完成了，前提是网络可以稍微拿到GitHub的yaml部署文件</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-28 17:18:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7e/25/3932dafd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GeekNEO</span>
  </div>
  <div class="_2_QraFYR_0">spec.nfs路径 不需要提前创建好，kubectl get pv也是可用状态，但是在创建pod状态时，pod状态一直在ContainerCreating的状态，kubectl describe pod nfs-static-pod查看问题，发现还是挂载的问题，挂载找不到目录1g-pv，报错如下：<br>Output: mount.nfs: mounting 172.17.40.171:&#47;tmp&#47;nfs&#47;1g-pv failed, reason given by server: No such file or directory<br>故删除kubectl delete -f nfs-static-pod.yml，在NFS服务器&#47;tmp&#47;nfs&#47;目录下创建目录1g-pv，重新创建容器kubectl apply -f nfs-static-pod.yml，pod创建启动正常。<br><br>但是在1.26.3中，前文我测试hostPath没问题哦，不需要提前创建目录，NFS的就需要。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-28 15:47:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_0e30b3</span>
  </div>
  <div class="_2_QraFYR_0">老师有一个问题就是 ， 假设我的NFS capacity 是10G， 那么我动态创建PV的时候 ， PVC 申请容量大于10 G， 这个会检测到 超过了NFS server 的限制吗？ 不然没有限制， 我可以创建很多PV 即使我的NFS server容量很小</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: NFS server 管理存储空间，应该会检测到。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-23 10:14:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/3c/4d/3dec4bfe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蔡晓慧</span>
  </div>
  <div class="_2_QraFYR_0">nfs.path:&#47;opt&#47;nfs&#47;1g-pv目录我没有手动创建，Pod挂载的时候会提示Output: mount.nfs: mounting 192.168.72.5:&#47;opt&#47;nfs&#47;1g-pv failed, reason given by server: No such file or directory<br><br>但是HostPath挂载我自测是会自动创建的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 也许是kubernetes版本不同，我用1.23是要手工创建的，不过这也是小事，因为实际场景里很少用host path类型。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-16 17:50:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/73/52/8a4cf5e9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>骷髅骨头</span>
  </div>
  <div class="_2_QraFYR_0">pv和pvc的storageClassName都是nfs-client,  而storageClass的name是nfs-client-retain, 是笔误还是？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nfs-client-retained是一个新的storageClass，示范了定制存储特性。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-22 13:32:51</div>
  </div>
</div>
</div>
</li>
</ul>