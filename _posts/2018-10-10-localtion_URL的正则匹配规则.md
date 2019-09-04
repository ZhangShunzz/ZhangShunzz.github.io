---
layout: post
title:  Localtion URL的正则匹配规则
date:   2018-10-10 09:32:00 +0800
categories: 技术
tag: Nginx
---


匹配的优先级顺序
---
`(localtion =) > (localtion完整url) > (localtion ^~) > (localtion ~,~*) > (lcoaltion部分起始路径) > (/)`

- = 表示精确匹配
- ^~ 表示指定的路径开头
- ~ 表示区分大小写的正则匹配
- ~* 表示不区分大小写的正则匹配
- / 通用匹配,所有的URL都是以此为开头

### 示例

	location / {
                        try_files $uri @apache;
                        }
	#所有的路径都是/开头,表示匹配所有
                location @apache {
                        internal;
                        proxy_pass http://127.0.0.1:1080;
                        include proxy.conf;
                        }
	#url重定向至@apache规则
                location ~ .*\.(php|php5)?$
                        {
                                proxy_pass http://127.0.0.1:1080;
                                include proxy.conf;
                        }
	#匹配所有以.php或者.php5的URL, ~表示区分大小写
                location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
                        {
                                expires      30d;
                        }
	#匹配以.gif,.jpg,.jpeg,.png,.bmp,.swf结尾的url
                location ~ .*\.(js|css)?$
                        {
                                expires      12h;
                        }
	#匹配以.js或者.css结尾的url

### 使用建议

	localtion = / {
    	proxy_pass http://127.0.0.1:1080/index.php;
	}
	#匹配根路径
	localtion ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {
    	root /web/static/;
	}
	#匹配所有静态文件
	localtion / {
    	proxy_pass http://127.0.0.1:1080/index.php;
	}
	#匹配所有的路径