# 手写RPC框架（Java简易版）
这是一个简易的RPC框架，只实现了基本功能，用于学习RPC！


### 1.定义IDL接口和消息体

- 客户端和服务端通过接口描述语言（IDL）就知道要怎么调用接口，供客户端和服务端使用。客户端调用hello方法，服务端实现hello具体。
- 客户端hello是没有具体实现的接口，所以要用反射的动态代理特性实例化一个接口，将调用接口方法“代理”给InvocationHandler中的invoke来执行。

```Java
public interface HelloService {
  HelloResponse hello(HellorRequest request);
}
```

```Java
// 属于rpc协议body里的内容
@Data
@AllArgsConstructor
public class HelloRequest implements Serializable {
    //请求消息
    private String name;
}
```

```Java
// 属于rpc协议body里的内容
@Data
@AllArgsConstructor
public class HelloResponse implements Serializable {
    //响应消息
    private String msg;
}
```

### 2.编写RPC协议（protocol协议层）

header我们用String表示，body用**Java自带序列化**后的byte[]流表示。

```Java
@Data
// Builder类似于setter,链式调用，提供builder()方法
@Builder
// Serializable：对象变成可传输的字节序列
public class RpcRequest implements Serializable {
  // rpc协议头部分（这里仅存放version=1）
  private String header;
  // rpc协议体部分（存放接口名、方法名、参数、参数类型的Java自带序列化后的信息）
  private byte[] body;
}
```

```Java
@Data
@Builder
public class RpcResponse implements Serializable {
  // rpc协议头部分（这里仅存放version=1）
  private String header;
  // rpc协议体部分（存放接口名、方法名、参数、参数类型的Java自带序列化后的信息）
  private byte[] body;
}
```

### 3.编写序列化（codec层）

编解码层：将接口名、方法名、参数、参数类型（包括IDL）放进RpcRequestBody对象中，再序列化后放到protocol层的RpcRequest的body字段。

```Java
@Data
@Builder
// 调用编码
public class RpcRequestBody implements Serializable {
  private String interfaceName;
  private String methodName;
  private Object[] parameters;
  private Class<?>[] paramTypes;
}
```

```Java
@Data
@Builder
// 返回编码
public class RpcResponseBody implements Serializable {
  // 服务端返回对象
  private Object retObject;
}
```

### 4.1编写客户端（动态代理）

- 使用Java反射的动态代理特性，调用一个没有具体实现的接口，实例化该接口，将调用方法代理给`InvocationHandler`中的`invoke`来执行，在`invoke`中获取到接口名、方法名、参数、参数类型，包装成RPC协议，发送给服务端，然后同步等待服务端返回。

```Java
// 客户端使用
public class RpcClientProxy implements InvocationHandler {
  // 通过代理获得Service,实例化一个接口
  @SuppressWarnings("unchecked")
  public <T> T getService(Class<T> clazz) {
    // 参数：1.loader 2.interfaces 3.InvocationHandler
    return (T) Proxy.newProxyInstance(clazz.getClassLoader(), new Class<?>[]{clazz}, this);
  }
  
  // 代理给invoke方法执行客户端Service
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // **1、拿到HelloService接口中的接口名、方法名、参数、参数类型，包装成rpcRequestBody**
    // **把它写入body的字节流，将调用所需信息编码成bytes[]，即有了调用编码（序列化）【codec层】**
    RpcRequestBody rpcRequestBody = RpcRequestBody.builder()
             // 反射：通过方法获取该方法对应的接口类名
             .interfaceName(method.getDeclaringClass().getName())
             .methodName(method.getName())
             .paramTypes(method.getParameterTypes())
             .paramTypes(args)
             .build();
    
    // 字节数组输出流在内存中创建一个字节数组缓冲区
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    // 将对象转化成相应的字节序列输出到baos流（参数为相接的流）
    ObjectOutputStream oos = new ObjectOutputStream(baos);
    // **序列化**(rpcRequestBody需要实现Serializable接口)
    oos.writeObject(rpcRequestBody);
    // 将写入的流放到字节数组
    byte[] bytes = baos.toByteArray();
    
    // **2、创建RPC协议，将Header、Body的内容设置好（Body中存放调用编码上面编码的bytes流）【protocol层】**
    RpcRequest rpcRequest = RpcRequest.buider()
            .header("version=1")
            .body(bytes)
            .build();
            
    // **3、发送RpcRequest协议，获得RpcResponse(这里的网络传输使用的socket)【网络传输层】
    **RpcClientTransfer rpcClient = new RpcClientTransfer();
    // 同步等待
    RpcResponse rpcResponse = rpcClient.sendRequest(rpcRequest);
    
    // **4、解析RpcResponse，也就是在解析rpc协议【protocol层】**
    String header = rpcResponse.getHeader();
    byte[] body = rpcResponse.getBody();
    // 判断是否传输的是同一个协议（这里写死的）
    if(header.equals("version=1")){
      ByteArrayInputStream bais = new ByteArrayInputStream(body);
      ObjectInputStream ois = new ObjectInputStream(bais);
      // 将服务端传来的RpcResponse的body中的返回编码，解码成我们需要的对象Object并返回**（反序列化）**
      RpcResponseBody rpcResponseBody = (RpcResponseBody) ois.readObject();
      Object retObject = rpcResponseBody.getRetObject();
      return retObject;
    }
    return null; 
  }
}
```

### 4.2编写客户端（网络传输）

- 使用socket进行网络传输（这里没有服务发现，是写死的），可以换成其他网络传输。

```Java
public class RpcClientTransfer {
  public RpcResponse sendRequest(RpcRequest rpcRequest) {
    // 发送【transfer层】
    try (Socket socket = new Socket("loaclhost", 9000)) {
      // 输出流
      ObjectOutputStream objectOutputStream = new ObjectOutputStream(socket.getOutputStram());
      // 输入流
      ObjectInputStream objectInputStream = new ObjectInputStream(socket.getInputStream());
      // 输出rpcRequest协议
      objectOutputStream.writeObject(rpcRequest);
      objectOutputStream.flush();
      
      // 输入rpcResponse协议中的对象,并返回给客户端
      RpcResponse rpcResponse = (RpcResponse) objectInputStream.readObject();
      return rpcResponse;
    } catch (IOException | ClassNotFoundException e) {
      e.printStackTrace();
      return null;
    }
  }
}
```

### 5.1编写服务端（启动连接注册服务）

充当注册中心的功能，将端口号和对应的service注册到registeredService这个Map集合中。

```Java
public class RpcServer {
  private final ExecutorService threadPool;
  
  private final HashMap<String, Object> registeredService;
  
  // 构造函数，服务器启动时初始化
  public RpcServer() {
    // 线程池
    int corePoolSize = 5;
    int maxinumPoolSize = 50;
    long keepAliveTime = 60;
    BlockingQueue<Runnable> workingQueue = new ArrayBlockingQueue<>(100);
    ThreadFactory threadFactory = Executors.defaultThreadFactory();
    this.threadPool = new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, TimeUnit.SECONDS, workingQueue, threadFactory);
    this.registeredService = new HashMap<String, Object>();
  }
  
  // 参数service就是service接口的具体实现的service
  // 对外提供注册方法（把HelloService里的接口名和这个对应的service拿到存入Map【反射】）
  public void register(Object service) {
    // 利用动态反射机制获得service对应的接口名（第一个接口的名字：hello）
    registeredService.put(service.getClass().getInterfaces()[0].getName(), service);
  }
  
  public void serve(int port) {
    try (ServerSocket serverSocket = new ServerSocket(port)) {
      System.out.println("服务器启动");
      Socket handleSocket;
      // 当与客户端建立连接时返回一个Socket对象
      while((handleSocket = serverSocket.accept()) != null) {
        System.out.println("客户端连接，ip:" + handleSocket.getInetAddress());
        threadPool.execute(new RpcServerWorker(handleSocket, registeredService));
      }
     } catch (IOException e) {
        e.printStackTrace();
     }
}
```

### 5.2编写服务端（反射调用）

```Java
public class RpcServerWorker implements Runnable {
  private Socket socket;
  
  private HashMap<String, Object> registeredService;
  
  public RpcServerWorker(Socket socket, HashMap<String, Object> registeredService) {
    this.socket = socket;
    this.registeredService = registeredService; 
  }
  
  // 开启一个线程来执行
  public void run() {
    try {
       // 输入流（客户端向服务端输入）
       ObjectInputStream objectInputStream = new ObjectInputStream(socket.getInputStream());
       // 输出流（服务端向客户端输出）
       ObjectOutputStream objectOutputStream = new ObjectOutputStream(socket.getOutputStream());
       
       // **1、网络传输层获取到RpcRequest消息【transfer层】**
       RpcRequest rpcRequest = (RpcRequest) objectInputStream.readObject();
       
       // **2、解析版本号，并判断【protocol层】**
       if(rpcRequest.getHeader().equals("version=1")) {
         // **3、将rpcRequest中的body部分解码出来变成RpcRequestBody【codec层】**
         byte[] body = rpcRequest.getBody();
         ByteArrayInputStream bais = new ByteArrayInputStream(body);
         ObjectInputStream ois = new ObjectInputStream(bais);
         // 服务端将客户端传来的编码**反序列化**
         RpcRequestBody rpcRequestBody = (RpcRequestBody) ois.readObject();
         
         // 调用服务（类似于从注册中心拿到对应服务，并通过反射拿到该服务需调用的方法）
         Object service = registeredService.get(rpcRequestBody.getInterfaceName());
         Method method = service.getClass().getMethod(rpcRequestBody.getMethodName(), rpcRequestBody.getParamTypes());
         // invoke反射的方法调用本地的服务
         Object returnObject = method.invoke(service, rpcRequestBody.getParameters());
         
         // 1、**服务端序列化，将返回的Object编码成bytes[]即变成了返回编码,放入rpcResponse的body里面【codec层】**
         RpcResponseBody rpcResponseBody = RpcResponseBody.builder()
                     .retObject(returnObject)
                     .build();
         ByteArrayOutputStream baos = new ByteArrayOutputStream();
         ObjectOutputStream oos = new ObjectOutputStream(baos);
         oos.writeObject(rpcResponseBody);
         byte[] bytes = baos.toByteArray();
         
         //** 2、将返回编码作为body，加上header，生成RpcResponse协议（对比http协议里的header和body）【protocol层】**
         RpcResponse rpcResponse = RpcResponse.builder()
                     .header("version=1")
                     .body(bytes)
                     .build();
         // **3、将rpcResponse发送给客户端【transfer层】**
         objectOutputStream.writeObject(rpcResponse);
         objectOutputStream.flush();  
        }
    } catch (IOException | ClassNotFoundException | NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
        e.printStackTrace();
    }
  }
}
```

### 6.通过以上RPC框架进行测试调用

#### 6.1服务端接口的具体实现

```Java
public class HelloServiceImpl implements HelloService {
  public HelloResponse hello(HelloRequest request) {
    // 拿到HelloRequest参数里的name方法
    String name = request.getName();
    String retMsg = "hello:" + name;
    // 把上面拼接出来的字符串放入HelloResponse里的msg，包装起来最后返回
    HelloResponse response = new HelloResponse(retMsg);
    return response;
  }
}
```

#### 6.2服务端测试类

- 启动rpc服务器
- 将服务注册
- 开启服务端口

```Java
public class TestServer {
  public static void main(String[] args) {
    RpcServer rpcServer = new RpcServer();// 真正的rpc server
    HelloService helloService = new HelloServiceImpl();// 包含需要处理的方法的对象
    rpcServer.register(helloService);// 向rpc server注册对象里面的所有方法
    rpcServer.serve(9000);
  }
}
```

#### 6.3客户端调用测试类

```Java
public class TestClient {
  public static void main(Stringp[] args) {
    // 通过动态代理获取RpcService
    RpcClientProxy proxy = new RpcClientProxy();
    HelloService helloService = proxy.getService(HelloService.class);
    // 构造出请求对象HelloRequest
    HelloRequest helloRequest = new HelloRequest("Nevermore");
    // rpc调用并返回结果对象HelloResponse（本地是没有实现hello方法的）
    HelloResponse helloResponse = helloService.hello(helloRequest);
    // 从HelloResponse中获取msg
    String helloMsg = helloResponse.getMsg();
    System.out.println(helloMsg);
  }
}
```
