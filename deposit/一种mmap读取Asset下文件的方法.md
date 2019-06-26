
工作中遇到这样一种场景：需要读取一个很大的词典文件，载入内存后执行。
之前的做法是将文件内置在assets目录下，拷贝到应用内置路径，再读取后载入内存。这种方案有两个问题，一是拷贝的过程耗时且可能存在错误；二是直接载入后占用太多内存。对于第二个问题我们是用mmap解决的，而为了解决第一个问题，我们尝试不走拷贝操作，直接从asset读取。

而mmap一个重点是获得文件的fd，但是上网搜了一下，从asset读取fd的方法比较少，因此就在这里总结一下。

mmap的函数原型是这样的：

```
/**
 * mmap将一个文件或者其它对象映射进内存
 * @param start 映射区的开始地址，设置为0时表示由系统决定映射区的起始地址。
 * @param length 映射区的长度
 * @param prot 期望的内存保护标志
 * @param flags 指定映射对象的类型，映射选项和映射页是否可以共享
 * @param fd 文件描述符
 * @params offset 被映射对象内容的起点
 */
void* mmap(void* start, size_t length, int prot, int flags, int fd ,off_t offset);

```

可以看到要使用mmap，需要提供一个有效的文件描述符fd。正好AssetsManager中有个方法是`AAsset_openFileDescriptor64()`，只要调用这个方法就能获得fd了，赶紧试一下，代码如下。

```
/**
 * 从Asset下读取某文件的fileDescriptor
 * @param env JNIENV对象
 * @param assetManager AssetManager管理对象
 * @param fileName asset下的文件名称
 * @param outStart 返回值，文件对应的start偏移
 * @param outLength 返回值，文件对应的长度
 * @param fd 返回值，文件的fd
 * @return 返回0表示成功，返回-1表示失败
 */
int read_fd_from_asset(JNIEnv *env, jobject assetManager, const char* fileName, off64_t *outStart, off64_t *outLength, int *fd)
{
    AAssetManager *mgr = AAssetManager_fromJava(env, assetManager);
    if (mgr == NULL) {
        LOGI("AAssetManager==NULL");
        return -1;
    }
    jboolean iscopy;
    AAsset *asset = AAssetManager_open(mgr, fileName, AASSET_MODE_UNKNOWN);
    if (asset == NULL) {
        LOGI("open asset %s==NULL", fileName);
        return -1;
    }
    off_t bufferSize = AAsset_getLength(asset);
    LOGI("asset cz file: %s, size: %d\n", fileName, bufferSize);

    (*fd) = AAsset_openFileDescriptor64(asset, outStart, outLength);
    LOGI("AAsset_openFileDescriptor64: start=%lld, len=%lld, fd=%d", *outStart, *outLength, *fd);

    AAsset_close(asset);
    return 0;
}
```
发现不行，返回的是-1。

查看这个方法的注释，举了一个例子，如果asset文件是compressed，则fd固定返回-1。

```
/**
 * Open a new file descriptor that can be used to read the asset data.
 *
 * Uses a 64-bit number for the offset and length instead of 32-bit instead of
 * as AAsset_openFileDescriptor does.
 *
 * Returns < 0 if direct fd access is not possible (for example, if the asset is
 * compressed).
 */
int AAsset_openFileDescriptor64(AAsset* asset, off64_t* outStart, off64_t* outLength);
```
因此可能跟compress有点关系。稍微看下源码：

```
int AAsset_openFileDescriptor64(AAsset* asset, off64_t* outStart, off64_t* outLength)
{
    return asset->mAsset->openFileDescriptor(outStart, outLength);
}
```

跟进去，Asset类是一个抽象类，有两个子类：（源码位置：[这里](https://android.googlesource.com/platform/frameworks/base/+/master/libs/androidfw/Asset.cpp)）

```
class Asset {

    /*
     * Open a new file descriptor that can be used to read this asset.
     * Returns -1 if you can not use the file descriptor (for example if the
     * asset is compressed).
     */
    virtual int openFileDescriptor(off64_t* outStart, off64_t* outLength) const = 0;
}

/*
 * An asset based on an uncompressed file on disk.  It may encompass the
 * entire file or just a piece of it.  Access is through fread/fseek.
 */
class _FileAsset : public Asset {
     //...
     virtual int openFileDescriptor(off64_t* outStart, off64_t* outLength) const;
}

/*
 * An asset based on compressed data in a file.
 */
class _CompressedAsset : public Asset {
    // ...
    virtual int openFileDescriptor(off64_t* outStart, off64_t* outLength) const { return -1; }
}
```
其实这里已经比较明确了，如果是_CompressedAsset则fd返回-1，只有_FileAsset才有正确的fd。我们还需要确认下Asset是如何构建出来的。

回到原先`AAsset *asset = AAssetManager_open(mgr, fileName, AASSET_MODE_UNKNOWN);`的位置，看源码。可以一直追溯到ApkAssets.cpp（源码在[这里](https://android.googlesource.com/platform/frameworks/base/+/master/libs/androidfw/ApkAssets.cpp)）：

```
std::unique_ptr<Asset> ApkAssets::Open(const std::string& path, Asset::AccessMode mode) const {
  CHECK(zip_handle_ != nullptr);
  ::ZipString name(path.c_str());
  ::ZipEntry entry;
  int32_t result = ::FindEntry(zip_handle_.get(), name, &entry);
  if (result != 0) {
    return {};
  }
  if (entry.method == kCompressDeflated) {
    std::unique_ptr<FileMap> map = util::make_unique<FileMap>();
    if (!map->create(path_.c_str(), ::GetFileDescriptor(zip_handle_.get()), entry.offset,
                     entry.compressed_length, true /*readOnly*/)) {
      LOG(ERROR) << "Failed to mmap file '" << path << "' in APK '" << path_ << "'";
      return {};
    }
    std::unique_ptr<Asset> asset =
        Asset::createFromCompressedMap(std::move(map), entry.uncompressed_length, mode);
    if (asset == nullptr) {
      LOG(ERROR) << "Failed to decompress '" << path << "'.";
      return {};
    }
    return asset;
  } else {
    std::unique_ptr<FileMap> map = util::make_unique<FileMap>();
    if (!map->create(path_.c_str(), ::GetFileDescriptor(zip_handle_.get()), entry.offset,
                     entry.uncompressed_length, true /*readOnly*/)) {
      LOG(ERROR) << "Failed to mmap file '" << path << "' in APK '" << path_ << "'";
      return {};
    }
    std::unique_ptr<Asset> asset = Asset::createFromUncompressedMap(std::move(map), mode);
    if (asset == nullptr) {
      LOG(ERROR) << "Failed to mmap file '" << path << "' in APK '" << path_ << "'";
      return {};
    }
    return asset;
  }
}

```

可以大概看出，apk中的asset是一个zip文件(可以看到整个asset文件也是通过mmap读取的)，asset中的每个文件对应一个entry，这个entry有一个method成员，如果是kCompressDeflated，则调用Asset::createFromCompressedMap，否则调用Asset::createFromUncompressedMap。

也就是说，要获得正确的fd，必须要让我们的目标文件不压缩。上网搜一下，确实可以单独制定apk内某一部分的asset文件不被压缩：

```
android {
    ...
    aaptOptions {
       noCompress 'png'
    }
}
```

到这一步，fd已经获得了。但是再进行mmap的时候，发现还是读不出来，读出来内容为空。

不过有了上面的分析就很好解决了，因为asset下的文件是作为整体存在的，所以每个文件都有一个自己的startOffset和Length，所以用`mmap(NULL, (size_t) outLength, PROT_READ, MAP_PRIVATE, fd, 0)`来读取文件是不行的，必须正确使用startOffset。而实验下来，不能直接从startOffset读取，必须从0开始读取到start+length才可以。

最终的代码如下：

```

JNIEXPORT int JNICALL Java_com_citrus_util_readFromAssetMmap
        (JNIEnv *env, jobject clazz, jobject assetManager, jstring jfilename)
{
    jboolean iscopy;
    const char *file_name = env->GetStringUTFChars(jfilename, &iscopy);
    off64_t outStart;
    off64_t outLength;
    int fd;
    int ret = read_fd_from_asset(env, assetManager, file_name, &outStart, &outLength, &fd);
    if (ret < 0) {
        LOGI("read_fd_from_asset fail");
        return -1;
    }
    if (fd < 0) {
        LOGI("read_fd_from_asset fail, fd < 0, direct fd access is not possible");
        return -1;
    }
    char *data = (char *) mmap(NULL, (size_t) (outLength + outStart), PROT_READ, MAP_PRIVATE, fd,
                               0);
    if (data == (void *) -1) {
        LOGI("mmap failed");
        return -1;
    }
    return 0;
}
```



