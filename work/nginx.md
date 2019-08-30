## nginx docs
[offical docs](http://nginx.org/en/docs/)  
[Location Block Selection Algorithms](https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms)

## regex in nginx

### How to I get variables from location in nginx?
```
location ~/photos/resize/(\d+)/(\d+) {
  # use $1 for the first \d+ and $2 for the second, and so on.
}
or
location ~/photos/resize/(?<width>(\d+))/(?<height>(\d+)) {
  # so here you can use the $width and the $height variables
}
```
$1 refers to the first capturing group (...). When you added another group it referred to that one instead. You can use a non-capturing group (?:...) instead, or refer to the second capturing group $2


## http related

### 跨域(cors)
- This means that a web application using those APIs can only request HTTP resources from the same origin the application was   loaded from, unless the response from the other origin includes the right CORS headers.
- [详细解释](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- nginx支持跨域相关配置
```
location / {
                proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504 http_404;
                proxy_pass          http://groupall;
                include             conf/proxy_params;
                proxy_http_version  1.1;
                proxy_set_header    Connection "";
                client_max_body_size 8M;

                if ( $request_method = 'POST' ) {
                    add_header Access-Control-Allow-Origin "$http_origin";
                    add_header Access-Control-Allow-Methods POST;
                    add_header Access-Control-Allow-Headers 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
                    add_header Access-Control-Max-Age 1;
                    add_header Access-Control-Allow-Credentials false; #false表示请求不允许带cookie
                }
                if ( $request_method = 'OPTIONS' ) {
                    add_header Access-Control-Allow-Origin "$http_origin";
                    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
                    add_header Access-Control-Allow-Headers 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
                    add_header Access-Control-Max-Age 1;
                    add_header Access-Control-Allow-Credentials false; 
                }
        }
```

### nginx做http正向代理
- [方法](https://blog.csdn.net/micwing/article/details/84774568)
- [模块链接](https://github.com/chobits/ngx_http_proxy_connect_module)


