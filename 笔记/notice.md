#### Andorid 创建文件夹和文件

都是基于一个File

创建文件夹  `file.makeDir()`

创建文件 `file.createNewFile()`

#### Android context.getDir

[Context  |  Android 开发者 ](https://developer.android.com/reference/android/content/Context.html#getDir(java.lang.String,%20int))

```
// 在内部存储下创建一个 app_ + cb_setting的文件夹
// MODE_APPEND 如果文件存在，会接着文件末端继续写入数据，而不是擦除原有数据
getApplicationContext().getDir("cb_setting", Context.MODE_APPEND);
```



### Android 清除缓存 && 清除全部数据

```
// 在内存存储中创建app_cb_setting文件夹，下面创建childFile123.txt文件
File dirFile = getApplicationContext().getDir("cb_setting", Context.MODE_APPEND);
                dirFile.mkdir();
                File dirChildFile = new File(dirFile, "chidFile123.txt");
                try {
                    dirChildFile.createNewFile();
                } catch (IOException e) {
                    e.printStackTrace();
                }

                // 获取内部存储的file文件夹，下面创建childFileFile123.txt
                File fileFile = getApplicationContext().getFilesDir();
                File fileChildFile = new File(fileFile, "childFileFile123.txt");
                try {
                    fileChildFile.createNewFile();
                } catch (IOException e) {
                    e.printStackTrace();
                }

                // 获取内部存储的cache文件夹，下面创建childCacheFile123.txt
                File cacheFile = getApplicationContext().getCacheDir();
                File cacheChildFile = new File(cacheFile, "childCacheFile123.txt");
                try {
                    cacheChildFile.createNewFile();
                } catch (IOException e) {
                    e.printStackTrace();
                }
```



![](https://tva1.sinaimg.cn/large/006tNbRwly1gax3c8z5t2j30hc05caag.jpg)

亲测 

清除缓存 会删除cache下面的内容

![屏幕快照 2020-01-15 上午11.31.58](https://tva1.sinaimg.cn/large/006tNbRwly1gax3cjqw02j30h604smxs.jpg)

清除全部数据 会删除内部存储的全部数据