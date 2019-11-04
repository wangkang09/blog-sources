- Java 通过 java.net 包 实现了 Java 程序的网络通信
- 本文，首先通过实现简单的 Socket 通信实例来讲解 Socket 的原理



#### 1 Socket 服务端

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.ServerSocket;
import java.net.Socket;
public class BIOServer {

    public static void main(String[] args) throws Exception {
        //创建一个服务端 socket，此 socket 监听 8080 端口
        ServerSocket serverSocket = new ServerSocket(8080);
        System.out.println("服务器启动成功");
        //
        while (!serverSocket.isClosed()) {
            // 创建一个新的 socket 
            // 为此 socket 设置 SocksSocketImpl() —— 默认！
            // 为此 SocksSocketImpl 的 address 赋值一个新的 new InetAddress() 对象
            // 为此 SocksSocketImpl 的 fd 赋值一个新的 new FileDescriptor() 对象
            // 调用 SocksSocketImpl 的 accept 方法，最终通过 InetSocketAddress 对象接收连接，并为 SocksSocketImpl 对象 赋值： port/address
            //一直阻塞到，有连接建立为止，此时返回的 request 主要有了 连接的 prot 和 address 属性
            Socket request = serverSocket.accept();
            System.out.println("收到新连接 : " + request.toString());
            try {
                // 创建 socketInputStream = new SocketInputStream(this); 并返回 
                InputStream inputStream = request.getInputStream(); // net + i/o
                BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream, "utf-8"));
                String msg;
                // 读取 客户端传过来的 输入流（这里要做好协议，什么时候，双方断开连接，以免出问题）
                while ((msg = reader.readLine()) != null) { 
                    if (msg.length() == 0) {
                        break;
                    }
                    System.out.println(msg);
                }
                System.out.println("收到数据,来自："+ request.toString());
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                try {
                    request.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        serverSocket.close();
    }
}
```

#### 2 Socket 客户端

```java
import java.io.OutputStream;
import java.net.Socket;
import java.nio.charset.Charset;
import java.util.Scanner;
public class BIOClient {
   private static Charset charset = Charset.forName("UTF-8");

   public static void main(String[] args) throws Exception {
      //建立于服务端的 socket 连接（connect）
      // 为此 socket 设置 SocksSocketImpl() —— 默认！
      // 为此 SocksSocketImpl 的 fd 赋值一个新的 new FileDescriptor() 对象
      // 最终通过 connect0(nativefd, address, port) 建立连接，建立连接后，立马返回（使用 fd 表示）
      Socket s = new Socket("localhost", 8080);
      // 创建 socketOutputStream = new SocketOutputStream(this); 并返回 
      OutputStream out = s.getOutputStream();

      Scanner scanner = new Scanner(System.in);
      System.out.println("请输入：");
      String msg = scanner.nextLine();
      //触发服务端 socket 的 inputstream 读事件 
      //向 输出流中输出数据流
      out.write(msg.getBytes(charset)); 
      scanner.close();
      //这里，只有关闭 socket，服务端才接收成功，是有问题的！！ 
      s.close();
   }

}
```

