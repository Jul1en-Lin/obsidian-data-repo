# 文件操作与IO流
> 相关笔记：[[JavaEE 初阶|JavaEE 知识总结]]


# 文件概念

日常生活中我们习惯把信息写在纸上，并将这种纸张放进文件夹中，方便日后的查找和使用，而在计算机中“文件”也就这样的角色，它是数据存储的基本单位，可以保存各种数据（文字，图片，程序，文件等）。

通常文件保存的都是长期的数据，保存在硬盘上，对数据进行分类管理，文件除了保存数据，还记录着文件名，大小，时间戳，权限等信息

文件把数据抽象化，屏蔽了数据复杂的存储细节

‍

在Java中，也对这种文件函数进行了封装，并提供了一系列方法，方便对文件进行读写操作，大致可分为两部分

1. 文件系统操作——编写代码对文件进行改名、删除、复制、移动，查看目录等操作
2. 文件内容操作——对文件的内容进行读写

## 文件系统操作（File类）

File类是Java.io包下的一个类，可以将文件抽象为一个file对象对其进行操作

### 构造方法

![image](image-20250916162449-zs74kf7.png)一个file对象对应着一个文件/目录（目录也算是文件），***<u>==里面传入参数可以是绝对路径，也可以是相对路径==</u>***

#### 绝对路径 vs 相对路径

- 从根目录开始，一直到文件/目录 例如D:\Siyuan data\repo\refs这就是绝对路径
- 相对路径，需要有一个基准目录，例如D:\Code\J2025 基准目录就以"==./=="表示“D:\Code\J2025”，而在J2025后的文件/目录就表示为"./J20250915-IO/text.txt "

### 方法

![image](image-20250916163129-t09fmdo.png)![image](image-20250916163140-6mfp3le.png)其中移动文件与改名在实现的逻辑上是一样的，都在“改名”。父目录就是上一级的目录

‍

## 文件内容操作（IO流）

文件内容操作主要针对的是读和写，操作系统也提供了相关的api函数，Java对其进行了进一步的封装

Java针对文件内容的操作，采用了一组“流对象”进行表示，“流”形似“水流”，数据像水流一样，生生不息，绵延不断。就像我们平时需要接水，比如一次接100ml的水，那我可以一次性装满，也可以分多次。

Java对数据的处理也是这样，比如要读1000字节的文件数据，可以一次性读完，也可以分十次、二十次，一百次....总之，将数据比喻成“IO流”，即Input / Output，对文件进行读写的操作![image](image-20250916171145-96cdljz.png)

在Java标准库中，对这样的IO流对象分成了两大类

1. 字节流——读写的时候，以字节为基本单位
2. 字符流——读写的时候，以字符为基本单位

![image](image-20250916170735-gicik5t.png)

### 字节流

适合处理二进制的文本内容

字节流对应的IO类有InputStream / OutputStream，他们都是抽象类，需要FileInputStream / FileOutputStream 或其他子类实现

#### InputStream

##### **方法**

![image](image-20250916171319-u8ys1fh.png)

##### 构造方法

我们以FileInputStream实现为例，构造方法中可传入file文件或相对路径/绝对路径，同样表示一个流对象代表一个文件

![image](image-20250916172308-g79cu5c.png "普通try catch")![image](image-20250916171918-texgarj.png "try with resources")

---

不难发现在实现InputStream类中需要try处理抛出异常，那两个版本的try有何不同？

这就涉及到InputStream的核心操作了

1. 打开文件（构建流对象）
2. 读文件
3. 关闭文件（close）

在打开文件中，文件是需要放到文件描述符表中的，这会消耗一定的资源，所以文件的资源关闭是很有必要的。除了线程自身关闭的特殊原因外，我们都需要手动关闭输入/输出流，这样才不会导致**<u>*==文件资源泄露，在实际开发中这是非常严重的问题。在程序运行过程中，也许存在很慢的资源泄露，还不好定位，因此在以后的工作中，服务器程序经常会有操作“定期例行重启”，将资源重新释放掉，这样有小的资源泄露也没有多大的影响~~==*</u>**

---

##### **常用read方法**

![image](image-20250916184805-24s1aqw.png)

此处可传入byte数组，并指定选择读入的byte[]的起始位置与长度，**==*注意此处的byte[]是一个“*==**​[[输出型参数|输出型参数]]​ ***==”==***​ **==，即传入的数组是一个“空白”数组，在方法内部再对数组进行内容填充，方法执行完毕之后，就可以通过数组的参数得到函数返回的结果了，并且read方法读到了文件末尾会返回“-1”，作为特殊的标识，需要在代码中依靠-1来跳出读代码的循环==**

```java
StringBuilder stringBuilder = new StringBuilder();
	try (Reader reader = new FileReader(file)){
    	while (true) {
        	char[] chars = new char[1024];
            int n = reader.read(chars);
            if (n == -1) break;
            stringBuilder.append(chars,0,n);
        }
    }catch (IOException e){
        e.printStackTrace();
    }
```

##### **<u>以下是close的使用</u>**

但是如果用普通的try-catch我们容易忘记关闭文件，或者在try中间的代码抛出异常了，进而没有执行到close操作，想解决这个问题可以把close操作在finally中

![image](image-20250916175757-0fk9nwf.png "普通try-catch")

💫**==用try with resources就很好的解决了这个问题，在try代码执行完毕之后会自动调用close，并妥善处理close的异常，并且代码看起来更加的优雅简洁，并且在try中可以创建多个流对象，之间用；分开==**

![image](image-20250916180007-ox71roj.png "try with resources")

另外 try with resources要求try（）里面的对象要实现Closeable接口，Closeable提供了close方法，如果需要在自己定义的类中实现需要手动实现Closeable接口![image](image-20250916183228-d4umthv.png)

另外，Scanner也可以搭配InputStream使用的，在构造方法中对输入的内容进行格式解析（类似System.in——本质上也是输入内容）

![image](image-20250916183531-ul47pp1.png)![image](image-20250916183541-smyzjz4.png)

#### OutputStream

##### **方法**

![image](image-20250916183728-890pk3r.png)![image](image-20250916183749-xg1dly4.png)

##### 构造方法

以FileOutputStream实现为例，同样需要try处理异常

![image](image-20250916184559-f5h25hv.png)

##### 常用write方法

![image](image-20250917003241-xsmto0x.png)

‍

##### 构造参数传入

***==<u>注意：OutputStream默认情况下，打开文件会先清空文件的内容！！！为了避免源文件内容的损失，可以在构造方法中添加参数，使OutputStream不清空~~，这样新写入的内容就会在文件末尾</u>==***

![image](image-20250917001125-njfovck.png)

### 字符流

适合处理文本内容

字符流对应的IO类有Reader / Writer，同样也是抽象类，需要FileReader / FileWriter 或其他子类实现，与InputStream / OutputStream的方法类似

#### Reader

##### 方法

![image](image-20250917001729-qfbxuyp.png)

##### 构造方法

一样的，同样需要处理异常，用try with resources 更好处理文件关闭的情况

![image](image-20250917001938-oyx96d4.png)

##### 常用read方法

![image](image-20250917002217-nnms7m2.png)

与InputStream的read方法一样，返回-1代表已读到文件末尾

#### Writer

##### 方法

![image](image-20250917002523-npmh8ll.png)

##### 构造方法

![image](image-20250917002633-0dnbv67.png)

##### 常用writer方法

![image](image-20250917003331-9vawssf.png)

‍

‍
