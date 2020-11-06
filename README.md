

# WWDC2020 run time 类结构的优化

## iOS  类的数据结构

首先了解一下类自身的结构是这样的，包含了指向元类、超类和方法缓存的指针。

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkeea44tjtj30kw09mjup.jpg" alt="myclass" style="zoom:25%;" />

但同时它还有一个`class_ro_t`指针来指向存储其它更多的数据。

ro(readonly),在编译阶段就已确定。

iOS 13的runtime 源码`class_ro_t`结构（iOS14源码还未放开）：

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkecqzz7r6j30u70u0th7.jpg" alt="class_ro_t" style="zoom: 15%;" />

### 类被使用时，发生了变化

当类被使用时，也就是被第一次加载到内存时，他们一经使用就会发生变化。要了解变化，就必须先了解一下`clean memory`和`dirty memory` 的区别

- **clean memory：**加载后不会发生更改的内存，`class_ro_t` 就属于`clean memory`因为它是readonly的，可以进行移除当需要时可以从磁盘中重新加载。
- **dirty memory：**指在进程（app）运行时会发生更改的内存，要比clean memory 重量级的多，只要进程在运行，它就必须一直存在。

“这个变化”，就是类结构一旦被使用就会变成dirty memory ，因为运行时可能会向它写新的数据，比如创建一个新的方法动态指向类。

dirty memory是类数据被分成两部分（MYClass、class_ro_t）的原因。

通过分开那些永远不会更改的数据，可以把大部分数据存储为clean memory

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkee9ytumrj30la0bujvb.jpg" alt="Dirty Memory" style="zoom:25%;" />

运行时需要追踪每个类的更多信息，所以当类首次被使用时，运行时会为它分配额外的存储容量`class_rw_t`，可读写。

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkee9u807bj30wi0ak7af.jpg" alt="class_rw_t" style="zoom:25%;" />

源码：

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkeed68lzpj30ym0re43k.jpg" alt="class_rw_t_code" style="zoom:25%;" />

在这个数据结构中，存储了只有在运行时才会生成的新数据

例如都会生成一个树状结构，这是通过使用FirstSubckass和Next Sibling Class 指针实现，这便于运行时遍历当前使用的所有类，对于使方法缓存无效非常有用。

### 这就是为什么可以动态添加属性，而不能动态添加成员变量

这里也可以看到为什么可以动态的去添加Methods和properties，也可以知道为什么不能动态添加成员变量，因为这个`class_rw_t`可读写结构里面就没有`IVars`

## iOS14  runtime 类的优化

### 为什么优化？

通过上面，我们可以看出，所有被使用到的类都会多存储一份class_rw_t结构，而大部分类（90%）都没有动态修改的需求，Apple官方统计class_rw_t的内存占用大约会在30M

### 怎么优化，能优化多少？

对class_rw_t进行拆分，常用的和不常用的分两部分，然后按需决定是否需要加载class_rw_ext_t进行分配内存，数据结构体积几乎小了一半，据apple官方统计平均可以节约14M内存。

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkfb3triuxj30w80h8q9n.jpg" alt="class_rw_t 2" style="zoom:25%;" />

在iOS我们可能没有好的方法通过数值来感知到这个优化，而在mac os 上我们可以通过在终端看到

拿微信举例，先打开微信进程，然后在终端执行命令：

```shell
heap WeChat | egrep 'class_rw|COUNT'  #egrep用于在文件内查找指定的字符串，COUNT数量
```

![heap_wechat](https://tva1.sinaimg.cn/large/0081Kckwgy1gkfbb87r7xj317e03yq3u.jpg)
