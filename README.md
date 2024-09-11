nginx
=====

This role installs and configures the nginx web server. The user can specify
any http configuration parameters they wish to apply their site. Any number of
sites can be added with configurations of your choice.

Requirements
------------

This role requires Ansible 2.6.2 or higher and platform requirements are listed
in the metadata file.

Role Variables
--------------

The variables that can be passed to this role and a brief description about
them are as follows.

```
# The max clients allowed
nginx_max_clients: 512 

# The user to run nginx
nginx_user: "www-data"

# A list of hashs that define the servers for nginx,
# as with http parameters. Any valid server parameters
# can be defined here.
nginx_sites:
 default:
     - listen 80
     - server_name _
     - root "/usr/share/nginx/html"
     - index index.html
 foo:
     - listen 8080
     - server_name localhost
     - root "/tmp/site1"
     - location / { try_files $uri $uri/ /index.html; }
     - location /images/ { try_files $uri $uri/ /index.html; }
 bar:
     - listen 9090
     - server_name ansible
     - root "/tmp/site2"
     - location / { try_files $uri $uri/ /index.html; }
     - location /images/ {
         try_files $uri $uri/ /index.html;
         allow 127.0.0.1;
         deny all;
       }

# A list of hashs that define additional configuration
nginx_configs:
  proxy:
      - proxy_set_header X-Real-IP  $remote_addr
      - proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
  upstream:
      - upstream foo { server 127.0.0.1:8080 weight=10; }
  geo:
      - geo $local {
          default 0;
          127.0.0.1 1;
        }
  gzip:
      - gzip on
      - gzip_disable msie6

# A list of hashs that define user/password files
nginx_auth_basic_files:
   demo:
     - foo:$apr1$mEJqnFmy$zioG2q1iDWvRxbHuNepIh0 # foo:demo , generated by : htpasswd -nb foo demo
     - bar:$apr1$H2GihkSo$PwBeV8cVWFFQlnAJtvVCQ. # bar:demo , generated by : htpasswd -nb bar demo

```

Configuring protection against bots
===================================

Some ways to limit the bots' request rates:

1) Using the `robots.txt` file. But unfortunately some bots don't follow the
robots.txt settings.

2) Denying access to bots based on the `user-agent` header. This solution is
only for bots that are considered unnecessary or harmful.

3) Using nginx to limit requests: https://www.nginx.com/blog/rate-limiting-nginx/

The three options mentioned above can be configured with this ansible role.

The following links contain very interesting examples and settings:

* https://www.freecodecamp.org/news/nginx-rate-limiting-in-a-nutshell-128fe9e0126c/

* https://www.unixteacher.org/blog/blocking-access-by-user-agent-in-nginx/

* https://serverfault.com/questions/549332/how-to-set-robots-txt-globally-in-nginx-for-all-virtual-hosts

* https://www.ryadel.com/en/nginx-request-rate-limit-protect-web-site-http-request-flood-dos-brute-force/

* https://medium.com/@christopherphillips_88739/rate-limiting-in-nginx-5af7511ab3ce

Using a global robots.txt for all sites
------------------------------------------

It is enabled with:

```
nginx_use_global_robots_txt: True
```

By default, it adds the following `robots.txt` to all sites:

```
"User-agent: *"
"Crawl-delay: 60"
```

To use a different `robots.txt` file, please use the `nginx_global_robots_txt`
list. For instance:

```
nginx_global_robots_txt:
  - "User-agent: Googlebot"
  - "Allow: /"
  - "Crawl-delay: 60"
```

Denying access to bots based on `user-agent` header
------------------------------------------------------

Just enable the following variable:

```
nginx_use_bots_deny_list: True
```

It blocks the access of bots whose `user-agent` header matches the
`nginx_bots_deny_list` list. You can use a custom list of `user-agent` strings
that will be proccessed by nginx as a regex:

```
nginx_bots_deny_list:
  - bot_user_agent_header_1
  - bot_user_agent_header_2
  - bot_user_agent_header_3
```

The default `http-status` output is "403", but you can redefine it with the
`nginx_bots_deny_list_status` variable.

You can add exceptions to IP addresses or networks. In case you want to apply default rate limit to `1.2.3.0/24` network and "4.5.6.7" IP address regardless the User-Agent value:

```
nginx_bots_allowed_networks:
  - "1.2.3.0/24"
  - "4.5.6.7"
```

And about exceptions to User-Agent list, you can set default rate limit for `Googlebot` regardless the values in `nginx_bots_request_limit_user_agents_regex`: 

```
nginx_bots_request_limit_user_agents_regex_allowed:
  - "Googlebot\/"
```

Using nginx request limit
----------------------------

This role allows to configure a request limit for bots (based on `user-agent`
header) and the other connections (the ones that do not match our bot filter). 

To enable bots request limit, set:

```
nginx_use_bots_request_limit: True
```

To enable request rate limit on other connections:

```
nginx_use_default_request_limit: True
```

Both settings can be activated at the same time or independently.

There are some variables that you can use for fine tuning the request rate limit:

```
nginx_bots_request_limit_rate: "1r/m"
nginx_bots_request_limit_burst: 2
nginx_bots_request_limit_nodelay: "yes"
nginx_bots_request_limit_shared_memory_zone: "10m"
nginx_bots_request_limit_use_x_forwarded_for: False

nginx_default_request_limit_rate: "100r/s"
nginx_default_request_limit_burst: 15
nginx_default_request_limit_nodelay: "yes"
nginx_default_request_limit_shared_memory_zone: "50m"
nginx_default_request_limit_use_x_forwarded_for: False

nginx_request_limit_status: 429
```

More info at: https://www.nginx.com/blog/rate-limiting-nginx/

Sometimes it is better to apply the rate limit based on `x_forwarded_for`
header instead of source ip address, for instance when there's a reverse proxy
as frontend and we are configuring the request rate limit in the backend
server.

To use the `x_forwarded_for` header, just enable them:

```
nginx_bots_request_limit_use_x_forwarded_for: True
nginx_default_request_limit_use_x_forwarded_for: True
```

The default http status when reaching the rate limit is 429. It can be changed
in `nginx_request_limit_status` variable:

```
nginx_request_limit_status: 429
```

Examples
========

1) Install nginx with HTTP directives of choices, but with no sites
configured and no additionnal configuration:

    - hosts: all
      roles:
      - {role: nginx,
         nginx_http_params: ["sendfile on", "access_log /var/log/nginx/access.log"]
                              }


2) Install nginx with different HTTP directives than previous example, but no
sites configured and no additionnal configuration.

    - hosts: all
      roles:
      - {role: nginx,
         nginx_http_params: ["tcp_nodelay on", "error_log /var/log/nginx/error.log"]}

Note: Please make sure the HTTP directives passed are valid, as this role
won't check for the validity of the directives. See the nginx documentation
for details.

3) Install nginx and add a site to the configuration.

    - hosts: all

      roles:
      - role: nginx,
        nginx_http_params:
          - sendfile "on"
          - access_log "/var/log/nginx/access.log"
        nginx_sites:
          bar:
            - listen 8080
            - location / { try_files $uri $uri/ /index.html; }
            - location /images/ { try_files $uri $uri/ /index.html; }
        nginx_configs:
          proxy:
            - proxy_set_header X-Real-IP  $remote_addr
            - proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for

Note: Each site added is represented by list of hashes, and the configurations
generated are populated in /etc/nginx/site-available/, a link is from /etc/nginx/site-enable/ to /etc/nginx/site-available

The file name for the specific site configurtaion is specified in the hash
with the key "file_name", any valid server directives can be added to hash.
Additional configuration are created in /etc/nginx/conf.d/

4) Install Nginx , add 2 sites (different method) and add additional configuration

    ---
    - hosts: all
      roles:
        - role: nginx
          nginx_http_params:
            sendfile: "on"
            access_log: "/var/log/nginx/access.log"
          nginx_sites:
             foo:
               - listen 8080
               - server_name localhost
               - root /tmp/site1
               - location / { try_files $uri $uri/ /index.html; }
               - location /images/ { try_files $uri $uri/ /index.html; }
             bar:
               - listen 9090
               - server_name ansible
               - root /tmp/site2
               - location / { try_files $uri $uri/ /index.html; }
               - location /images/ { try_files $uri $uri/ /index.html; }
          nginx_configs:
             proxy:
                - proxy_set_header X-Real-IP  $remote_addr
                - proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for

5) Install Nginx , add 2 sites, add additional configuration and an upstream configuration block

    ---
    - hosts: all
      roles:
        - role: nginx
          nginx_http_params:
            sendfile: "on"
            access_log: "/var/log/nginx/access.log"
          nginx_sites:
            foo:
               - listen 8080
               - server_name localhost
               - root /tmp/site1
               - location / { try_files $uri $uri/ /index.html; }
               - location /images/ { try_files $uri $uri/ /index.html; }
            bar:
               - listen 9090
               - server_name ansible
               - root /tmp/site2
               - if ( $host = example.com ) { rewrite ^(.*)$ http://www.example.com$1 permanent; }
               - location / { try_files $uri $uri/ /index.html; }
               - location /images/ { try_files $uri $uri/ /index.html; }
               - auth_basic            "Restricted"
               - auth_basic_user_file  auth_basic/demo
          nginx_configs:
            proxy:
                - proxy_set_header X-Real-IP  $remote_addr
                - proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
            upstream:
                # Results in:
                # upstream foo_backend {
                #   server 127.0.0.1:8080 weight=10;
                # }
                - upstream foo_backend { server 127.0.0.1:8080 weight=10; }
          nginx_auth_basic_files:
            demo:
               - foo:$apr1$mEJqnFmy$zioG2q1iDWvRxbHuNepIh0 # foo:demo , generated by : htpasswd -nb foo demo
               - bar:$apr1$H2GihkSo$PwBeV8cVWFFQlnAJtvVCQ. # bar:demo , generated by : htpasswd -nb bar demo

Create a default site to deny all requests without a configured virtualhost
---------------------------------------------------------------------------

All these requests to non configured virtualhosts are going to be denied with 444 status code (specific to the Nginx - it closes connection to the server without any response). This new site will also not send any line to logs (access or error).

Just set as `True` the `nginx_use_default_deny` variable (default False). Then the role will create
a site `default-deny` (and it's self-signed certificate):

```
server {

   listen 80 default_server;
   listen [::]:80 default_server;
   listen 443 ssl http2 default_server;
   listen [::]:443 ssl http2 default_server;
   server_name _;
   ssl_ciphers aNULL;
   ssl_session_tickets off;
   ssl_certificate /etc/nginx/ssl-certs/default_deny.crt;
   ssl_certificate_key /etc/nginx/ssl-certs/default_deny.pem;
   return 444;
   access_log off;
   error_log /dev/null;

}
```

Dependencies
------------

None

License
-------

BSD

Author Information
------------------

- Original : Benno Joy
- Modified by : DAUPHANT Julien
- Modified by : Artefactual Systems

