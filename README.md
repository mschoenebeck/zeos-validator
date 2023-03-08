How to install Janus:

install:
`sudo apt-get install libavutil-dev libavcodec-dev libavformat-dev`
for post processing of video (https://ourcodeworld.com/articles/read/1198/how-to-join-the-audio-and-video-mjr-from-a-recorded-session-of-janus-gateway-in-ubuntu-18-04)

make sure to `--enable-post-processing` when calling `./configure` when following the instructions on:
./configure --prefix=/opt/janus --enable-post-processing
https://github.com/meetecho/janus-gateway

install libsrtp 2.2.0 as mentioned in the readme (newer versions don't seem to work as of now):

```
wget https://github.com/cisco/libsrtp/archive/v2.2.0.tar.gz
tar xfv v2.2.0.tar.gz
cd libsrtp-2.2.0
./configure --prefix=/usr --enable-openssl
make shared_library && sudo make install
```

create systemd and log rotate file as described here:
https://facsiaginsa.com/janus/install-janus-webrtc-server-on-ubuntu


# Postprocess videos:
janus-pp-rec ./room-1234-user-0001-video.mjr ./video-track.webm
janus-pp-rec ./room-1234-user-0001-audio.mjr ./audio-track.opus
ffmpeg -i ./video-track.webm -i ./audio-track.opus  -c:v copy -c:a opus -strict experimental ./final-video.webm
ffmpeg -i ./final-video.webm -vcodec libx264 -crf 24 final-video.avi

# zeos-validator
