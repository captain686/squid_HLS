# squid_HLS
```bash
mkdir -p /etc/squid/ssl_cert
cd /etc/squid/

# 生成 CA 私钥 ca.key 和自签名证书 ca.pem（有效期 10 年）
openssl req -new -x509 -days 3650 -nodes \
  -newkey rsa:2048 \
  -keyout ssl_cert/SquidCA.key \
  -out ssl_cert/SquidCA.pem \
  -subj "/C=CN/ST=Beijing/L=Beijing/O=SquidProxy/OU=Squid/CN=SquidCA"


openssl x509 -in ssl_cert/SquidCA.pem -outform DER -out ssl_cert/SquidCA.der
# openssl x509 -in ssl_cert/SquidCA.pem -outform DER -out ssl_cert/SquidCA.crt

# 自动获取 Squid 配置文件中定义的运行用户（默认为 proxy 或 squid）
SQUID_USER=$(grep -E '^cache_effective_user' /etc/squid/squid.conf | awk '{print $2}')
[ -z "$SQUID_USER" ] && SQUID_USER="proxy" # 如果没搜到，默认尝试使用 proxy

# 执行合并后的初始化流程
sudo mkdir -p /var/lib/squid && \
sudo chown -R $SQUID_USER:$SQUID_USER /var/lib/squid && \
sudo -u $SQUID_USER /usr/lib/squid/security_file_certgen -c -s /var/lib/squid/ssl_db -M 4MB

```

## squid.conf
```conf

# ============ 基本端口设置 =============
#http_port 3128
http_port 3128 ssl-bump cert=/etc/squid/ssl_cert/SquidCA.pem key=/etc/squid/ssl_cert/SquidCA.key generate-host-certificates=on dynamic_cert_mem_cache_size=4MB

sslcrtd_program /usr/lib/squid/security_file_certgen -s /var/lib/squid/ssl_db -M 4MB
sslcrtd_children 5

# TLS 连接参数（替代旧指令）
tls_outgoing_options options=NO_SSLv3 flags=DONT_VERIFY_PEER cipher=EECDH+ECDSA+AESGCM:EECDH+aRSA+AESGCM:EECDH+ECDSA+SHA384:EECDH+ECDSA+SHA256:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH+aRSA+RC4:EECDH:EDH+aRSA:!RC4:!aNULL:!eNULL:!LOW:!3DES:!MD5:!EXP:!PSK:!SRP:!DSS

# ============ SSL Bump 设置 ============
# SSL Bump（统一 bump，防止 pinning）
acl step1 at_step SslBump1
ssl_bump bump step1
ssl_bump bump all


# ============ 拦截 .m3u8 请求 ============
acl m3u8_files urlpath_regex -i \.m3u8$
# 定义 ACL：匹配所有 URL 路径中包含 .m3u8 的请求

# URL 重写程序配置，重写只针对匹配 m3u8_files 的请求
url_rewrite_program /etc/squid/squid-urlrewrite
url_rewrite_children 5

# 允许匹配的请求走 URL 重写程序
url_rewrite_access allow m3u8_files
# 其他请求不走重写程序
url_rewrite_access deny all

# ============ 基本访问控制 ============
acl localnet src 192.168.0.0/16
acl localnet src 172.16.0.0/12
http_access allow localnet
http_access allow localhost
http_access deny all

########################################
#            缓存配置设置
########################################

# 磁盘缓存设置
# 格式：cache_dir <类型> <目录> <大小(MB)> <一级目录数> <二级目录数>
cache_dir aufs /var/spool/squid 100 16 256

# 内存缓存大小（用于存储频繁访问的数据）
cache_mem 1024 MB

# 内存中缓存单个对象的最大大小
maximum_object_size_in_memory 1024 KB

# 磁盘上缓存单个对象的最大大小
maximum_object_size 128 MB

# 缓存刷新规则：
# refresh_pattern 正则匹配 URL，后面三个值分别表示最小过期时间、百分比过期时间和最大过期时间（单位均为分钟）
refresh_pattern ^ftp:      1440    20%     10080
refresh_pattern ^gopher:   1440    0%      1440
refresh_pattern -i (/cgi-bin/|\?) 0     0%      0
refresh_pattern .          0       20%     4320

# ============ 日志配置 ============
logformat custom_log %{ %Y-%m-%d %H:%M:%S}tl %tr %la %lp %>a %>p %>Hs %<st HTTP/%rv %rm "%ru" "%{Referer}>h" "%{User-Agent}>h" %Ss:%Sh/%<a

# 全量日志优化
access_log /var/log/squid/access.log custom_log
pid_filename /tmp/squid.pid

hosts_file /etc/hosts
# debug_options ALL,1 33,2 28,9

```

## squid-urlrewrite.conf
```conf
# example

# loglevel
# info: default
# debug: more detail info
# log messages are write to syslog
loglevel debug

# rewrite  <regexp> <target>
# redirect <regexp> [301;]<target>

# rewrite  ^https?://webserver\.domain\.com/file/(\d+)/  http://192.168.1.3:1234/backend/file/read?file_id=$1

# redirect ^(https?://)domain\.com/(.*)$                    301;$1www.domain.com/$2

# redirect ^(https?://)www\.domain2\.com/(.*)$      $1www2.domain2.com/$2
#
#redirect ^http[s]?://.*\.m3u8$       301;https://hlsfilter.com:8443/?url=$0

exclude_keyword hlsfilter.com

rewrite  ^http[s]?://[^/]+(?:/[^?#]*)*\.m3u8$      https://hlsfilter.com:8443/$0
# rewrite  ^http[s]?://.*\.m3u8$   https://www.baidu.com
#
```
