### 1. 源码

先放上源码，方便看

```java
@Override public Response intercept(Chain chain) throws IOException {
   // 1. 注意cache.get方法
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();

    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    // If we're forbidden from using the network and the cache is insufficient, fail.
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    // If we don't need the network, we're done.
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        // 2. 注意cache.update方法
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        // 3. 注意cache.put方法
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          // 4. 注意cache.remove方法
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
```



### 2. 整体流程

- **cache.get(chain.request())**，取出**cacheCandidate**(候选缓存)
- 通过chain.request 和 **cacheCandidate**,创建 **CacheStrategy** 缓存策略。并从中拿到**networkRequest**和**cacheResponse**
- 如果**networkRequest**和**cacheResponse**都为null，返回一个504code的**response**--------END
- 如果只有**networkRequest**为null，将缓存返回---------END
- 否则，**networkRequest**不为null，使用`networkRequest = chain.proceed(networkRequest)` 将请求传递给下一个Interceptor。
- 如果**cacheResponse**也不为null
  - **networkResponse** code为304，返回**CacheResponse** ---------------- END
  - 否则判断是否符合缓存条件，可以的话，使用 **cache.put** 进行缓存并将其返回。否则直接返回---------------------END



### 3. 流程细节

 #### 3.1 cache.get

> 这应该是整个流程中最复杂的部分了，代码只有下面简洁的一句话

```
Response cacheCandidate = cache != null ? cache.get(chain.request()): null;
```

需要先解释下，cache的由来, Cache.java 在 okhttp 的jar包里。

Okhttp默认配置下，cache为null。如果想要用，可以按照下面的例子使用：

```
//缓存文件夹
File cacheFile = new File(getExternalCacheDir().toString(),"cache");
//缓存大小为10M
int cacheSize = 10 * 1024 * 1024;
//创建缓存对象
Cache cache = new Cache(cacheFile,cacheSize);
OkHttpClient client = new OkHttpClient.Builder().cache(cache).build();
```

***

下面来看Cache的get方法源码，行数不多

```
@Nullable Response get(Request request) {
  String key = key(request.url());
  DiskLruCache.Snapshot snapshot;
  Entry entry;
  try {
  	// 注意这里的cache是DiskLruCache.java, 也在Okhttp的jar包里
    snapshot = cache.get(key);
    if (snapshot == null) {
      return null;
    }
  } catch (IOException e) {
    // Give up because the cache cannot be read.
    return null;
  }

  try {
    entry = new Entry(snapshot.getSource(ENTRY_METADATA));
  } catch (IOException e) {
    Util.closeQuietly(snapshot);
    return null;
  }

  Response response = entry.response(snapshot);

  if (!entry.matches(request, response)) {
    Util.closeQuietly(response.body());
    return null;
  }

  return response;
}
```

1. DiskLruCache的get方法，返回Snapshot对象,步骤如下

- initialize()

> 主要是配置journalFile文件 和 lruEntries (LinkedHashMap<String, Entry>).

- 从lruEntries中取出key对应的Entry对象

- 同时，<u>**向journalFile中添加一行READ数据**</u>

- 返回Entry中的Snapshot对象



2. 回到源码第8行。使用Snapshot对象中，长度为2的sources数组，构建Response并返回。

> 其中ENTRY_METADATA和ENTRY_BODY分别为0和1，代表请求报文的head和body信息。

***

#### 3.2 cache.update

```java
void update(Response cached, Response network) {
  Entry entry = new Entry(network);
  DiskLruCache.Snapshot snapshot = ((CacheResponseBody) cached.body()).snapshot;
  DiskLruCache.Editor editor = null;
  try {
    editor = snapshot.edit(); // Returns null if snapshot is not current.
    if (editor != null) {
      entry.writeTo(editor);
      editor.commit();
    }
  } catch (IOException e) {
    abortQuietly(editor);
  }
}
```

> 整体步骤与 3.3 put方法基本一致

***



#### 3.3 cache.put(response)

1. 用 **response** 构建 **Entry** 对象
2. **DiskLruCache.edit** 获取 **Editor**

- **initialize()**
- 通过**response**的url生成key
- 调用 **DiskLruCache.edit**   方法，**lruEntries.get(key)**获得**Entry** 
- **向journalFile中写入一行 DIRTY 数据**
- 用**Entry**对象，构建**Editor**对象返回

3. **entry.writeTo(editor)** 写入数据

***

#### 3.4 cache.remove(request)

只有短短一行代码

```java
void remove(Request request) throws IOException {
  // cache 是 LruDiskCache对象
  cache.remove(key(request.url()));
}
```

来看下 **LruDiskCache **的 **remove** 方法

```
public synchronized boolean remove(String key) throws IOException {
// 这四步真的和edit，get方法一模一样
  initialize();
  checkNotClosed();
  validateKey(key);
  Entry entry = lruEntries.get(key);
  
  if (entry == null) return false;
  boolean removed = removeEntry(entry);
  if (removed && size <= maxSize) mostRecentTrimFailed = false;
  return removed;
}
```

**removeEntry**

- 向 **journalFile** 写入一行REMOVE数据
- 从 **lruEntries** 中 remove 对应的 key