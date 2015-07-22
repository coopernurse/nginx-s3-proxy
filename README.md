
## Motivation

This image was created for use with dogestry. We wanted a caching HTTP proxy between our
servers and S3 so that images were only downloaded once from S3.

## Usage

The image assumes a config file in the container at: `/nginx.conf` so use the `-v` option to
mount one from your host.


```
docker run -p 8000:8000 -v /path/to/nginx.conf:/nginx.conf coopernurse/nginx-s3-proxy 
```

If you want to store the cache on the host, bind a path to `/data/cache`:

```
docker run -p 8000:8000 -v /path/to/nginx.conf:/nginx.conf -v /my/path:/data/cache coopernurse/nginx-s3-proxy 
```

Feel free to alter the `-p` param if you wish to bind the port differently onto the host.


Example nginx.conf file:

```
worker_processes 2;
pid /run/nginx.pid;
daemon off;

events {
	worker_connections 768;
}

http {
	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 65;
	types_hash_max_size 2048;
	server_names_hash_bucket_size 64;

	include /usr/local/nginx/conf/mime.types;
	default_type application/octet-stream;

	access_log /usr/local/nginx/logs/access.log;
	error_log  /usr/local/nginx/logs/error.log;

	gzip on;
	gzip_disable "msie6";
	gzip_http_version 1.1;
	gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

    proxy_cache_lock on;
    proxy_cache_lock_timeout 60s;
    proxy_cache_path /data/cache levels=1:2 keys_zone=s3cache:10m max_size=30g;

    server {
        listen     8000;

        location / {
            proxy_pass https://your-bucket.s3.amazonaws.com;

            aws_access_key your-access-key;
            aws_secret_key your-secret-key;
            s3_bucket your-bucket;

            proxy_set_header Authorization $s3_auth_token;
            proxy_set_header x-amz-date $aws_date;

            proxy_cache        s3cache;
            proxy_cache_valid  200 302  24h;
        }
    }
}
```

Things you want to tweak include:

* proxy_cache_path
  * alter max_size as desired
  * if you want the cache stored external to the container, alter the path
* proxy_pass
* aws_access_key
* aws_secret_key
* s3_bucket
* proxy_cache_valid - change 24h to your cache duration as desired.


