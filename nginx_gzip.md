# How to enable the GZIP on nginx

## Enable dynamic gzip compression

Dynamic compression means the nginx dynamically compress contents when serving the content, it's not pre compressed.

By default, the gzip compression is no enabled. Let's take a look:

```
# curl -o test.txt -D - -s -H "Accept-Encoding: gzip,deflate" http://localhost/test.html
HTTP/1.1 200 OK
Server: nginx/1.10.3 (Ubuntu)
Date: Wed, 25 Nov 2020 08:58:27 GMT
Content-Type: text/html
Content-Length: 1550  <<<<<
Last-Modified: Tue, 26 Nov 2019 09:09:36 GMT
Connection: keep-alive
ETag: "5ddcebd0-60e"
Accept-Ranges: bytes
```

Refer to this [doc](https://docs.nginx.com/nginx/admin-guide/web-server/compression/), I enable the gzip in the http section as below:

```
http {
    ...
    gzip  on;
    
    server {
        ...
    }
}
```

Remember to reload the configure
```
# service nginx reload
```

Then let's test again

```
# curl -o test.txt -D - -s -H "Accept-Encoding: gzip,deflate" http://localhost/test.html
HTTP/1.1 200 OK
Server: nginx/1.10.3 (Ubuntu)
Date: Wed, 25 Nov 2020 09:01:47 GMT
Content-Type: text/html
Last-Modified: Tue, 26 Nov 2019 09:09:36 GMT
Transfer-Encoding: chunked  <<<<<<<<
Connection: keep-alive
ETag: W/"5ddcebd0-60e"
Content-Encoding: gzip <<<<<<<
```

We can see there is gzip response header returned.

### Question
- Why there is no `Content-Length` returned, but there is `Transfer-Encoding: chunked`?

    Get answer from [here](https://github.com/metacpan/metacpan-api/issues/240) 
    >If the message body doesn't need modifying, then it can know the Content-Length in advance, and can reliably report as such, and then just stream the message body from whatever is creating it as-is.
    >
    >However, if the message body does need modifying ( ie: gzip ), then nginx will do so in a streaming mechanism, that is: instead of gzipping a large file, finding the content length, and then starting the transfer, it adds a gzip filter to the output stream, and uses "Chunked" transfer encoding instead.

## Serve the static / pre compressed file

In some cases, we want the nginx to serve the compressed content with the content-Length, this is useful in the cases, for example, the caching that use the length to judge whether the content changed, and the precompressed file can save servers energy since it does not compressed for each request.

Referring to this [doc](https://docs.nginx.com/nginx/admin-guide/web-server/compression/#sending-compressed-files), it needs to use the `gzip_static`.

Before you use enable this configure, you need to make sure your `ngx_http_gzip_static_module` is enabled. You can check your nginx configure by `nginx -V`.

```
# nginx -V
nginx version: nginx/1.10.3 (Ubuntu)
built with OpenSSL 1.0.2g  1 Mar 2016
TLS SNI support enabled
configure arguments: --with-cc-opt='-g -O2 -fPIE -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2' --with-ld-opt='-Wl,-Bsymbolic-functions -fPIE -pie -Wl,-z,relro -Wl,-z,now' --prefix=/usr/share/nginx --conf-path=/etc/nginx/nginx.conf --http-log-path=/var/log/nginx/access.log --error-log-path=/var/log/nginx/error.log --lock-path=/var/lock/nginx.lock --pid-path=/run/nginx.pid --http-client-body-temp-path=/var/lib/nginx/body --http-fastcgi-temp-path=/var/lib/nginx/fastcgi --http-proxy-temp-path=/var/lib/nginx/proxy --http-scgi-temp-path=/var/lib/nginx/scgi --http-uwsgi-temp-path=/var/lib/nginx/uwsgi --with-debug --with-pcre-jit --with-ipv6 --with-http_ssl_module --with-http_stub_status_module --with-http_realip_module --with-http_auth_request_module --with-http_addition_module --with-http_dav_module --with-http_geoip_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_image_filter_module --with-http_v2_module --with-http_sub_module --with-http_xslt_module --with-stream --with-stream_ssl_module --with-mail --with-mail_ssl_module --with-threads
```
This is my configure, and the `--with-http_gzip_static_module` is listed.  If you can not find it, you can refer this [doc](https://www.cnblogs.com/yanjieli/p/10615361.html) to add this module.

Then let's enable the static gzip and reload nginx:
```
http {
    ...
    # gzip  on;
    gzip_static  on;
 
    server {
        ...
    }
}
```

```
# service nginx reload
```

Let's do a test without the gzip header
```
# curl -o test.txt -D - -s  http://localhost/test.html
HTTP/1.1 200 OK
Server: nginx/1.10.3 (Ubuntu)
Date: Thu, 26 Nov 2020 06:54:26 GMT
Content-Type: text/html
Content-Length: 1550  <<<<<<
Last-Modified: Thu, 26 Nov 2020 06:31:38 GMT
Connection: keep-alive
ETag: "5fbf4bca-60e"
Accept-Ranges: bytes
```
The content is not compressed.


Let's add a gzip request header:
```
# curl -H "Accept-Encoding: gzip" -o test.txt -D - -s  http://localhost/test.html
HTTP/1.1 200 OK
Server: nginx/1.10.3 (Ubuntu)
Date: Thu, 26 Nov 2020 06:55:01 GMT
Content-Type: text/html
Content-Length: 1550 <<<<<<<<<
Last-Modified: Thu, 26 Nov 2020 06:31:38 GMT
Connection: keep-alive
ETag: "5fbf4bca-60e"
Accept-Ranges: bytes
```

Still not compressed!!!!

Reading the [doc](https://docs.nginx.com/nginx/admin-guide/web-server/compression/#sending-compressed-files) 
>In this case, to service a request for /path/to/file, NGINX tries to find and send the file /path/to/file.gz. If the file doesn’t exist, or the client does not support gzip, NGINX sends the uncompressed version of the file.

So we need a compressed file first
```
# gzip -k test.html
# ll | grep test.html
-rw-r--r--  1 root   root   1550 Nov 26 06:31 test.html
-rw-r--r--  1 root   root    894 Nov 26 06:31 test.html.gz
```

Let's try again
```
# curl -o test.txt -D - -s  http://localhost/test.html
HTTP/1.1 200 OK
Server: nginx/1.10.3 (Ubuntu)
Date: Thu, 26 Nov 2020 07:11:00 GMT
Content-Type: text/html
Content-Length: 1550
Last-Modified: Thu, 26 Nov 2020 06:31:38 GMT
Connection: keep-alive
ETag: "5fbf4bca-60e"
Accept-Ranges: bytes

# curl -H "Accept-Encoding: gzip" -o test.txt -D - -s  http://localhost/test.html
HTTP/1.1 200 OK
Server: nginx/1.10.3 (Ubuntu)
Date: Thu, 26 Nov 2020 07:11:09 GMT
Content-Type: text/html
Content-Length: 894  <<<<<<<<<<<
Last-Modified: Thu, 26 Nov 2020 06:31:38 GMT
Connection: keep-alive
ETag: "5fbf4bca-37e"
Content-Encoding: gzip  <<<<<<<<<
```

Now we can fine the compressed file with the content-length.


