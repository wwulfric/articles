大纲：

stupidBio（服务端，客户端）；c 语言实现对照，查看源码；流输入输出和c的对比

socket 编程的 jdk 实现 

类比 C 语言的服务端和客户端代码，画图

http://c.biancheng.net/cpp/html/3030.html
https://www.runoob.com/java/net-serversocket-socket.html
https://blog.csdn.net/defonds/article/details/7971259

java socket bio 最简实现：每次只能接受一个连接，阻塞

```java
public class Test {
    public static void main(String[] args) throws IOException {
        // 监听 8888 端口
        ServerSocket serverSocket = new ServerSocket(8888);
        while (true) {
            // 一旦有堵塞, 则表示服务器与客户端获得了连接
            Socket clientSocket = serverSocket.accept();
            // 处理这次连接
            InputStream inputStream = clientSocket.getInputStream();
            // 读 4 个字节，输出
            byte[] bytes = new byte[4];
            inputStream.read(bytes);
            System.out.println(Arrays.toString(bytes));
            // 再读 4 个字节，输出
            inputStream.read(bytes);
            System.out.println(Arrays.toString(bytes));
            OutputStream outputStream = clientSocket.getOutputStream();
            // 输出
            outputStream.write("OK".getBytes());
        }
    }
}
```

连接未释放导致的泄漏：

- 通过 `telnet localhost 8888` 建立连接，并保持
- 通过 `telnet localhost 8888` 建立连接，并退出 telnet 进程
- 通过 `telnet localhost 8888` 建立连接，并退出 telnet 进程

查看连接情况：`netstat -an | grep 8888`: 结果如下图所示，有一个连接是`ESTABLISHED`在线状态，另外两个客户端退出了，但是服务端没有处理结束的连接，这两个连接一直没有释放。至于为什么未释放的状态是`CLOSE_WAIT`和`FIN_WAIT_2`，可以参考下 TCP 的四次挥手过程。

```bash
tcp6       6      0  ::1.8888               ::1.51023              CLOSE_WAIT
tcp6       0      0  ::1.51023              ::1.8888               FIN_WAIT_2
tcp6       6      0  ::1.8888               ::1.50995              ESTABLISHED
tcp6       0      0  ::1.50995              ::1.8888               ESTABLISHED
tcp6      10      0  ::1.8888               ::1.50986              CLOSE_WAIT
tcp6       0      0  ::1.50986              ::1.8888               FIN_WAIT_2
tcp46      0      0  *.8888                 *.*                    LISTEN
```



介绍如何创建 socket，如何 bind，如何 listen，如何 accept，如何流写入和读取

对流的一些介绍

bio 的多线程模型(用线程模拟通道)

bio socket 的输入输出流

FileInputStream#read(int) 方法最终调用的是 JNI 的 readBytes 方法，如下图所示。

```c
static jdwpError
readBytes(PacketInputStream *stream, void *dest, int size)
{
    // 错误处理
    if (stream->error) {
        return stream->error;
    }
    // 错误处理
    if (size > stream->left) {
        stream->error = JDWP_ERROR(INTERNAL);
        return stream->error;
    }
    // socket 接收的内容复制到 dest
    if (dest) {
        (void)memcpy(dest, stream->current, size);
    }
    // 更新指针
    stream->current += size;
    stream->left -= size;

    return stream->error;
}
```



