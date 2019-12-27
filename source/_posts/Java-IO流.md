---
title: Java-IO流
abbrlink: 3845166172
date: 2019-12-25 16:15:51
categories:
  - Java
  - Basic
tags:
  - Io
  - Java
author: 长歌
---

{% cq %}
**Io流**,用于处理设备上的数据。在流中一般以字节的形式存放着数据。
**File**,Java将文件和文件夹封装为对象
{% endcq %}
<!-- More -->

# 文件
## 文件介绍
Java将系统中的文件和文件夹封装成了对象。提供了属性以及方法对这些文件和文件夹进行操作。

## File类
### 创建
```java
    /**
     * 在指定目录下创建文件
     * 文件上级目录必须存在,文件已存在则不创建.
     * 流对象操作时,文件已存在,会覆盖,除非使用续写模式
     *
     * @param filePath 文件路径
     * @return 是否创建成功
     * @throws IOException IO异常
     */
    public boolean createNewFile(String filePath) throws IOException {
        File file = new File(filePath);
        return file.createNewFile();
    }

    /**
     * 创建文件夹
     * 上级目录需存在,文件夹存在时不会创建
     *
     * @param filePath 文件路径
     * @return 是否创建成功
     */
    public boolean mkdir(String filePath) {
        File file = new File(filePath);
        return file.mkdir();
    }

    /**
     * 创建多级目录
     *
     * @param filePath 文件路径
     * @return 是否创建成功
     */
    public boolean mkdirs(String filePath) {
        File file = new File(filePath);
        return file.mkdirs();
    }
```

### 删除
```java
    /**
     * 删除文件/目录
     * 文件/目录不存在返回<code>false</code>,若删除目录下存在文件/目录,不进行删除,返回<code>false</code>
     *
     * @param filePath 文件路径
     */
    public void delete(String filePath) {
        File file = new File(filePath);

        boolean isDeleted = file.delete();
        System.out.println("删除文件" + filePath + "是否成功 : " + isDeleted);
    }

    /**
     * 虚拟机退出后删除文件/目录
     * 文件/目录不存在返回<code>false</code>,若删除目录下存在文件/目录,不进行删除,返回<code>false</code>
     *
     * @param filePath 文件路径
     */
    public void deleteOnExit(String filePath) {
        File file = new File(filePath);
        try {
            file.deleteOnExit();
            System.out.println("设置退出虚拟机后删除文件" + filePath);
            Thread.sleep(3000); // 3s
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

### 获取
```java
    /**
     * 文件获取方法
     *
     * @param filePath 文件路径
     */
    public void getProperties(String filePath) {
        File file = new File(filePath);
        long length = file.length();    // 文件大小
        String name = file.getName();   // 文件/目录 名称
        String path = file.getPath();   // 路径字符串
        String absolutePath = file.getAbsolutePath();   // 绝对路径字符串
        String parent = file.getParent();   // 获取上级路径字符串表示，若无，返回null
        long lastModified = file.lastModified();    // 返回文件/目录 最后一次修改的时间
        String pathSeparator = File.pathSeparator;  // 当前系统默认的路径分隔符,window默认为";"
        String separator = File.separator;  // 当前系统默认的目录分隔符,window默认为"\"
        System.out.println(String.format("文件大小=%s,文件名称=%s,文件路径=%s,文件绝对路径=%s,文件上级路径=%s,文件最后修改时间=%s,路径分隔符=%s,目录分隔符=%s",
                length, name, path, absolutePath, parent,String.valueOf(lastModified),pathSeparator,separator));
    }
```

### 判断
```java
    /**
     * 文件相关判断方法
     *
     * @param filePath 文件路径
     */
    public void judge(String filePath){
        File file = new File(filePath);
        boolean isExists = file.exists();
        boolean isDirectory = file.isDirectory();
        boolean isFile = file.isFile();
        boolean isHidden = file.isHidden();
        boolean isAbsolute = file.isAbsolute();
        System.out.println(String.format("文件是否存在=%s,是目录=%s,是文件=%s,是否是隐藏文件=%s,是否是绝对路径=%s",
                isExists,isDirectory,isFile,isHidden,isAbsolute));
    }
```

### 重命名
```java
    /**
     * 重命名
     * 上级路径不同可以实现文件移动,上级路径相同时实现重命名
     *
     * @param srcPath 源文件路径
     * @param destPath 目标文件路径
     */
    public void renameTo(String srcPath,String destPath){
        File srcFile = new File(srcPath);
        File destFile = new File(destPath);
        boolean isSuccess = srcFile.renameTo(destFile);
        System.out.println("文件重命名是否成功 : "+isSuccess);
    }
```

# IO流
## IO流介绍
### 流的分类
- 按操作分为输入流、输出流
- 按类型分为字节流、字符流

### 基类
> 流的功能向上抽象，共包含4个基类，且都是抽象类
- 字节流 InputStream OutputStream
- 字符流 Reader Writer
- 对应子类的后缀为其基类名称
- 结构如下:
{% asset_img Java_IO.png Java-IO结构 %}
{% asset_img Java_IO_Detail.png Java-IO结构图，含方法 %}

## 字符流
### Reader 和 Writer
{% asset_img Reader.png Reader的UML图 %}
{% asset_img Writer.png Writer的UML图 %}
### FileWriter 和 FileReader
```java
    /**
     * FileWriter 继承自 OutputStreamWriter 继承自 Writer <br/>
     * 支持通过String 类型路径、File 类以及 FileDescriptor 进行初始化<br/>
     * 支持使用父类OutputStreamWriter及Writer的write()方法输出
     * 
     * @param filePath 文件路径
     * @param content  要输出文件内容
     */
    public static void writeByFileWriter(String filePath, String content) {
        FileWriter fw = null;
        try {
            fw = new FileWriter(filePath);
//            fw = new FileWriter(new File(filePath));
            fw.write(content);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (fw != null) {
                try {
                    fw.close();
                } catch (Exception ex) {
                    System.out.println("Exception when close FileWriter" + ex);
                }
            }
        }
    }

    /**
     * FileReader 继承自 InputStreamReader 继承自 Reader <br/>
     * 支持通过String 类型路径、File 类以及 FileDescriptor 进行初始化<br/>
     * 支持使用父类的 read():int 及read(char[]):int 方法进行读取
     *
     * @param filePath 文件路径
     * @return 文件内容
     */
    public static String readByFileReader(String filePath) {
        char[] cbuf = new char[1024];
        FileReader fr = null;
        StringBuilder sb = new StringBuilder();
        try {
            fr = new FileReader(filePath);  // 支持 String 类型路径、File 类以及 FileDescriptor初始化
//            fr = new FileReader(new File(filePath));
            while (fr.read(cbuf) != -1) {
                sb.append(String.valueOf(cbuf));
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (fr != null) {
                try {
                    fr.close();
                } catch (Exception ex) {
                    System.out.println("Exception when close FileReader " + ex);
                }
            }
        }
        return sb.toString();
    }
```

### BufferedWriter、BufferedReader
> 通过缓冲区实现的高效输出及输入操作

```java
    /**
     * BufferedWriter 使用缓冲区实现的高效输出流
     * 继承自Writer.
     * @param filePath 文件路径
     * @param content 输出内容
     */
    public void writeByBufferedWriter(String filePath,String content){
        FileWriter fw = null;
        BufferedWriter bw = null;
        try{
            fw = new FileWriter(filePath);
            bw = new BufferedWriter(fw);
            bw.write(content);
            bw.newLine();   // 输出换行
            bw.flush();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            if (fw != null) {
                try{
                    fw.close();
                }catch (Exception ex){
                    System.out.println("execption when closing FileWriter");
                }
            }
            if(bw!=null){
                try{
                    bw.close();
                }catch (Exception ey){
                    System.out.println("execption when closing BufferedWriter");
                }
            }
        }
    }

    /**
     * BufferedReader 使用缓冲区实现的高效输入流
     * 继承自 Reader
     * 可以使用readLine():String 按行读取
     *
     * @param filePath 文件路径
     * @return 文件内容
     */
    public String readByBufferedReader(String filePath) {
        StringBuilder sb = new StringBuilder();
        FileReader fr = null;
        BufferedReader br = null;
        try {
            fr = new FileReader(filePath);
            br = new BufferedReader(fr);
            String line;
            while ((line = br.readLine())!=null){
                sb.append(line);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return sb.toString();
    }
```

### OutputStreamWriter 和 InputStreamReader
> 转换流，可以将字节流转换为字符流并指定编码

```java
    /**
     * OutputStreamWriter 转换流，可以转换字节流->字符流并指定编码
     * 继承自Writer，支持write()方法
     *
     * @param filePath 文件路径
     * @param content 输出内容
     */
    public void writeByOutputStreamWriter(String filePath, String content) {
        File file;
        FileOutputStream fos = null;
        OutputStreamWriter osw = null;
        try {
            file = new File(filePath);
            fos = new FileOutputStream(file);
            osw = new OutputStreamWriter(fos, "utf8");
            osw.write(content);
            osw.flush();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (fos != null) {
                try {
                    fos.close();
                } catch (Exception ex) {
                    System.out.println("exception when closing FileOutputStream");
                }
            }

            if (osw != null) {
                try{
                    osw.close();
                }catch(Exception ey){
                    System.out.println("exception when closing OutputStreamWriter");
                }
            }
        }
    }

    /**
     * InputStreamReader 转换流，可以转换字节流->字符流并指定编码
     * 继承自Reader, 使用read(char[] cbuf)方法
     *
     * @param filePath 文件路径
     * @return 文件内容
     */
    public String readByInputStreamReader(String filePath) {
        StringBuilder sb = new StringBuilder();
        FileInputStream fis = null;
        InputStreamReader isr = null;
        try {
            fis = new FileInputStream(filePath);
            isr = new InputStreamReader(fis, "utf8");
            char[] cbuf = new char[1024];
            int len;
            while ((len = isr.read(cbuf)) != -1) {
                sb.append(new String(cbuf, 0, len));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }finally {
            if (fis != null) {
                try{
                    fis.close();
                }catch(Exception ex){
                    System.out.println("exception when closing FileInputStream");
                }
            }

            if (isr != null) {
                try{
                    isr.close();
                }catch (Exception ey){
                    System.out.println("exception when closing InputStreamReader");
                }
            }
        }
        return sb.toString();
    }
```

### PrintWriter
> 字符输出流，提供更多功能，打印方法可以打印任意类型数据。有自动刷新机制。使用本机默认编码


```java
    /**
     * PrintWirter 格式化输出流
     * 继承自Writer 支持包装String,File,OutputStream,Writer
     * 除Writer支持方法外，还支持各种println、print以及format方法进行格式化输出
     *
     * @param filePath 文件路径
     */
    public void writeByPrintWriter(String filePath){
        FileWriter fw = null;
        PrintWriter pw = null;
        try{
            fw = new FileWriter(filePath);
            pw = new PrintWriter(filePath);
            pw.println(true);
            pw.print('a');
            pw.print(3);
            pw.println(2.1);
            pw.println("字符串");
            pw.format("%s--%d\n","test",2);
        }catch (Exception e){
            e.printStackTrace();
        }finally{
            if (fw != null) {
                try{
                    fw.close();
                }catch(Exception ex){
                    System.out.println("exception when closing FileWriter");
                }
            }

            if (pw != null) {
                try{
                    pw.close();
                }catch (Exception ey){
                    System.out.println("exception when closing PrintWriter");
                }
            }
        }
    }
```

### CharArrayWriter
> 内存流，底层使用char[]保存数据，在我看来是一个工具类，处理好数据后通过writeTo(Writer)方法借由其他Writer输出。

```java
    /**
     * CharArrayWriter 字符数组输出流.
     * 底层使用char[] 保存内容,可以调用write(CharSequence)进行输出
     * 支持通过writeTo(Writer)讲内容输出到其他输出流(CharArrayWriter或者其他Writer)
     *
     * @param filePath 文件路径
     */
    public void writerByCharArrayWriter(String filePath){
        FileWriter fw = null;
        CharArrayWriter caw = new CharArrayWriter();    // 只操作内存,内存流的close()无效,关闭还可以正常使用
        try {
            caw.write(1);
            caw.write("字符串");
            caw.write(new char[]{'1','a','\r','\n'});
//            CharArrayWriter caw2 = new CharArrayWriter();
//            caw.writeTo(caw2);
            fw = new FileWriter(filePath);
            caw.writeTo(fw);
        }catch(Exception e){
            e.printStackTrace();
        }finally{
            if (fw != null) {
                try{
                    fw.close();
                }catch (Exception ex){
                    System.out.println("exception when closing FileWriter");
                }
            }
        }
    }
```

### StringWriter
> 内存流，提供字符串的操作方法，通过toString()方法返回收集到的数据。作用同CharArraySequence类似。区别是底层保存数据不同，char[] 可以直接通过Writer输出。所以StringWriter没有 writeTo(Writer) 方法

```java
    /**
     * StringWriter 字符串输出流
     * 底层使用 StringBuffer 保存数据, 需要数据时,通过 toString() 方法返回收集的数据
     * 支持 append 以及 write 方法收集数据
     *
     * @param filePath 文件路径
     */
    public void writeByStringWriter(String filePath){
        BufferedWriter bw = null;
        StringWriter sw = new StringWriter();
        try{
            sw.write("字符串");
            sw.append('c');
            sw.write(1);
            sw.write("123456",1,3);
            sw.append("abced",0,3);     // append(CharSequence)
            bw = new BufferedWriter(new FileWriter(filePath));
            bw.write(sw.toString());    // 实际使用时,通过toString方法返回收集到的数据
        }catch (Exception e){
            e.printStackTrace();
        }finally{
            if (bw != null) {
                try{
                    bw.close();
                }catch (Exception ex){
                    System.out.println("exception when closing BufferedWriter");
                }
            }
        }
    }
```

## 字节流
> 相比较与字符流，字节流都是操作byte[]的操作，相对来说使用的很少。更多的是将字节流转换为字符流，并通过字符流的各个类进行包装使用。

### InputStream 和 OutputStream
{% asset_img InputStream.png InputStream的UML图 %}
{% asset_img OutputStream.png OutputStream的UML图 %}

### FileOutputStream 和 FileInputStream
```java
    /**
     * FileOutputStream 字节输出流
     *
     * @param filePath 文件路径
     */
    public void writeByFileOutputStream(String filePath){
        File file;
        FileOutputStream fos;

        try{
            file = new File(filePath);
            // 需要对文件进行操作时可以先实例化File类，只需要进行流操作则不用
            fos = new FileOutputStream(file);
//            fos = new FileOutputStream(filePath);
            fos.write(1);
            fos.write(new byte[]{97,98,13,10}); // 13 10 表示 \r\n
        }catch(Exception e){
            e.printStackTrace();
        }
    }

    /**
     * FileInputSream 字节输入流
     *
     * @param filePath 文件路径
     */
    public void readByFileInputStream(String filePath){
        File file;
        FileInputStream fis = null;
        try{
            file = new File(filePath);
            fis = new FileInputStream(file);
            byte[] bytes = new byte[1024];
            int len;
            while((len = fis.read(bytes)) != -1){
                for (byte aByte : bytes) {
                    System.out.println(aByte);
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }finally{
            if (fis != null) {
                try{
                    fis.close();
                }catch (Exception ex){
                    System.out.println("exception when close FileInputSream");
                }
            }
        }
    }
```

###  BufferedOutputStream 和 BufferedInputStream
```java
    /**
     * 字节输出流,使用缓冲区的高效输出流
     *
     * @param filePath 文件路径
     */
    public void writeByBufferedOutputStream(String filePath){
        FileOutputStream fos = null;
        BufferedOutputStream bos = null;
        try{
            fos = new FileOutputStream(filePath);
            bos = new BufferedOutputStream(fos);
            bos.write(new byte[]{99,100,13,10});
            bos.flush();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            if (fos != null) {
                try{
                    fos.close();
                }catch(Exception ex){
                    System.out.println("exception when closing FileOutputStream");
                }
            }

            if (bos != null) {
                try{
                    bos.close();
                }catch (Exception ey){
                    System.out.println("exception when closing BufferedOutputStream");
                }
            }
        }
    }

    /**
     * 字节输入流，使用缓冲区的高效输人流
     *
     * @param filePath 文件路径
     */
    public void readByBufferedInputStream(String filePath){
        FileInputStream fis = null;
        BufferedInputStream bis = null;
        try{
            fis = new FileInputStream(filePath);
            bis = new BufferedInputStream(fis);
            byte[] bytes = new byte[1024];
            int len;
            while((len = bis.read(bytes))!=-1){
                for (byte aByte : bytes) {
                    System.out.println(aByte);
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            if (fis != null) {
                try{
                    fis.close();
                }catch (Exception ex){
                    System.out.println("exception when closing FileInputStream");
                }
            }
            if (bis != null) {
                try{
                    bis.close();
                }catch (Exception ey){
                    System.out.println("exception when closing BufferedInputStream");
                }
            }
        }
    }
```

### PrintStream

> 字节打印流，提供更多功能，打印方法可以打印任意类型数据。有自动刷新机制。使用本机默认编码

```java
    /**
     * 字节打印流，提供更多功能，打印方法可以打印任意类型数据。有自动刷新机制。使用本机默认编码
     * 可以包装 File、字符串路径、OutputStream
     * 
     * @param filePath 文件路径
     */
    public void writeByPrintStream(String filePath){
        FileOutputStream fos = null;
        PrintStream ps = null;
        try{
            fos = new FileOutputStream(filePath);
            ps = new PrintStream(fos);
            ps.print(1);
            ps.println("test");
        }catch(Exception e){
            e.printStackTrace();
        }finally {
            if (fos != null) {
                try {
                    fos.close();
                }catch (Exception ex){
                    System.out.println("exception when closing FileOutputStream");
                }
            }

            if (ps != null) {
                try{
                    ps.close();
                }catch(Exception ey){
                    System.out.println("exception when closing Print");
                }
            }
        }
    }
```

### ObjectOutputStream 和 ObjectInputStream
```java
    /**
     * 对象输出字节流,可以序列化对象
     *
     * @param filePath 文件路径
     */
    public void writeByObjectOutputStream(String filePath){
        FileOutputStream fos = null;
        ObjectOutputStream oos = null;
        try{
            fos = new FileOutputStream(filePath);
            oos = new ObjectOutputStream(fos);
            oos.writeObject(new Date());
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            if (fos != null) {
                try{
                    fos.close();
                }catch(Exception ex){
                    System.out.println("exception when closing FileOutputStream");
                }
            }
            if (oos != null) {
                try{
                    oos.close();
                }catch(Exception ex){
                    System.out.println("exception when closing ObjectOutputStream");
                }
            }
        }
    }

    /**
     *对象输入字节流,可以读取序列化对象
     *
     * @param filePath 文件路径
     * @return 文件内容
     */
    public String readByObjectInputStream(String filePath) {
        Date date = new Date();
        FileInputStream fis = null;
        ObjectInputStream ois = null;
        try {
            fis = new FileInputStream(filePath);
            ois = new ObjectInputStream(fis);
            date = (Date) ois.readObject();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (fis != null) {
                try {
                    fis.close();
                } catch (Exception ex) {
                    System.out.println("exception when closing FileInputStream");
                }
            }

            if (ois != null) {
                try {
                    ois.close();
                } catch (Exception ey) {
                    System.out.println("exception when closing ObjectInputStreamm");
                }
            }
        }
        return new SimpleDateFormat("yyyy-MM-dd").format(date).toString();
    }
```

### PipedOutputStream 和  PipedInputStream
```java
public class PipedStreamTest {

    class Receiver implements Runnable{

        private PipedInputStream in = new PipedInputStream();

        public PipedInputStream getInputStream() {
            return in;
        }

        public void readMessageOnce(){
            // 虽然buf的大小是2048个字节，但最多只会从“管道输入流”中读取1024个字节。
            // 因为，“管道输入流”的缓冲区大小默认只有1024个字节。
            byte[] buf = new byte[2048];
            try {
                int len = in.read(buf);
                System.out.println(new String(buf,0,len));
                in.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        public void readMessageContinued() {
            int total=0;
            while(true) {
                byte[] buf = new byte[1024];
                try {
                    int len = in.read(buf);
                    total += len;
                    System.out.println(new String(buf,0,len));
                    // 若读取的字节总数>1024，则退出循环。
                    if (total > 1024)
                        break;
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }

            try {
                in.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        @Override
        public void run() {
            readMessageOnce();
        }
    }

    class Sender extends Thread {

        // 管道输出流对象。
        // 它和“管道输入流(PipedInputStream)”对象绑定，
        // 从而可以将数据发送给“管道输入流”的数据，然后用户可以从“管道输入流”读取数据。
        private PipedOutputStream out = new PipedOutputStream();

        // 获得“管道输出流”对象
        public PipedOutputStream getOutputStream() {
            return out;
        }

        @Override
        public void run() {
            writeShortMessage();
            //writeLongMessage();
        }

        // 向“管道输出流”中写入一则较简短的消息："this is a short message"
        private void writeShortMessage() {
            String strInfo = "this is a short message";
            try {
                out.write(strInfo.getBytes());
                out.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        // 向“管道输出流”中写入一则较长的消息
        private void writeLongMessage() {
            StringBuilder sb = new StringBuilder();
            // 通过for循环写入1020个字节
            for (int i = 0; i < 102; i++)
                sb.append("0123456789");
            // 再写入26个字节。
            sb.append("abcdefghijklmnopqrstuvwxyz");
            // str的总长度是1020+26=1046个字节
            String str = sb.toString();
            try {
                // 将1046个字节写入到“管道输出流”中
                out.write(str.getBytes());
                out.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 多个线程间可以通过　PipedInputStream 和 PipedOutputStream 完成通信
     * 输入流同输出流绑定，一次最多传送1024字节信息
     */
    public void testPipedStream(){
        Sender sender = new Sender();

        Receiver receiver = new Receiver();

        PipedOutputStream out = sender.getOutputStream();

        PipedInputStream in = receiver.getInputStream();

        try {
            //out.connect(in);
            in.connect(out);
            sender.start();
            new Thread(receiver).start();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 流操作规律
### 明确目的
- 数据源：　需要读取，可以使用两类流对象：`InputStream`、`Reader`
- 数据汇：　需要写入，可以使用两类流对象：`OutputStream`、`Writer`

### 是否纯文本
- 如果是
    + 数据源：`Reader`
    + 数据汇: `Writer`
- 如果不是
    + 数据源: `InputStream`
    + 数据汇: `OutputStream`

### 明确操作的数据设备
- 硬盘 : `File`
- 内存 : `数组`
- 键盘/控制台：　`System.in/System.out`

### 是否需要在基础操作上附加其他功能
如果需要就使用其他流对象进行装饰，以提高效率或者降低编码难度。

### 读取键盘录入

```java
BufferedReader bufr = new BufferedReader(new InputStreamReader(System.in));	// 源: 键盘输入
```

