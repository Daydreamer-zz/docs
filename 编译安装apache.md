```
yum install zlib zlib-devel pcre pcre-devel openssl openssl-devel -y
```

apr

```
tar xf apr-1.4.5.tar.gz && cd apr-1.4.5/ && ./configure --prefix=/usr/local/apr && make -j8 && make install
```

apr-util

```
tar xf apr-util-1.3.12.tar.gz && cd apr-util-1.3.12/ && ./configure --prefix=/usr/local/apr-util --with-apr=/usr/local/apr/bin/apr-1-config && make -j8 && make install
```





```
tar xf httpd-2.4.38.tar.gz && cd httpd-2.4.38/ &&  ./configure --prefix=/usr/local/httpd --enable-deflate --enable-expires --enable-headers --enable-modules=most --enable-so --with-mpm=worker --enable-rewrite --with-pcre --with-apr=/usr/local/apr --with-apr-util=/usr/local/apr-util && make -j8 && make install
```



```
tar xf apr-1.4.5.tar.gz && cd apr-1.4.5/ && ./configure --prefix=/usr/local/apr && make -j8 && make install && cd .. && tar xf apr-util-1.3.12.tar.gz && cd apr-util-1.3.12/ && ./configure --prefix=/usr/local/apr-util --with-apr=/usr/local/apr/bin/apr-1-config && make -j8 && make install && cd .. && tar xf httpd-2.4.38.tar.gz && cd httpd-2.4.38/ &&  ./configure --prefix=/usr/local/httpd --enable-deflate --enable-expires --enable-headers --enable-modules=most --enable-so --with-mpm=worker --enable-rewrite --with-pcre --with-apr=/usr/local/apr --with-apr-util=/usr/local/apr-util && make -j8 && make install
```

