http://subosito.com/fix-302-redirect-response-for-github-hosted-site/

ALIAS, not CNAME(ALIAS)

https://help.github.com/articles/tips-for-configuring-an-a-record-with-your-dns-provider/

~~~bash
dig wulfric.me +nocomments +nocmd +nostats
~~~

~~~
; <<>> DiG 9.8.3-P1 <<>> wulfric.me +nocomments +nocmd +nostats
;; global options: +cmd
;wulfric.me.			IN	A
wulfric.me.		3600	IN	A	xxx.xxx.xxx.xxx
~~~

这里的 IP 地址不再是 Github 给出的 IP 了

~~~bash
curl -I wulfric.me/archive/
~~~

~~~
HTTP/1.1 200 OK
Server: GitHub.com
Content-Type: text/html; charset=utf-8
Last-Modified: Fri, 17 Oct 2014 16:29:11 GMT
Expires: Sat, 18 Oct 2014 11:55:16 GMT
Cache-Control: max-age=600
Content-Length: 4786
Accept-Ranges: bytes
Date: Sat, 18 Oct 2014 11:45:16 GMT
Via: 1.1 varnish
Age: 0
Connection: keep-alive
X-Served-By: cache-lcy1125-LCY
X-Cache: MISS
X-Cache-Hits: 0
X-Timer: S1413632716.122644,VS0,VE85
Vary: Accept-Encoding
~~~

常规的绑定域名方式：A CNAME(ALIAS) to github 给出的 IP 地址