<audio title="24 _ 深入解析声明式API（一）：API对象的奥秘" src="https://static001.geekbang.org/resource/audio/ab/70/ab2cf4d567d3ced9caa349bd0b9dbe70.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：深入解析声明式API之API对象的奥秘。</p><p>在上一篇文章中，我为你详细讲解了Kubernetes声明式API的设计、特点，以及使用方式。</p><p>而在今天这篇文章中，我就来为你讲解一下Kubernetes声明式API的工作原理，以及如何利用这套API机制，在Kubernetes里添加自定义的API对象。</p><p>你可能一直就很好奇：当我把一个YAML文件提交给Kubernetes之后，它究竟是如何创建出一个API对象的呢？</p><p>这得从声明式API的设计谈起了。</p><p>在Kubernetes项目中，一个API对象在Etcd里的完整资源路径，是由：Group（API组）、Version（API版本）和Resource（API资源类型）三个部分组成的。</p><p>通过这样的结构，整个Kubernetes里的所有API对象，实际上就可以用如下的树形结构表示出来：</p><p><img src="https://static001.geekbang.org/resource/image/70/da/709700eea03075bed35c25b5b6cdefda.png?wh=1820*783" alt=""><br>
在这幅图中，你可以很清楚地看到<strong>Kubernetes里API对象的组织方式，其实是层层递进的。</strong></p><p>比如，现在我要声明要创建一个CronJob对象，那么我的YAML文件的开始部分会这么写：</p><pre><code>apiVersion: batch/v2alpha1
kind: CronJob
...
</code></pre><p>在这个YAML文件中，“CronJob”就是这个API对象的资源类型（Resource），“batch”就是它的组（Group），v2alpha1就是它的版本（Version）。</p><!-- [[[read_end]]] --><p>当我们提交了这个YAML文件之后，Kubernetes就会把这个YAML文件里描述的内容，转换成Kubernetes里的一个CronJob对象。</p><p>那么，<span class="orange">Kubernetes是如何对Resource、Group和Version进行解析，从而在Kubernetes项目里找到CronJob对象的定义呢？</span></p><p><strong>首先，Kubernetes会匹配API对象的组。</strong></p><p>需要明确的是，对于Kubernetes里的核心API对象，比如：Pod、Node等，是不需要Group的（即：它们的Group是“”）。所以，对于这些API对象来说，Kubernetes会直接在/api这个层级进行下一步的匹配过程。</p><p>而对于CronJob等非核心API对象来说，Kubernetes就必须在/apis这个层级里查找它对应的Group，进而根据“batch”这个Group的名字，找到/apis/batch。</p><p>不难发现，这些API Group的分类是以对象功能为依据的，比如Job和CronJob就都属于“batch” （离线业务）这个Group。</p><p><strong>然后，Kubernetes会进一步匹配到API对象的版本号。</strong></p><p>对于CronJob这个API对象来说，Kubernetes在batch这个Group下，匹配到的版本号就是v2alpha1。</p><p>在Kubernetes中，同一种API对象可以有多个版本，这正是Kubernetes进行API版本化管理的重要手段。这样，比如在CronJob的开发过程中，对于会影响到用户的变更就可以通过升级新版本来处理，从而保证了向后兼容。</p><p><strong>最后，Kubernetes会匹配API对象的资源类型。</strong></p><p>在前面匹配到正确的版本之后，Kubernetes就知道，我要创建的原来是一个/apis/batch/v2alpha1下的CronJob对象。</p><p>这时候，<span class="orange">APIServer就可以继续创建这个CronJob对象了</span>。为了方便理解，我为你总结了一个如下所示流程图来阐述这个创建过程：</p><p><img src="https://static001.geekbang.org/resource/image/df/6f/df6f1dda45e9a353a051d06c48f0286f.png?wh=1867*748" alt=""><br>
<strong>首先</strong>，当我们发起了创建CronJob的POST请求之后，我们编写的YAML的信息就被提交给了APIServer。</p><p>而APIServer的第一个功能，就是过滤这个请求，并完成一些前置性的工作，比如授权、超时处理、审计等。</p><p><strong>然后</strong>，请求会进入MUX和Routes流程。如果你编写过Web Server的话就会知道，MUX和Routes是APIServer完成URL和Handler绑定的场所。而APIServer的Handler要做的事情，就是按照我刚刚介绍的匹配过程，找到对应的CronJob类型定义。</p><p><strong>接着</strong>，APIServer最重要的职责就来了：根据这个CronJob类型定义，使用用户提交的YAML文件里的字段，创建一个CronJob对象。</p><p>而在这个过程中，APIServer会进行一个Convert工作，即：把用户提交的YAML文件，转换成一个叫作Super Version的对象，它正是该API资源类型所有版本的字段全集。这样用户提交的不同版本的YAML文件，就都可以用这个Super Version对象来进行处理了。</p><p><strong>接下来</strong>，APIServer会先后进行Admission()和Validation()操作。比如，我在上一篇文章中提到的Admission Controller和Initializer，就都属于Admission的内容。</p><p>而Validation，则负责验证这个对象里的各个字段是否合法。这个被验证过的API对象，都保存在了APIServer里一个叫作Registry的数据结构中。也就是说，只要一个API对象的定义能在Registry里查到，它就是一个有效的Kubernetes API对象。</p><p><strong>最后</strong>，APIServer会把验证过的API对象转换成用户最初提交的版本，进行序列化操作，并调用Etcd的API把它保存起来。</p><p>由此可见，声明式API对于Kubernetes来说非常重要。所以，<strong>APIServer这样一个在其他项目里“平淡无奇”的组件，却成了Kubernetes项目的重中之重</strong>。它不仅是Google Borg设计思想的集中体现，也是Kubernetes项目里唯一一个被Google公司和RedHat公司双重控制、其他势力根本无法参与其中的组件。</p><p>此外，由于同时要兼顾性能、API完备性、版本化、向后兼容等很多工程化指标，所以Kubernetes团队在APIServer项目里大量使用了Go语言的代码生成功能，来自动化诸如Convert、DeepCopy等与API资源相关的操作。这部分自动生成的代码，曾一度占到Kubernetes项目总代码的20%~30%。</p><p>这也是为何，在过去很长一段时间里，在这样一个极其“复杂”的APIServer中，添加一个Kubernetes风格的API资源类型，是一个非常困难的工作。</p><p>不过，在Kubernetes v1.7 之后，这个工作就变得轻松得多了。这，当然得益于一个全新的API插件机制：CRD。</p><p>CRD的全称是Custom Resource Definition。顾名思义，它指的就是，允许用户在Kubernetes中添加一个跟Pod、Node类似的、新的API资源类型，即：自定义API资源。</p><p>举个例子，<span class="orange">我现在要为Kubernetes添加一个名叫Network的API资源类型</span>。</p><p>它的作用是，一旦用户创建一个Network对象，那么Kubernetes就应该使用这个对象定义的网络参数，调用真实的网络插件，比如Neutron项目，为用户创建一个真正的“网络”。这样，将来用户创建的Pod，就可以声明使用这个“网络”了。</p><p>这个Network对象的YAML文件，名叫example-network.yaml，它的内容如下所示：</p><pre><code>apiVersion: samplecrd.k8s.io/v1
kind: Network
metadata:
  name: example-network
spec:
  cidr: &quot;192.168.0.0/16&quot;
  gateway: &quot;192.168.0.1&quot;
</code></pre><p>可以看到，我想要描述“网络”的API资源类型是Network；API组是<code>samplecrd.k8s.io</code>；API 版本是v1。</p><p>那么，<span class="orange">Kubernetes又该如何知道这个API（<code>samplecrd.k8s.io/v1/network</code>）的存在呢？</span></p><p>其实，上面的这个YAML文件，就是一个具体的“自定义API资源”实例，也叫CR（Custom Resource）。而为了能够让Kubernetes认识这个CR，你就需要让Kubernetes明白这个CR的宏观定义是什么，也就是CRD（Custom Resource Definition）。</p><p>这就好比，你想让计算机认识各种兔子的照片，就得先让计算机明白，兔子的普遍定义是什么。比如，兔子“是哺乳动物”“有长耳朵，三瓣嘴”。</p><p>所以，接下来，我就先编写一个CRD的YAML文件，它的名字叫作network.yaml，内容如下所示：</p><pre><code>apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: networks.samplecrd.k8s.io
spec:
  group: samplecrd.k8s.io
  version: v1
  names:
    kind: Network
    plural: networks
  scope: Namespaced
</code></pre><p>可以看到，在这个CRD中，我指定了“<code>group: samplecrd.k8s.io</code>”“<code>version: v1</code>”这样的API信息，也指定了这个CR的资源类型叫作Network，复数（plural）是networks。</p><p>然后，我还声明了它的scope是Namespaced，即：我们定义的这个Network是一个属于Namespace的对象，类似于Pod。</p><p>这就是一个Network API资源类型的API部分的宏观定义了。这就等同于告诉了计算机：“兔子是哺乳动物”。所以这时候，Kubernetes就能够认识和处理所有声明了API类型是“<code>samplecrd.k8s.io/v1/network</code>”的YAML文件了。</p><p>接下来，我还需要让Kubernetes“认识”这种YAML文件里描述的“网络”部分，比如“cidr”（网段），“gateway”（网关）这些字段的含义。这就相当于我要告诉计算机：“兔子有长耳朵和三瓣嘴”。</p><p>这时候呢，我就需要稍微做些代码工作了。</p><p><strong>首先，我要在GOPATH下，创建一个结构如下的项目：</strong></p><blockquote>
<p>备注：在这里，我并不要求你具有完备的Go语言知识体系，但我会假设你已经了解了Golang的一些基本知识（比如，知道什么是GOPATH）。而如果你还不了解的话，可以在涉及到相关内容时，再去查阅一些相关资料。</p>
</blockquote><pre><code>$ tree $GOPATH/src/github.com/&lt;your-name&gt;/k8s-controller-custom-resource
.
├── controller.go
├── crd
│   └── network.yaml
├── example
│   └── example-network.yaml
├── main.go
└── pkg
    └── apis
        └── samplecrd
            ├── register.go
            └── v1
                ├── doc.go
                ├── register.go
                └── types.go
</code></pre><p>其中，pkg/apis/samplecrd就是API组的名字，v1是版本，而v1下面的types.go文件里，则定义了Network对象的完整描述。我已经把这个项目<a href="https://github.com/resouer/k8s-controller-custom-resource">上传到了GitHub上</a>，你可以随时参考。</p><p><strong>然后，我在pkg/apis/samplecrd目录下创建了一个register.go文件，用来放置后面要用到的全局变量</strong>。这个文件的内容如下所示：</p><pre><code>package samplecrd

const (
 GroupName = &quot;samplecrd.k8s.io&quot;
 Version   = &quot;v1&quot;
)
</code></pre><p><strong>接着，我需要在pkg/apis/samplecrd目录下添加一个doc.go文件（Golang的文档源文件）</strong>。这个文件里的内容如下所示：</p><pre><code>// +k8s:deepcopy-gen=package

// +groupName=samplecrd.k8s.io
package v1
</code></pre><p>在这个文件中，你会看到+&lt;tag_name&gt;[=value]格式的注释，这就是Kubernetes进行代码生成要用的Annotation风格的注释。</p><p>其中，+k8s:deepcopy-gen=package意思是，请为整个v1包里的所有类型定义自动生成DeepCopy方法；而<code>+groupName=samplecrd.k8s.io</code>，则定义了这个包对应的API组的名字。</p><p>可以看到，这些定义在doc.go文件的注释，起到的是全局的代码生成控制的作用，所以也被称为Global Tags。</p><p><strong>接下来，我需要添加types.go文件</strong>。顾名思义，它的作用就是定义一个Network类型到底有哪些字段（比如，spec字段里的内容）。这个文件的主要内容如下所示：</p><pre><code>package v1
...
// +genclient
// +genclient:noStatus
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// Network describes a Network resource
type Network struct {
 // TypeMeta is the metadata for the resource, like kind and apiversion
 metav1.TypeMeta `json:&quot;,inline&quot;`
 // ObjectMeta contains the metadata for the particular object, including
 // things like...
 //  - name
 //  - namespace
 //  - self link
 //  - labels
 //  - ... etc ...
 metav1.ObjectMeta `json:&quot;metadata,omitempty&quot;`
 
 Spec networkspec `json:&quot;spec&quot;`
}
// networkspec is the spec for a Network resource
type networkspec struct {
 Cidr    string `json:&quot;cidr&quot;`
 Gateway string `json:&quot;gateway&quot;`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// NetworkList is a list of Network resources
type NetworkList struct {
 metav1.TypeMeta `json:&quot;,inline&quot;`
 metav1.ListMeta `json:&quot;metadata&quot;`
 
 Items []Network `json:&quot;items&quot;`
}
</code></pre><p>在上面这部分代码里，你可以看到Network类型定义方法跟标准的Kubernetes对象一样，都包括了TypeMeta（API元数据）和ObjectMeta（对象元数据）字段。</p><p>而其中的Spec字段，就是需要我们自己定义的部分。所以，在networkspec里，我定义了Cidr和Gateway两个字段。其中，每个字段最后面的部分比如<code>json:"cidr"</code>，指的就是这个字段被转换成JSON格式之后的名字，也就是YAML文件里的字段名字。</p><blockquote>
<p>如果你不熟悉这个用法的话，可以查阅一下Golang的文档。</p>
</blockquote><p>此外，除了定义Network类型，你还需要定义一个NetworkList类型，用来描述<strong>一组Network对象</strong>应该包括哪些字段。之所以需要这样一个类型，是因为在Kubernetes中，获取所有X对象的List()方法，返回值都是<x>List类型，而不是X类型的数组。这是不一样的。</x></p><p>同样地，在Network和NetworkList类型上，也有代码生成注释。</p><p>其中，+genclient的意思是：请为下面这个API资源类型生成对应的Client代码（这个Client，我马上会讲到）。而+genclient:noStatus的意思是：这个API资源类型定义里，没有Status字段。否则，生成的Client就会自动带上UpdateStatus方法。</p><p>如果你的类型定义包括了Status字段的话，就不需要这句+genclient:noStatus注释了。比如下面这个例子：</p><pre><code>// +genclient

// Network is a specification for a Network resource
type Network struct {
 metav1.TypeMeta   `json:&quot;,inline&quot;`
 metav1.ObjectMeta `json:&quot;metadata,omitempty&quot;`
 
 Spec   NetworkSpec   `json:&quot;spec&quot;`
 Status NetworkStatus `json:&quot;status&quot;`
}
</code></pre><p>需要注意的是，+genclient只需要写在Network类型上，而不用写在NetworkList上。因为NetworkList只是一个返回值类型，Network才是“主类型”。</p><p>而由于我在Global Tags里已经定义了为所有类型生成DeepCopy方法，所以这里就不需要再显式地加上+k8s:deepcopy-gen=true了。当然，这也就意味着你可以用+k8s:deepcopy-gen=false来阻止为某些类型生成DeepCopy。</p><p>你可能已经注意到，在这两个类型上面还有一句<code>+k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object</code>的注释。它的意思是，请在生成DeepCopy的时候，实现Kubernetes提供的runtime.Object接口。否则，在某些版本的Kubernetes里，你的这个类型定义会出现编译错误。这是一个固定的操作，记住即可。</p><p>不过，你或许会有这样的顾虑：这些代码生成注释这么灵活，我该怎么掌握呢？</p><p>其实，上面我所讲述的内容，已经足以应对99%的场景了。当然，如果你对代码生成感兴趣的话，我推荐你阅读<a href="https://blog.openshift.com/kubernetes-deep-dive-code-generation-customresources/">这篇博客</a>，它详细地介绍了Kubernetes的代码生成语法。</p><p><strong>最后，我需要再编写一个pkg/apis/samplecrd/v1/register.go文件</strong>。</p><p>在前面对APIServer工作原理的讲解中，我已经提到，“registry”的作用就是注册一个类型（Type）给APIServer。其中，Network资源类型在服务器端注册的工作，APIServer会自动帮我们完成。但与之对应的，我们还需要让客户端也能“知道”Network资源类型的定义。这就需要我们在项目里添加一个register.go文件。它最主要的功能，就是定义了如下所示的addKnownTypes()方法：</p><pre><code>package v1
...
// addKnownTypes adds our types to the API scheme by registering
// Network and NetworkList
func addKnownTypes(scheme *runtime.Scheme) error {
 scheme.AddKnownTypes(
  SchemeGroupVersion,
  &amp;Network{},
  &amp;NetworkList{},
 )
 
 // register the type in the scheme
 metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
 return nil
}
</code></pre><p>有了这个方法，Kubernetes就能够在后面生成客户端的时候，“知道”Network以及NetworkList类型的定义了。</p><p>像上面这种<strong>register.go文件里的内容其实是非常固定的，你以后可以直接使用我提供的这部分代码做模板，然后把其中的资源类型、GroupName和Version替换成你自己的定义即可。</strong></p><p>这样，Network对象的定义工作就全部完成了。可以看到，它其实定义了两部分内容：</p><ul>
<li>第一部分是，自定义资源类型的API描述，包括：组（Group）、版本（Version）、资源类型（Resource）等。这相当于告诉了计算机：兔子是哺乳动物。</li>
<li>第二部分是，自定义资源类型的对象描述，包括：Spec、Status等。这相当于告诉了计算机：兔子有长耳朵和三瓣嘴。</li>
</ul><p>接下来，<span class="orange">我就要使用Kubernetes提供的代码生成工具，为上面定义的Network资源类型自动生成clientset、informer和lister。</span>其中，clientset就是操作Network对象所需要使用的客户端，而informer和lister这两个包的主要功能，我会在下一篇文章中重点讲解。</p><p>这个代码生成工具名叫<code>k8s.io/code-generator</code>，使用方法如下所示：</p><pre><code># 代码生成的工作目录，也就是我们的项目路径
$ ROOT_PACKAGE=&quot;github.com/resouer/k8s-controller-custom-resource&quot;
# API Group
$ CUSTOM_RESOURCE_NAME=&quot;samplecrd&quot;
# API Version
$ CUSTOM_RESOURCE_VERSION=&quot;v1&quot;

# 安装k8s.io/code-generator
$ go get -u k8s.io/code-generator/...
$ cd $GOPATH/src/k8s.io/code-generator

# 执行代码自动生成，其中pkg/client是生成目标目录，pkg/apis是类型定义目录
$ ./generate-groups.sh all &quot;$ROOT_PACKAGE/pkg/client&quot; &quot;$ROOT_PACKAGE/pkg/apis&quot; &quot;$CUSTOM_RESOURCE_NAME:$CUSTOM_RESOURCE_VERSION&quot;
</code></pre><p>代码生成工作完成之后，我们再查看一下这个项目的目录结构：</p><pre><code>$ tree
.
├── controller.go
├── crd
│   └── network.yaml
├── example
│   └── example-network.yaml
├── main.go
└── pkg
    ├── apis
    │   └── samplecrd
    │       ├── constants.go
    │       └── v1
    │           ├── doc.go
    │           ├── register.go
    │           ├── types.go
    │           └── zz_generated.deepcopy.go
    └── client
        ├── clientset
        ├── informers
        └── listers
</code></pre><p>其中，pkg/apis/samplecrd/v1下面的zz_generated.deepcopy.go文件，就是自动生成的DeepCopy代码文件。</p><p>而整个client目录，以及下面的三个包（clientset、informers、 listers），都是Kubernetes为Network类型生成的客户端库，这些库会在后面编写自定义控制器的时候用到。</p><p>可以看到，到目前为止的这些工作，其实并不要求你写多少代码，主要考验的是“复制、粘贴、替换”这样的“基本功”。</p><p>而有了这些内容，现在你就可以<span class="orange">在Kubernetes集群里创建一个Network类型的API对象</span>了。我们不妨一起来试验下。</p><p><strong>首先</strong>，使用network.yaml文件，在Kubernetes中创建Network对象的CRD（Custom Resource Definition）：</p><pre><code>$ kubectl apply -f crd/network.yaml
customresourcedefinition.apiextensions.k8s.io/networks.samplecrd.k8s.io created
</code></pre><p>这个操作，就告诉了Kubernetes，我现在要添加一个自定义的API对象。而这个对象的API信息，正是network.yaml里定义的内容。我们可以通过kubectl get命令，查看这个CRD：</p><pre><code>$ kubectl get crd
NAME                        CREATED AT
networks.samplecrd.k8s.io   2018-09-15T10:57:12Z
</code></pre><p><strong>然后</strong>，我们就可以创建一个Network对象了，这里用到的是example-network.yaml：</p><pre><code>$ kubectl apply -f example/example-network.yaml 
network.samplecrd.k8s.io/example-network created
</code></pre><p>通过这个操作，你就在Kubernetes集群里创建了一个Network对象。它的API资源路径是<code>samplecrd.k8s.io/v1/networks</code>。</p><p>这时候，你就可以通过kubectl get命令，查看到新创建的Network对象：</p><pre><code>$ kubectl get network
NAME              AGE
example-network   8s
</code></pre><p>你还可以通过kubectl describe命令，看到这个Network对象的细节：</p><pre><code>$ kubectl describe network example-network
Name:         example-network
Namespace:    default
Labels:       &lt;none&gt;
...API Version:  samplecrd.k8s.io/v1
Kind:         Network
Metadata:
  ...
  Generation:          1
  Resource Version:    468239
  ...
Spec:
  Cidr:     192.168.0.0/16
  Gateway:  192.168.0.1
</code></pre><p>当然 ，你也可以编写更多的YAML文件来创建更多的Network对象，这和创建Pod、Deployment的操作，没有任何区别。</p><h2>总结</h2><p>在今天这篇文章中，我为你详细解析了Kubernetes声明式API的工作原理，讲解了如何遵循声明式API的设计，为Kubernetes添加一个名叫Network的API资源类型。从而达到了通过标准的kubectl create和get操作，来管理自定义API对象的目的。</p><p>不过，创建出这样一个自定义API对象，我们只是完成了Kubernetes声明式API的一半工作。</p><p>接下来的另一半工作是：为这个API对象编写一个自定义控制器（Custom Controller）。这样， Kubernetes才能根据Network API对象的“增、删、改”操作，在真实环境中做出相应的响应。比如，“创建、删除、修改”真正的Neutron网络。</p><p>而这，正是Network这个API对象所关注的“业务逻辑”。</p><p>这个业务逻辑的实现过程，以及它所使用的Kubernetes API编程库的工作原理，就是我要在下一篇文章中讲解的主要内容。</p><h2>思考题</h2><p>在了解了CRD的定义方法之后，你是否已经在考虑使用CRD（或者已经使用了CRD）来描述现实中的某种实体了呢？能否分享一下你的思路？（举个例子：某技术团队使用CRD描述了“宿主机”，然后用Kubernetes部署了Kubernetes）</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/96/50/bde525b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>北卡</span>
  </div>
  <div class="_2_QraFYR_0">运维人员会心碎?<br>我是运维人员，此刻看完我感到很兴奋。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没错。我觉得这部分才是运维最喜欢的。谁不喜欢写代码呢？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-18 01:46:25</div>
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
  <div class="_2_QraFYR_0">一般来说，扩展api server (或者说添加自定义 resource )有两种方式：<br>1. 通过创建CRDs, 主API server可以处理 CRDs 的 REST 请求（CRUD）和持久性存储。简单，不需要其他的编程。更适用于声明式的API，和kubernetes高度集成统一。<br>2. API Aggregation, 一个独立的API server。主API server委托此独立的API server处理自定义resource。 需要编程，但能够更加灵活的控制API的行为，更加灵活的自定义存储，以及与API不同版本之间的转换。一般更适用于命令模式，或者复用已经存在REST API代码，不直接支持kubectl 和 k8s UI, 不支持scope resource in a cluster&#47;namespace.<br><br>自定义 resource 可以使你方便的存取结构化的resource 数据。但是只有和controller组合在一起才是声明式API。声明式API允许你定义一个期望的状态。controller通过解读结构化的resource数据，获得期望状态，从而不断的调协期望状态和实际状态。<br><br>综上，今天文档中的types.go 应该是给controller来理解CRDs的schema用的。只有掌握了resource的schema，才能解释并得到用户创建的resource API object。<br>而 kubectl create -f resourcedefinition.yaml 或者 自定义API server， 则定义了RESTful API endpoint. 用于接受 REST 请求，改变 resource 的期望状态。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-17 17:16:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8e/c9/9f783069.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小金刚</span>
  </div>
  <div class="_2_QraFYR_0">可以用 kubebuild 自动生成项目框架，添加自己的 CRD 并实现 controller 即可。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-28 15:49:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ca/d5/a2cd57c9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小明root</span>
  </div>
  <div class="_2_QraFYR_0">我是运维人员，我很心碎.....我不会go语言</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-22 14:50:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/27/4b/170654bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>千寻</span>
  </div>
  <div class="_2_QraFYR_0">我从代码开始，就按着步骤走，最后创建network的CRD和example-network都成功了，<br>但是我直接将cdr&#47;network.yml和example&#47;example-network.yml文件单独拿出来，并没有执行代码生成那些步骤，发现也创建成功了，搞得有点懵。<br>老师可以说一下这大概是什么回事吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为生成的代码是给下一篇写控制器代码用的，哈哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-17 14:06:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">我是开发人员，会go,也会C++，也自己设计过业务协议，看到这篇异常兴奋</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-26 16:19:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/44/b8/d5fe40fb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vndi</span>
  </div>
  <div class="_2_QraFYR_0">CRD就像你去数据库定义一张表，CR就像表里的记录</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-02 15:30:17</div>
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
  <div class="_2_QraFYR_0">好好好。之前学习过这些内容，但是很不系统，只是大概知道怎么回事，很多细节没有穿起来。跟老师比起来，我太不善于总结了。<br>磊哥大神，等我把你的课都学会了，能跟着你干吗？<br><br>另外我看到有人问需要把代码重新编译进k8s中么？不需要。作为插件应该是可以热插拔的。比如service catalog，flannel，都不要求你重启k8s。<br>在k8s里定义了你的CR后，controller可以作为一个container跑在k8s集群里，来响应CR的增删改。<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-17 10:52:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b6/11/e8506a04.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小宇宙</span>
  </div>
  <div class="_2_QraFYR_0">用CRD来开发定义自主化的operator ，将有状态的应用自动化</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-17 20:11:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/05/92/b609f7e3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>骨汤鸡蛋面</span>
  </div>
  <div class="_2_QraFYR_0">自定义resource的Controller 单独运行，只是通过client-go 与api 交互？ 是否可以认为，k8s内建的Resource 对应的Controller，由Controller-manager 统一管理呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，内置和外挂的区别</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-05 21:06:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/nhLS152kEs5J65bBpM2fzMn4agfoow8xibFzNSDcmo9Oiby2lNB4hWRcetRWFyY2y05IJu8GbkZer9BUiahtadU0w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yuanlinios</span>
  </div>
  <div class="_2_QraFYR_0">.&#47;generate-groups.sh all &quot;$ROOT_PACKAGE&#47;pkg&#47;client&quot; &quot;$ROOT_PACKAGE&#47;pkg&#47;apis&quot; &quot;$CUSTOM_RESOURCE_NAME:$CUSTOM_RESOURCE_VERSION&quot;<br>Generating deepcopy funcs<br>F1228 01:11:20.543446    2908 deepcopy.go:866] Hit an unsupported type invalid type.<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-28 14:21:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/46/5b/07858c33.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Pixar</span>
  </div>
  <div class="_2_QraFYR_0">有一个问题张老师，一直不是特别清楚… 通过 crd 创建的自定义资源我还并没有定义他的结构，为什么就可以通过 kubectl get 拿到这个资源的详情呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你提交了一个CR yaml了不是？反正是存在etcd里，给你返回即可。不过kubernetes 并不懂字段的意思。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-23 09:15:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/96/89/9312b3a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vincen</span>
  </div>
  <div class="_2_QraFYR_0">从kubernetes用户到kubernetes玩家成长中...</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-17 09:04:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/81/8dd164fb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>丶海口</span>
  </div>
  <div class="_2_QraFYR_0">使用最新版本 cd 到 code-generator 目录去执行会报错，改成这样就好了：<br># 代码生成的工作目录，也就是我们的项目路径<br>ROOT_PACKAGE=&quot;github.com&#47;resouer&#47;k8s-controller-custom-resource&quot;<br># API Group<br>CUSTOM_RESOURCE_NAME=&quot;samplecrd&quot;<br># API Version<br>CUSTOM_RESOURCE_VERSION=&quot;v1&quot;<br># Code-Generator Directory<br>EXEC_DIR=$GOPATH&#47;src&#47;k8s.io&#47;code-generator<br><br># 安装 k8s.io&#47;code-generator<br>go get -u k8s.io&#47;code-generator&#47;...<br><br># 执行代码自动生成，其中pkg&#47;client是生成目标目录，pkg&#47;apis是类型定义目录<br>&quot;${EXEC_DIR}&quot;&#47;generate-groups.sh all &quot;$ROOT_PACKAGE&#47;pkg&#47;client&quot; &quot;$ROOT_PACKAGE&#47;pkg&#47;apis&quot; &quot;$CUSTOM_RESOURCE_NAME:$CUSTOM_RESOURCE_VERSION&quot; --go-header-file &quot;${EXEC_DIR}&quot;&#47;hack&#47;boilerplate.go.txt -v 10</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-30 12:35:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/93/38/71615300.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DayDayUp</span>
  </div>
  <div class="_2_QraFYR_0">看了好兴奋，真的感觉自己能够从一个旁观者慢慢变成一个开发贡献者</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-21 11:34:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b5/ca/c3949f49.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>疯狂的小企鹅</span>
  </div>
  <div class="_2_QraFYR_0">提示 Hit an unsupported type invalid type的同学，可以先安装下k8s.io&#47;apimachinery包。应该是deepcopy找不到metav1.ObjectMeta和metav1.TypeMeta<br>go get -u k8s.io&#47;apimachinery</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-09 20:14:34</div>
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
  <div class="_2_QraFYR_0">register.go会将自定义Type注册到APIServer，那register.go本身是怎么交给APIServer，然后被APIServer调用注册过程的？<br><br>$ kubectl apply -f crd&#47;network.yaml<br>$ kubectl apply -f example&#47;example-network.yaml <br><br>执行完这两步，自定义的Newwork对象被创建出来，怎么感觉register.go并没有被用到</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为你还没有调用生成的client，register.go是客户端用来进行初始化的代码的一部分。你这两步做完，服务端已经知道这个crd了，客户端也得有办法知道啊。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-18 10:42:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c4/4a/c02c597b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Joe</span>
  </div>
  <div class="_2_QraFYR_0">Generating deepcopy funcs<br>F0110 15:05:38.168635   11946 deepcopy.go:866] Hit an unsupported type invalid type.<br><br>按文档走了一半，报错~ 这个是什么问题呀？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-10 15:08:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/97/95/aad51e9b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>waterjiao</span>
  </div>
  <div class="_2_QraFYR_0">deepcopy的作用是什么？runtime.Object是什么呢？是Network这个资源在k8s客户端的形态呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-25 15:11:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b0/57/a84d633e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>圣诞使者</span>
  </div>
  <div class="_2_QraFYR_0">老师，我照着你的代码敲了一遍，这个pkg&#47;signals目录是自己创建的吗？我这个生成完代码也没有这个目录。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是个自己的帮助库</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-22 17:23:10</div>
  </div>
</div>
</div>
</li>
</ul>