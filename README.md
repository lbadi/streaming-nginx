# How-to streaming-server

## 1-Clonar el repositorio

```
	$git clone https://github.com/lbadi/streaming-nginx.git
```

## 2-Compilar e instalar nginx

### 2.1 - Configurar la compilación

Ir al directorio donde se clono el proyecto.

Configurar la compilación
```
	-/configure --add-module=/Absolute/path/to/nginx-rtmp-module/ --with-openssl=/absolute/path/to/openssl-1.0.1t --without-http_rewrite_module
```

### 2.2 - Compilar e instalar

Ejecutar
```
	 make
	 make install
```

### 2.3 - Instalar FFMPEG (Opcional)

FFMPEG es una herramienta para convertir video y audio en diferentes formatos. Se puede obtener desde el sitio oficial https://ffmpeg.org . En algunas configuraciones que mostraremos a continuación se utiliza para cambiar los codec.

```
# cd /my/path/where/i/keep/compiled/stuff
# git clone http://source.ffmpeg.org/git/ffmpeg.git
# cd ffmpeg
# ./configure --enable-gpl --enable-libx264
# make
# make install
# ldconfig
```

## 3-Configurar el servidor

Para configurar un servidor primer necesita tener un archivo de configuración. Aca adentro podra encontrar un archivo de configuración con varias aplicaciones rtmp. Si usted lo desea puede crear su propio archivo de configuración. Los archivos de configuración se encuentran en la carpeta conf.

### 3-1 Configuración de ejemplo

```
events {
	worker_connections 1024;
}

rtmp {

    server {

        listen 1935;

        chunk_size 4000;

        # TV mode: one publisher, many subscribers
        application mytv {

            # enable live streaming
            live on;

            # record first 1K of stream
            record all;
            record_path /tmp/av;
            record_max_size 1K;

            # append current timestamp to each flv
            record_unique on;

            # publish only from localhost
            allow publish 127.0.0.1;
            deny publish all;

            #allow play all;
        }

        # Transcoding (ffmpeg needed)
        application big {
            live on;

            # On every pusblished stream run this command (ffmpeg)
            # with substitutions: $app/${app}, $name/${name} for application & stream name.
            #
            # This ffmpeg call receives stream from this application &
            # reduces the resolution down to 32x32. The stream is the published to
            # 'small' application (see below) under the same name.
            #
            # ffmpeg can do anything with the stream like video/audio
            # transcoding, resizing, altering container/codec params etc
            #
            # Multiple exec lines can be specified.

            exec ffmpeg -re -i rtmp://localhost:1935/$app/$name -vcodec flv -acodec copy -s 32x32
                        -f flv rtmp://localhost:1935/small/${name};
        }

        application small {
            live on;
            # Video with reduced resolution comes here from ffmpeg
        }

        application webcam {
            live on;

            # Stream from local webcam
            exec_static ffmpeg -f video4linux2 -i /dev/video0 -c:v libx264 -an
                               -f flv rtmp://localhost:1935/webcam/mystream;
        }

        # video on demand
        application vod {
            play /var/flvs;
        }

        application vod2 {
            play /var/mp4s;
        }

        # Many publishers, many subscribers
        # no checks, no recording
        application videochat {

            live on;

            # The following notifications receive all
            # the session variables as well as
            # particular call arguments in HTTP POST
            # request

            # Make HTTP request & use HTTP retcode
            # to decide whether to allow publishing
            # from this connection or not
            on_publish http://localhost:8080/publish;

            # Same with playing
            on_play http://localhost:8080/play;

            # Publish/play end (repeats on disconnect)
            on_done http://localhost:8080/done;

            # All above mentioned notifications receive
            # standard connect() arguments as well as
            # play/publish ones. If any arguments are sent
            # with GET-style syntax to play & publish
            # these are also included.
            # Example URL:
            #   rtmp://localhost/myapp/mystream?a=b&c=d

            # record 10 video keyframes (no audio) every 2 minutes
            record keyframes;
            record_path /tmp/vc;
            record_max_frames 10;
            record_interval 2m;

            # Async notify about an flv recorded
            on_record_done http://localhost:8080/record_done;

        }


        # HLS

        # For HLS to work please create a directory in tmpfs (/tmp/hls here)
        # for the fragments. The directory contents is served via HTTP (see
        # http{} section in config)
        #
        # Incoming stream must be in H264/AAC. For iPhones use baseline H264
        # profile (see ffmpeg example).
        # This example creates RTMP stream from movie ready for HLS:
        #
        # ffmpeg -loglevel verbose -re -i movie.avi  -vcodec libx264
        #    -vprofile baseline -acodec libmp3lame -ar 44100 -ac 1
        #    -f flv rtmp://localhost:1935/hls/movie
        #
        # If you need to transcode live stream use 'exec' feature.
        #
        application hls {
            live on;
            hls on;
            hls_path /tmp/hls;
        }

        # MPEG-DASH is similar to HLS

        application dash {
            live on;
            dash on;
            dash_path /tmp/dash;
        }
    }
}

# HTTP can be used for accessing RTMP stats
http {

    server {

        listen      8081;

        # This URL provides RTMP statistics in XML
        location /stat {
            rtmp_stat all;

            # Use this stylesheet to view XML as web page
            # in browser
            rtmp_stat_stylesheet stat.xsl;
        }

        location /stat.xsl {
            # XML stylesheet to view RTMP stats.
            # Copy stat.xsl wherever you want
            # and put the full directory path here
            root /path/to/stat.xsl/;
        }

        location /hls {
            # Serve HLS fragments
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root /tmp;
            add_header Cache-Control no-cache;
        }

        location /dash {
            # Serve DASH fragments
            root /tmp;
            add_header Cache-Control no-cache;
        }
    }
}
```
### 3-2 Explicación de algunas configuraciones

```
    application vod {
        play /var/flvs;
    }
```

En la carpeta /var/flvs se deberian poner los archivos para ser reproducidos. Esta configuración se puede cambiar por cualquier otra.

```
application webcam {
            live on;

            # Stream from local webcam
            exec_static ffmpeg -f video4linux2 -i /dev/video0 -c:v libx264 -an
                               -f flv rtmp://localhost:1935/webcam/mystream;
        }
```

Esta aplicación lee el feed de video desde /dev/video0 , lo convierte utilizando ffmpeg con el codec libx264 y lo publica en rtmp://localhost:1935/webcam/mystream;

```
# This URL provides RTMP statistics in XML
        location /stat {
            rtmp_stat all;

            # Use this stylesheet to view XML as web page
            # in browser
            rtmp_stat_stylesheet stat.xsl;
        }
```

En esta sección definimos donde obtener los stats de nginx.

## Ejecutar

Para ejecutar hay que correr ./nginx . Si se quiere ejecutar con una configuración diferente ./nginx -c conf/laconfiguracion.