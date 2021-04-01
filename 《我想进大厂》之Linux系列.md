说真的，这就是《我想进大厂》系列第八篇，但是Linux的问题确实很少，就这样，强行编几个没有营养的问题也没啥意义。

### 1.CPU负载和CPU利用率的区别是什么？

首先，我们可以通过`uptime`，`w`或者`top`命令看到CPU的平均负载。

![](https://tva1.sinaimg.cn/large/0081Kckwgy1glcycyltujj30yc0b4jsu.jpg)

![](https://tva1.sinaimg.cn/large/0081Kckwly1glcye7y6ixj31300rgq9x.jpg)

**Load Average** ：负载的3个数字，比如上图的4.86，5.28，5.00，分别代表系统在过去的1分钟，5分钟，15分钟内的系统平均负载。他代表的是**当前系统正在运行的和处于等待运行的进程数之和**。也指的是处于**可运行状态**和**不可中断状态**的平均进程数。

如果单核CPU的话，负载达到1就代表CPU已经达到满负荷的状态了，超过1，后面的进行就需要排队等待处理了。

如果是是多核多CPU的话，假设现在服务器是2个CPU，每个CPU2个核，那么总负载不超过4都没什么问题。

怎么查看CPU有多少核呢？

通过命令`cat /proc/cpuinfo | grep "model name"`查看CPU的情况。

![](https://tva1.sinaimg.cn/large/0081Kckwly1glcyf0ym8yj30w00ca0vv.jpg)

通过`cat /proc/cpuinfo | grep "cpu cores"`查看CPU的核数

![](https://tva1.sinaimg.cn/large/0081Kckwly1glcyfr9zfkj30mo0cagmo.jpg)

**CPU 利用率**：和负载不同，CPU利用率指的是当前**正在运行**的进程实时占用CPU的百分比，他是对一段时间内CPU使用状况的统计。

我举个栗子🌰：

假设你们公司厕所有1个坑位，有一个人占了坑位，这时候负载就是1，如果还有一个人在排队，那么负载就是2。

如果在1个小时内，A上厕所花了10分钟，B上厕所花了20分钟，剩下30分钟厕所都没人使用，那么这一个小时内利用率就是50%。



### 2.那如果CPU负载很高，利用率却很低该怎么办？

CPU负载很高，利用率却很低，说明处于等待状态的任务很多，负载越高，代表可能很多僵死的进程。通常这种情况是IO密集型的任务，大量请求在请求相同的IO，导致任务队列堆积。

同样，可以先通过`top`命令观察(截图只是示意，不代表真实情况)，假设发现现在确实是高负载低使用率。

![](https://tva1.sinaimg.cn/large/0081Kckwly1glcyge7dkpj31300lmafv.jpg)

然后，再通过命令`ps -axjf`查看是否存在状态为`D+`状态的进程，这个状态指的就是不可中断的睡眠状态的进程。处于这个状态的进程无法终止，也无法自行退出，只能通过恢复其依赖的资源或者重启系统来解决。(对不起，我截不到D+的状态)

![](https://tva1.sinaimg.cn/large/0081Kckwly1glcyh7p91aj310o0i4tcg.jpg)



### 3.那如果负载很低，利用率却很高呢？

如果你的公司只有一个厕所，外面没人排队，却有一个人在里面上了大半个小时，这说明什么？

两种可能：他没带纸，或者一些奇怪的事情发生了？

这表示CPU的任务并不多，但是任务执行的时间很长，大概率就是你写的代码本身有问题，通常是计算密集型任务，生成了大量耗时短的计算任务。

怎么排查？直接`top`命令找到使用率最高的任务，定位到去看看就行了。如果代码没有问题，那么过段时间CPU使用率就会下降的。



### 4.那如果CPU使用率达到100%呢？怎么排查？

1. 通过`top`找到占用率高的进程。

![](https://tva1.sinaimg.cn/large/0081Kckwly1glcyi2me90j31300dgdir.jpg)

2. 通过`top -Hp pid`找到占用CPU高的线程ID。这里找到958的线程ID

![](https://tva1.sinaimg.cn/large/0081Kckwly1glcyiw3r0ej31300gyaej.jpg)

3. 再把线程ID转化为16进制，`printf "0x%x\n" 958`，得到线程ID`0x3be`

![](https://tva1.sinaimg.cn/large/0081Kckwly1glcyjilh9pj30eq07m74l.jpg)

4. 通过命令`jstack 163 | grep '0x3be' -C5 --color` 或者 `jstack 163|vim +/0x3be -` 找到有问题的代码

![](https://tva1.sinaimg.cn/large/0081Kckwly1glcyk25joaj31kw0msn2s.jpg)



### 5.说说常见的Linux命令吧？

**常用的文件、目录命令**

`ls`：用户查看目录下的文件，`ls -a`可以用来查看隐藏文件，`ls -l`可以用于查看文件的详细信息，包括权限、大小、所有者等信息。

![](https://tva1.sinaimg.cn/large/0081Kckwly1glcx1jw2k3j30ys0dgq55.jpg)

`touch`：用于创建文件。如果文件不存在，则创建一个新的文件，如果文件已存在，则会修改文件的时间戳。

`cat`：cat是英文`concatenate`的缩写，用于查看文件内容。使用`cat`查看文件的话，不管文件的内容有多少，都会一次性显示，所以他不适合查看太大的文件。

`more`：more和cat有点区别，more用于分屏显示文件内容。可以用`空格键`向下翻页，`b`键向上翻页

`less`：和more类似，less用于分行显示

`tail`：可能是平时用的最多的命令了，查看日志文件基本靠他了。一般用户`tail -fn 100 xx.log`查看最后的100行内容

**常用的权限命令**

`chmod`：修改权限命令。一般用`+`号添加权限，`-`号删除权限，`x`代表执行权限，`r`代表读取权限，`w`代表写入权限，常见写法比如`chmod +x 文件名` 添加执行权限。

还有另外一种写法，使用数字来授权，因为`r`=4，`w`=2，`x`=1，平时执行命令`chmod 777 文件名`这就是最高权限了。

第一个数字7=4+2+1代表着所有者的权限，第二个数字7代表所属组的权限，第三个数字代表其他人的权限。

常见的权限数字还有644，所有者有读写权限，其他人只有只读权限，755代表其他人有只读和执行权限。

`chown`：用于修改文件和目录的所有者和所属组。一般用法`chown user 文件`用于修改文件所有者，`chown user:user 文件`修改文件所有者和组，冒号前面是所有者，后面是组。

**常用的压缩命令**

`zip`：压缩zip文件命令，比如`zip test.zip 文件`可以把文件压缩成zip文件，如果压缩目录的话则需添加`-r`选项。

`unzip`：与zip对应，解压zip文件命令。`unzip xxx.zip`直接解压，还可以通过`-d`选项指定解压目录。

![](https://tva1.sinaimg.cn/large/0081Kckwgy1glcxvu03d7j31eo0b4jt1.jpg)

`gzip`：用于压缩.gz后缀文件，gzip命令不能打包目录。需要注意的是直接使用`gzip 文件名`源文件会消失，如果要保留源文件，可以使用`gzip -c 文件名 > xx.gz`，解压缩直接使用`gzip -d xx.gz`

`tar`：tar常用几个选项，`-x`解打包，`-c`打包，`-f`指定压缩包文件名，`-v`显示打包文件过程，一般常用`tar -cvf xx.tar 文件`来打包，解压则使用`tar -xvf xx.tar`。

Linux的打包和压缩是分开的操作，如果要打包并且压缩的话，按照前面的做法必须先用tar打包，然后再用gzip压缩。当然，还有更好的做法就是`-z`命令，打包并且压缩。

使用命令`tar -zcvf xx.tar.gz 文件`来打包压缩，使用命令`tar -zxvf xx.tar.gz`来解压缩



