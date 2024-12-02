### 方式1:文件传输

将Bitmap转换为文件存储到手机存储空间，并将对应的文件路径进行传递。

### 方式2:使用intent的putBinder

bitmap的底层实现中，writeToParcel会根据图片的大小是否大于16k来选择直接传输，
或者通过Ashmem进行处理后，将对应的fd传输过去。

### 方式3:使用MemoryFile

通过使用MemoryFile得到FileDescriptor，将得到的文件描述符转为ParcelFileDescriptor，
通过aidl函数的参数传递到对端，再解析出来使用。

其中方法2、3都是利用了安卓中共享内存的方式。