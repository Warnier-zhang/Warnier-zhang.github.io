---
layout: post
title: Nginx解惑

---

1. `/aaa/`等价于`/aaa/index.html`；

2. `location`匹配规则的优先级是什么？

   > To find the location that best matches a URI, NGINX Plus first compares the URI to the locations with a prefix string. It then searches the locations with a regular expression.
   >
   > Higher priority is given to regular expressions, unless the `^~` modifier is used. Among the prefix strings NGINX Plus selects the most specific one (that is, the longest and most complete string). The exact logic for selecting a location to process a request is given below:
   >
   > 1. Test the URI against all prefix strings.
   > 2. The `=` (equals sign) modifier defines an exact match of the URI and a prefix string. If the exact match is found, the search stops.
   > 3. If the `^~` (caret-tilde) modifier prepends the longest matching prefix string, the regular expressions are not checked.
   > 4. Store the longest matching prefix string.
   > 5. Test the URI against regular expressions.
   > 6. Stop processing when the first matching regular expression is found and use the corresponding location.
   > 7. If no regular expression matches, use the location corresponding to the stored prefix string.

3. `location [修饰符： = | ~ | ~* | ^~ ] 规则：uri { ... }`中`=`、`~`、`~*`、`^~`的区别是什么？

   1. 若修饰符是`=`，则是**完全匹配**，即HTTP请求地址与`uri`相同；
   2. 若修饰符是`^~`，则是**前缀匹配**，且如果匹配成功，就不再进行**正则表达式匹配**；
   3. 若没有修饰符，则是**前缀匹配**，即HTTP请求地址必须以`uri`开头；
   4. 若修饰符是`~ `，则是**正则表达式匹配**，且**字符大小写敏感**；
   5. 若修饰符是`~*`，则是**正则表达式匹配**，且**字符大小写不敏感**；

4. `$host`、`$http_host`、`$proxy_host`等的区别是什么？

   - `$host`：若HTTP请求头设置了`Host`，则值是`Host`去掉端口号的部分，否则就是`$server_name`；
   - `$http_host`：若HTTP请求头设置了`Host`，则值是`Host`，否则就是空；
   - `$proxy_host`：`proxy_pass`命令指定的HTTP地址中的域名和端口号；
   - `$remote_addr`：客户端地址；
   - `$server_addr`：服务器地址；
   - `$server_name`：主机名 / 域名；

5. 怎么给Nginx启用HTTPS？

   ```
   listen       443 ssl;
   server_name  localhost;
   
   ssl_certificate      cert.pem;
   ssl_certificate_key  cert.key;
   
   ssl_session_cache    shared:SSL:1m;
   ssl_session_timeout  5m;
   
   ssl_ciphers  HIGH:!aNULL:!MD5;
   ssl_prefer_server_ciphers  on;
   ```

6. 在处理HTTP请求的时候，`location /aaa { ... }`和`location /aaa/ { ... }`的区别是什么？

   - 若仅配置`location /aaa { ... }`，则无论请求地址是`/aaa`，还是`/aaa/`，返回结果都相同；
   - 若同时配置`location /aaa { ... }`和`location /aaa/ { ... }`，则根据请求地址返回对应结果；
   - 若仅配置``location /aaa/ { ... }`，则请求地址`/aaa`的返回结果是`404`，`/aaa/`正常返回；

7. 在代理、转发请求的时候，`location /aaa { ... }`和`location /aaa/ { ... }`的区别是什么？

   > 如果`proxy_pass`指定的转发uri以`/`结尾，就会替换请求地址中相同的部分。

   当HTTP请求由`proxy_pass`进行转发的时候，规则uri若以/结尾，即/aaa/，此时，如果请求地址是/aaa，就会直接返回状态码301，然后重定向到/aaa/；

   - 若请求URL以`/aaa`或者`/aaa/`结尾，则2种配置**没有区别**；

   - 若`/aaa`或者`/aaa/`是请求URL的前缀、一部分，则2种配置有区别，以转发到http://www.baidu.com/s?wd=ddd举例说明：

     如果匹配规则是：

     ```
     location /ddd/ {
         proxy_pass http://www.baidu.com/;
         ...
     }
     ```

     就需要访问：http://localhost/ddd/s?wd=ddd；

     如果匹配规则是：

     ```
     location /ddd {
         proxy_pass http://www.baidu.com/;
         ...
     }
     ```

     就需要访问：http://localhost/ddds?wd=ddd；此时，如果访问http://localhost/ddd/s?wd=ddd，就会被转发到http://www.baidu.com//s?wd=ddd，返回状态码404。

8. 在代理、转发请求的时候，如何设置HTTP请求头`Host`？

   默认可以这样做：

   ```
   proxy_set_header Host $proxy_host;
   ```

   如果需要传递客户端的域名（/IP）、**端口**等等，就：

   ```
   proxy_set_header Host $http_host;
   或者
   proxy_set_header Host $host:$server_port;
   ```

9. 怎么处理HTTP状态码`413`？

   413，即Payload Too Large，需要设置：

   ```
   client_max_body_size 50m;
   ```

10. 如何使用Nginx实现负载均衡？



参考资料：

- [Nginx文档——location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location)；
- [Nginx文档——proxy_pass](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)；
- [Understanding Nginx Server and Location Block Selection Algorithms](https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms)；