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
