```text
listen              1935;
max_connections     1000;
srs_log_tank        file;
srs_log_file        ./objs/srs.log;
daemon              off;
http_api {
    enabled         on;
    listen          1985;
}
http_server {
    enabled         on;
    listen          8080;
    dir             ./objs/nginx/html;
}
stats {
    network         0;
    disk            sda sdb xvda xvdb;
}
vhost __defaultVhost__ {
    gop_cache       off;
    queue_length    10;
    tcp_nodelay     on;
    hls {
        enabled         off;
    }
    http_remux {
        enabled     on;
    }
    transcode live-low {
        enabled     off;
        ffmpeg      /usr/local/srs/objs/ffmpeg/bin/ffmpeg;
        engine ff {
            enabled         on;
            vfilter {
            }
            vcodec          copy;
            vbitrate        5000;
            vfps            0;
            vwidth          1280;
            vheight         720;
            vthreads        2;
            vprofile        main;
            vpreset         medium;
            vparams {
            }
            acodec          copy;
            output          rtmp://test:12345;
        }
    }
    forward {
        enabled on;
        destination test.com:21935;
    }

}

vhost test {
    gop_cache       off;
    queue_length    10;
    tcp_nodelay     on;
    hls {
        enabled         off;
    }
    http_remux {
        enabled     on;
    }
    forward {
        enabled on;
        destination test.com:21935;
    }

}

```

```yaml
version: '3'
services:
  srs:
    image: ossrs/srs:3
    ports:
      - "1935:1935"
      - "1985:1985"
      - "8080:8080"
    volumes:
      - "/home/fi/tools/srs-docker/srs.conf:/usr/local/srs/conf/srs.conf"
      - "/home/fi/tools/srs-docker/srs.log:/usr/local/srs/objs/srs.log"
```