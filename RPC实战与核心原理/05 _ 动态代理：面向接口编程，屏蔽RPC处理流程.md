<audio title="05 _ 动态代理：面向接口编程，屏蔽RPC处理流程" src="https://static001.geekbang.org/resource/audio/31/65/3116e2aac6cda68241c512ea34069c65.mp3" controls="controls"></audio> 
<p>你好，我是何小锋。上一讲我分享了网络通信，其实要理解起来也很简单，RPC 是用来解决两个应用之间的通信，而网络则是两台机器之间的“桥梁”，只有架好了桥梁，我们才能把请求数据从一端传输另外一端。其实关于网络通信，你只要记住一个关键字就行了——可靠的传输。</p><p>那么接着上一讲的内容，我们再来聊聊动态代理在 RPC 里面的应用。</p><p>如果我问你，你知道动态代理吗？ 你可能会如数家珍般地告诉我动态代理的作用以及好处。那我现在接着问你，你在项目中用过动态代理吗？这时候可能有些人就会犹豫了。那我再换一个方式问你，你在项目中有实现过统一拦截的功能吗？比如授权认证、性能统计等等。你可能立马就会想到，我实现过呀，并且我知道可以用 Spring 的 AOP 功能来实现。</p><p>没错，进一步再想，在 Spring AOP 里面我们是怎么实现统一拦截的效果呢？并且是在我们不需要改动原有代码的前提下，还能实现非业务逻辑跟业务逻辑的解耦。这里的核心就是采用动态代理技术，通过对字节码进行增强，在方法调用的时候进行拦截，以便于在方法调用前后，增加我们需要的额外处理逻辑。</p><p>那话说回来，动态代理跟 RPC 又有什么关系呢？</p><h2>远程调用的魔法</h2><p>我说个具体的场景，你可能就明白了。</p><!-- [[[read_end]]] --><p>在项目中，当我们要使用 RPC 的时候，我们一般的做法是先找服务提供方要接口，通过 Maven 或者其他的工具把接口依赖到我们项目中。我们在编写业务逻辑的时候，如果要调用提供方的接口，我们就只需要通过依赖注入的方式把接口注入到项目中就行了，然后在代码里面直接调用接口的方法 。</p><p>我们都知道，接口里并不会包含真实的业务逻辑，业务逻辑都在服务提供方应用里，但我们通过调用接口方法，确实拿到了想要的结果，是不是感觉有点神奇呢？想一下，在 RPC 里面，我们是怎么完成这个魔术的。</p><p><strong>这里面用到的核心技术就是前面说的动态代理。</strong>RPC 会自动给接口生成一个代理类，当我们在项目中注入接口的时候，运行过程中实际绑定的是这个接口生成的代理类。这样在接口方法被调用的时候，它实际上是被生成代理类拦截到了，这样我们就可以在生成的代理类里面，加入远程调用逻辑。</p><p>通过这种“偷梁换柱”的手法，就可以帮用户屏蔽远程调用的细节，实现像调用本地一样地调用远程的体验，整体流程如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/05/53/05cd18e7e33c5937c7c39bf8872c5753.jpg?wh=2508*1568" alt="" title="调用流程"></p><h2>实现原理</h2><p>动态代理在 RPC 里面的作用，就像是个魔术。现在我不妨给你揭秘一下，我们一起看看这是怎么实现的。之后，学以致用自然就不难了。</p><p>我们以 Java 为例，看一个具体例子，代码如下所示：</p><pre><code>/**
 * 要代理的接口
 */
public interface Hello {
    String say();
}

/**
 * 真实调用对象
 */
public class RealHello {

    public String invoke(){
        return &quot;i'm proxy&quot;;
    }
}

/**
 * JDK代理类生成
 */
public class JDKProxy implements InvocationHandler {
    private Object target;

    JDKProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] paramValues) {
        return ((RealHello)target).invoke();
    }
}

/**
 * 测试例子
 */
public class TestProxy {

    public static void main(String[] args){
        // 构建代理器
        JDKProxy proxy = new JDKProxy(new RealHello());
        ClassLoader classLoader = ClassLoaderUtils.getCurrentClassLoader();
        // 把生成的代理类保存到文件
System.setProperty(&quot;sun.misc.ProxyGenerator.saveGeneratedFiles&quot;,&quot;true&quot;);
        // 生成代理类
        Hello test = (Hello) Proxy.newProxyInstance(classLoader, new Class[]{Hello.class}, proxy);
        // 方法调用
        System.out.println(test.say());
    }
}
</code></pre><p>这段代码想表达的意思就是：给 Hello 接口生成一个动态代理类，并调用接口 say() 方法，但真实返回的值居然是来自 RealHello 里面的 invoke() 方法返回值。你看，短短50行的代码，就完成了这个功能，是不是还挺有意思的？</p><p>那既然重点是代理类的生成，那我们就去看下 Proxy.newProxyInstance 里面究竟发生了什么？</p><p>一起看下下面的流程图，具体代码细节你可以对照着 JDK 的源码看（上文中有类和方法，可以直接定位），我是按照 1.7.X 版本梳理的。</p><p><img src="https://static001.geekbang.org/resource/image/50/41/5042cf1b79e6b9233f2152e1e0aca741.jpg?wh=4442*926" alt="" title="代理类生成流程"></p><p>在生成字节码的那个地方，也就是 ProxyGenerator.generateProxyClass() 方法里面，通过代码我们可以看到，里面是用参数  saveGeneratedFiles  来控制是否把生成的字节码保存到本地磁盘。同时为了更直观地了解代理的本质，我们需要把参数 saveGeneratedFiles 设置成true，但这个参数的值是由key为“sun.misc.ProxyGenerator.saveGeneratedFiles”的Property来控制的，动态生成的类会保存在工程根目录下的 com/sun/proxy 目录里面。现在我们找到刚才生成的 $Proxy0.class，通过反编译工具打开class文件，你会看到这样的代码：</p><pre><code>package com.sun.proxy;

import com.proxy.Hello;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements Hello {
  private static Method m3;
  
  private static Method m1;
  
  private static Method m0;
  
  private static Method m2;
  
  public $Proxy0(InvocationHandler paramInvocationHandler) {
    super(paramInvocationHandler);
  }
  
  public final String say() {
    try {
      return (String)this.h.invoke(this, m3, null);
    } catch (Error|RuntimeException error) {
      throw null;
    } catch (Throwable throwable) {
      throw new UndeclaredThrowableException(throwable);
    } 
  }
  
  public final boolean equals(Object paramObject) {
    try {
      return ((Boolean)this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
    } catch (Error|RuntimeException error) {
      throw null;
    } catch (Throwable throwable) {
      throw new UndeclaredThrowableException(throwable);
    } 
  }
  
  public final int hashCode() {
    try {
      return ((Integer)this.h.invoke(this, m0, null)).intValue();
    } catch (Error|RuntimeException error) {
      throw null;
    } catch (Throwable throwable) {
      throw new UndeclaredThrowableException(throwable);
    } 
  }
  
  public final String toString() {
    try {
      return (String)this.h.invoke(this, m2, null);
    } catch (Error|RuntimeException error) {
      throw null;
    } catch (Throwable throwable) {
      throw new UndeclaredThrowableException(throwable);
    } 
  }
  
  static {
    try {
      m3 = Class.forName(&quot;com.proxy.Hello&quot;).getMethod(&quot;say&quot;, new Class[0]);
      m1 = Class.forName(&quot;java.lang.Object&quot;).getMethod(&quot;equals&quot;, new Class[] { Class.forName(&quot;java.lang.Object&quot;) });
      m0 = Class.forName(&quot;java.lang.Object&quot;).getMethod(&quot;hashCode&quot;, new Class[0]);
      m2 = Class.forName(&quot;java.lang.Object&quot;).getMethod(&quot;toString&quot;, new Class[0]);
      return;
    } catch (NoSuchMethodException noSuchMethodException) {
      throw new NoSuchMethodError(noSuchMethodException.getMessage());
    } catch (ClassNotFoundException classNotFoundException) {
      throw new NoClassDefFoundError(classNotFoundException.getMessage());
    } 
  }
}

</code></pre><p>我们可以看到 $Proxy0 类里面有一个跟 Hello 一样签名的 say() 方法，其中 this.h 绑定的是刚才传入的 JDKProxy 对象，所以当我们调用 Hello.say() 的时候，其实它是被转发到了JDKProxy.invoke()。到这儿，整个魔术过程就透明了。</p><h2>实现方法</h2><p>其实在 Java 领域，除了JDK 默认的nvocationHandler能完成代理功能，我们还有很多其他的第三方框架也可以，比如像 Javassist、Byte Buddy 这样的框架。</p><p>单纯从代理功能上来看，JDK 默认的代理功能是有一定的局限性的，它要求被代理的类只能是接口。原因是因为生成的代理类会继承 Proxy 类，但Java 是不支持多重继承的。</p><p>这个限制在RPC应用场景里面还是挺要紧的，因为对于服务调用方来说，在使用RPC的时候本来就是面向接口来编程的，这个我们刚才在前面已经讨论过了。使用JDK默认的代理功能，最大的问题就是性能问题。它生成后的代理类是使用反射来完成方法调用的，而这种方式相对直接用编码调用来说，性能会降低，但好在JDK8及以上版本对反射调用的性能有很大的提升，所以还是可以期待一下的。</p><p>相对 JDK 自带的代理功能，Javassist的定位是能够操纵底层字节码，所以使用起来并不简单，要生成动态代理类恐怕是有点复杂了。但好的方面是，通过Javassist生成字节码，不需要通过反射完成方法调用，所以性能肯定是更胜一筹的。在使用中，我们要注意一个问题，通过Javassist生成一个代理类后，此 CtClass 对象会被冻结起来，不允许再修改；否则，再次生成时会报错。</p><p>Byte Buddy 则属于后起之秀，在很多优秀的项目中，像Spring、Jackson都用到了Byte Buddy来完成底层代理。相比Javassist，Byte Buddy提供了更容易操作的API，编写的代码可读性更高。更重要的是，生成的代理类执行速度比Javassist更快。</p><p>虽然以上这三种框架使用的方式相差很大，但核心原理却是差不多的，区别就只是通过什么方式生成的代理类以及在生成的代理类里面是怎么完成的方法调用。同时呢，也正是因为这些细小的差异，才导致了不同的代理框架在性能方面的表现不同。因此，我们在设计RPC框架的时候，还是需要进行一些比较的，具体你可以综合它们的优劣以及你的场景需求进行选择。</p><h2>总结</h2><p>今天我们介绍了动态代理在RPC里面的应用，虽然它只是一种具体实现的技术，但我觉得只有理解了方法调用是怎么被拦截的，才能厘清在RPC里面我们是怎么做到面向接口编程，帮助用户屏蔽RPC调用细节的，最终呈现给用户一个像调用本地一样去调用远程的编程体验。</p><p>既然动态代理是一种具体的技术框架，那就会涉及到选型。我们可以从这样三个角度去考虑：</p><ul>
<li>因为代理类是在运行中生成的，那么代理框架生成代理类的速度、生成代理类的字节码大小等等，都会影响到其性能——生成的字节码越小，运行所占资源就越小。</li>
<li>还有就是我们生成的代理类，是用于接口方法请求拦截的，所以每次调用接口方法的时候，都会执行生成的代理类，这时生成的代理类的执行效率就需要很高效。</li>
<li>最后一个是从我们的使用角度出发的，我们肯定希望选择一个使用起来很方便的代理类框架，比如我们可以考虑：API设计是否好理解、社区活跃度、还有就是依赖复杂度等等。</li>
</ul><p>最后，我想再强调一下。动态代理在RPC里面，虽然看起来只是一个很小的技术点，但就是这个创新使得用户可以不用关注细节了。其实，我们在日常设计接口的时候也是一样的，我们会想尽一切办法把细节对调用方屏蔽，让调用方的接入尽可能的简单。这就好比，让你去设计一个商品发布的接口，你并不需要暴露给用户一些细节，比如，告诉他们商品数据是怎么存储的。</p><h2>课后思考</h2><p>请你设想一下，如果没有动态代理帮我们完成方法调用拦截，用户该怎么完成RPC调用？</p><p>欢迎留言和我分享你的答案，也欢迎你把文章分享给你的朋友，邀请他加入学习。我们下节课再见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/53/ae/26faa892.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邵邵</span>
  </div>
  <div class="_2_QraFYR_0">这一节RPC讲解重度依赖java知识，希望老师在后续的文章中多提炼一些跨语言层面的原理，或者用一些伪代码的方式讲解，谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-01 00:20:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/e7/76/79c1f23a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李玥</span>
  </div>
  <div class="_2_QraFYR_0">我在工作中有幸看了不少小锋老师的代码，京东很多的基础类库中，都有小峰老师的代码。从中学到了很多。包括文章中提到的对调用方尽量屏蔽实现细节的思想，以及通过定义扩展点来剥离实现等等。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-29 15:53:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83epY2JtiahsBz4LK1k00DbYoR2Zk56wnYaoBfP63v5X3Xu1kuH1ethAnMOMu6vT9eKQSqHqn3HrTK9Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_7d1d6d</span>
  </div>
  <div class="_2_QraFYR_0">感觉这节课的通用型不好，语言选型太侧重java，以后能不能使用伪代码讲流程呢？非常感谢老师</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-24 16:55:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0f/bf/ee93c4cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雨霖铃声声慢</span>
  </div>
  <div class="_2_QraFYR_0">如果没有动态代理帮我们完成方法调用拦截，那么就需要使用静态代理来实现，就需要用户对原始类中所有的方法都重新实现一遍，并且为每个方法附加相似的代码逻辑，如果在RPC中，这种需要代理的类有很多个，就需要针对每个类都创建一个代理类。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很棒</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-01 12:00:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/3a/df/ea0fc831.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>周天航</span>
  </div>
  <div class="_2_QraFYR_0">这个文章的名字应该叫《Java 中使用rpc和原理》</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-16 08:55:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7b/98/8f1aecf4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>楼下小黑哥</span>
  </div>
  <div class="_2_QraFYR_0">没有动态代理的情况下，最简单的情况，<br>服务消费者将类信息序列化---》按照协议拼接报文----》调用网络程序发送报文----》收到提供者返回信息----》根据协议解析出返回信息-----》再反序列化成返回结果。<br>上面每一步都挺复杂的，RPC 框架使用动态代理帮我们屏蔽这种细节。<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很棒</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-06 07:43:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKhuGLVRYZibOTfMumk53Wn8Q0Rkg0o6DzTicbibCq42lWQoZ8lFeQvicaXuZa7dYsr9URMrtpXMVDDww/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hello</span>
  </div>
  <div class="_2_QraFYR_0">Byte Buddy，老师这个后起之秀，是怎么完成动态代理的，能否剖析下，多谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 相比于其他框架，其API还是很简单的。举个例子：Class&lt;? extends T&gt; clazz = BYTE_BUDDY.subclass(clz)<br>                .method(ElementMatchers.isDeclaredBy(clz))<br>                .intercept(MethodDelegation.to(new ByteBuddyInvocationHandler(invoker)))<br>                .make()<br>                .load(classLoader, ClassLoadingStrategy.Default.INJECTION)<br>                .getLoaded();<br>        try {<br>            return clazz.newInstance();<br>        } catch (Exception e) {<br>            throw new ProxyException(&quot;Error occurred while creating bytebuddy proxy of &quot; + clz.getName(), e);<br>        }</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-28 08:23:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/03/5b/3cdbc9fa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>宁悦</span>
  </div>
  <div class="_2_QraFYR_0">补充一个动态代理实现RPC的简单例子，可以辅助理解。例子来源于隔壁王争老师的《设计模式之美》专栏。<br>https:&#47;&#47;github.com&#47;wangzheng0822&#47;codedesign&#47;tree&#47;master&#47;com&#47;xzg&#47;cd&#47;rpc</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-11 10:46:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/81/0e/ec667c01.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wallace Pang</span>
  </div>
  <div class="_2_QraFYR_0">其实在 Java 领域，除了 JDK 默认的 nvocationHandler 能完成代理功能，应该是InvocationHandler</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-06 07:05:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/03/5b/3cdbc9fa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>宁悦</span>
  </div>
  <div class="_2_QraFYR_0">我的jdk版本中找不到ClassLoaderUtils这个类，只能找到ClassLoader这个类，发现用下面的代码也能实现老师文中的例子。<br> ClassLoader classLoader = ClassLoader.getSystemClassLoader();</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-05 10:03:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/f4/9a1feb59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱</span>
  </div>
  <div class="_2_QraFYR_0">请你设想一下，如果没有动态代理帮我们完成方法调用拦截，用户该怎么完成 RPC 调用？<br>1：使用静态代理实现<br>2：远程调度的代码和业务处理逻辑写在一块，序列化、编码、网络建连、网络传输、网络断连、以及负载均衡、失败重试等等都需要写出来，这块大家都重复造轮子，上线一个RPC调用功能现在是一天之后可能是一个月<br><br>代理模式我觉得很神奇，尤其是动态代理，真像魔法一样，许多代码你看不到不是因为他不存在而是她隐藏了起来，在代码运行的时候才出现，但是计算机不可能平白无故的运行的，你不告诉它怎么运行它是不知道怎么走的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-13 08:24:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cb/44/22d65cf1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>winy</span>
  </div>
  <div class="_2_QraFYR_0">1.静态代理的效率是远高于动态代理的<br>2.类似于lombok我们可以用注解处理器现实静态代理<br>3.因为是编译时生成的字节码，调试的时候方法间调用逻辑更简单清晰<br><br>所以我一直不明白为什么大家都用动态代理，很多框架组合起来用一层层代理下去，在调试的时候真的令人头大2333</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-25 20:21:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/cb/c7541d52.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cwfighter</span>
  </div>
  <div class="_2_QraFYR_0">不太明白为啥要用运行时动态生成这么损耗性能的方式，在c&#47;c++里面都是autogen自动生成框架代码以及对应的stub调用封装，业务也不需要关心rpc调用，这种静态的方式不是更好？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-23 09:19:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f2/62/f873cd8f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tongmin_tsai</span>
  </div>
  <div class="_2_QraFYR_0">老师，可以理解为，动态代理生成的类，就是stub吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 作用是一样的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-26 10:12:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/da/36/ac0ff6a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wusiration</span>
  </div>
  <div class="_2_QraFYR_0">如果没有动态代理完成方法拦截，那么被调用方需要有调用方的接口实现，就失去了面向接口编程的意义</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很棒</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-28 22:51:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/9a/76c0af70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每天晒白牙</span>
  </div>
  <div class="_2_QraFYR_0">如果没有动态代理类帮我们完成方法调用拦截，需要用户自己加入远程调用的逻辑，这样就麻烦，且使用不方便了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，动态代理让用户API调用透明</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-28 07:26:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIQMh5jMlvAibnTgiaVXmsb333JBjpdvLVcptc232zQey5wkBOEPiauepQpv8UlRcOFqnTyhEQiadaHMA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>winfield</span>
  </div>
  <div class="_2_QraFYR_0">我呆的两家大厂，目前都是用静态代理，每个 rpc 的客户端都要生成代码</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-28 15:08:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/ca/82/85f6a1a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>番茄炒西红柿</span>
  </div>
  <div class="_2_QraFYR_0">为什么不用静态代理？aspectJ之类的。提前把代理类在编译的时候创建？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理论上编译期创建是可以的，但实际问题就是怎么确定哪些需要创建呢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-08 23:44:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c2/fe/038a076e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿卧</span>
  </div>
  <div class="_2_QraFYR_0">从动态代理的原理和特点出发。动态代理帮我们生成代理类，屏蔽具体实现细节，在代理方法前实现统一封装。<br>如果没有动态代理，就需要手动实现每个接口类的远程调用细节了。<br>被面试官问到这个问题，没有回答上来...</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 现在可以了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-16 07:37:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/82/3d/356fc3d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忆水寒</span>
  </div>
  <div class="_2_QraFYR_0">其实也是看需求的。如果没有动态代理，那么调用双方可以通过定义一套消息id和消息结构（才有protobuf定义），也是可以完成远程调用的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-01 22:01:05</div>
  </div>
</div>
</div>
</li>
</ul>