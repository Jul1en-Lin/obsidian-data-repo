# HTTP ／ HTTPS（背）
> 相关笔记：[[JavaEE 初阶|JavaEE 知识总结]]


在网络原理中已经初次认识了HTTP，接下来介绍更深层次的内容，深究HTTP

# HTTP

## 方法

先从HTTP请求的首行Method来讲，常用的方法有GET、POST、PUT、DELETE......

|方法|说明|
| --------| --------------|
|GET|获取资源|
|POST|传输实体主体|
|PUT|传输文件|
|DELETE|删除文件|
|TRACE|追踪路径|

虽然分了很多的方法，但大多数都是用到的GET与POST方法，且各个方法之间的运用界限也没有那么的清，GET有时候也能做POST的事情，POST也能干GET的事情

那既然GET和POST的界限不是很清，但它们有没有很大的区别呢？（面试题）

- 回答是没有本质的区别，只是在使用习惯上有一定的区别
- GET通常用来表示“获取数据”的语义，POST表示“提交数据”的语义
- GET通常把服务器传输的数据放到query string中，而POST一般放到body中

🤔另外，基于GET和POST在网络上的讨论包括但不限于

1. GET请求一般是实现成“幂等”，POST请求则没有“幂等”的要求——“幂等”是请求出现重复的时候，结果是明确的 / 一致的，否之则“不幂等”

   幂等场景：第一次问1+1 = ？，回答是2。那第二次问它的结果就是明确的，是2

   不幂等场景：以打开b站为例，请求相同，都是刷新网页，但是看到的内容都不一样的，这是大模型在给你做个性化推荐，必然不幂等

   这话来源于HTTP RFC标准文档，建议程序员这么做，但实际上，程序员并没有很遵守这套规则
2. ❌ GET请求不安全，POST请求安全，因为GET把数据放query string里，POST则放到了body中

   这个结论是错误的。安不安全是针对数据有没有加密来说的，POST即使放到了body中，用抓包工具一抓也能看到，这种掩耳盗铃的说法站不住脚跟
3. ❌ GET单次请求传输的数据量小，POST大

   这个言论并不准确。这是以前非常久的说法，受限于久远的IE浏览器来说是这样的，但现在URL的长度也可以很长
4. ❌针对GET请求只能传输文本数据，POST可以传输文本和二进制数据

   这个言论并不准确。对于GET来说，二进制数据可以通过urlencode / base64来进行转码成文本的数据，再通过GET的URL来传输，所以GET实际上也可以传输二进制的数据（URL中确实不能放二进制数据）

网络上充斥着杂乱的信息，所以我们需要怀有求真的精神来学习 ᕙ(`▿´)ᕗ~

在实际开发中，这几个常见方法可以一起搭配使用，规范化以Restful风格来编写更有效率

## Host

通过抓包其中一个HTTP请求来看，在请求报头第一行有Host

![image](image-20251013163456-z87ztw7.png)

Host表示了服务器的地址和端口

## Content-Length

表示了body中的数据长度，单位是字节

![image](image-20251013163923-2dnss1w.png)

## Content-Type

表示了请求的body的解码格式

![image](image-20251013164128-2rmk7rd.png)

常见的数据格式有

1. text/html
2. text/css
3. application/json
4. application/javascript
5. image/gif
6. image/jpg
7. image/png
8. text/plain

在网页前端开发中

- html 负责网页结构 （骨）
- css 负责网页的样式  （皮）
- json 负责网页的交互逻辑  （灵魂）

在Content-Type中，告诉了浏览器body的数据格式，浏览器会根据不同的数据格式来解析，来渲染网页

## User-Agent

用来告诉浏览器的所在的用户的设备、操作系统、浏览器版本号.....

![image](image-20251013164906-93hdww8.png)

浏览器通过UA来区分用户是PC端or手机端，以此来渲染不同尺寸的页面，所以以前的前端程序员需要写两套代码来适配不同的人群

除此之外，根据UA的不同，程序员来写不同的代码以此适配不同的浏览器版本

随着技术发展，还有别的方案来代替写两套代码的情况——“响应式编程”（基于css3），工作原理是根据用户端屏幕的尺寸，自动进行不同的排版，效率更高，且不需要写两套代码

## Referer

是记录从哪个页面跳转过来的

注意：这个Referer是面向服务器的，与浏览器的“前进” / “后退”不一样，后者是浏览器记录的，前者是发给服务器的

![image](image-20251013165837-69864yn.png)

Referer的作用一般有收集数据来源 / 溯源 / 统计 的作用，是重要的数据

## Cookie

我们一般在新下载一个浏览器的时候都会弹出“接受Cookie”的选项，那什么是Cookie呢？

Cookie是浏览器给网页提供的向本地存储数据的机制，但这个存储权限是有限的，且经过浏览器的安全操作再写入到本地硬盘中

类似的机制还有Local Storage，IndexDB（浏览器通过类似SQL语句方式来操作表）

在HTTP请求抓包中可以看到，Cookie保存的也是一系列的键值对，键值之间用 = 分开，键值对之间用 ；分开

![image](image-20251013170324-kun4iv5.png)

网页本身是不允许向本地存储 / 修改数据的，因为这种操作很危险。那为什么会允许写入本地存储数据呢？接着看下去👇

‍

当用户首次访问网页时，服务器会为该客户端创建一个会话，并通过 Cookie 中的一系列键值对来标识该会话。这些键值对会组成字符串，由浏览器存储在本地。（Cookie 存储在本地 / 客户端）

由于服务器会维护多个会话，不同用户对应着不同会话。所以用户通过SessionId都能找到，至于**<u>==Session==</u>**对象（对象里存什么，是程序员之间决定的）

---

当本地用户存储了一系列的Cookie后，如果后续再访问同一个网页，HTTP请求首先会把Cookie发给服务器，此时服务器维护了一个类似HashMap的一个表，里面存储了<会话ID>与<会话对象>的一系列键值对（<sessionID>与<session对象>，其中session对象中又存放着一个个键值对）

当客户端发来的Cookie，服务器检查后有与之对应的的时候，服务器就会直接返回对应的信息，例如登录信息、用户权限、配置等其他功能，避免了重复访问请求 / 对错误的session对象回答 （能认出对应的session对象）的情况出现

![image](image-20251013172915-q62exw0.png)

## Session

由上述描述可以看出，Cookie 与 session 是有一定的联系的

但上面提到的都比较泛，这里就详细讲一讲他们之间的区别与联系

假设一个场景：假如在一家医院，每个医生是需要每天看上百号病人的，有的还需要回来复诊，那么光凭记忆力是无法记住全部的，那如何记住呢？它根据病人的诊疗卡的id能读出你的就诊记录、开药记录、挂号....等

1. 现在去医院都需要拿病历本建个档，需要你的姓名、身份证号、家庭住址、电话号码.....

2. 建档完成后，医院会发张诊疗卡，上面有属于你的卡号id（Cookie）
3. 病人拿了这个卡就能去就诊、复诊、取药了
4. 那医生如何知道你的信息？——根据诊疗卡上的id，去医院的服务器上查询资料，里面就存储了病人的信息（Session）

**其中，诊疗卡拿在病人手上，病人的信息在服务器上，分别对应了 Cookie 存储在客户端上，Session 存储在服务器上的规律。**

举个具体的例子：

在医院中，有三个人一起去柜台开发票。医院为了区分这三个人，就给这三个人分别建档开诊疗卡。

这三个人拿着各自的诊疗卡，医院就根据诊疗卡的id，就能区分这三个人，给出响应的发票

如果这三个人没有诊疗卡，那医院也分辨不出来谁是谁，要给谁开哪一种发票~

服务器同一时刻收到的请求是很多的，服务器需要清楚的区分每个请求是属于哪个用户，也就是属于哪个会话，就需要在服务器这边记录每个会话以及与用户的信息的对应关系，所以 Cookie 中存储Session（会话的） id。

Session 是服务器为了保存用户信息而创建的一个特殊的对象。

![image](image-20251105235911-oegc53t.png)

Q: **如果负载均衡，用户请求访问到不同服务器上，每个服务器都有独立的Session，你该怎么解决？**

1. 可以通过技术手段，让用户只访问特定的服务器，确保用户访问不会跨服务器

2. 可以通过 Redis 的共享Session机制，服务器的Session都放到 Redis 中，无论用户请求访问哪个服务器，服务器都要从 Redis 取Session，确保 Session 不会丢失。

## Session / Cookie 区别

- Cookie 是客户端保存用户信息的一种机制，Session 是服务器端保存用户信息的一种机制
- Cookie 和 Session 之间主要是通过 SessionId 关联起来的，SessionId 是 Cookie 和 Session 之间的桥梁
- Cookie 和 Session 经常会搭配在一起使用，但不是必须配合

  - 完全可以用 Cookie 来保存一些数据在客户端，这些数据不一定是用户身份信息，也不一定是 SessionId
  - Session 中的 SessionId 也不要非得通过 Cookie / Set-Cookie 传递，比如还有通过 URL 传递

## 无状态协议

**无状态** 是指 **服务器在处理每一个请求时，都是独立的**，不会主动记忆客户端之前的任何请求或状态信息

HTTP 是无状态的协议，意味着服务器不会自动保存客户端的历史交互信息。每个请求必须独立携带所需数据，这简化了设计但也需要额外机制来维持会话状态

- 优点：使得服务器设计变得简单，且可扩展性强，服务器无需记忆客户端的请求状态，方便方便负载均衡与横向扩展（加服务器）

- 缺点：需要额外机制保存状态，例如 Cookie / Session，还有 Token / JWT

## 状态码

响应的状态码在HTTP响应的首行中

最常见的就是200 请求成功，2xx开头的都可以视为成功，只是成功的内容有不一样

![image](image-20251013174340-dvg5psf.png)

404 NotFound 意思是访问的资源在服务器上不存在，一般是客户端填写的URL有问题，服务器找不到

![image](image-20251013174640-k63n6km.png)

403 Forbidden 拒绝访问，说明访问的页面无权限，以码云的403举栗

![image](image-20251013174809-4ycotz2.png)

4xx 都属于是“客户端错误”

除此之外还有 

500 Internal Server Error 服务器出错——服务器挂了 / 服务器非常繁忙   5xx都属于“服务器错误”

302 Move temporarily 临时重定向——belike手机号的呼叫转移，当你呼叫手机号码1的时候，运营商会自动转接到手机号码2

客户端想要请求URL1的时候，服务器会告诉客户端你去访问URL2就能得到你想要的东西了！！然后客户端就去访问URL2了~~

![image](image-20251013213257-16yqfy8.png)

有临时重定向，那就会有永久重定向

301 redirect   永久重定向——即URL2的地址永远不会变，不过一般永久重定向的问题，浏览器可以做缓存，也没有必要再访问服务器重定向，这样效率更高，所以301很少出现

总结一下状态码

|状态码|类别|小幽默（以服务器的视角）|
| --------| --------------| --------------------------|
|1xx|请求正在处理|别急，我有我自己的节奏 ~|
|2xx|成功|Here you go ~|
|3xx|重定向|Go away ！|
|4xx|客户端错误|You fucked up|
|5xx|服务器错误|I fucked up|

## 构造HTTP请求

前面通过Fiddler，我们抓包HTTP请求，那我们能不能自己构造一个HTTP请求，手动打包发给服务器呢？答案是可以的

HTTP协议本质上就是往TCP socket中写一个特定格式的字符串，发给服务器，我们手动构造一个特定的字符串也是可以的

主流的构造HTTP请求的方式有

- 通过form表单来构造HTTP请求
- 通过ajax来构造HTTP请求
- 通过Java socket来构造HTTP请求——这是最原始的方案

  以最基础的Java socket来讲，这是一个最简单的HTTP请求

  ```java
  public class MyHttpClient {
      private Socket socket;
      private String ip;
      private int port;

      public MyHttpClient(String ip,int port) throws IOException {
          this.ip = ip;
          this.port = port;
          socket = new Socket(ip,port);
      }

      public String GET(String url) throws IOException {
          // 构造字符串
          StringBuilder request = new StringBuilder();
          // 构造首行
          request.append("GET " + url + "HTTP/1.1\n");
          // 构造header
          request.append("Host: " + ip + ":" + port + "\n");
          // 构造空行
          request.append("\n");
          // 发送数据
          OutputStream stream = socket.getOutputStream();
          stream.write(request.toString().getBytes());
          // 读取 request 响应
          InputStream inputStream = socket.getInputStream();
          byte[] bytes = new byte[1024 * 1024];
          int n = inputStream.read(bytes);
          return new String(bytes,0,n,"UTF-8");
      }

      public String POST(String url,String body) throws IOException {
          StringBuilder request = new StringBuilder();
          // 构造首行
          request.append("POST " + url + "HTTP/1.1\n");
          // 构造 header
          request.append("Host: " + ip + ":" + port + "\n");
          request.append("Content-Length: " + body.getBytes().length + "\n");
          request.append("Content-Type: text/plain\n");
          // 构造空行
          request.append("\n");
          // 构造 body
          request.append(body);
          // 发送数据
          OutputStream outputStream = socket.getOutputStream();
          outputStream.write(request.toString().getBytes());
          // 接收数据
          InputStream inputStream = socket.getInputStream();
          byte[] bytes = new byte[1024 * 1024];
          int n = inputStream.read(bytes);
          return new String(bytes,0,n,"utf-8");
      }
  }
  ```

  在Java标准库中，也封装了一系列方案，让我们能直接访问HTTP服务器 / HTTPS服务器

  ![image](image-20251020121333-vyk0et6.png)

  ```java
  public static void main(String[] args) throws IOException, InterruptedException {
          // 创建一个 HTTP 对象
          // 创建对象的时候使用工厂类
          HttpClient client = HttpClient.newHttpClient();

          // 这样 new 出来需要重写许多方法，非常的麻烦，有另一种构造 Http 请求的方法
      /*HttpRequest request = new HttpRequest() {
          @Override
          public Optional<BodyPublisher> bodyPublisher() {
              return Optional.empty();
          }

          @Override
          public String method() {
              return "";
          }

          @Override
          public Optional<Duration> timeout() {
              return Optional.empty();
          }

          @Override
          public boolean expectContinue() {
              return false;
          }

          @Override
          public URI uri() {
              return null;
          }

          @Override
          public Optional<HttpClient.Version> version() {
              return Optional.empty();
          }

          @Override
          public HttpHeaders headers() {
              return null;
          }
      }*/

          // 创建 request 使用工厂类
          HttpRequest request = HttpRequest.newBuilder()
                  .uri(URI.create("https://www.gitee.com"))
                  .GET()
                  .header("User-Agent","xxxx")
                  .build();
  		// 链式调用 最后以 build 方法结尾

          // 发送请求，获取响应,且以 String 的格式处理响应
          // send 执行之后会阻塞等待，直到响应返回
          HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
  		// 查看返回的HTTP响应报文
          System.out.println(response.statusCode());
          System.out.println(response.headers());
          System.out.println(response.body());
      }
  ```

  其中看到发送请求用send方法，send方法也分同步发送和异步发送

  ![image](image-20251020123106-z08sv5r.png)

用编写代码的方式来构造HTTP请求，是测试的其中一种方案

另外，可以用更加专业的专门的构造HTTP请求的工具，例如Postman、Apifox等...

---

# HTTPS

通过上述了解，我们对HTTP有了相对深刻的认识，但是HTTP有个问题——它 “明文传输” 的特点，导致了它有安全的漏洞，为了应对这种情况，就基于HTTP发明了HTTPS加密协议（SSL/TLS 协议）

## SSL/TLS 协议

为了加密传输的内容，服务器和客户端就引入 “密钥” 这个东西，负责加密&解密

主流的HTTPS加密方式有对称加密、非对称加密和应对中间人攻击的证书加密

1. 对称加密

   客户端 / 服务器生成一个密钥，同时对方也拥有这个密钥。这个密钥既可以加密，也可以解密，是客户端随机生成的，不同客户端对应不同的密钥，再通过网络传输给服务器

   像这种一把锁控制全部的方法，传输效率高，但安全系数低，上面说是客户端通过网络传输密钥，那传输的过程中就很容易被窃取，毕竟传输 “密钥是xxx” 这个信息并没有加密

   ![image](image-20251020134034-w6v61k5.png "对称加密方式")
2. 非对称加密

   是对称加密的更进一步的的加密方式

   拥有两把钥匙，pub公钥是公共的，负责加密；pri私钥是私有的，负责解密，公匙谁都可以拥有，但私钥只在服务器手里，黑客如果想得到私钥就只能黑进服务器，安全系数增加

   公钥就想是锁头，谁都能上锁，私钥就是钥匙，只有一把

   这种传输方式虽然更加安全，但是又加多了一把锁，占用的系统资源是比较多的，效率不高

   ![image](image-20251020134048-vbuohea.png "非对称加密方式")
3. 对称加密+非对称加密

   聪明的人们就想到了通过两者结合的方式。第一次把对称加密的钥匙，通过非对称加密的方式，传输给对方，这样黑客就不知道对称密钥是多少了，接下来的文件就只需要通过对称密钥进行加密，效率增加

   ![image](image-20251020135450-xa412pm.png "对称加密 + 非对称加密结合")
4. 中间人攻击

   但是上述这种方式就绝对安全吗？道高一尺魔高一丈，黑客可以冒充对方的方式，来间接的窃取，belike双面间谍

   具体的流程如图所示

   ![image](image-20251020170241-6v9w7wr.png)
5. 证书加密

   为了应对这种情况，引入了证书的机制

   证书是一个结构化的字符串，由第三方公证机构办法给服务器的

   所以在客户端申请访问服务器时，都会先索要服务器的证书，证明 “你是你”

   具体流程如图所示

   ![image](image-20251020194428-52mkyds.png)

   从图中能看出，pub3 / pri3 跟平时的用途不太一样？ 这里的pub3是负责解密，pri3负责加密；跟加密通信的pub加密，pri解密不同？

   - 前者是签名机制
   - 后者是通信机制

   ---

   这是因为两者的目标不一样。

   - 签名机制是 “我要让别人知道这真的是我发的，且内容没被改过” ，所以私钥要我自己保存着，签名确实是我发出的（pri在我这，只有我能加密）

   - 通信机制是 “我要让别人（只有拥有私钥的人）能解密”，所以只有他有pri能解密

这就是HTTPS，在HTTP的基础上，引入了加密层。像这种加密方式就是 SSL/TLS 协议。

所以能大致理解成HTTPS == HTTP + SSL/TLS 协议(●'◡'●)
