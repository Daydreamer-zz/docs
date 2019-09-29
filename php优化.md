# php优化插件

##  一、xcache

php版本5.6.26

xcache 3.2.0

### 1.安装相关包

```bash
yum install -y autoconf libtool
```

### 2.解压xcache

```
tar xf xcache-3.2.0.tar.gz
```

### 3.编译

必须执行这项，否则不生成configure文件 

```
/application/php/bin/phpize
```

编译

```
./configure --enable-xcache --with-php-config=/application/php/bin/php-config
```

安装

```
make && make install
```

### 4.拷贝相关文件 

将xcache配置文件追加到`php.ini`

```
cat xcache.ini >> /application/php/etc/php.ini
```

拷贝管理页面

```
cp ~/xcache-3.2.0/htdocs/* /application/nginx/html/xadmin
```



### 5.配置文件修改

其中`xcache.admin.user`和`xcache.admin.pass`需要手动配置，这是用于xcache管理页面登录的账户和密码

生成`xcache.admin.pass`

```
echo -n "shz123456" |md5sum
```



配置文件解释

xcache管理界面用户和密码

```
xcache.admin.user = "shz"
xcache.admin.pass = "98807f210c86d8038ed2bcea0639ae21"
```

xcache缓存类型

```
xcache.shm_scheme =        "mmap"
```

缓存空间大小

```
xcache.size  =               60M
```

xcache缓存文件(需要手动touch)



```bash
[xcache-common]
;; non-Windows example:
extension = xcache.so
;; Windows example:
; extension = php_xcache.dll

[xcache.admin]
xcache.admin.enable_auth = On

; use http://xcache.lighttpd.net/demo/cacher/mkpassword.php to generate your encrypted password
xcache.admin.user = "shz"
xcache.admin.pass = "98807f210c86d8038ed2bcea0639ae21"

[xcache]
; ini only settings, all the values here is default unless explained

; select low level shm implemenation
xcache.shm_scheme =        "mmap"
; to disable: xcache.size=0
; to enable : xcache.size=64M etc (any size > 0) and your system mmap allows
xcache.size  =               60M          
; set to cpu count (cat /proc/cpuinfo |grep -c processor)
xcache.count =                 1
; just a hash hints, you can always store count(items) > slots
xcache.slots =                8K
; ttl of the cache item, 0=forever
xcache.ttl   =                 0
; interval of gc scanning expired items, 0=no scan, other values is in seconds
xcache.gc_interval =           0

; same as aboves but for variable cache
xcache.var_size  =            4M
xcache.var_count =             1
xcache.var_slots =            8K
; default value for $ttl parameter of xcache_*() functions
xcache.var_ttl   =             0
; hard limit ttl that cannot be exceed by xcache_*() functions. 0=unlimited
xcache.var_maxttl   =          0
xcache.var_gc_interval =     300

; mode:0, const string specified by xcache.var_namespace
; mode:1, $_SERVER[xcache.var_namespace]
; mode:2, uid or gid (specified by xcache.var_namespace)
xcache.var_namespace_mode =    0
xcache.var_namespace =        ""

; N/A for /dev/zero
xcache.readonly_protection = Off
; for *nix, xcache.mmap_path is a file path, not directory. (auto create/overwrite)
; Use something like "/tmp/xcache" instead of "/dev/*" if you want to turn on ReadonlyProtection
; different process group of php won't share the same /tmp/xcache
; for win32, xcache.mmap_path=anonymous map name, not file path
xcache.mmap_path =    "/dev/zero"


; Useful when XCache crash. leave it blank(disabled) or "/tmp/phpcore/" (writable by php)
xcache.coredump_directory =   ""
; Windows only. leave it as 0 (default) until you're told by XCache dev
xcache.coredump_type =         0

; disable cache after crash
xcache.disable_on_crash =    Off

; enable experimental documented features for each release if available
xcache.experimental =        Off

; per request settings. can ini_set, .htaccess etc
xcache.cacher =               On
xcache.stat   =               On
xcache.optimizer =           Off

[xcache.coverager]
; enabling this feature will impact performance
; enabled only if xcache.coverager == On && xcache.coveragedump_directory == "non-empty-value"

; per request settings. can ini_set, .htaccess etc
; enable coverage data collecting and xcache_coverager_start/stop/get/clean() functions
xcache.coverager =           Off
xcache.coverager_autostart =  On

; set in php ini file only
; make sure it's readable (open_basedir is checked) by coverage viewer script
xcache.coveragedump_directory = ""
```

## 二、mamcache

### 1. 解压

```
tar xf memcache-2.2.5.tgz
```

### 2.生成configure文件

```
cd memcache-2.2.5/ && /application/php/bin/phpize
```

### 3.编译

```
./configure --with-php-config=/application/php/bin/php-config
```

### 4.安装

```
make && make install
```

### 5.配置

```
vim /application/php/etc/php.ini
```

```bash
extension_dir = "/application/php/lib/php/extensions/no-debug-non-zts-20131226/"  #修改路径

extension = memcache.so         #最下面添加
```

### 6.检查配置文件语法

```bash
/application/php/sbin/php-fpm -t    #出现successful说明成功
```



## 三、PDO_MYSQL(源码安装时没有指定时需要单独安装的情况)

### 1.进入php源码目录

```
/root/php-5.6.8/ext/pdo_mysql
```

### 2.生产configure文件

```
/application/php/bin/phpize
```

### 3.编译

```
./configure --with-php-config=/application/php/bin/php-config --with-pdo-mysql=/application/mysql
```

### 4.安装

在make之前还要做一个mysql的header文件的软连接。因为mysql安装的时候指定了目录，不做软连接的话，还是找不到header文件

```
ln -s /application/mysql/include/* /usr/local/include/
```

```
make && make install
```

### 5.配置

到`php.ini`中添加`extension=pdo_mysql.so`

```
extension = pdo_mysql.so
```
### 6.检查配置文件语法

```bash
/application/php/sbin/php-fpm -t    #出现successful说明成功
```

##  四、图像处理程序以及imagick扩展模块

### 1.安装ImageMagick(imagick扩展模块需要用到ImageMagick)

**关于什么是ImageMagick**

ImageMagick是一套软件系列，主要用于图片的创建、编辑以及转换等

安装ImageMagick

```
tar xf ImageMagick-6.9.10-29.tar.gz && cd ImageMagick-6.9.10-29/ && ./configure --prefix=/application/imagemagick && make && make install
```

或

```
yum install ImageMagick ImageMagick-devel -y
```



安装imagick模块

```
tar xf imagick-3.3.0.tgz
```

```
cd imagick-3.3.0/
```

```
/application/php/bin/phpize
```

```
./configure --with-php-config=/application/php/bin/php-config --with-imagick=/application/imagemagick
```



配置(在php.ini中添加)

```
extension = imagick.so
```










