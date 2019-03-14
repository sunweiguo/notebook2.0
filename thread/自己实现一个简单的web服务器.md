title: 自己实现一个简单的web服务器
tag: java多线程
---

这里涉及网络编程的基本知识以及HTTP协议的基本认识，下面来一步一步实现一下最简单的一个web服务器。

<!--more-->

## 一、用户请求域名到底发送了什么信息来

如果服务端都不知道客户端发来的是什么，那何谈对内容的解析呢？所以我们首先在服务端解析客户端的访问信息，比如我们比较关心的是请求的是什么路径：

![image](http://bloghello.oursnail.cn/thread17-1.png)

此时我们访问地址： localhost:8888 打印出来的结果为：

![image](http://bloghello.oursnail.cn/thread17-2.png)

其实对于我们这种比较简单的实现来说，红框的信息已经足够了。我们只要知道客户端要发来的资源名字是什么，我们根据这个名字取找响应的资源返回给客户端展示即可。由于我其实请求的是根路径，所以是`/`。如果我在这里请求 localhost:8888/index.html 那么就会显示 `GET /index.html HTTP/1.1`这样的信息。不再演示。

但是上面的写法是存在问题的，仅仅是演示而用，因为它会一直等待输入。

那么，不用`while`一直等待，而且只需要第一行即可，那么可以这样写：

![image](http://bloghello.oursnail.cn/thread17-3.png)

好了，不能总之只接收吧，我们服务端要给客户端点什么。

## 二、服务端如何响应资源给客户端展示

首先我们得有资源才能展示，假设我们要展示`index.html`，我们将其暂时放在`F:/webrooot`下。

内容暂时为：

![image](http://bloghello.oursnail.cn/thread17-4.png)

只是展示一下欢迎信息而已。服务端就需要读取这个文件，然后以流的形式发送给客户端的浏览器上，浏览器再解析展示。

![image](http://bloghello.oursnail.cn/thread17-5.png)

此时再去访问页面，就会显示欢迎的信息啦！

![image](http://bloghello.oursnail.cn/thread17-6.png)


这里没有关闭流，也没有关闭`socket`，不过下面都会关闭掉。这个不重要，重要的是，这玩意都是在主线程中做的，显然不如多线程来的快，并且全写在主线程里，肯定是不够好的。下面就用多线程来实现。

## 三、普通的多线程实现方式

![image](http://bloghello.oursnail.cn/thread17-7.png)

就是在这个线程中处理数据和返回数据。

![image](http://bloghello.oursnail.cn/thread17-8.png)


其很简单，但是如果我想显示一张图片呢？

比如我的根目录下有一张图片叫做：`hh.jpg`

![image](http://bloghello.oursnail.cn/thread17-9.png)

很显然现在还是无法展示的，原因是我这里写死了是以`text/html`的形式响应，但是图片正确的响应是`image/jpeg`这种格式。

所以我们不能无脑写死，要进行适当的判断才行。下面贴出代码：

```java
public class ServerThread implements Runnable{
    //根目录
    private static final String webroot = "F:\\webroot\\";

    private Socket client;
    InputStream ins;
    OutputStream out;
    
    //存放类型，比如jpg对应的是image/jpeg，这是http协议规定的每种类型的响应格式
    private static Map<String,String> contentMap =  new HashMap<>();
    static {
        contentMap.put("html","text/html;charset=utf-8");
        contentMap.put("jpg","image/jpeg");
        contentMap.put("png","image/jpeg");
    }

    public ServerThread(Socket client){
        this.client = client;
        init();
    }

    private void init() {
        try {
            ins = client.getInputStream();
            out = client.getOutputStream();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        try {
            go();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void go() throws IOException {
        //获取请求内容
        BufferedReader reader = new BufferedReader(new InputStreamReader(ins));
        //获取到请求的资源名称
        String line = reader.readLine().split(" ")[1].replace("/","\\");
        if(line.equals("\\")){
            line += "index.html";
        }
        System.out.println(line);
        //拼接起来就是资源的完整路径
        File file = new File(webroot + line);
        //判断是否存在，存在则响应内容，不存在则告知不存在
        if (file.exists()) {
            //给用户响应
            PrintWriter pw = new PrintWriter(out);
            InputStream i = new FileInputStream(webroot + line);
            //由于需要将图片也要传给前端，再用这个就不好办了，得用普通的文件输入流
//          BufferedReader fr = new BufferedReader(new InputStreamReader(i));
            pw.println("HTTP/1.1 200 OK");
            //返回的类型是动态判断的，图片用图片的类型，文本用文本的类型
            String s = contentMap.get(line.substring(line.lastIndexOf(".")+1,line.length()));
            System.out.println("返回的类型为："+ s);
            pw.println("Content-Type: " + s);
            pw.println("Content-Length:" + i.available());
            pw.println("Server: hello-server");
            pw.println("Date:"+ new Date());
            pw.println("");
            pw.flush();
            
            //写入输出流中通过PrintWriter发给浏览器
            byte[] buff = new byte[1024];
            int len = 0;
            while ( (len = i.read(buff)) != -1){
                out.write(buff,0,len);
            }
            pw.flush();
            pw.close();
            i.close();
            reader.close();
            client.close();
        }else {
            StringBuffer error = new StringBuffer();
            error.append("HTTP /1.1 400 file not found /r/n");
            error.append("Content-Type:text/html \r\n");
            error.append("Content-Length:20 \r\n").append("\r\n");
            error.append("<h1 >File Not Found..</h1>");
            try {
                out.write(error.toString().getBytes());
                out.flush();
                out.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

ok,你会发现，我还添加了判断资源存在不存在的逻辑，这样显得更加健全一点。

当找得到资源得时候：


![image](http://bloghello.oursnail.cn/thread17-10.png)

当找不到资源得时候：

![image](http://bloghello.oursnail.cn/thread17-11.png)


## 四、线程池方式

其实很简单，线程池的优势以前也说过，不赘述，直接贴一下代码结束。

![image](http://bloghello.oursnail.cn/thread17-12.png)


至此，我们完成了一个比较简单的web服务器的开发。