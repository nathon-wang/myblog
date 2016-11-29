title:  "使用OpenResty防缩略图盗链"
date: 2015-07-14 11:35:07
tags:
    - openresty
    - 技术
    - 原创
---

![基本的缩略图生成流程图](thumbnail-convert.png)

一个基本的缩略图生成流程如上图所示，最后生成的缩略图缓存储在服务器的本地缓存，通过nginx的虚拟目录机制，客户端可以进行下载。但是存在一个问题，因为通过虚拟目录，只要复制这个地址就可以绕过
验证机制随时访问缩略图，这很显然不是我们想要的。
我们的需求是：
1. 通过step1发起的请求能够正常访问缩略图，注意step1是有认证的请求
2. 绕过step1直接复制缩略图地址的请求静止访问缩略图

这不就是防盗链么，google一下，发现有一个叫Referer的头，但是Referer是能轻松绕过的。于是乎，我就在想，能不能在返回的链接里加个标记并缓存这个标记，访问虚拟目录时将缓存中的标记delete。如果再访问这个虚拟目录
发现没有这个标记，那么就将请求deny调。这样就可以防止盗链了。说干就干
首先要将虚拟目录请求设置为内部访问

~~~
location  /olc/image/thumbnail {
    internal; #此指令标示只能内部访问
    alias /caches/olc/thumbnail;
    autoindex off;
}
~~~

其次我要做一个到虚拟目录的代理

~~~
location  ~ /olc/image/thumbnail/([a-z0-9]*)/(.*) {
    set $access_id $1;
    set $resource_id $2;
    set $access_type "image/thumbnail";
    rewrite_by_lua_file 'scripts/olc_access_check.lua';
}
~~~

其中规则([a-z0-9]*)就是我们要加的tag，这个tag是根据某种规则生成的一个md5值。然后获取去tag我们要去做验证，如果通过验证就重写请求到/olc/image/thumbnail，否则就返回403。这就是scripts/olc_access_check.lua中的内容

~~~lua
local cache_ngx = ngx.shared.share_cache
local value = cache_ngx:get(ngx.var.access_id)
cache_ngx:delete(ngx.var.access_id)
if value == nil then
  ngx.exit(403)
else
    ngx.exec("/olc/" .. ngx.var.access_type .. "/" ..ngx.var.resource_id);
end
~~~~

当然因为get和delete不是原子操作，所以并发的情况下，还是有可能会有一些漏网之鱼，不过我认为完全是可以接受的。


这个方案最大的好处还在于，只用在nginx内部完成，不用改其他代码。下面的代码通过拦截从下载代理返回的链接地址，并提出其中的Location字段，然后附加上tag，再更新Location。整个过程对无论对客户端还是下载代理都是透明的。

~~~
location ~ /olc/image/view {
     proxy_pass http://127.0.0.1:6666;
     include vhosts/cfgs/proxy.cfg;
     header_filter_by_lua_file 'scripts/olc_access_set.lua';
}
~~~

~~~
local str = require "resty.string"

local location = ngx.resp.get_headers()["Location"]
if location ~= nil then
  local resource_path = string.match(location, ".*/")
  local resource_name = string.match(location, ".*/(.*)")
  local resource_type = string.match(string.sub(resource_path, 1, -1), ".*/(.*)")
  local stripped_resource_path = string.sub(resource_path, 1, -2)
  local resource_type = string.match(stripped_resource_path, ".*/(.*)")
  if resource_type == "thumbnail" then
    local tag = some_alg()

    local cache_ngx = ngx.shared.share_cache
    local succ, err, forcible = cache_ngx:set(tag, 1, 20)
    local new_location =  resource_path .. tag .. '/' .. resource_name
    ngx.header["Location"] = new_location
  end
end
~~~





