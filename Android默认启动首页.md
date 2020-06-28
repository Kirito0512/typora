### Android流程图



![启动流程](http://ww1.sinaimg.cn/mw690/e99d92cfly1g5r6lzvhs6j20zw2ag7bc.jpg)

### 需要注意的点

### 1.强制登录的相关条件

-  从未登陆过
- 当前未登录
- 不是从老版本更新上来的用户

当用户同时满足，以上三个条件时，会执行强制登录逻辑

强制登录后，走到 LoginBaseActivity 的 processOnLoginSuccess 方法， isForcedLogin 为true，跳转MainActivity的点歌台。

### 2. ServerConfig指定tab

服务器控制选中tab的字段，是ServerConfig里的startTab。但是目前被客户端写死成"my_changba"了。所以服务器对这方面的控制，应该是名存实亡了。

### 3. MainActivity选择Tab

- 判断是 新安装用户，并且 安装时间小于 24h

- 检查INTENT中存在值为DEFAULT_TAB的key

上面两个判断，满足任意一个，都会指定到点歌台TAB。如果都不满足，那就是去首页。

### 4. 首页下属三个TAB的选择

服务器对首页TAB下属的，关注，推荐，附近和探索，四个tab的控制，体现在OptionalConfigs中的feedStartTab字段。

如果服务器配置到 后三个tab，会直接切换。

#### 4.1 服务器配置到关注TAB

查看用户关注的数量，如果不为0，就切换到关注TAB。否则，判断是第一次进到当前逻辑。

是则进入关注TAB，否则进入推荐TAB。



即当第一次，被服务器指定到关注页，并且当时没关注过别人，才会切到推荐页。



### 5. 当前存在的问题

- saveInstanceState的相关逻辑，保存的tab为TAB_MESSAGE时，没必要再跳回点歌台
- startTab的值，应该由服务器控制，不该被写死
- 上面的第三点，判断上有些问题。AppUtil.isNewInstall 方法，在用户第一次关闭app后，CURRENT_VERSION会被赋值，之后就只会返回false。所以后面对24h的判断就失去意义了。这里应该可以改为，由服务器统一下发startTab，不再由客户端去判断
- 第三点中的24h，时间与iOS有一些出入，之后需要统一
- 首页tab的下属tab的选择，可以直接使用服务器配置的feedStartTab，多余的判断可以移除



### 6. 后续优化后的流程

![优化后的流程](http://ww1.sinaimg.cn/mw690/e99d92cfly1g5tj2ca863j20mk110dhw.jpg)

### 友情链接

[iOS启动默认页面整理](https://wiki.changba.com/pages/viewpage.action?pageId=28153220)

