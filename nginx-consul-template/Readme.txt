1. 根据教程配置consul-template以及负载均衡

https://learn.hashicorp.com/consul/integrations/nginx-consul-template

install consul consul-template nginx


/etc/nginx/conf.d/consulwebs.conf


server {
	listen       8081;
	server_name  _;
	root         /usr/share/nginx/html;
	index  8081.html;

	# Load configuration files for the default server block.
	include /etc/nginx/default.d/*.conf;

	location / {
	}

	error_page 404 /404.html;
		location = /40x.html {
	}

	error_page 500 502 503 504 /50x.html;
		location = /50x.html {
	}
}
server {
	listen       8082;
	server_name  _;
	root         /usr/share/nginx/html;
	index  8082.html;

	# Load configuration files for the default server block.
	include /etc/nginx/default.d/*.conf;

	location / {
	}

	error_page 404 /404.html;
		location = /40x.html {
	}

	error_page 500 502 503 504 /50x.html;
		location = /50x.html {
	}
}
server {
	listen       8083;
	server_name  _;
	root         /usr/share/nginx/html;
	index  8083.html;

	# Load configuration files for the default server block.
	include /etc/nginx/default.d/*.conf;

	location / {
	}

	error_page 404 /404.html;
		location = /40x.html {
	}

	error_page 500 502 503 504 /50x.html;
		location = /50x.html {
	}
}

将html/下的文件放入 /usr/share/nginx/html

service nginx reload


/etc/consul.d/web-service.json

{
  "services": [
  {
	"Id":"web1",
    "Name": "web",
    "Port": 8081,
    "check": {
      "args": ["curl", "localhost:8081"],
      "interval": "3s"
    }
  },
  {
	"Id":"web2",
    "Name": "web",
    "Port": 8082,
    "check": {
      "args": ["curl", "localhost:8082"],
      "interval": "3s"
	}
  },
  {
	"Id":"web3",
    "Name": "web",
    "Port": 8083,
    "check": {
      "args": ["curl", "localhost:8083"],
      "interval": "3s"
    }
  }
  ]
}

consul reload



/root/consul/consul-template-config.hcl：

consul {
address = "localhost:8500"
retry {
enabled = true
attempts = 12
backoff = "250ms"
}
}
template {
source      = "/etc/nginx/conf.d/load-balancer.conf.ctmpl"
destination = "/etc/nginx/conf.d/load-balancer.conf"
perms = 0600
command = "service nginx reload"
}


/etc/nginx/conf.d/load-balancer.conf.ctmpl:

upstream backend {
{{ range service "web" }}
  server {{ .Address }}:{{ .Port }};
{{ end }}
}

2.通过startup.sh启动consul及template


/root/startup.sh

#!/usr/bin/env bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
cd /root/consul
nohup consul-template -config=consul-template-config.hcl > consul-template.log 2>&1 &
nohup consul agent -server -ui -client=0.0.0.0 -bind=192.168.56.10 -bootstrap-expect=1 \
-data-dir=/tmp/consul -config-dir /etc/consul.d/ -enable-local-script-checks \
-node=node01 -datacenter=dc1 > /tmp/consul/consul.log 2>&1 &



3.校验是否成功
check:
consul port: 8500
cat /etc/nginx/conf.d/load-balancer.conf