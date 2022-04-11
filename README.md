# Live Stream

A live-streaming app working over RTMP protocol using Apache,node.js and Nginx which will work just fine on any OS and
any server

https://obsproject.com/forum/threads/how-to-set-up-your-own-private-rtmp-server-using-nginx.12891/

jwplayer("myElement").setup({
  file: "rtmp://<Ubuntu VM server ip>/live/flv:test",
  height: 360,
  width: 640
});

### Dig Notes
https://askubuntu.com/questions/460710/why-is-the-index-php-downloaded-instead-of-rendered-by-nginx

https://blog.twitch.tv/en/2017/10/10/live-video-transmuxing-transcoding-f-fmpeg-vs-twitch-transcoder-part-i-489c1c125f28/

https://www.nginx.com/blog/video-streaming-for-remote-learning-with-nginx/

ffmpeg -re -I bbb_sunflower_1080p_60fps_normal.mp4 -vcodec copy -loop -1 -c:a aac -b:a 160k -ar 44100 -strict -2 -f flv rtmp:192.168.1.138/live/bbb

https://github.com/arut/nginx-rtmp-module/issues/1174
rec folder must be created manually

https://github.com/arut/nginx-rtmp-module/issues/1184

    exec_record_done /bin/ffmpeg -i $path  -f mp4 /tmp/live/$basename.mp4;

https://github.com/arut/nginx-rtmp-module/wiki/Directives#record_path

```
exec ffmpeg -i rtmp://localhost/src/$name
              -c:a aac -b:a 32k  -c:v libx264 -b:v 128K -f flv rtmp://localhost/hls/$name_low
              -c:a aac -b:a 64k  -c:v libx264 -b:v 256k -f flv rtmp://localhost/hls/$name_mid
              -c:a aac -b:a 128k -c:v libx264 -b:v 512K -f flv rtmp://localhost/hls/$name_hi;
```


https://github.com/arut/nginx-rtmp-module/wiki/Directives#record_path


application naming - @link https://stackoverflow.com/questions/59922977/how-to-stream-with-ffmpeg-and-nginx-rtmp

```
exec_push ffmpeg
                -i rtmp://localhost/live/$name 
                -c:v libx264 
                -c:a copy 
                -preset veryfast 
                -profile:v high 
                -level 4.1
                -x264-params "nal-hrd=cbr" "opencl=true"
                -b:v 8000K 
                -minrate 8000K 
                -maxrate 8000K
                -keyint 2
                -s 1920x1080
                push rtmp://live-lhr03.twitch.tv/app/STREAM_KEY;
```

# Configuration and Usage

- install an Apache server and run it
- download and put the content of **public_html** into your Apache server index directory
- Download and install nginx
- Go to the path where you have installed nginx, Go to conf directory, open ```nginx.conf``` or create it if not exists
- clear the file content and paste the following configuration into the file and save it

```
worker_processes  1;


events {
  worker_connections  1024;
}


rtmp {

    server {

        listen 9935;

       application live {
           live on;
           
           deny play all;

           push rtmp://127.0.0.1:9935/hls;

           on_publish http://127.0.0.1:8000/;
           on_publish_done http://127.0.0.1:8000/;
           
           # Push, restream RTMP
           # push rtmp://live.twitch.tv/app/YOUR_TWITCH_KEY;
           # push rtmp://a.rtmp.youtube.com/live2/YOUR_YOUTUBE_KEY;
           # or dynamically using exec and ffmpeg @link https://stackoverflow.com/questions/70282613/nginx-rtmp-module-restream-to-a-dynamic-address/70749254#70749254
           
       
       }

      application src {
      
            live on;

            exec ffmpeg -i rtmp://localhost/src/$name
              -c:a aac -b:a 32k  -c:v libx264 -b:v 128K -f flv rtmp://localhost/hls/$name_low
              -c:a aac -b:a 64k  -c:v libx264 -b:v 256k -f flv rtmp://localhost/hls/$name_mid
              -c:a aac -b:a 128k -c:v libx264 -b:v 512K -f flv rtmp://localhost/hls/$name_hi;
        
      }

       application hls {
           live on;
           record all;
           allow publish 127.0.0.1;
           deny publish all;
           deny play all;

           hls on;
           hls_path /Users/richardmiles/IdeaProjects/nginx_live_stream/live;
           hls_nested on;
          
           hls_variant _low BANDWIDTH=160000;
           hls_variant _mid BANDWIDTH=320000;
           hls_variant _hi  BANDWIDTH=640000;
        
           record_unique on;
           record_path  /Users/richardmiles/IdeaProjects/nginx_live_stream/rec;
           record_interval 15m;

       }
    }
}

http {
    server {
        listen 9090;
    
        location /stat {
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }
        location /stat.xsl {
            root /path/to/stat/xsl/file;
        }
    }
}
```

    **_Notice:_** don't forget to change __replace__ ```path_to_public_html``` with absolute path to where you put the content of **public_html**

- download the node.js app, install it and run server.js
- download and install OBS (download from [here](https://obsproject.com/download))
- open OBS, go to settings->stream
- set RTMP url on __rtmp://your.host/live__
- set the stream key on whatever you want and apply settings
- go back and hit **Start Streaming**
- after you started streaming, open your browser and type __http://your.host__, you'll see a list of live streams on
  your server, click on one of them to play
- BOM ! you have implemented a live steram

# How It Works?

- OBS (or other RTMP streaming client) records the stream and sends it to RTMP host that you enter in OBS settings (on
  port 9935 which Nginx is listening)
- Nginx recieves the stream, triggers publish event, which sends stream data (not content of stream, jsut general data
  like stream name, the event name ,etc) to __http://127.0.0.1:8000__ which node.js server is listening on, this event
  callback is defined by this line:

  ```on_publish http://127.0.0.1:8000/;```

- node.js server gets the data, checks event and sends response to Nginx, Nginx catches data from __HTTP headers__, so
  that if server.js return 3** status with Location header, Nginx will publish the incoming stream on the value of
  Location header

  consider that when you set **on_publish** event in Nginx server config, Nginx will wait for response
  from __http://127.0.0.1:8000/__ after getting each stream, if it recieves status code 3**, will redirect stream to the
  name passed in __Location header__, if 2**, it will continue RTMP session and otherwise, RTMP connection will drop

  also note that server.js defines an array named ```streams``` which holds name of live streams, the array is pushed on
  each ```on_publish``` and is spliced on each ```on_publish_done``` (triggers when stream ends) event

- since we have the line :

  ```hls_nested on;```

  in ```nginx.cong``` file, Nginx will create HLS manifest and segments in the path specified by this line :

  ```hls_path path_to_public_html/live/<stream_name>;```

  <stream_name> is the value which server.js returns in __Location header__
- now, when you enter __http://your.host__ in your browser, apache returns index.html which then loads index.js, then
  javascript sends XHR HTTP to port 333 which server.js is listening on and will return a JSON encoded array of live
  stream names
- JS decodes string and displays live streams in links like :

  ```http://your.host/display.html?name=<stream_name>```

- by clicking on any link, you'll go to ```display.html``` page which has a ```display.js``` aside
- ```display.js``` gets stream name and sets

```http://your.host/live/<stream_name>/index.m3u8```

as HLS manifest and finally HLS gets segments and play them by turn

For more details on Nginx configuration and what every line means, take a
look [here](https://github.com/arut/nginx-rtmp-module/wiki/Directives)

# Notice

This app is implemented just to give you initial imagination of how you can create a live streaming website and needs
lots of completions (especially in security aspects)

# At Last

Feel free to open issues if detected and ask any question you have
