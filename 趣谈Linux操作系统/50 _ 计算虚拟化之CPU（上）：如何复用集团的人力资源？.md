<audio title="50 _ 计算虚拟化之CPU（上）：如何复用集团的人力资源？" src="https://static001.geekbang.org/resource/audio/86/8d/864b26555d79e4635efeac6b4580bb8d.mp3" controls="controls"></audio> 
<p>上一节，我们讲了一下虚拟化的基本原理，以及qemu、kvm之间的关系。这一节，我们就来看一下，用户态的qemu和内核态的kvm如何一起协作，来创建虚拟机，实现CPU和内存虚拟化。</p><p>这里是上一节我们讲的qemu启动时候的命令。</p><pre><code>qemu-system-x86_64 -enable-kvm -name ubuntutest  -m 2048 -hda ubuntutest.qcow2 -vnc :19 -net nic,model=virtio -nettap,ifname=tap0,script=no,downscript=no
</code></pre><p>接下来，我们在<a href="https://www.qemu.org/">这里下载</a>qemu的代码。qemu的main函数在vl.c下面。这是一个非常非常长的函数，我们来慢慢地解析它。</p><h2>1.初始化所有的Module</h2><p>第一步，初始化所有的Module，调用下面的函数。</p><pre><code>module_call_init(MODULE_INIT_QOM);
</code></pre><p>上一节我们讲过，qemu作为中间人其实挺累的，对上面的虚拟机需要模拟各种各样的外部设备。当虚拟机真的要使用物理资源的时候，对下面的物理机上的资源要进行请求，所以它的工作模式有点儿类似操作系统对接驱动。驱动要符合一定的格式，才能算操作系统的一个模块。同理，qemu为了模拟各种各样的设备，也需要管理各种各样的模块，这些模块也需要符合一定的格式。</p><p>定义一个qemu模块会调用type_init。例如，kvm的模块要在accel/kvm/kvm-all.c文件里面实现。在这个文件里面，有一行下面的代码：</p><pre><code>type_init(kvm_type_init);

#define type_init(function) module_init(function, MODULE_INIT_QOM)

#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_module_init(function, type);                                   \
}

void register_module_init(void (*fn)(void), module_init_type type)
{
    ModuleEntry *e;
    ModuleTypeList *l;

    e = g_malloc0(sizeof(*e));
    e-&gt;init = fn;
    e-&gt;type = type;

    l = find_type(type);

    QTAILQ_INSERT_TAIL(l, e, node);
}
</code></pre><p>从代码里面的定义我们可以看出来，type_init后面的参数是一个函数，调用type_init就相当于调用module_init，在这里函数就是kvm_type_init，类型就是MODULE_INIT_QOM。是不是感觉和驱动有点儿像？</p><!-- [[[read_end]]] --><p>module_init最终要调用register_module_init。属于MODULE_INIT_QOM这种类型的，有一个Module列表ModuleTypeList，列表里面是一项一项的ModuleEntry。KVM就是其中一项，并且会初始化每一项的init函数为参数表示的函数fn，也即KVM这个module的init函数就是kvm_type_init。</p><p>当然，MODULE_INIT_QOM这种类型会有很多很多的module，从后面的代码我们可以看到，所有调用type_init的地方都注册了一个MODULE_INIT_QOM类型的Module。</p><p>了解了Module的注册机制，我们继续回到main函数中module_call_init的调用。</p><pre><code>void module_call_init(module_init_type type)
{
    ModuleTypeList *l;
    ModuleEntry *e;
    l = find_type(type);
    QTAILQ_FOREACH(e, l, node) {
        e-&gt;init();
    }
}
</code></pre><p>在module_call_init中，我们会找到MODULE_INIT_QOM这种类型对应的ModuleTypeList，找出列表中所有的ModuleEntry，然后调用每个ModuleEntry的init函数。这里需要注意的是，在module_call_init调用的这一步，所有Module的init函数都已经被调用过了。</p><p>后面我们会看到很多的Module，当你看到它们的时候，你需要意识到，它的init函数在这里也被调用过了。这里我们还是以对于kvm这个module为例子，看看它的init函数都做了哪些事情。你会发现，其实它调用的是kvm_type_init。</p><pre><code>static void kvm_type_init(void)
{
    type_register_static(&amp;kvm_accel_type);
}

TypeImpl *type_register_static(const TypeInfo *info)
{
    return type_register(info);
}

TypeImpl *type_register(const TypeInfo *info)
{
    assert(info-&gt;parent);
    return type_register_internal(info);
}

static TypeImpl *type_register_internal(const TypeInfo *info)
{
    TypeImpl *ti;
    ti = type_new(info);

    type_table_add(ti);
    return ti;
}

static TypeImpl *type_new(const TypeInfo *info)
{
    TypeImpl *ti = g_malloc0(sizeof(*ti));
    int i;

    if (type_table_lookup(info-&gt;name) != NULL) {
    }

    ti-&gt;name = g_strdup(info-&gt;name);
    ti-&gt;parent = g_strdup(info-&gt;parent);

    ti-&gt;class_size = info-&gt;class_size;
    ti-&gt;instance_size = info-&gt;instance_size;

    ti-&gt;class_init = info-&gt;class_init;
    ti-&gt;class_base_init = info-&gt;class_base_init;
    ti-&gt;class_data = info-&gt;class_data;

    ti-&gt;instance_init = info-&gt;instance_init;
    ti-&gt;instance_post_init = info-&gt;instance_post_init;
    ti-&gt;instance_finalize = info-&gt;instance_finalize;

    ti-&gt;abstract = info-&gt;abstract;

    for (i = 0; info-&gt;interfaces &amp;&amp; info-&gt;interfaces[i].type; i++) {
        ti-&gt;interfaces[i].typename = g_strdup(info-&gt;interfaces[i].type);
    }
    ti-&gt;num_interfaces = i;

    return ti;
}

static void type_table_add(TypeImpl *ti)
{
    assert(!enumerating_types);
    g_hash_table_insert(type_table_get(), (void *)ti-&gt;name, ti);
}

static GHashTable *type_table_get(void)
{
    static GHashTable *type_table;

    if (type_table == NULL) {
        type_table = g_hash_table_new(g_str_hash, g_str_equal);
    }

    return type_table;
}

static const TypeInfo kvm_accel_type = {
    .name = TYPE_KVM_ACCEL,
    .parent = TYPE_ACCEL,
    .class_init = kvm_accel_class_init,
    .instance_size = sizeof(KVMState),
};
</code></pre><p>每一个Module既然要模拟某种设备，那应该定义一种类型TypeImpl来表示这些设备，这其实是一种面向对象编程的思路，只不过这里用的是纯C语言的实现，所以需要变相实现一下类和对象。</p><p>kvm_type_init会注册kvm_accel_type，定义上面的代码，我们可以认为这样动态定义了一个类。这个类的名字是TYPE_KVM_ACCEL，这个类有父类TYPE_ACCEL，这个类的初始化应该调用函数kvm_accel_class_init（看，这里已经直接叫类class了）。如果用这个类声明一个对象，对象的大小应该是instance_size。是不是有点儿Java语言反射的意思，根据一些名称的定义，一个类就定义好了。</p><p>这里的调用链为：kvm_type_init-&gt;type_register_static-&gt;type_register-&gt;type_register_internal。</p><p>在type_register_internal中，我们会根据kvm_accel_type这个TypeInfo，创建一个TypeImpl来表示这个新注册的类，也就是说，TypeImpl才是我们想要声明的那个class。在qemu里面，有一个全局的哈希表type_table，用来存放所有定义的类。在type_new里面，我们先从全局表里面根据名字找这个类。如果找到，说明这个类曾经被注册过，就报错；如果没有找到，说明这是一个新的类，则将TypeInfo里面信息填到TypeImpl里面。type_table_add会将这个类注册到全局的表里面。到这里，我们注意，class_init还没有被调用，也即这个类现在还处于纸面的状态。</p><p>这点更加像Java的反射机制了。在Java里面，对于一个类，首先我们写代码的时候要写一个class xxx的定义，编译好就放在.class文件中，这也是出于纸面的状态。然后，Java会有一个Class对象，用于读取和表示这个纸面上的class xxx，可以生成真正的对象。</p><p>相同的过程在后面的代码中我们也可以看到，class_init会生成XXXClass，就相当于Java里面的Class对象，TypeImpl还会有一个instance_init函数，相当于构造函数，用于根据XXXClass生成Object，这就相当于Java反射里面最终创建的对象。和构造函数对应的还有instance_finalize，相当于析构函数。</p><p>这一套反射机制放在qom文件夹下面，全称QEMU Object Model，也即用C实现了一套面向对象的反射机制。</p><p>说完了初始化Module，我们还回到main函数接着分析。</p><h2>2.解析qemu的命令行</h2><p>第二步我们就要开始解析qemu的命令行了。qemu的命令行解析，就是下面这样一长串。还记得咱们自己写过一个解析命令行参数的程序吗？这里的opts是差不多的意思。</p><pre><code>    qemu_add_opts(&amp;qemu_drive_opts);
    qemu_add_opts(&amp;qemu_chardev_opts);
    qemu_add_opts(&amp;qemu_device_opts);
    qemu_add_opts(&amp;qemu_netdev_opts);
    qemu_add_opts(&amp;qemu_nic_opts);
    qemu_add_opts(&amp;qemu_net_opts);
    qemu_add_opts(&amp;qemu_rtc_opts);
    qemu_add_opts(&amp;qemu_machine_opts);
    qemu_add_opts(&amp;qemu_accel_opts);
    qemu_add_opts(&amp;qemu_mem_opts);
    qemu_add_opts(&amp;qemu_smp_opts);
    qemu_add_opts(&amp;qemu_boot_opts);
    qemu_add_opts(&amp;qemu_name_opts);
    qemu_add_opts(&amp;qemu_numa_opts);
</code></pre><p>为什么有这么多的opts呢？这是因为，我们上一节给的参数都是简单的参数，实际运行中创建的kvm参数会复杂N倍。这里我们贴一个开源云平台软件OpenStack创建出来的KVM的参数，如下所示。不要被吓坏，你不需要全部看懂，只需要看懂一部分就行了。具体我来给你解析。</p><pre><code>qemu-system-x86_64
-enable-kvm
-name instance-00000024
-machine pc-i440fx-trusty,accel=kvm,usb=off
-cpu SandyBridge,+erms,+smep,+fsgsbase,+pdpe1gb,+rdrand,+f16c,+osxsave,+dca,+pcid,+pdcm,+xtpr,+tm2,+est,+smx,+vmx,+ds_cpl,+monitor,+dtes64,+pbe,+tm,+ht,+ss,+acpi,+ds,+vme
-m 2048
-smp 1,sockets=1,cores=1,threads=1
......
-rtc base=utc,driftfix=slew
-drive file=/var/lib/nova/instances/1f8e6f7e-5a70-4780-89c1-464dc0e7f308/disk,if=none,id=drive-virtio-disk0,format=qcow2,cache=none
-device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1
-netdev tap,fd=32,id=hostnet0,vhost=on,vhostfd=37
-device virtio-net-pci,netdev=hostnet0,id=net0,mac=fa:16:3e:d1:2d:99,bus=pci.0,addr=0x3
-chardev file,id=charserial0,path=/var/lib/nova/instances/1f8e6f7e-5a70-4780-89c1-464dc0e7f308/console.log
-vnc 0.0.0.0:12
-device cirrus-vga,id=video0,bus=pci.0,addr=0x2
</code></pre><ul>
<li>
<p>-enable-kvm：表示启用硬件辅助虚拟化。</p>
</li>
<li>
<p>-name instance-00000024：表示虚拟机的名称。</p>
</li>
<li>
<p>-machine pc-i440fx-trusty,accel=kvm,usb=off：machine是什么呢？其实就是计算机体系结构。不知道什么是体系结构的话，可以订阅极客时间的另一个专栏《深入浅出计算机组成原理》。<br>
qemu会模拟多种体系结构，常用的有普通PC机，也即x86的32位或者64位的体系结构、Mac电脑PowerPC的体系结构、Sun的体系结构、MIPS的体系结构，精简指令集。如果使用KVM hardware-assisted virtualization，也即BIOS中VD-T是打开的，则参数中accel=kvm。如果不使用hardware-assisted virtualization，用的是纯模拟，则有参数accel = tcg，-no-kvm。</p>
</li>
<li>
<p>-cpu SandyBridge,+erms,+smep,+fsgsbase,+pdpe1gb,+rdrand,+f16c,+osxsave,+dca,+pcid,+pdcm,+xtpr,+tm2,+est,+smx,+vmx,+ds_cpl,+monitor,+dtes64,+pbe,+tm,+ht,+ss,+acpi,+ds,+vme：表示设置CPU，SandyBridge是Intel处理器，后面的加号都是添加的CPU的参数，这些参数会显示在/proc/cpuinfo里面。</p>
</li>
<li>
<p>-m 2048：表示内存。</p>
</li>
<li>
<p>-smp 1,sockets=1,cores=1,threads=1：SMP我们解析过，叫对称多处理器，和NUMA对应。qemu仿真了一个具有1个vcpu，一个socket，一个core，一个threads的处理器。<br>
socket、core、threads是什么概念呢？socket就是主板上插cpu的槽的数目，也即常说的“路”，core就是我们平时说的“核”，即双核、4核等。thread就是每个core的硬件线程数，即超线程。举个具体的例子，某个服务器是：2路4核超线程（一般默认为2个线程），通过cat /proc/cpuinfo，我们看到的是2<em>4</em>2=16个processor，很多人也习惯成为16核了。</p>
</li>
<li>
<p>-rtc base=utc,driftfix=slew：表示系统时间由参数-rtc指定。</p>
</li>
<li>
<p>-device cirrus-vga,id=video0,bus=pci.0,addr=0x2：表示显示器用参数-vga设置，默认为cirrus，它模拟了CL-GD5446PCI VGA card。</p>
</li>
<li>
<p>有关网卡，使用-net参数和-device。</p>
</li>
<li>
<p>从HOST角度：-netdev tap,fd=32,id=hostnet0,vhost=on,vhostfd=37。</p>
</li>
<li>
<p>从GUEST角度：-device virtio-net-pci,netdev=hostnet0,id=net0,mac=fa:16:3e:d1:2d:99,bus=pci.0,addr=0x3。</p>
</li>
<li>
<p>有关硬盘，使用-hda -hdb，或者使用-drive和-device。</p>
</li>
<li>
<p>从HOST角度：-drive file=/var/lib/nova/instances/1f8e6f7e-5a70-4780-89c1-464dc0e7f308/disk,if=none,id=drive-virtio-disk0,format=qcow2,cache=none</p>
</li>
<li>
<p>从GUEST角度：-device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1</p>
</li>
<li>
<p>-vnc 0.0.0.0:12：设置VNC。</p>
</li>
</ul><p>在main函数中，接下来的for循环和大量的switch case语句，就是对于这些参数的解析，我们不一一解析，后面真的用到这些参数的时候，我们再仔细看。</p><h2>3.初始化machine</h2><p>回到main函数，接下来是初始化machine。</p><pre><code>machine_class = select_machine();
current_machine = MACHINE(object_new(object_class_get_name(
                          OBJECT_CLASS(machine_class))));
</code></pre><p>这里面的machine_class是什么呢？这还得从machine参数说起。</p><pre><code>-machine pc-i440fx-trusty,accel=kvm,usb=off
</code></pre><p>这里的pc-i440fx是x86机器默认的体系结构。在hw/i386/pc_piix.c中，它定义了对应的machine_class。</p><pre><code>DEFINE_I440FX_MACHINE(v4_0, &quot;pc-i440fx-4.0&quot;, NULL,
                      pc_i440fx_4_0_machine_options);

#define DEFINE_I440FX_MACHINE(suffix, name, compatfn, optionfn) \
    static void pc_init_##suffix(MachineState *machine) \
    { \
......
        pc_init1(machine, TYPE_I440FX_PCI_HOST_BRIDGE, \
                 TYPE_I440FX_PCI_DEVICE); \
    } \
    DEFINE_PC_MACHINE(suffix, name, pc_init_##suffix, optionfn)


#define DEFINE_PC_MACHINE(suffix, namestr, initfn, optsfn) \
    static void pc_machine_##suffix##_class_init(ObjectClass *oc, void *data
) \
    { \
        MachineClass *mc = MACHINE_CLASS(oc); \
        optsfn(mc); \
        mc-&gt;init = initfn; \
    } \
    static const TypeInfo pc_machine_type_##suffix = { \
        .name       = namestr TYPE_MACHINE_SUFFIX, \
        .parent     = TYPE_PC_MACHINE, \
        .class_init = pc_machine_##suffix##_class_init, \
    }; \
    static void pc_machine_init_##suffix(void) \
    { \
        type_register(&amp;pc_machine_type_##suffix); \
    } \
    type_init(pc_machine_init_##suffix)
</code></pre><p>为了定义machine_class，这里有一系列的宏定义。入口是DEFINE_I440FX_MACHINE。这个宏有几个参数，v4_0是后缀，"pc-i440fx-4.0"是名字，pc_i440fx_4_0_machine_options是一个函数，用于定义machine_class相关的选项。这个函数定义如下：</p><pre><code>static void pc_i440fx_4_0_machine_options(MachineClass *m)
{
    pc_i440fx_machine_options(m);
    m-&gt;alias = &quot;pc&quot;;
    m-&gt;is_default = 1;
}

static void pc_i440fx_machine_options(MachineClass *m)
{
    PCMachineClass *pcmc = PC_MACHINE_CLASS(m);
    pcmc-&gt;default_nic_model = &quot;e1000&quot;;

    m-&gt;family = &quot;pc_piix&quot;;
    m-&gt;desc = &quot;Standard PC (i440FX + PIIX, 1996)&quot;;
    m-&gt;default_machine_opts = &quot;firmware=bios-256k.bin&quot;;
    m-&gt;default_display = &quot;std&quot;;
    machine_class_allow_dynamic_sysbus_dev(m, TYPE_RAMFB_DEVICE);
}
</code></pre><p>我们先不看pc_i440fx_4_0_machine_options，先来看DEFINE_I440FX_MACHINE。</p><p>这里面定义了一个pc_init_##suffix，也就是pc_init_v4_0。这里面转而调用pc_init1。注意这里这个函数只是定义了一下，没有被调用。</p><p>接下来，DEFINE_I440FX_MACHINE里面又定义了DEFINE_PC_MACHINE。它有四个参数，除了DEFINE_I440FX_MACHINE传进来的三个参数以外，多了一个initfn，也即初始化函数，指向刚才定义的pc_init_##suffix。</p><p>在DEFINE_PC_MACHINE中，我们定义了一个函数pc_machine_##suffix##<em>class_init。从函数的名字class_init可以看出，这是machine_class从纸面上的class初始化为Class对象的方法。在这个函数里面，我们可以看到，它创建了一个MachineClass对象，这个就是Class对象。MachineClass对象的init函数指向上面定义的pc_init</em>##suffix，说明这个函数是machine这种类型初始化的一个函数，后面会被调用。</p><p>接着，我们看DEFINE_PC_MACHINE。它定义了一个pc_machine_type_##suffix的TypeInfo。这是用于生成纸面上的class的原材料，果真后面调用了type_init。</p><p>看到了type_init，我们应该能够想到，既然它定义了一个纸面上的class，那上面的那句module_call_init，会和我们上面解析的type_init是一样的，在全局的表里面注册了一个全局的名字是"pc-i440fx-4.0"的纸面上的class，也即TypeImpl。</p><p>现在全局表中有这个纸面上的class了。我们回到select_machine。</p><pre><code>static MachineClass *select_machine(void)
{
    MachineClass *machine_class = find_default_machine();
    const char *optarg;
    QemuOpts *opts;
......
    opts = qemu_get_machine_opts();
    qemu_opts_loc_restore(opts);

    optarg = qemu_opt_get(opts, &quot;type&quot;);
    if (optarg) {
        machine_class = machine_parse(optarg);
    }
......
    return machine_class;
}

MachineClass *find_default_machine(void)
{
    GSList *el, *machines = object_class_get_list(TYPE_MACHINE, false);
    MachineClass *mc = NULL;
    for (el = machines; el; el = el-&gt;next) {
        MachineClass *temp = el-&gt;data;
        if (temp-&gt;is_default) {
            mc = temp;
            break;
        }
    }
    g_slist_free(machines);
    return mc;
}

static MachineClass *machine_parse(const char *name)
{
    MachineClass *mc = NULL;
    GSList *el, *machines = object_class_get_list(TYPE_MACHINE, false);

    if (name) {
        mc = find_machine(name);
    }
    if (mc) {
        g_slist_free(machines);
        return mc;
    }
......
}
</code></pre><p>在select_machine中，有两种方式可以生成MachineClass。一种方式是find_default_machine，找一个默认的；另一种方式是machine_parse，通过解析参数生成MachineClass。无论哪种方式，都会调用object_class_get_list获得一个MachineClass的列表，然后在里面找。object_class_get_list定义如下：</p><pre><code>GSList *object_class_get_list(const char *implements_type,
                              bool include_abstract)
{
    GSList *list = NULL;

    object_class_foreach(object_class_get_list_tramp,
                         implements_type, include_abstract, &amp;list);
    return list;
}

void object_class_foreach(void (*fn)(ObjectClass *klass, void *opaque), const char *implements_type, bool include_abstract,
                          void *opaque)
{
    OCFData data = { fn, implements_type, include_abstract, opaque };

    enumerating_types = true;
    g_hash_table_foreach(type_table_get(), object_class_foreach_tramp, &amp;data);
    enumerating_types = false;
}
</code></pre><p>在全局表type_table_get()中，对于每一项TypeImpl，我们都执行object_class_foreach_tramp。</p><pre><code>static void object_class_foreach_tramp(gpointer key, gpointer value,
                                       gpointer opaque)
{
    OCFData *data = opaque;
    TypeImpl *type = value;
    ObjectClass *k;

    type_initialize(type);
    k = type-&gt;class;
......
    data-&gt;fn(k, data-&gt;opaque);
}

static void type_initialize(TypeImpl *ti)
{
    TypeImpl *parent;
......
    ti-&gt;class_size = type_class_get_size(ti);
    ti-&gt;instance_size = type_object_get_size(ti);
    if (ti-&gt;instance_size == 0) {
        ti-&gt;abstract = true;
    }
......
    ti-&gt;class = g_malloc0(ti-&gt;class_size);
......
    ti-&gt;class-&gt;type = ti;

    while (parent) {
        if (parent-&gt;class_base_init) {
            parent-&gt;class_base_init(ti-&gt;class, ti-&gt;class_data);
        }
        parent = type_get_parent(parent);
    }

    if (ti-&gt;class_init) {
        ti-&gt;class_init(ti-&gt;class, ti-&gt;class_data);
    }
}
</code></pre><p>在object_class_foreach_tramp中，会调用将type_initialize，这里面会调用class_init将纸面上的class也即TypeImpl变为ObjectClass，ObjectClass是所有Class类的祖先，MachineClass是它的子类。</p><p>因为在machine的命令行里面，我们指定了名字为"pc-i440fx-4.0"，就肯定能够找到我们注册过了的TypeImpl，并调用它的class_init函数。</p><p>因而pc_machine_##suffix##<em>class_init会被调用，在这里面，pc_i440fx_machine_options才真正被调用初始化MachineClass，并且将MachineClass的init函数设置为pc_init</em>##suffix。也即，当select_machine执行完毕后，就有一个MachineClass了。</p><p>接着，我们回到object_new。这就很好理解了，MachineClass是一个Class类，接下来应该通过它生成一个Instance，也即对象，这就是object_new的作用。</p><pre><code>Object *object_new(const char *typename)
{
    TypeImpl *ti = type_get_by_name(typename);

    return object_new_with_type(ti);
}

static Object *object_new_with_type(Type type)
{
    Object *obj;
    type_initialize(type);
    obj = g_malloc(type-&gt;instance_size);
    object_initialize_with_type(obj, type-&gt;instance_size, type);
    obj-&gt;free = g_free;

    return obj;
}
</code></pre><p>object_new中，TypeImpl的instance_init会被调用，创建一个对象。current_machine就是这个对象，它的类型是MachineState。</p><p>至此，绕了这么大一圈，有关体系结构的对象才创建完毕，接下来很多的设备的初始化，包括CPU和内存的初始化，都是围绕着体系结构的对象来的，后面我们会常常看到current_machine。</p><h2>总结时刻</h2><p>这一节，我们学到，虚拟机对于设备的模拟是一件非常复杂的事情，需要用复杂的参数模拟各种各样的设备。为了能够适配这些设备，qemu定义了自己的模块管理机制，只有了解了这种机制，后面看每一种设备的虚拟化的时候，才有一个整体的思路。</p><p>这里的MachineClass是我们遇到的第一个，我们需要掌握它里面各种定义之间的关系。</p><p><img src="https://static001.geekbang.org/resource/image/07/30/078dc698ef1b3df93ee9569e55ea2f30.png?wh=2308*2053" alt=""></p><p>每个模块都会有一个定义TypeInfo，会通过type_init变为全局的TypeImpl。TypeInfo以及生成的TypeImpl有以下成员：</p><ul>
<li>name表示当前类型的名称</li>
<li>parent表示父类的名称</li>
<li>class_init用于将TypeImpl初始化为MachineClass</li>
<li>instance_init用于将MachineClass初始化为MachineState</li>
</ul><p>所以，以后遇到任何一个类型的时候，将父类和子类之间的关系，以及对应的初始化函数都要看好，这样就一目了然了。</p><h2>课堂练习</h2><p>你可能会问，这么复杂的qemu命令，我是怎么找到的，当然不是我一个字一个字打的，这是著名的云平台管理软件OpenStack创建虚拟机的时候自动生成的命令行。所以，给你留一道课堂练习题，请你看一下OpenStack的基本原理，看它是通过什么工具来管理如此复杂的命令行的。</p><p>欢迎留言和我分享你的疑惑和见解，也欢迎可以收藏本节内容，<span class="orange">反复研读</span>。你也可以把今天的内容分享给你的朋友，和他一起学习和进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/f9/caf27bd3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大王叫我来巡山</span>
  </div>
  <div class="_2_QraFYR_0">感觉设计这个软件的真厉害，怪不得我们自己做的业务系统自己都信不过，差距实在是太远，国内很多大公司在分享技术的时候也就是个PPT，根本不敢把代码放出来给大家看，也没有把实际用的效果展示给大家，只是给别人的感觉很牛逼而已。我感觉我从事这个工作这么久，真没遇到过这种大神</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，绝对牛</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-22 07:42:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/74/c9/d3439ca4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>why</span>
  </div>
  <div class="_2_QraFYR_0">解析 qemu 的执行步骤<br><br>1. 初始化所有模块<br><br>模块的信息(包括名称, 模块类型初始化函数等), 存在 TypeInfo 中, 通过调用 type_init, 将这些信息统一以 ModelEntry 的格式存储到 ModelTypeList 中. <br><br>module_call_init() 会调用 ModelTypeList 中所有模块的初始化函数,  从 ModelEntry 里存储的 TypeInfo 信息生成 TypeImpl(类似于 class 文件), 这个 TypeImpl 会存储到 qemu 的一个全局 hash 表中.  <br><br>2. 解析命令行参数<br><br>命令行参数比较多. <br><br>其中 -machine 参数用于指定计算机体系结构. 另外 网卡&#47;硬盘的配置要分表从 Host&#47;Guest 较多进行设置.<br><br>3. 初始化 machine<br><br>会在 qemu 的全局 hash 表中注册对应计算机体系结构的 TypeImpl 信息. 然后会调用所有 TypeImpl 的初始化方法 class_init 生成对应的 Class. 最后会得到一个 MachineClass. 然后调用 TypeImple 中的 instance_init 方法生成 MachineClass 的一个实例. </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-24 23:08:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/25/4b/4cbd001e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>佳俊</span>
  </div>
  <div class="_2_QraFYR_0">看了很久都没有搞明白一个type_init 宏定义出来的函数是怎么被调用的，直到发现在module_init 里面一个这个属性的定义__attribute__((constructor))，才明白是GNU C 里面的一个特性，在main函数调用前系统会自动先调用这个函数。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-28 09:23:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4a/2c/f8451d77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石维康</span>
  </div>
  <div class="_2_QraFYR_0">请问老师pc_machine_type_##suffix所对应的TypeImpl的instance_init是在哪初始化的？也就是从代码里如何体现从MachineClass生成MachineState？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-23 00:23:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5e/aa/b29ee77d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>robin</span>
  </div>
  <div class="_2_QraFYR_0">期望老师出一个讲openstack的专栏</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-01 14:25:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b2/e0/bf56878a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kkxue</span>
  </div>
  <div class="_2_QraFYR_0">nova通过libvirt驱动，将配置数据转化成XML格式的文件，用于创建虚拟机。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-26 22:56:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/4d/97/1d99a0a3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>柒城</span>
  </div>
  <div class="_2_QraFYR_0">老师，之后会不会开个专门讲虚拟化的专栏啊？吧QEMU和KVM好好讲讲</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-15 20:42:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5e/aa/b29ee77d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>robin</span>
  </div>
  <div class="_2_QraFYR_0">openstack是调用libvirt，libevirtd再调用kvm吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-01 14:30:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/90/8f/9c691a5f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>奔跑的码仔</span>
  </div>
  <div class="_2_QraFYR_0">请问，qemu使用的是哪个版本？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-23 17:06:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/e3/ad/a47728fd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>泡泡</span>
  </div>
  <div class="_2_QraFYR_0">请问代码是用的哪个内核版本</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 4.13</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-22 09:57:13</div>
  </div>
</div>
</div>
</li>
</ul>