### API网关

> 由于此项目是模块化、微服务化，因此需要在借助API网关，需要在API网关注册，此步骤是必须的。

**安装openresty**

```shell
yum update
yum install yum-utils -y
yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo
yum install openresty -y
yum install openresty-resty -y
```

**部署网关**
```shell
cd /opt/codo/ && git clone https://github.com/ss1917/api-gateway.git
\cp -arp api-gateway/* /usr/local/openresty/nginx/
```

**修改配置**
> 主要修改`nginx.conf`配置信息和`config.lua`配置，具体参考API网关块：[API网关修改配置](https://github.com/ss1917/api-gateway/blob/master/README.md#%E4%BA%8C-%E4%BF%AE%E6%94%B9%E9%85%8D%E7%BD%AE)

接下来配置：

    因为我把前端静态文件也使用 网关进行代理 所以配置文件如下

**全局nginx配置**

```nginx
#  /usr/local/openresty/nginx/conf/nginx.conf
user root;
worker_processes auto;
worker_rlimit_nofile 51200;
error_log logs/error.log;
events {
    use epoll;
    worker_connections 51024;
}
http {
    #设置默认lua搜索路径
    lua_package_path '$prefix/lua/?.lua;/blah/?.lua;;';
    lua_code_cache on;      #线上环境设置为on, off时可以热加载lua文件
    lua_shared_dict user_info 1m;
    lua_shared_dict my_limit_conn_store 100m;   #100M可以放1.6M个键值对
    include             mime.types;    #代理静态文件

    client_header_buffer_size 64k;
    large_client_header_buffers 4 64k;

    init_by_lua_file lua/init_by_lua.lua;       # nginx启动时就会执行
    include ./conf.d/*.conf;                    # lua生成upstream
    resolver 10.10.10.12;                       # 内部DNS服务器地址
}
```


**网关配置**
```shell

# /usr/local/openresty/nginx/conf/conf.d/gw.conf
    server {
        listen 80;
        server_name gw.yanghongfei.me;
        lua_need_request_body on;           # 开启获取body数据记录日志

        location / {
            ### ws 支持
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            ### 获取真实IP
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            access_by_lua_file lua/access_check.lua;
            set $my_upstream $my_upstream;
            proxy_pass http://$my_upstream;

            ### 跨域
            add_header Access-Control-Allow-Methods *;
            add_header Access-Control-Max-Age 3600;
            add_header Access-Control-Allow-Credentials true;
            add_header Access-Control-Allow-Origin $http_origin;
            add_header Access-Control-Allow-Headers $http_access_control_request_headers;
            if ($request_method = OPTIONS){
                return 204;}
        }
    }

```

**前端资源配置**

```shell
#前端vhosts
mkdir -p /usr/local/openresty/nginx/conf/conf.d/
# /usr/local/openresty/nginx/conf/conf.d/demo.conf
# 这里是前端的访问入口，如果不使用网关代理静态的话，可以使用nginx代理，请根据自身情况修改配置。
server {
        listen       80;
        server_name demo.opendevops.cn;
        access_log /var/log/nginx/f_access.log;
        error_log  /var/log/nginx/f_error.log;
        root /var/www/codo;

        location / {
                    root /var/www/codo;
                    index index.html index.htm;
                    try_files $uri $uri/ /index.html;
                    }

        location /api {
                ### ws 支持
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

                add_header 'Access-Control-Allow-Origin' '*';
                proxy_pass http://gw.opendevops.cn;
        }

        location ~ /(.svn|.git|admin|manage|.sh|.bash)$ {
            return 403;
        }
}

```

**注册API网关**
```
vim /usr/local/openresty/nginx/lua/configs.lua
#修改lua配置文件里面数据库连接信息，和域名信息

json = require("cjson")

--mysql_config = {
--    host = "127.0.0.1",
--    port = 3306,
--    database = "lua",
--    user = "root",
--    password = "",
--    max_packet_size = 1024 * 1024
--}

redis_config = {
    host = '${DEFAULT_REDIS_HOST}',
    --host = '172.16.0.121',
    port = ${DEFAULT_REDIS_PORT},
    auth_pwd = '${DEFAULT_REDIS_PASSWORD}',
    db = 8,
    alive_time = 3600 * 24 * 7,
    channel = 'gw'
}

--mq_conf = {
--  host = '172.16.0.121',
--  port = 5672,
--  username = 'sz',
--  password = '123456',
--  vhost = '/'
--}

token_secret = "${token_secret}"
logs_file = '/var/log/gw.log'

--刷新权限到redis接口
rewrite_cache_url = 'http://${mg_domain}:8010/v2/accounts/verify/'
rewrite_cache_token = '8b888a62-3edb-4920-b446-697a472b4001'  #这里需要和codo_admin settings里面token一致

--并发限流配置
limit_conf = {
    rate = 10, --限制ip每分钟只能调用n*60次接口
    burst = 10, --桶容量,用于平滑处理,最大接收请求次数
}

--upstream匹配规则
gw_domain_name = '${api_gw_url}'

rewrite_conf = {
    [gw_domain_name] = {
        rewrite_urls = {
            {
                uri = "/cmdb",
                rewrite_upstream = "${cmdb_domain}:8002"
            },
            {
                uri = "/task",
                rewrite_upstream = "${task_domain}:8020"
            },
            {
                uri = "/cron",
                rewrite_upstream = "${LOCALHOST_IP}:9900"
            },
            {
                uri = "/mg",
                rewrite_upstream = "${mg_domain}:8010"
            },
            {
                uri = "/accounts",
                rewrite_upstream = "${mg_domain}:8010"
            },
        }
    }
}
```

**API网关启动**
```shell
#OpenResty 是一个基于 Nginx 与 Lua 的高性能 Web 平台，使用的也是80端口，若不能启动请检查你的80端口是否被占用了

openresty -t   #测试
systemctl start openresty
systemctl enable openresty

```

