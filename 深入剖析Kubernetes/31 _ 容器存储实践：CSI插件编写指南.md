<audio title="31 _ 容器存储实践：CSI插件编写指南" src="https://static001.geekbang.org/resource/audio/41/ab/4139f282750e199c1ade2843b7bbc0ab.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：容器存储实践之CSI插件编写指南。</p><p>在上一篇文章中，我已经为你详细讲解了CSI插件机制的设计原理。今天我将继续和你一起实践一个CSI插件的编写过程。</p><p>为了能够覆盖到CSI插件的所有功能，我这一次选择了DigitalOcean的块存储（Block Storage）服务，来作为实践对象。</p><p>DigitalOcean是业界知名的“最简”公有云服务，即：它只提供虚拟机、存储、网络等为数不多的几个基础功能，其他功能一概不管。而这，恰恰就使得DigitalOcean成了我们在公有云上实践Kubernetes的最佳选择。</p><p><span class="orange">我们这次编写的CSI插件的功能，就是：让我们运行在DigitalOcean上的Kubernetes集群能够使用它的块存储服务，作为容器的持久化存储。</span></p><blockquote>
<p>备注：在DigitalOcean上部署一个Kubernetes集群的过程，也很简单。你只需要先在DigitalOcean上创建几个虚拟机，然后按照我们在第11篇文章<a href="https://time.geekbang.org/column/article/39724">《从0到1：搭建一个完整的Kubernetes集群》</a>中从0到1的步骤直接部署即可。</p>
</blockquote><p>而有了CSI插件之后，持久化存储的用法就非常简单了，你只需要创建一个如下所示的StorageClass对象即可：</p><!-- [[[read_end]]] --><pre><code>kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: do-block-storage
  namespace: kube-system
  annotations:
    storageclass.kubernetes.io/is-default-class: &quot;true&quot;
provisioner: com.digitalocean.csi.dobs
</code></pre><p>有了这个StorageClass，External Provisoner就会为集群中新出现的PVC自动创建出PV，然后调用CSI插件创建出这个PV对应的Volume，这正是CSI体系中Dynamic Provisioning的实现方式。</p><blockquote>
<p>备注：<code>storageclass.kubernetes.io/is-default-class: "true"</code>的意思，是使用这个StorageClass作为默认的持久化存储提供者。</p>
</blockquote><p>不难看到，这个StorageClass里唯一引人注意的，是provisioner=com.digitalocean.csi.dobs这个字段。显然，这个字段告诉了Kubernetes，请使用名叫com.digitalocean.csi.dobs的CSI插件来为我处理这个StorageClass相关的所有操作。</p><p>那么，<span class="orange">Kubernetes又是如何知道一个CSI插件的名字的呢？</span></p><p><strong>这就需要从CSI插件的第一个服务CSI Identity说起了。</strong></p><p>其实，一个CSI插件的代码结构非常简单，如下所示：</p><pre><code>tree $GOPATH/src/github.com/digitalocean/csi-digitalocean/driver  
$GOPATH/src/github.com/digitalocean/csi-digitalocean/driver 
├── controller.go
├── driver.go
├── identity.go
├── mounter.go
└── node.go
</code></pre><p>其中，CSI Identity服务的实现，就定义在了driver目录下的identity.go文件里。</p><p>当然，为了能够让Kubernetes访问到CSI Identity服务，我们需要先在driver.go文件里，定义一个标准的gRPC Server，如下所示：</p><pre><code>// Run starts the CSI plugin by communication over the given endpoint
func (d *Driver) Run() error {
 ...
 
 listener, err := net.Listen(u.Scheme, addr)
 ...
 
 d.srv = grpc.NewServer(grpc.UnaryInterceptor(errHandler))
 csi.RegisterIdentityServer(d.srv, d)
 csi.RegisterControllerServer(d.srv, d)
 csi.RegisterNodeServer(d.srv, d)
 
 d.ready = true // we're now ready to go!
 ...
 return d.srv.Serve(listener)
}
</code></pre><p>可以看到，只要把编写好的gRPC Server注册给CSI，它就可以响应来自External Components的CSI请求了。</p><p><strong>CSI Identity服务中，最重要的接口是GetPluginInfo</strong>，它返回的就是这个插件的名字和版本号，如下所示：</p><blockquote>
<p>备注：CSI各个服务的接口我在上一篇文章中已经介绍过，你也可以在这里找到<a href="https://github.com/container-storage-interface/spec/blob/master/csi.proto">它的protoc文件</a>。</p>
</blockquote><pre><code>func (d *Driver) GetPluginInfo(ctx context.Context, req *csi.GetPluginInfoRequest) (*csi.GetPluginInfoResponse, error) {
 resp := &amp;csi.GetPluginInfoResponse{
  Name:          driverName,
  VendorVersion: version,
 }
 ...
}
</code></pre><p>其中，driverName的值，正是"com.digitalocean.csi.dobs"。所以说，Kubernetes正是通过GetPluginInfo的返回值，来找到你在StorageClass里声明要使用的CSI插件的。</p><blockquote>
<p>备注：CSI要求插件的名字遵守<a href="https://en.wikipedia.org/wiki/Reverse_domain_name_notation">“反向DNS”格式</a>。</p>
</blockquote><p>另外一个<strong>GetPluginCapabilities接口也很重要</strong>。这个接口返回的是这个CSI插件的“能力”。</p><p>比如，当你编写的CSI插件不准备实现“Provision阶段”和“Attach阶段”（比如，一个最简单的NFS存储插件就不需要这两个阶段）时，你就可以通过这个接口返回：本插件不提供CSI Controller服务，即：没有csi.PluginCapability_Service_CONTROLLER_SERVICE这个“能力”。这样，Kubernetes就知道这个信息了。</p><p>最后，<strong>CSI Identity服务还提供了一个Probe接口</strong>。Kubernetes会调用它来检查这个CSI插件是否正常工作。</p><p>一般情况下，我建议你在编写插件时给它设置一个Ready标志，当插件的gRPC Server停止的时候，把这个Ready标志设置为false。或者，你可以在这里访问一下插件的端口，类似于健康检查的做法。</p><blockquote>
<p>备注：关于健康检查的问题，你可以再回顾一下第15篇文章<a href="https://time.geekbang.org/column/article/40466">《深入解析Pod对象（二）：使用进阶》</a>中的相关内容。</p>
</blockquote><p>然后，<span class="orange">我们要开始编写CSI 插件的第二个服务，即CSI Controller服务了。</span>它的代码实现，在controller.go文件里。</p><p>在上一篇文章中我已经为你讲解过，这个服务主要实现的就是Volume管理流程中的“Provision阶段”和“Attach阶段”。</p><p><strong>“Provision阶段”对应的接口，是CreateVolume和DeleteVolume</strong>，它们的调用者是External Provisoner。以CreateVolume为例，它的主要逻辑如下所示：</p><pre><code>func (d *Driver) CreateVolume(ctx context.Context, req *csi.CreateVolumeRequest) (*csi.CreateVolumeResponse, error) {
 ...
 
 volumeReq := &amp;godo.VolumeCreateRequest{
  Region:        d.region,
  Name:          volumeName,
  Description:   createdByDO,
  SizeGigaBytes: size / GB,
 }
 
 ...
 
 vol, _, err := d.doClient.Storage.CreateVolume(ctx, volumeReq)
 
 ...
 
 resp := &amp;csi.CreateVolumeResponse{
  Volume: &amp;csi.Volume{
   Id:            vol.ID,
   CapacityBytes: size,
   AccessibleTopology: []*csi.Topology{
    {
     Segments: map[string]string{
      &quot;region&quot;: d.region,
     },
    },
   },
  },
 }
 
 return resp, nil
}
</code></pre><p>可以看到，对于DigitalOcean这样的公有云来说，CreateVolume需要做的操作，就是调用DigitalOcean块存储服务的API，创建出一个存储卷（d.doClient.Storage.CreateVolume）。如果你使用的是其他类型的块存储（比如Cinder、Ceph RBD等），对应的操作也是类似地调用创建存储卷的API。</p><p>而“<strong>Attach阶段”对应的接口是ControllerPublishVolume和ControllerUnpublishVolume</strong>，它们的调用者是External Attacher。以ControllerPublishVolume为例，它的逻辑如下所示：</p><pre><code>func (d *Driver) ControllerPublishVolume(ctx context.Context, req *csi.ControllerPublishVolumeRequest) (*csi.ControllerPublishVolumeResponse, error) {
 ...
 
  dropletID, err := strconv.Atoi(req.NodeId)
  
  // check if volume exist before trying to attach it
  _, resp, err := d.doClient.Storage.GetVolume(ctx, req.VolumeId)
 
 ...
 
  // check if droplet exist before trying to attach the volume to the droplet
  _, resp, err = d.doClient.Droplets.Get(ctx, dropletID)
 
 ...
 
  action, resp, err := d.doClient.StorageActions.Attach(ctx, req.VolumeId, dropletID)

 ...
 
 if action != nil {
  ll.Info(&quot;waiting until volume is attached&quot;)
 if err := d.waitAction(ctx, req.VolumeId, action.ID); err != nil {
  return nil, err
  }
  }
  
  ll.Info(&quot;volume is attached&quot;)
 return &amp;csi.ControllerPublishVolumeResponse{}, nil
}
</code></pre><p>可以看到，对于DigitalOcean来说，ControllerPublishVolume在“Attach阶段”需要做的工作，是调用DigitalOcean的API，将我们前面创建的存储卷，挂载到指定的虚拟机上（d.doClient.StorageActions.Attach）。</p><p>其中，存储卷由请求中的VolumeId来指定。而虚拟机，也就是将要运行Pod的宿主机，则由请求中的NodeId来指定。这些参数，都是External Attacher在发起请求时需要设置的。</p><p>我在上一篇文章中已经为你介绍过，External Attacher的工作原理，是监听（Watch）了一种名叫VolumeAttachment的API对象。这种API对象的主要字段如下所示：</p><pre><code>// VolumeAttachmentSpec is the specification of a VolumeAttachment request.
type VolumeAttachmentSpec struct {
 // Attacher indicates the name of the volume driver that MUST handle this
 // request. This is the name returned by GetPluginName().
 Attacher string
 
 // Source represents the volume that should be attached.
 Source VolumeAttachmentSource
 
 // The node that the volume should be attached to.
 NodeName string
}
</code></pre><p>而这个对象的生命周期，正是由AttachDetachController负责管理的（这里，你可以再回顾一下第28篇文章<a href="https://time.geekbang.org/column/article/42698">《PV、PVC、StorageClass，这些到底在说啥？》</a>中的相关内容）。</p><p>这个控制循环的职责，是不断检查Pod所对应的PV，在它所绑定的宿主机上的挂载情况，从而决定是否需要对这个PV进行Attach（或者Dettach）操作。</p><p>而这个Attach操作，在CSI体系里，就是创建出上面这样一个VolumeAttachment对象。可以看到，Attach操作所需的PV的名字（Source）、宿主机的名字（NodeName）、存储插件的名字（Attacher），都是这个VolumeAttachment对象的一部分。</p><p>而当External Attacher监听到这样的一个对象出现之后，就可以立即使用VolumeAttachment里的这些字段，封装成一个gRPC请求调用CSI Controller的ControllerPublishVolume方法。</p><p>最后，<span class="orange">我们就可以编写CSI Node服务了。</span></p><p>CSI Node服务对应的，是Volume管理流程里的“Mount阶段”。它的代码实现，在node.go文件里。</p><p>我在上一篇文章里曾经提到过，kubelet的VolumeManagerReconciler控制循环会直接调用CSI Node服务来完成Volume的“Mount阶段”。</p><p>不过，在具体的实现中，这个“Mount阶段”的处理其实被细分成了NodeStageVolume和NodePublishVolume这两个接口。</p><p>这里的原因其实也很容易理解：我在第28篇文章<a href="https://time.geekbang.org/column/article/42698">《PV、PVC、StorageClass，这些到底在说啥？》</a>中曾经介绍过，对于磁盘以及块设备来说，它们被Attach到宿主机上之后，就成为了宿主机上的一个待用存储设备。而到了“Mount阶段”，我们首先需要格式化这个设备，然后才能把它挂载到Volume对应的宿主机目录上。</p><p>在kubelet的VolumeManagerReconciler控制循环中，这两步操作分别叫作<strong>MountDevice和SetUp。</strong></p><p>其中，MountDevice操作，就是直接调用了CSI Node服务里的NodeStageVolume接口。顾名思义，这个接口的作用，就是格式化Volume在宿主机上对应的存储设备，然后挂载到一个临时目录（Staging目录）上。</p><p>对于DigitalOcean来说，它对NodeStageVolume接口的实现如下所示：</p><pre><code>func (d *Driver) NodeStageVolume(ctx context.Context, req *csi.NodeStageVolumeRequest) (*csi.NodeStageVolumeResponse, error) {
 ...
 
 vol, resp, err := d.doClient.Storage.GetVolume(ctx, req.VolumeId)
 
 ...
 
 source := getDiskSource(vol.Name)
 target := req.StagingTargetPath
 
 ...
 
 if !formatted {
  ll.Info(&quot;formatting the volume for staging&quot;)
  if err := d.mounter.Format(source, fsType); err != nil {
   return nil, status.Error(codes.Internal, err.Error())
  }
 } else {
  ll.Info(&quot;source device is already formatted&quot;)
 }
 
...

 if !mounted {
  if err := d.mounter.Mount(source, target, fsType, options...); err != nil {
   return nil, status.Error(codes.Internal, err.Error())
  }
 } else {
  ll.Info(&quot;source device is already mounted to the target path&quot;)
 }
 
 ...
 return &amp;csi.NodeStageVolumeResponse{}, nil
}
</code></pre><p>可以看到，在NodeStageVolume的实现里，我们首先通过DigitalOcean的API获取到了这个Volume对应的设备路径（getDiskSource）；然后，我们把这个设备格式化成指定的格式（ d.mounter.Format）；最后，我们把格式化后的设备挂载到了一个临时的Staging目录（StagingTargetPath）下。</p><p>而SetUp操作则会调用CSI Node服务的NodePublishVolume接口。有了上述对设备的预处理工作后，它的实现就非常简单了，如下所示：</p><pre><code>func (d *Driver) NodePublishVolume(ctx context.Context, req *csi.NodePublishVolumeRequest) (*csi.NodePublishVolumeResponse, error) {
 ...
 source := req.StagingTargetPath
 target := req.TargetPath
 
 mnt := req.VolumeCapability.GetMount()
 options := mnt.MountFlag
    ...
    
 if !mounted {
  ll.Info(&quot;mounting the volume&quot;)
  if err := d.mounter.Mount(source, target, fsType, options...); err != nil {
   return nil, status.Error(codes.Internal, err.Error())
  }
 } else {
  ll.Info(&quot;volume is already mounted&quot;)
 }
 
 return &amp;csi.NodePublishVolumeResponse{}, nil
}
</code></pre><p>可以看到，在这一步实现中，我们只需要做一步操作，即：将Staging目录，绑定挂载到Volume对应的宿主机目录上。</p><p>由于Staging目录，正是Volume对应的设备被格式化后挂载在宿主机上的位置，所以当它和Volume的宿主机目录绑定挂载之后，这个Volume宿主机目录的“持久化”处理也就完成了。</p><p>当然，我在前面也曾经提到过，对于文件系统类型的存储服务来说，比如NFS和GlusterFS等，它们并没有一个对应的磁盘“设备”存在于宿主机上，所以kubelet在VolumeManagerReconciler控制循环中，会跳过MountDevice操作而直接执行SetUp操作。所以对于它们来说，也就不需要实现NodeStageVolume接口了。</p><p><span class="orange">在编写完了CSI插件之后，我们就可以把这个插件和External Components一起部署起来。</span></p><p>首先，我们需要创建一个DigitalOcean client授权需要使用的Secret对象，如下所示：</p><pre><code>apiVersion: v1
kind: Secret
metadata:
  name: digitalocean
  namespace: kube-system
stringData:
  access-token: &quot;a05dd2f26b9b9ac2asdas__REPLACE_ME____123cb5d1ec17513e06da&quot;
</code></pre><p>接下来，我们通过一句指令就可以将CSI插件部署起来：</p><pre><code>$ kubectl apply -f https://raw.githubusercontent.com/digitalocean/csi-digitalocean/master/deploy/kubernetes/releases/csi-digitalocean-v0.2.0.yaml
</code></pre><p>这个CSI插件的YAML文件的主要内容如下所示（其中，非重要的内容已经被略去）：</p><pre><code>kind: DaemonSet
apiVersion: apps/v1beta2
metadata:
  name: csi-do-node
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: csi-do-node
  template:
    metadata:
      labels:
        app: csi-do-node
        role: csi-do
    spec:
      serviceAccount: csi-do-node-sa
      hostNetwork: true
      containers:
        - name: driver-registrar
          image: quay.io/k8scsi/driver-registrar:v0.3.0
          ...
        - name: csi-do-plugin
          image: digitalocean/do-csi-plugin:v0.2.0
          args :
            - &quot;--endpoint=$(CSI_ENDPOINT)&quot;
            - &quot;--token=$(DIGITALOCEAN_ACCESS_TOKEN)&quot;
            - &quot;--url=$(DIGITALOCEAN_API_URL)&quot;
          env:
            - name: CSI_ENDPOINT
              value: unix:///csi/csi.sock
            - name: DIGITALOCEAN_API_URL
              value: https://api.digitalocean.com/
            - name: DIGITALOCEAN_ACCESS_TOKEN
              valueFrom:
                secretKeyRef:
                  name: digitalocean
                  key: access-token
          imagePullPolicy: &quot;Always&quot;
          securityContext:
            privileged: true
            capabilities:
              add: [&quot;SYS_ADMIN&quot;]
            allowPrivilegeEscalation: true
          volumeMounts:
            - name: plugin-dir
              mountPath: /csi
            - name: pods-mount-dir
              mountPath: /var/lib/kubelet
              mountPropagation: &quot;Bidirectional&quot;
            - name: device-dir
              mountPath: /dev
      volumes:
        - name: plugin-dir
          hostPath:
            path: /var/lib/kubelet/plugins/com.digitalocean.csi.dobs
            type: DirectoryOrCreate
        - name: pods-mount-dir
          hostPath:
            path: /var/lib/kubelet
            type: Directory
        - name: device-dir
          hostPath:
            path: /dev
---
kind: StatefulSet
apiVersion: apps/v1beta1
metadata:
  name: csi-do-controller
  namespace: kube-system
spec:
  serviceName: &quot;csi-do&quot;
  replicas: 1
  template:
    metadata:
      labels:
        app: csi-do-controller
        role: csi-do
    spec:
      serviceAccount: csi-do-controller-sa
      containers:
        - name: csi-provisioner
          image: quay.io/k8scsi/csi-provisioner:v0.3.0
          ...
        - name: csi-attacher
          image: quay.io/k8scsi/csi-attacher:v0.3.0
          ...
        - name: csi-do-plugin
          image: digitalocean/do-csi-plugin:v0.2.0
          args :
            - &quot;--endpoint=$(CSI_ENDPOINT)&quot;
            - &quot;--token=$(DIGITALOCEAN_ACCESS_TOKEN)&quot;
            - &quot;--url=$(DIGITALOCEAN_API_URL)&quot;
          env:
            - name: CSI_ENDPOINT
              value: unix:///var/lib/csi/sockets/pluginproxy/csi.sock
            - name: DIGITALOCEAN_API_URL
              value: https://api.digitalocean.com/
            - name: DIGITALOCEAN_ACCESS_TOKEN
              valueFrom:
                secretKeyRef:
                  name: digitalocean
                  key: access-token
          imagePullPolicy: &quot;Always&quot;
          volumeMounts:
            - name: socket-dir
              mountPath: /var/lib/csi/sockets/pluginproxy/
      volumes:
        - name: socket-dir
          emptyDir: {}
</code></pre><p>可以看到，我们编写的CSI插件只有一个二进制文件，它的镜像是digitalocean/do-csi-plugin:v0.2.0。</p><p>而我们<strong>部署CSI插件的常用原则是：</strong></p><p><strong>第一，通过DaemonSet在每个节点上都启动一个CSI插件，来为kubelet提供CSI Node服务</strong>。这是因为，CSI Node服务需要被kubelet直接调用，所以它要和kubelet“一对一”地部署起来。</p><p>此外，在上述DaemonSet的定义里面，除了CSI插件，我们还以sidecar的方式运行着driver-registrar这个外部组件。它的作用，是向kubelet注册这个CSI插件。这个注册过程使用的插件信息，则通过访问同一个Pod里的CSI插件容器的Identity服务获取到。</p><p>需要注意的是，由于CSI插件运行在一个容器里，那么CSI Node服务在“Mount阶段”执行的挂载操作，实际上是发生在这个容器的Mount Namespace里的。可是，我们真正希望执行挂载操作的对象，都是宿主机/var/lib/kubelet目录下的文件和目录。</p><p>所以，在定义DaemonSet Pod的时候，我们需要把宿主机的/var/lib/kubelet以Volume的方式挂载进CSI插件容器的同名目录下，然后设置这个Volume的mountPropagation=Bidirectional，即开启双向挂载传播，从而将容器在这个目录下进行的挂载操作“传播”给宿主机，反之亦然。</p><p><strong>第二，通过StatefulSet在任意一个节点上再启动一个CSI插件，为External Components提供CSI Controller服务</strong>。所以，作为CSI Controller服务的调用者，External Provisioner和External Attacher这两个外部组件，就需要以sidecar的方式和这次部署的CSI插件定义在同一个Pod里。</p><p>你可能会好奇，为什么我们会用StatefulSet而不是Deployment来运行这个CSI插件呢。</p><p>这是因为，由于StatefulSet需要确保应用拓扑状态的稳定性，所以它对Pod的更新，是严格保证顺序的，即：只有在前一个Pod停止并删除之后，它才会创建并启动下一个Pod。</p><p>而像我们上面这样将StatefulSet的replicas设置为1的话，StatefulSet就会确保Pod被删除重建的时候，永远有且只有一个CSI插件的Pod运行在集群中。这对CSI插件的正确性来说，至关重要。</p><p>而在今天这篇文章一开始，我们就已经定义了这个CSI插件对应的StorageClass（即：do-block-storage），所以你接下来只需要定义一个声明使用这个StorageClass的PVC即可，如下所示：</p><pre><code>apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: do-block-storage
</code></pre><p>当你把上述PVC提交给Kubernetes之后，你就可以在Pod里声明使用这个csi-pvc来作为持久化存储了。这一部分使用PV和PVC的内容，我就不再赘述了。</p><h2>总结</h2><p>在今天这篇文章中，我以一个DigitalOcean的CSI插件为例，和你分享了编写CSI插件的具体流程。</p><p>基于这些讲述，你现在应该已经对Kubernetes持久化存储体系有了一个更加全面和深入的认识。</p><p>举个例子，对于一个部署了CSI存储插件的Kubernetes集群来说：</p><p>当用户创建了一个PVC之后，你前面部署的StatefulSet里的External Provisioner容器，就会监听到这个PVC的诞生，然后调用同一个Pod里的CSI插件的CSI Controller服务的CreateVolume方法，为你创建出对应的PV。</p><p>这时候，运行在Kubernetes Master节点上的Volume Controller，就会通过PersistentVolumeController控制循环，发现这对新创建出来的PV和PVC，并且看到它们声明的是同一个StorageClass。所以，它会把这一对PV和PVC绑定起来，使PVC进入Bound状态。</p><p>然后，用户创建了一个声明使用上述PVC的Pod，并且这个Pod被调度器调度到了宿主机A上。这时候，Volume Controller的AttachDetachController控制循环就会发现，上述PVC对应的Volume，需要被Attach到宿主机A上。所以，AttachDetachController会创建一个VolumeAttachment对象，这个对象携带了宿主机A和待处理的Volume的名字。</p><p>这样，StatefulSet里的External Attacher容器，就会监听到这个VolumeAttachment对象的诞生。于是，它就会使用这个对象里的宿主机和Volume名字，调用同一个Pod里的CSI插件的CSI Controller服务的ControllerPublishVolume方法，完成“Attach阶段”。</p><p>上述过程完成后，运行在宿主机A上的kubelet，就会通过VolumeManagerReconciler控制循环，发现当前宿主机上有一个Volume对应的存储设备（比如磁盘）已经被Attach到了某个设备目录下。于是kubelet就会调用同一台宿主机上的CSI插件的CSI Node服务的NodeStageVolume和NodePublishVolume方法，完成这个Volume的“Mount阶段”。</p><p>至此，一个完整的持久化Volume的创建和挂载流程就结束了。</p><h2>思考题</h2><p>请你根据编写FlexVolume和CSI插件的流程，分析一下什么时候该使用FlexVolume，什么时候应该使用CSI？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/05/fc/bceb3f2b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>开心哥</span>
  </div>
  <div class="_2_QraFYR_0">果然是云原生技术，听的云里雾里。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-20 16:08:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a5/98/a65ff31a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>djfhchdh</span>
  </div>
  <div class="_2_QraFYR_0">flexVolume插件只负责attach和mount，使用简单。而CSI插件包括了一部分原来kubernetes中存储管理的功能，实现、部署起来比较复杂。所以，如果场景简单，不需要Dynamic Provisioning，则可以使用flexVolume；如果场景复杂，需要支持Dynamic Provisioning，则用CSI插件。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-15 17:37:25</div>
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
  <div class="_2_QraFYR_0">CSI的工作原理： 步骤分为存储插件注册、创建磁盘、挂载磁盘到虚拟机、挂载磁盘到Volume。其中<br><br>插件注册： Driver Register调用CSI的CSI Identify来完成注册，将插件注册到kubelet里面（这可以类比，将可执行文件放在插件目录下）。<br>存储创建：External Provisioner调用CSI的CSI Controller来创建PV和(远程)存储Volume，PV和PVC绑定之后，需要经过Attach和Mount这两阶段之后才能变成宿主机可用的Volume。所以，PV和PVC绑定之后，在Pod所在的宿主机上，执行Attach和Mount，即：<br><br>挂载磁盘到虚拟机： External Attacher调用CSI Controller来将新创建的存储卷挂载到虚拟机上（Attach）<br>格式化并挂载到Volume：k8s的Node节点调用CSI Node 将虚拟机上的存储卷格式化并挂载到容器的Volume上（Mount）<br><br>例：<br><br>当用户创建了一个PVC之后，External Provisioner会监听到这个PVC的诞生，然后调用同一个Pod里的CSI插件的CSI Controller服务的CreateVolume方法，为你创建出对应的PV。这时候，运行在Kubernetes Master节点上的Volume Controller就会通过PersistentVolumeController控制循环，发现这对新创建出来的PV和PVC，并且看到它们声明的是同一个StorageClass。所以，它会把这一对PV和PVC绑定，使PVC进入Bound状态。然后，用户创建一个声明使用上述PVC的Pod，并且这个Pod被调度到了宿主机A上，这时，Volume Controller的AttachDetachController控制循环就会发现，上述PVC对应的Volume，需要被Attach到宿主机A上。所以，AttachDetachController就会创建一个VolumeAttachment对象，这个对象携带了宿主机A和待处理的Volume名字。External  Attacher监听到VolumeAttachment对象的诞生。于是，它就会使用这个对象里的宿主机和Volume名字，调用同一个Pod里的CSI插件的CSI Controller服务的ControllerPublishVolume，完成Attach阶段。上述过程完成后，运行在宿主机A的kubelet，就会通过VolumeManagerReconciler控制循环，发现当前宿主机上有一个Volume对应的存储设备（比如磁盘）已经被Attach到了某个设备目录下。于是kubelet就会调用同一宿主机上的CSI插件的CSI Node服务的NodeStageVolume和NodePublishVolume完成这个Volume的“Mount阶段”。至此，一个完成的持久化Volume的创建和挂载就结束了。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-29 09:18:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/69/4d/81c44f45.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拉欧</span>
  </div>
  <div class="_2_QraFYR_0">老师对k8s的理解真心让人敬佩</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-16 20:43:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e9/91/4219d305.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>初学者</span>
  </div>
  <div class="_2_QraFYR_0">一般来说一个块存储在被宿主机使用之前，需要先将该块存储load 到宿主机的&#47;dev 下成为linux 的设备文件，然后format还设备文件，然后挂载到一个目录下就可以使用了，我觉得nodestagevolume这步挂载操作更像是为了同一台宿主机上的pod 可以共享一块盘</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-07 01:55:11</div>
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
  <div class="_2_QraFYR_0">大师，请教一个问题：&quot;将Staging目录，绑定挂载到Volume对应的宿主机目录上&quot;这个绑定挂载是指mount -bind吗？ 为什么要挂载到一个临时目录，再绑定挂载Volume对应的宿主机目录上，而不是一步挂载到目标目录上？另外Staging目录具体是哪个目录？<br>谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 预处理完成前volume并不是可用的，直接挂载目的目录上显然太早了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-02 16:27:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIh7iatqAeGsJuDNxsDlmCQx64ktJl7ATAkBtDO6iczIqsLFPXkF6GPGJpMBCxbl4DJ5obHwAK0bSAQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱东辉</span>
  </div>
  <div class="_2_QraFYR_0">张大佬真的天花板一样的存储，二刷依然收获满满，多谢大佬提供的这么好的学习资料</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-31 17:28:08</div>
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
  <div class="_2_QraFYR_0">找到了，文中有出现。是在 https:&#47;&#47;raw.githubusercontent.com&#47;digitalocean&#47;csi-digitalocean<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-24 09:41:13</div>
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
  <div class="_2_QraFYR_0">块处理设备从挂载到staging，到挂载到宿主机目录具体做了哪些预处理呢？我和前面几位一样，对需要分两部挂载不是很理解</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-05 04:28:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/30/f3/03/24a0e652.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🍊 🐱</span>
  </div>
  <div class="_2_QraFYR_0">有的同学跟我有一样的疑惑，就是 staging 这一步为什么需要再格式化后挂载到一个临时目录，而不是直接留给 publish 阶段挂在到 pod 中，根据作者给其他小伙伴的回复和我阅读 sample 代码的理解如下：kubelet 的 VolumeManagerReconciler 分两步 mount 的原因是假如直接一步 format 的过程非常久，可能会导致 reconciler 阻塞，而分开两步可以解决这个问题，另外在第一步 staging 后挂载到临时目录（由 req 获取）的目的是方便 reconciler 判断 volum（通过传给第一步 req 的临时目录） 是否 format 完毕，可以进入第二步挂载到 pod 中。如果理解错了还希望作者指出。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-25 14:29:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJutT9JkFAcOk1JxOBdPuLgROpvcuxD9ROP9ACILAHITjcaYGNrZ5lHMZORYM6ibCuScDibYlgRvAIw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>车小勺的男神</span>
  </div>
  <div class="_2_QraFYR_0">请教一下 能用对象存储来作为持久化存储么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以啊 你搜下S3的Kubernetes 存储插件</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-13 13:23:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/5a/65/b80035a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ryan</span>
  </div>
  <div class="_2_QraFYR_0">好像看懂了，又好像没看懂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-22 11:48:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f1b96b</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好：  ControllerPublishVolume 这个方法是如何将 块设备map到node上的？ 它应该是external-attacher （deployment 或daemonset）中的方法，它和node中的哪个进程通信nodeplugin or kubelet？tks</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-30 01:55:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/7b/2b/97e4d599.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Podman</span>
  </div>
  <div class="_2_QraFYR_0">请教一下，glusterfs+heketi+k8s实现的自动绑定PV的模式 我是否可以理解为与ceph+rook+k8s模式一致？CSI实现的自定义存储插件与rook的角色有何不同？glusterfs+heketi+k8s的架构是不是也可以通过CSI来实现？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-17 23:07:48</div>
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
  <div class="_2_QraFYR_0">对于文件系统类型的存储服务，例如: NFS，它并没有对一个的磁盘设备存在与宿主机上，有些nfs类型的csi driver上并没有实现ControllerPublishVolume这个操作，是不是可以理解这里nfs存储卷的attach阶段只是创建了VolumeAttachment对象，并不需要通过ControllerPublishVolume完成nfs volume挂载到虚拟机上？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-07 00:22:16</div>
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
  <div class="_2_QraFYR_0">请问在上一节里提到 “ CSI 的 api 不会直接使用 Kubernetes 定义的 PV 类型， 而是会自己定义一个单独的 volume 类型。 这个在digitalocean csi 里具体体现是什么？是一个cdr吗，我好像没找到。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: csi自己有一套types.go，这跟kubernetes已经没关系了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-03 13:29:03</div>
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
  <div class="_2_QraFYR_0">DJH 竟然和我的问题一模一样，握手！<br>为什么不直接把设备挂载到 volume宿主机目录？在pv&#47;pvc到底讲什么那么一节里就是这么讲的。<br>在这里有什么特殊的考虑吗？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 块存储设备不经过预处理就能直接挂载使用的情况，我还真想不出来。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-03 12:40:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/72/1f/9ddfeff7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>文进</span>
  </div>
  <div class="_2_QraFYR_0">1，CSI一开始部署（每个节点部署）只是向Driver Registrar通过CSI Identify注册了身份，以及部署CSI NODE服务<br>2，第二次部署，集群唯一，部署了CSI Controller服务。<br>3，用户创建PVC时，并且声明StorageClass使用指定身份的CSI插件时，External Provisioner组件调用CSI Controller的createV来创建一个PV，可能是通过远程API创建。此时只是创建，还未进行attach和mount。<br>4，PersistentVolumeController循环会自动将PV和PVC已经绑定在一起。<br>5，当用户的创建需要使用这个PVC的pod时，在kubelet进行调度时，会把此pod和宿主机先绑定。然后AttachDetachController会识别到有个pod未设置attach，然后就会创建VolumeAttachment对象，携带宿主机和pvc的信息。<br>6，集群中的External Attacher会监听到VolumeAttachment创建，然后调用CSI的attach逻辑。<br>7，运行在宿主机 A 上的 kubelet，就会通过 VolumeManagerReconciler 控制循环，发现当前宿主机上有一个 Volume 对应的存储设备（比如磁盘）已经被 Attach 到了某个设备目录下。于是 kubelet 就会调用同一台宿主机上的 CSI 插件的 CSI Node 服务的 NodeStageVolume 和 NodePublishVolume 方法，完成这个 Volume 的“Mount 阶段”</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-03 18:08:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/48/ce/9978ed21.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>linyy</span>
  </div>
  <div class="_2_QraFYR_0">老师问下 存储插件本身的高可用怎么处理呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-21 20:43:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/f7/cb/771876e5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Double f</span>
  </div>
  <div class="_2_QraFYR_0">一章看了4遍，感觉每一遍收获都不一样，感谢大师</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-10 18:25:57</div>
  </div>
</div>
</div>
</li>
</ul>