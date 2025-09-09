# [首页](/blog/)

> version: **jdk17**

## IO

### InputStream

- read()：返回输入流中下一个字节的数据。返回的值介于 0 到 255 之间。如果未读取任何字节，则代码返回 -1 ，表示文件结束。

- transferTo

    ```java
    public long transferTo(OutputStream out) throws IOException {
        Objects.requireNonNull(out, "out");
        long transferred = 0;
        byte[] buffer = new byte[DEFAULT_BUFFER_SIZE];
        int read;
        while ((read = this.read(buffer, 0, DEFAULT_BUFFER_SIZE)) >= 0) {
            out.write(buffer, 0, read);
            transferred += read;
        }
        return transferred;
    }
    ```

### OutputStream

- write(int b)：将一个字节写入输出流。

- flush()
    刷新此输出流并强制写出所有缓冲的字节，在流关闭前会自动执行一次。大部分的输出流都在内部维护了一个缓冲区，调用write(int b)并不会每次都触发一次I/O操作，而是写入缓冲区内，当缓存区满了之后才会执行一次I/O操作。

### Reader

### Writer

### BufferedInputStream / BufferedOutputStream

装饰器模式实现，为输入流及输出流增加缓冲区，对于write(int b)和read()每次只读取和写入一个字节的操作，性能与read(byte b[])和write(byte b[])相当，实际是读取缓冲区的数据和将数据写入缓冲区，能大幅减少IO次数。

### InputStreamReader / OutputStreamWriter

适配器模式实现，实现字节流和字符流之间的转换。

## Net

### ServerSocket

服务器端需要主动监听端口，执行accpet()等待客户端连接，连接建立后创建Socket对象，通过getOutputStream()向客户端写入数据，通过getInputStream()读取客户端写入的数据。

```java
    public static void main(String[] args) {
        try (var serverSocket = new ServerSocket(10990);
             var socket = serverSocket.accept();
             Scanner sc = new Scanner(System.in);
             var writer = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
             var reader = new BufferedReader(new InputStreamReader(socket.getInputStream()))) {
            writer.write("hello\n");
            writer.flush();
            for (; ; ) {
                // readline需要读取到换行符才能结束
                var readLine = reader.readLine();
                System.out.println(readLine);
                if (readLine.equals("bye")) {
                    break;
                }
                writer.write(sc.nextLine() + "\n");
                writer.flush();
            }
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
```

### Socket

客户端直接创建Socket对象，并主动与服务器端连接，建立连接后，通过getOutputStream()向服务器端写入数据，通过getInputStream()读取服务器端写入的数据。

```java
    public static void main(String[] args) {
        try (var socket = new Socket("127.0.0.1", 10990);
             Scanner sc = new Scanner(System.in);
             var writer = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
             var reader = new BufferedReader(new InputStreamReader(socket.getInputStream()))) {
            for (; ; ) {
                var readLine = reader.readLine();
                System.out.println(readLine);
                var input = sc.nextLine();
                writer.write(input + "\n");
                writer.flush();
                if (input.equals("bye")) {
                    break;
                }
            }
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
```

## HttpClient

Since 11，支持HTTP/1.1和HTTP/2，支持异步。

```java
    public static void main(String[] args) throws Exception {
        // 构建 HttpClient，可重用，线程安全
        HttpClient client = HttpClient.newHttpClient();
        // 构建 HttpRequest
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://www.baidu.com"))
                .header("Content-Type", "application/json")
                .GET()
                .build();
        // 同步发送请求
        // HttpResponse.BodyHandlers.ofString() 指定将响应体转换为 String
        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
        // 处理响应
        System.out.println("Status Code: " + response.statusCode());
        System.out.println("Response Body: " + response.body());
        System.out.println("Headers: " + response.headers().map());
        // 异步发送请求
        var responseCompletableFuture = client.sendAsync(request, HttpResponse.BodyHandlers.ofString());
        // 处理响应
        responseCompletableFuture.thenAccept(aysncResponse -> {
            System.out.println("Status Code: " + aysncResponse.statusCode());
            System.out.println("Response Body: " + aysncResponse.body());
            System.out.println("Headers: " + aysncResponse.headers().map());
        }).join();
    }
```

## File

### RandomAccessFile

- public RandomAccessFile(File file, String mode)

模式支持（"r", "rw", "rws", "rwd"），w模式下如果文件不存在则会自动创建。

- write(int b)

磁盘写入是调用操作系统write指令，该指令不需要flush且无法保证数据写入磁盘，如需强制输盘需要调用sync指令。

```java
file.write("aaa".getBytes());
file.getFD().sync();
```

- public native long length()

获取文件实际字节数

- public void seek(long pos)

设置Position，不设置则默认从0开始，写入数据会进行覆盖。

- public long getFilePointer()

获取当前Position

- public final FileChannel getChannel()

获取NIO文件通道。
