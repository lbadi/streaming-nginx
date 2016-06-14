# How-to streaming-server

## 0 - Instalar paquetes de dependencias

```
    apt-get install git gcc make libaio1 libpcre3-dev openssl libssl-dev ffmpeg -y
```

## 1 - Descargar nginx y el modulo rtmp

```
    wget http://nginx.org/download/nginx-1.9.4.tar.gz
    git clone https://github.com/arut/nginx-rtmp-module.git
```

## 2 - Compilar e instalar nginx

### 2.1 - Configurar la compilación

Ir al directorio donde se descargo nginx.

Configurar la compilación
```
	-/configure --prefix=/usr/local/nginx --with-file-aio --add-module=/Absolute/path/to/nginx-rtmp-module/
```

### 2.2 - Compilar e instalar

Ejecutar
```
	 make
	 make install
     ln -s /usr/local/nginx/sbin/nginx /usr/bin/nginx
```


## 3 - Configurar el servidor

Para configurar un servidor primer necesita tener un archivo de configuración. Aca adentro podra encontrar un archivo de configuración con varias aplicaciones rtmp. Si usted lo desea puede crear su propio archivo de configuración. Los archivos de configuración se encuentran en la carpeta conf.

### 3.1 Configuración de ejemplo

```
worker_processes  auto;
events {
    # Allows up to 1024 connections, can be adjusted
    worker_connections  1024;
}

# RTMP configuration
rtmp {
    server {
        listen 1935; # Listen on standard RTMP port
        chunk_size 4000; 
    
    # Application for Video on Demand (Vod)
    application vod {
       play /var/mp4s;
    }

    # Application for Webcam

    application webcam {
       live on;
       exec_static ffmpeg -f video4linux2 -i /dev/video0 -c:v libx264  -f flv rtmp://localhost:1935/webcam/mystream;
    }        

        # This application is to accept incoming stream
        application live {
            live on; # Allows live input
            
            # Once receive stream, transcode for adaptive streaming
            # This single ffmpeg command takes the input and transforms
            # the source into 4 different streams with different bitrate
            # and quality. P.S. The scaling done here respects the aspect
            # ratio of the input.
            exec ffmpeg -i rtmp://localhost/vod/sample.mp4 -async 1 -vsync -1
                        -c:v libx264 -c:a libvo_aacenc -b:v 256k -b:a 32k -vf "scale=480:trunc(ow/a/2)*2" -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://localhost/show/$name_low
                        -c:v libx264 -c:a libvo_aacenc -b:v 768k -b:a 96k -vf "scale=720:trunc(ow/a/2)*2" -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://localhost/show/$name_mid
                        -c:v libx264 -c:a libvo_aacenc -b:v 1024k -b:a 128k -vf "scale=960:trunc(ow/a/2)*2" -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://localhost/show/$name_high
                        -c:v libx264 -c:a libvo_aacenc -b:v 1920k -b:a 128k -vf "scale=1280:trunc(ow/a/2)*2" -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://localhost/show/$name_hd720
                        -c copy -f flv rtmp://localhost/show/$name_src;
        }
        
        # This application is for splitting the stream into HLS fragments
        application show {
            live on; # Allows live input from above
            hls on; # Enable HTTP Live Streaming
            
            # Pointing this to an SSD is better as this involves lots of IO
            hls_path /tmp/hls/;
            
            # Instruct clients to adjust resolution according to bandwidth
            hls_variant _low BANDWIDTH=288000; # Low bitrate, sub-SD resolution
            hls_variant _mid BANDWIDTH=448000; # Medium bitrate, SD resolution
            hls_variant _high BANDWIDTH=1152000; # High bitrate, higher-than-SD resolution
            hls_variant _hd720 BANDWIDTH=2048000; # High bitrate, HD 720p resolution
            hls_variant _src BANDWIDTH=4096000; # Source bitrate, source resolution
        }
    }
}

http {
    # See http://licson.net/post/optimizing-nginx-for-large-file-delivery/ for more detail
    # This optimizes the server for HLS fragment delivery
    sendfile off;
    tcp_nopush on;
    aio on;
    directio 512;
    
    # HTTP server required to serve the player and HLS fragments
    server {
        listen 9090;
        
    location /stat {
       rtmp_stat all;

       rtmp_stat_stylesheet stat.xsl;
    }

    location /stat.xsl {
       root /usr/local/nginx/conf/;
    }
    
        location / {
            root /path/to/web_player/;
        }
        
        location /hls {
            types {
                application/vnd.apple.mpegurl m3u8;
            }
            
            root /tmp/;
            add_header Cache-Control no-cache; # Prevent caching of HLS fragments
            add_header Access-Control-Allow-Origin *; # Allow web player to access our playlist
        }
    }
}

```
### 3.2 Explicación de algunas configuraciones

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

        location /stat.xsl {
            root /usr/local/nginx/conf/;
        }
```

En esta sección definimos donde obtener los stats de nginx.
Hay que colocar un .xsl para darle formato a las estadisticas.
En root hay que poner el path absoluto a la carpeta donde se encontrara el archivo stat.xsl

```
# This application is to accept incoming stream
        application live {
            live on; # Allows live input
            
            # Once receive stream, transcode for adaptive streaming
            # This single ffmpeg command takes the input and transforms
            # the source into 4 different streams with different bitrate
            # and quality. P.S. The scaling done here respects the aspect
            # ratio of the input.
            exec ffmpeg -i rtmp://localhost/vod/sample.mp4 -async 1 -vsync -1
                        -c:v libx264 -c:a libvo_aacenc -b:v 256k -b:a 32k -vf "scale=480:trunc(ow/a/2)*2" -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://localhost/show/$name_low
                        -c:v libx264 -c:a libvo_aacenc -b:v 768k -b:a 96k -vf "scale=720:trunc(ow/a/2)*2" -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://localhost/show/$name_mid
                        -c:v libx264 -c:a libvo_aacenc -b:v 1024k -b:a 128k -vf "scale=960:trunc(ow/a/2)*2" -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://localhost/show/$name_high
                        -c:v libx264 -c:a libvo_aacenc -b:v 1920k -b:a 128k -vf "scale=1280:trunc(ow/a/2)*2" -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://localhost/show/$name_hd720
                        -c copy -f flv rtmp://localhost/show/$name_src;
        }
```

En esta sección leemos de un stream y lo codificamos con diferentes calidades, para darle adaptabilidad al cliente dependiendo de su conexión.
Esto se utiliza con hls para variar el sub-stream dependiendo de la conexión del cliente.

```
application show {
            live on; # Allows live input from above
            hls on; # Enable HTTP Live Streaming
            
            # Pointing this to an SSD is better as this involves lots of IO
            hls_path /tmp/hls/;
            
            # Instruct clients to adjust resolution according to bandwidth
            hls_variant _low BANDWIDTH=288000; # Low bitrate, sub-SD resolution
            hls_variant _mid BANDWIDTH=448000; # Medium bitrate, SD resolution
            hls_variant _high BANDWIDTH=1152000; # High bitrate, higher-than-SD resolution
            hls_variant _hd720 BANDWIDTH=2048000; # High bitrate, HD 720p resolution
            hls_variant _src BANDWIDTH=4096000; # Source bitrate, source resolution
        }
```

En esta sección definimos el stream de http , como se puede ver definimos que substream de rtmp se utiliza dependiendo el bandwith que se tenga.


## 4 - Ejecutar

Para ejecutar hay basta correr.
```
nginx
```
Si se quiere ejecutar con una configuración diferente
```
nginx -c conf/laconfiguracion.
```
Para detener el server hay que ejecutar
```
nginx -s stop
```