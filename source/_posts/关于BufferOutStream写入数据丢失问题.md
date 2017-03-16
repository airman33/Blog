---
title: 关于BufferedOutputStream写入数据丢失
tags: 'BufferedOutputStream'
date: 2017-03-16 15:25:14
categories: Java IO
---
### 问题
使用封装了FileOutputStream的BufferedOutputStream进行write操作，当字节数组长度小于一定值时，写入文件失败。代码如下：

	byte[] retByt;
	………………
	File file = new File("test.txt");
	FileOutputStream fileOutput = new FileOutputStream(file);
	BufferedOutputStream bufferedOutput = new BufferedOutputStream(fileOutput);
	bufferedOutput.write(retByt);
<!--more-->
### 溯源
    Class FilterOutputStream extends OutputStream{
	
	    protected OutputStream out;
	    
	    public void write(byte b[]) throws IOException {
	        write(b, 0, b.length);
	    }
	    
	    public void write(byte b[], int off, int len) throws IOException {
	        if ((off | len | (b.length - (len + off)) | (off + len)) < 0)
	            throw new IndexOutOfBoundsException();
	
	        for (int i = 0 ; i < len ; i++) {
	            write(b[off + i]);
	        }
	    }
	}

**off**是写入的源数组偏移量，**len**是写入的长度。  

注意：虽然在FilterOutputStream中有*write(byte b[], int off, int len)* 方法，但是BufferedOutputStream中**重写**了这个方法。如下所示：

	Class BufferedOutputStream extends FilterOutputStream {
	
	    public BufferedOutputStream(OutputStream out) {
	        this(out, 8192);
	    }
	
	    public synchronized void write(byte b[], int off, int len) throws IOException {
	        if (len >= buf.length) {
	            flushBuffer();
	            out.write(b, off, len);
	            return;
	        }
	        if (len > buf.length - count) {
	            flushBuffer();
	        }
	        System.arraycopy(b, off, buf, count, len);
	        count += len;
	    }
	    
	    private void flushBuffer() throws IOException {
	        if (count > 0) {
	            out.write(buf, 0, count);
	            count = 0;
	        }
	    }
	    
	    public synchronized void flush() throws IOException {
	        flushBuffer();
	        out.flush();
	    }
	}
这里的字节数组*buf* 默认长度为8192，即8KB，内部保存数据用的，*count*是目前buffer中已经写入的长度。这里的逻辑是：

1. 如果写入的长度大于buffer的大小，先将buffer的数据写入，然后再直接全部写入
2. 如果写入的长度大于buffer剩余的空间，先将buffer的数据写入，然后进行拷贝操作
3. 如果写入的长度小于buffer剩余的空间，只进行拷贝操作

### BufferedOutputStream的作用
定义一个buffer数组，用于临时保存数据，然后再一次性写入。所以当使用BufferedOutputStream写入数据时，**一定要**使用 **flush()** 方法来保证数据全部写入成功，否则会出现数据丢失。

[IO系列资料](http://tutorials.jenkov.com/java-io/bufferedoutputstream.html)