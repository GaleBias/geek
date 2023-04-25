<audio title="30 _ 编写自己的存储插件：FlexVolume与CSI" src="https://static001.geekbang.org/resource/audio/bc/77/bc308ea748a442563757ca59218d3777.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：编写自己的存储插件之FlexVolume与CSI。</p><p>在上一篇文章中，我为你详细介绍了Kubernetes里的持久化存储体系，讲解了PV和PVC的具体实现原理，并提到了这样的设计实际上是出于对整个存储体系的可扩展性的考虑。</p><p>而在今天这篇文章中，我就和你分享一下如何借助这些机制，来开发自己的存储插件。</p><p>在Kubernetes中，存储插件的开发有两种方式：FlexVolume和CSI。</p><p>接下来，我就先为你剖析一下<span class="orange">Flexvolume的原理和使用方法</span>。</p><p>举个例子，现在我们要编写的是一个使用NFS实现的FlexVolume插件。</p><p>对于一个FlexVolume类型的PV来说，它的YAML文件如下所示：</p><pre><code>apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-flex-nfs
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  flexVolume:
    driver: &quot;k8s/nfs&quot;
    fsType: &quot;nfs&quot;
    options:
      server: &quot;10.10.0.25&quot; # 改成你自己的NFS服务器地址
      share: &quot;export&quot;
</code></pre><p>可以看到，这个PV定义的Volume类型是flexVolume。并且，我们<strong>指定了这个Volume的driver叫作k8s/nfs</strong>。这个名字很重要，我后面马上会为你解释它的含义。</p><p>而Volume的options字段，则是一个自定义字段。也就是说，它的类型，其实是map[string]string。所以，你可以在这一部分自由地加上你想要定义的参数。</p><p>在我们这个例子里，options字段指定了NFS服务器的地址（server: “10.10.0.25”），以及NFS共享目录的名字（share: “export”）。当然，你这里定义的所有参数，后面都会被FlexVolume拿到。</p><!-- [[[read_end]]] --><blockquote>
<p>备注：你可以使用<a href="https://github.com/ehough/docker-nfs-server">这个Docker镜像</a>轻松地部署一个试验用的NFS服务器。</p>
</blockquote><p>像这样的一个PV被创建后，一旦和某个PVC绑定起来，这个FlexVolume类型的Volume就会进入到我们前面讲解过的Volume处理流程。</p><p>你应该还记得，这个流程的名字叫作“两阶段处理”，即“Attach阶段”和“Mount阶段”。它们的主要作用，是在Pod所绑定的宿主机上，完成这个Volume目录的持久化过程，比如为虚拟机挂载磁盘（Attach），或者挂载一个NFS的共享目录（Mount）。</p><blockquote>
<p>备注：你可以再回顾一下第28篇文章<a href="https://time.geekbang.org/column/article/42698">《PV、PVC、StorageClass，这些到底在说啥？》</a>中的相关内容。</p>
</blockquote><p>而在具体的控制循环中，这两个操作实际上调用的，正是Kubernetes的pkg/volume目录下的存储插件（Volume Plugin）。在我们这个例子里，就是pkg/volume/flexvolume这个目录里的代码。</p><p>当然了，这个目录其实只是FlexVolume插件的入口。以“Mount阶段”为例，在FlexVolume目录里，它的处理过程非常简单，如下所示：</p><pre><code>// SetUpAt creates new directory.
func (f *flexVolumeMounter) SetUpAt(dir string, fsGroup *int64) error {
  ...
  call := f.plugin.NewDriverCall(mountCmd)
  
  // Interface parameters
  call.Append(dir)
  
  extraOptions := make(map[string]string)
  
  // pod metadata
  extraOptions[optionKeyPodName] = f.podName
  extraOptions[optionKeyPodNamespace] = f.podNamespace
  
  ...
  
  call.AppendSpec(f.spec, f.plugin.host, extraOptions)
  
  _, err = call.Run()
  
  ...
  
  return nil
}
</code></pre><p>上面这个名叫SetUpAt()的方法，正是FlexVolume插件对“Mount阶段”的实现位置。而SetUpAt()实际上只做了一件事，那就是封装出了一行命令（即：NewDriverCall），由kubelet在“Mount阶段”去执行。</p><p>在我们这个例子中，<strong>kubelet要通过插件在宿主机上执行的命令，如下所示</strong>：</p><pre><code>/usr/libexec/kubernetes/kubelet-plugins/volume/exec/k8s~nfs/nfs mount &lt;mount dir&gt; &lt;json param&gt;
</code></pre><p>其中，/usr/libexec/kubernetes/kubelet-plugins/volume/exec/k8s~nfs/nfs就是插件的可执行文件的路径。这个名叫nfs的文件，正是你要编写的插件的实现。它可以是一个二进制文件，也可以是一个脚本。总之，只要能在宿主机上被执行起来即可。</p><p>而且这个路径里的k8s~nfs部分，正是这个插件在Kubernetes里的名字。它是从driver="k8s/nfs"字段解析出来的。</p><p>这个driver字段的格式是：vendor/driver。比如，一家存储插件的提供商（vendor）的名字叫作k8s，提供的存储驱动（driver）是nfs，那么Kubernetes就会使用k8s~nfs来作为插件名。</p><p>所以说，<strong>当你编写完了FlexVolume的实现之后，一定要把它的可执行文件放在每个节点的插件目录下。</strong></p><p>而紧跟在可执行文件后面的“mount”参数，定义的就是当前的操作。在FlexVolume里，这些操作参数的名字是固定的，比如init、mount、unmount、attach，以及dettach等等，分别对应不同的Volume处理操作。</p><p>而跟在mount参数后面的两个字段：<code>&lt;mount dir&gt;</code>和<code>&lt;json params&gt;</code>，则是FlexVolume必须提供给这条命令的两个执行参数。</p><p>其中第一个执行参数<code>&lt;mount dir&gt;</code>，正是kubelet调用SetUpAt()方法传递来的dir的值。它代表的是当前正在处理的Volume在宿主机上的目录。在我们的例子里，这个路径如下所示：</p><pre><code>/var/lib/kubelet/pods/&lt;Pod ID&gt;/volumes/k8s~nfs/test
</code></pre><p>其中，test正是我们前面定义的PV的名字；而k8s~nfs，则是插件的名字。可以看到，插件的名字正是从你声明的driver="k8s/nfs"字段里解析出来的。</p><p>而第二个执行参数<code>&lt;json params&gt;</code>，则是一个JSON Map格式的参数列表。我们在前面PV里定义的options字段的值，都会被追加在这个参数里。此外，在SetUpAt()方法里可以看到，这个参数列表里还包括了Pod的名字、Namespace等元数据（Metadata）。</p><p>在明白了存储插件的调用方式和参数列表之后，<span class="orange">这个插件的可执行文件的实现部分</span>就非常容易理解了。</p><p>在这个例子中，我直接编写了一个简单的shell脚本来作为插件的实现，它对“Mount阶段”的处理过程，如下所示：</p><pre><code>domount() {
 MNTPATH=$1
 
 NFS_SERVER=$(echo $2 | jq -r '.server')
 SHARE=$(echo $2 | jq -r '.share')
 
 ...
 
 mkdir -p ${MNTPATH} &amp;&gt; /dev/null
 
 mount -t nfs ${NFS_SERVER}:/${SHARE} ${MNTPATH} &amp;&gt; /dev/null
 if [ $? -ne 0 ]; then
  err &quot;{ \&quot;status\&quot;: \&quot;Failure\&quot;, \&quot;message\&quot;: \&quot;Failed to mount ${NFS_SERVER}:${SHARE} at ${MNTPATH}\&quot;}&quot;
  exit 1
 fi
 log '{&quot;status&quot;: &quot;Success&quot;}'
 exit 0
}
</code></pre><p>可以看到，当kubelet在宿主机上执行“<code>nfs mount &lt;mount dir&gt; &lt;json params&gt;</code>”的时候，这个名叫nfs的脚本，就可以直接从<code>&lt;mount dir&gt;</code>参数里拿到Volume在宿主机上的目录，即：<code>MNTPATH=$1</code>。而你在PV的options字段里定义的NFS的服务器地址（options.server）和共享目录名字（options.share），则可以从第二个<code>&lt;json params&gt;</code>参数里解析出来。这里，我们使用了jq命令，来进行解析工作。</p><p>有了这三个参数之后，这个脚本最关键的一步，当然就是执行：<code>mount -t nfs ${NFS_SERVER}:/${SHARE} ${MNTPATH}</code> 。这样，一个NFS的数据卷就被挂载到了MNTPATH，也就是Volume所在的宿主机目录上，一个持久化的Volume目录就处理完了。</p><p>需要注意的是，当这个mount -t nfs操作完成后，你必须把一个JOSN格式的字符串，比如：{“status”: “Success”}，返回给调用者，也就是kubelet。这是kubelet判断这次调用是否成功的唯一依据。</p><p>综上所述，在“Mount阶段”，kubelet的VolumeManagerReconcile控制循环里的一次“调谐”操作的执行流程，如下所示：</p><pre><code>kubelet --&gt; pkg/volume/flexvolume.SetUpAt() --&gt; /usr/libexec/kubernetes/kubelet-plugins/volume/exec/k8s~nfs/nfs mount &lt;mount dir&gt; &lt;json param&gt;
</code></pre><blockquote>
<p>备注：这个NFS的FlexVolume的完整实现，在<a href="https://github.com/kubernetes/examples/blob/master/staging/volumes/flexvolume/nfs">这个GitHub库</a>里。而你如果想用Go语言编写FlexVolume的话，我也有一个<a href="https://github.com/kubernetes/frakti/tree/master/pkg/flexvolume">很好的例子</a>供你参考。</p>
</blockquote><p>当然，在前面文章中我也提到过，像NFS这样的文件系统存储，并不需要在宿主机上挂载磁盘或者块设备。所以，我们也就不需要实现attach和dettach操作了。</p><p>不过，<strong>像这样的FlexVolume实现方式，虽然简单，但局限性却很大。</strong></p><p>比如，跟Kubernetes内置的NFS插件类似，这个NFS FlexVolume插件，也不能支持Dynamic Provisioning（即：为每个PVC自动创建PV和对应的Volume）。除非你再为它编写一个专门的External Provisioner。</p><p>再比如，我的插件在执行mount操作的时候，可能会生成一些挂载信息。这些信息，在后面执行unmount操作的时候会被用到。可是，在上述FlexVolume的实现里，你没办法把这些信息保存在一个变量里，等到unmount的时候直接使用。</p><p>这个原因也很容易理解：<strong>FlexVolume每一次对插件可执行文件的调用，都是一次完全独立的操作</strong>。所以，我们只能把这些信息写在一个宿主机上的临时文件里，等到unmount的时候再去读取。</p><p>这也是为什么，我们需要有Container Storage Interface（CSI）这样更完善、更编程友好的插件方式。</p><p>接下来，我就来为你讲解一下开发存储插件的第二种方式CSI。我们先来看一下<span class="orange">CSI插件体系的设计原理</span>。</p><p>其实，通过前面对FlexVolume的讲述，你应该可以明白，默认情况下，Kubernetes里通过存储插件管理容器持久化存储的原理，可以用如下所示的示意图来描述：</p><p><img src="https://static001.geekbang.org/resource/image/6a/ef/6a553321623f6b58f5494b25091592ef.png?wh=1212*946" alt=""><br>
可以看到，在上述体系下，无论是FlexVolume，还是Kubernetes内置的其他存储插件，它们实际上担任的角色，仅仅是Volume管理中的“Attach阶段”和“Mount阶段”的具体执行者。而像Dynamic Provisioning这样的功能，就不是存储插件的责任，而是Kubernetes本身存储管理功能的一部分。</p><p>相比之下，<strong>CSI插件体系的设计思想，就是把这个Provision阶段，以及Kubernetes里的一部分存储管理功能，从主干代码里剥离出来，做成了几个单独的组件</strong>。这些组件会通过Watch API监听Kubernetes里与存储相关的事件变化，比如PVC的创建，来执行具体的存储管理动作。</p><p>而这些管理动作，比如“Attach阶段”和“Mount阶段”的具体操作，实际上就是通过调用CSI插件来完成的。</p><p>这种设计思路，我可以用如下所示的一幅示意图来表示：<br>
<img src="https://static001.geekbang.org/resource/image/d4/ad/d4bdc7035f1286e7a423da851eee89ad.png?wh=1880*941" alt=""><br>
可以看到，这套存储插件体系多了三个独立的外部组件（External Components），即：Driver Registrar、External Provisioner和External Attacher，对应的正是从Kubernetes项目里面剥离出来的那部分存储管理功能。</p><p>需要注意的是，External Components虽然是外部组件，但依然由Kubernetes社区来开发和维护。</p><p>而图中最右侧的部分，就是需要我们编写代码来实现的CSI插件。一个CSI插件只有一个二进制文件，但它会以gRPC的方式对外提供三个服务（gRPC Service），分别叫作：CSI Identity、CSI Controller和CSI Node。</p><p><span class="orange">我先来为你讲解一下这三个External Components</span>。</p><p>其中，<strong>Driver Registrar组件，负责将插件注册到kubelet里面</strong>（这可以类比为，将可执行文件放在插件目录下）。而在具体实现上，Driver Registrar需要请求CSI插件的Identity服务来获取插件信息。</p><p>而<strong>External Provisioner组件，负责的正是Provision阶段</strong>。在具体实现上，External Provisioner监听（Watch）了APIServer里的PVC对象。当一个PVC被创建时，它就会调用CSI Controller的CreateVolume方法，为你创建对应PV。</p><p>此外，如果你使用的存储是公有云提供的磁盘（或者块设备）的话，这一步就需要调用公有云（或者块设备服务）的API来创建这个PV所描述的磁盘（或者块设备）了。</p><p>不过，由于CSI插件是独立于Kubernetes之外的，所以在CSI的API里不会直接使用Kubernetes定义的PV类型，而是会自己定义一个单独的Volume类型。</p><p><strong>为了方便叙述，在本专栏里，我会把Kubernetes里的持久化卷类型叫作PV，把CSI里的持久化卷类型叫作CSI Volume，请你务必区分清楚。</strong></p><p>最后一个<strong>External Attacher组件，负责的正是“Attach阶段”</strong>。在具体实现上，它监听了APIServer里VolumeAttachment对象的变化。VolumeAttachment对象是Kubernetes确认一个Volume可以进入“Attach阶段”的重要标志，我会在下一篇文章里为你详细讲解。</p><p>一旦出现了VolumeAttachment对象，External Attacher就会调用CSI Controller服务的ControllerPublish方法，完成它所对应的Volume的Attach阶段。</p><p>而Volume的“Mount阶段”，并不属于External Components的职责。当kubelet的VolumeManagerReconciler控制循环检查到它需要执行Mount操作的时候，会通过pkg/volume/csi包，直接调用CSI Node服务完成Volume的“Mount阶段”。</p><p>在实际使用CSI插件的时候，我们会将这三个External Components作为sidecar容器和CSI插件放置在同一个Pod中。由于External Components对CSI插件的调用非常频繁，所以这种sidecar的部署方式非常高效。</p><p>接下来，<span class="orange">我再为你讲解一下CSI插件的里三个服务</span>：CSI Identity、CSI Controller和CSI Node。</p><p>其中，<strong>CSI插件的CSI Identity服务，负责对外暴露这个插件本身的信息</strong>，如下所示：</p><pre><code>service Identity {
  // return the version and name of the plugin
  rpc GetPluginInfo(GetPluginInfoRequest)
    returns (GetPluginInfoResponse) {}
  // reports whether the plugin has the ability of serving the Controller interface
  rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
    returns (GetPluginCapabilitiesResponse) {}
  // called by the CO just to check whether the plugin is running or not
  rpc Probe (ProbeRequest)
    returns (ProbeResponse) {}
}
</code></pre><p>而<strong>CSI Controller服务，定义的则是对CSI Volume（对应Kubernetes里的PV）的管理接口</strong>，比如：创建和删除CSI Volume、对CSI Volume进行Attach/Dettach（在CSI里，这个操作被叫作Publish/Unpublish），以及对CSI Volume进行Snapshot等，它们的接口定义如下所示：</p><pre><code>service Controller {
  // provisions a volume
  rpc CreateVolume (CreateVolumeRequest)
    returns (CreateVolumeResponse) {}
    
  // deletes a previously provisioned volume
  rpc DeleteVolume (DeleteVolumeRequest)
    returns (DeleteVolumeResponse) {}
    
  // make a volume available on some required node
  rpc ControllerPublishVolume (ControllerPublishVolumeRequest)
    returns (ControllerPublishVolumeResponse) {}
    
  // make a volume un-available on some required node
  rpc ControllerUnpublishVolume (ControllerUnpublishVolumeRequest)
    returns (ControllerUnpublishVolumeResponse) {}
    
  ...
  
  // make a snapshot
  rpc CreateSnapshot (CreateSnapshotRequest)
    returns (CreateSnapshotResponse) {}
    
  // Delete a given snapshot
  rpc DeleteSnapshot (DeleteSnapshotRequest)
    returns (DeleteSnapshotResponse) {}
    
  ...
}
</code></pre><p>不难发现，CSI Controller服务里定义的这些操作有个共同特点，那就是它们都无需在宿主机上进行，而是属于Kubernetes里Volume Controller的逻辑，也就是属于Master节点的一部分。</p><p>需要注意的是，正如我在前面提到的那样，CSI Controller服务的实际调用者，并不是Kubernetes（即：通过pkg/volume/csi发起CSI请求），而是External Provisioner和External Attacher。这两个External Components，分别通过监听 PVC和VolumeAttachement对象，来跟Kubernetes进行协作。</p><p>而CSI Volume需要在宿主机上执行的操作，都定义在了CSI Node服务里面，如下所示：</p><pre><code>service Node {
  // temporarily mount the volume to a staging path
  rpc NodeStageVolume (NodeStageVolumeRequest)
    returns (NodeStageVolumeResponse) {}
    
  // unmount the volume from staging path
  rpc NodeUnstageVolume (NodeUnstageVolumeRequest)
    returns (NodeUnstageVolumeResponse) {}
    
  // mount the volume from staging to target path
  rpc NodePublishVolume (NodePublishVolumeRequest)
    returns (NodePublishVolumeResponse) {}
    
  // unmount the volume from staging path
  rpc NodeUnpublishVolume (NodeUnpublishVolumeRequest)
    returns (NodeUnpublishVolumeResponse) {}
    
  // stats for the volume
  rpc NodeGetVolumeStats (NodeGetVolumeStatsRequest)
    returns (NodeGetVolumeStatsResponse) {}
    
  ...
  
  // Similar to NodeGetId
  rpc NodeGetInfo (NodeGetInfoRequest)
    returns (NodeGetInfoResponse) {}
}
</code></pre><p>需要注意的是，“Mount阶段”在CSI Node里的接口，是由NodeStageVolume和NodePublishVolume两个接口共同实现的。我会在下一篇文章中，为你详细介绍这个设计的目的和具体的实现方式。</p><h2>总结</h2><p>在本篇文章里，我为你详细讲解了FlexVolume和CSI这两种自定义存储插件的工作原理。</p><p>可以看到，相比于FlexVolume，CSI的设计思想，把插件的职责从“两阶段处理”，扩展成了Provision、Attach和Mount三个阶段。其中，Provision等价于“创建磁盘”，Attach等价于“挂载磁盘到虚拟机”，Mount等价于“将该磁盘格式化后，挂载在Volume的宿主机目录上”。</p><p>在有了CSI插件之后，Kubernetes本身依然按照我在第28篇文章<a href="https://time.geekbang.org/column/article/42698">《PV、PVC、StorageClass，这些到底在说啥？》</a>中所讲述的方式工作，唯一区别在于：</p><ul>
<li>当AttachDetachController需要进行“Attach”操作时（“Attach阶段”），它实际上会执行到pkg/volume/csi目录中，创建一个VolumeAttachment对象，从而触发External Attacher调用CSI Controller服务的ControllerPublishVolume方法。</li>
<li>当VolumeManagerReconciler需要进行“Mount”操作时（“Mount阶段”），它实际上也会执行到pkg/volume/csi目录中，直接向CSI Node服务发起调用NodePublishVolume方法的请求。</li>
</ul><p>以上，就是CSI插件最基本的工作原理了。</p><p>在下一篇文章里，我会和你一起实践一个CSI存储插件的完整实现过程。</p><h2>思考题</h2><p>假设现在，你的宿主机是阿里云的一台虚拟机，你要实现的容器持久化存储，是基于阿里云提供的云盘。你能准确地描述出，在Provision、Attach和Mount阶段，CSI插件都需要做哪些操作吗？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/94/47/75875257.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>虎虎❤️</span>
  </div>
  <div class="_2_QraFYR_0">思考题：<br><br>1. Register过程： csi 插件应该作为 daemonSet 部署到每个节点（node）。然后插件 container 挂载 hostpath 文件夹，把插件可执行文件放在其中，并启动rpc服务（identity, controller, node）。External component Driver Registrar 利用 kubelet plugin watcher 特性watch指定的文件夹路径来自动检测到这个存储插件。然后通过调用identity rpc服务，获得driver的信息，并完成注册。<br><br><br>2. Provision过程：部署External Provisioner。 Provisioner 将会 watch apiServer 中 PVC 资源的创建，并且PVC 所指定的 storageClass 的 provisioner是我们上面启动的插件。那么，External Provisioner 将会调用 插件的 controller.createVolume() 服务。其主要工作应该是通过阿里云的api 创建网络磁盘，并根据磁盘的信息创建相应的pv。<br><br>3. Attach过程：部署External Attacher。Attacher 将会监听 apiServer 中 VolumeAttachment 对象的变化。一旦出现新的VolumeAttachment，Attacher 会调用插件的 controller.ControllerPublish() 服务。其主要工作是调用阿里云的api，把相应的磁盘 attach 到声明使用此 PVC&#47;PV 的 pod 所调度到的 node 上。挂载的目录：&#47;var&#47;lib&#47;kubelet&#47;pods&#47;&lt;Pod ID&gt;&#47;volumes&#47;aliyun~netdisk&#47;&lt;name&gt;<br><br><br>4. Mount过程：mount 不可能在远程的container里完成，所以这个工作需要kubelet来做。kubelet 的 VolumeManagerReconciler 控制循环，检测到需要执行 Mount 操作的时候，通过调用 pkg&#47;volume&#47;csi 包，调用 CSI Node 服务，完成 volume 的 Mount 阶段。具体工作是调用 CRI 启动带有 volume 参数的container，把上阶段准备好的磁盘 mount 到 container指定的目录。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-31 14:24:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ch_ort</span>
  </div>
  <div class="_2_QraFYR_0">PVC描述的，是Pod想要使用的持久化存储的属性，比如存储的大小、读写权限等<br>PV描述的，则是一个具体的Volume的属性，比如Volume的类型、挂载目录、远程存储服务器地址等<br><br>有两种管理PV的方式：  人工管理（Static Provisioning），自动创建（Dynamic Provisioning）。Dynamic Provisioning机制工作的核心，就在于一个名叫StorageClass的API对象。Kubernetes能够根据用户提交的PVC，找到一个对应的StorageClass了。然后，Kuberentes就会调用该StorageClass声明的存储插件，创建出需要的PV。<br><br>需要注意的是，StorageClass并不是专门为了Dynamic Provisioning而设计的。比如，我在PV和PVC里都声明了storageClassName=manual。而我的集群里，实际上并没有一个叫manual的StorageClass对象。这完全没有问题，这个时候Kubernetes进行的是Static Provisioning，但在做绑定决策的时候，它依然会考虑PV和PVC的StorageClass定义。而这么做的好处也很明显：这个PVC和PV的绑定关系，就完全在我自己的掌握之中。<br><br>存储插件的开发方式有两种：FlexVolume和CSI <br><br>FlexVolume： kubelet --&gt; pkg&#47;volume&#47;flexvolume.SetUpAt() --&gt; &#47;usr&#47;libexec&#47;kubernetes&#47;kubelet-plugins&#47;volume&#47;exec&#47;k8s~nfs&#47;nfs mount &lt;mount dir&gt; &lt;json param&gt;<br><br>FlexVolume的方式，需要手动先创建好PV，由FlexVolume来执行Attach和Mount操作。<br><br>相比于 FlexVolume，CSI 的设计思想是把插件的职责从“两阶段处理”，扩展成了 Provision、Attach 和 Mount 三个阶段。其中，Provision 等价于“创建磁盘”，Attach 等价于“挂载磁盘到虚拟机”，Mount 等价于“将该磁盘格式化后，挂载在 Volume 的宿主机目录上”。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-29 09:18:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_5baa01</span>
  </div>
  <div class="_2_QraFYR_0">Provision：调用阿里云 API  Create 云盘<br>Attach： 调用阿里云 API 挂载云盘到 ECS <br>Mount: 挂载云盘到对应的 pod volume 目录</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-06 18:30:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/1b/b4/a6db1c1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>silver</span>
  </div>
  <div class="_2_QraFYR_0">&#39;test 正是我们前面定义的 PV 的名字&#39;,这个是否是typo？PV的名字是pv-flex-nfs？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-01 13:26:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/09/e1/100a0526.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kakj</span>
  </div>
  <div class="_2_QraFYR_0">java程序员从入门到放弃到再入门到再放弃中</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 20:56:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b4/8a/ffc23116.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leo</span>
  </div>
  <div class="_2_QraFYR_0">厉害了 新的知识！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-20 23:06:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>vincent</span>
  </div>
  <div class="_2_QraFYR_0">简单说是两阶段：<br>attach + mount<br>细了说：1、创建卷资源  2、attach节点  3、node上创建设备格式化  4、挂在设备</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-29 12:39:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/94/47/75875257.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>虎虎❤️</span>
  </div>
  <div class="_2_QraFYR_0">问题：<br><br>1. 既然csi的PV是自己定义的类型，那么volume controller应该不会做这个红娘吧？所以问题是，他们是怎么完成绑定的？绑定后的状态会改变为 bound 吗？<br>2. 按照我的理解 driver 插件应该安装到每个node上，那么适合使用 daemonSet 去部署插件和 Driver Registerar sidecar。而 External Provisioner&#47;Attacher 则只需要一份部署就可以。为什么文中建议把三个 External components 都部署为sidecar？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-31 14:24:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/01/1d/12d716c9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>guolisen</span>
  </div>
  <div class="_2_QraFYR_0">kubelet为什么会知道 对应的可执行程序叫做nfs(k8s~nfs&#47;nfs)？是在哪里告诉kubelet的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-15 23:03:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Ezw43LrWwwqsCxuccwwQ7OibXq5sxJPOnicsPpFvBrkQQXcuhqfnxXq5ypcVY0Vg5AiaXZbh3tXmibH9icjcaFuBOsw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BobToGo</span>
  </div>
  <div class="_2_QraFYR_0">🐮🍺</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-12 18:32:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/e8/98/6adce1b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>chenkai-1</span>
  </div>
  <div class="_2_QraFYR_0">1.register(包含调用identity获取插件信息）：将插件注册到kubelet里面，将可执行文件放在插件目录下<br>2.External Provisioner：处理 Provision 的阶段。External Provisioner 监听APIServer 里的 PVC 对象。当一个 PVC 被创建时，调用 CSI Controller 的 CreateVolume 方法，创建PV。<br>3.External Attacher ：处理“Attach 阶段”。监听了 APIServer 里 VolumeAttachment 对象的变化。一旦出现了 VolumeAttachment 对象，External Attacher 就会调用 CSI Controller 服务的 ControllerPublish 方法，完成它所对应的 Volume 的 Attach 阶段。<br>4.mount：kubelet 的 VolumeManagerReconciler 控制循环检查到它需要执行 Mount 操作的时候，会通过 pkg&#47;volume&#47;csi 包，直接调用 CSI Node 服务完成 Volume 的“Mount 阶段”。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-01 20:52:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/07/d2/0d7ee298.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>惘 闻</span>
  </div>
  <div class="_2_QraFYR_0">脑壳疼... 没用过docker 没用过k8s,操作系统知识不扎实,导致我看到这里好累啊,还是基础差</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-05 15:28:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/3d/0e/92176eaa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>左氧佛沙星人</span>
  </div>
  <div class="_2_QraFYR_0">思考题，应该参考local path storage provisioner 或者 local volume storage provisioner，需要新增的是，讲创建好的云盘，attach到主机上，这样对吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-16 16:30:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/83/ae/c082bb25.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大星星</span>
  </div>
  <div class="_2_QraFYR_0">有个问题，请教下，三个external组建为什么要独立出来。这个不需要吧。只要csi 三个服务起来了，自动注册插件。他们三个服务也负责watch api，每当有请求过来，provision attatch.mount动作时候分别找对应服务请求就行。不知道都一个个分出来有必要么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-15 13:47:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/b6/97/93e82345.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陆培尔</span>
  </div>
  <div class="_2_QraFYR_0">老师的课讲得太好了，什么时候开始讲容器网络方面的内容？感觉这一块一直有很多地方搞不明白，service,ingress，lb,跨节点组网等等</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很快</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-02 14:08:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a5/cd/3aff5d57.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Alery</span>
  </div>
  <div class="_2_QraFYR_0">你好，自己实现的nfs flexvolume是否可以在pvc中指定呢？怎么指定呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-31 23:32:00</div>
  </div>
</div>
</div>
</li>
</ul>