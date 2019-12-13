## autoedit v 1.4
### A script to turn Kodi with TVHeadend into a fully automated DVR

Turn your system into a fully automated TV recorder. It's supposed to just work without further interaction with the end users (wife and kids). In fact they should not even notice anything is happening in the background. The system should just silently do it's task, record all the favorite series and movies, remove all the advertising junk, convert everything to a decently compressed format and cleanly add every recording to the movie or series library.

For each of these tasks there's good open source software available. But it is not just a simple download and run. Most needs to be compiled from source and the documentation is incomplete and scattered around the web. Nothing for a beginner at least. Also you have to manually run it on each recorded video. That's very inconvenient and absolutely not acceptable for a family box.

Follow the  steps below to set up the necessary software and install a little script to do all the processing. When it's done, you just choose what to record and let the system do all the work for you.

### Requirements:
This tutorial is supposed to be for everybody who is interested in an automated DVR. Even if you are an absolute beginner you should be able to succeed. No linux or programming skills whatsoever required!

#### All you need is:
- A Raspberry Pi (2 or 3 recommended, but even the first models should work). Everything except for the hardware encoding should also work on any other device including the Vero, but I can't test this.
- Sufficient storage attached, either via USB or network.
- OSMC installed and set up.
- TVHeadend server installed from the OSMC App Store. You need to have everything set up to stream and record TV. There are plenty of good articles on that if you haven't [see here](http://forum.kodi.tv/showthread.php?tid=270385&pid=2315547#pid2315547).
- Crontab from the OSMC App Store – if you want to run the CPU heavy commercial detection and transcoding on an overnight schedule (OPTIONAL).
- Ca. 2h of leisure time. Don't be scared, most of the time is compiling so you can do other things in between.

#### Let's start:

If you are not interested in one certain feature of my script you can just skip the corresponding steps. All optional components are marked so. Otherwise follow the instructions closely. For the impatient you can simply copy everything from the blue boxes to the command line.

#### Install ffmpeg
**On Debian 9 "Stretch":**

Please note that the default Debian package of ffmpeg does not support the RPi's hardware video decoder. This usually is no problem as long as you are not interested in transcoding. Otherwise you need to build ffmpeg from source – please refer to the section below:
```
# 1) Make temporary directories for sources and building:
mkdir $HOME/sources
# 2) Install ffmpeg standard package and dev libraries
sudo apt-get update
sudo apt-get install ffmpeg libavcodec-dev libavformat-dev libavutil-dev
## Continue with #7.
```

**_DEPRECATED!_ Build ffmpeg from source (only needed on Debian 8 "Jessie" and earlier):**
```
# 1) Install dependencies:
sudo apt-get update
sudo apt-get install autoconf automake build-essential libass-dev libfreetype6-dev libsdl1.2-dev libtheora-dev libtool libva-dev libvdpau-dev libvorbis-dev libxcb1-dev libxcb-shm0-dev libxcb-xfixes0-dev pkg-config texinfo zlib1g-dev yasm libx264-dev cmake mercurial libfdk-aac-dev libmp3lame-dev libopus-dev
# 2) Make temporary directories for sources and building:
mkdir $HOME/sources
mkdir $HOME/sources/ffmpeg
mkdir $HOME/ffmpeg_build
# 3) Compile h.265 software de-/encoder
cd $HOME/sources/ffmpeg
hg clone https://bitbucket.org/multicoreware/x265
cd ~/sources/ffmpeg/x265/build/linux
cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="$HOME/ffmpeg_build" -DENABLE_SHARED:bool=off ../../source
make -j4
make install
# 4) Compile VP8/VP9 de-/encoder
cd $HOME/sources/ffmpeg
wget http://storage.googleapis.com/downloads.webmproject.org/releases/webm/libvpx-1.5.0.tar.bz2
tar xjvf libvpx-1.5.0.tar.bz2
cd libvpx-1.5.0
./configure --prefix="$HOME/ffmpeg_build" --disable-examples --disable-unit-tests
make -j4
make install
make clean
# 5) Compile Raspberry Pi hardware de-/encoder (MMAL and OMX)
cd $HOME/sources/ffmpeg
sudo apt-get install git
git clone git://github.com/raspberrypi/userland
cd userland-master
./buildme
# 6) ffmpeg itself
cd $HOME/sources/ffmpeg
wget http://ffmpeg.org/releases/ffmpeg-snapshot.tar.bz2
tar xjvf ffmpeg-snapshot.tar.bz2
cd ffmpeg
export LDFLAGS="-L/opt/vc/lib -L/opt/vc/lib/pkgconfig -L/opt/vc/lib/plugins"
export CPPFLAGS='-I/opt/vc/include'
PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig" ./configure --pkg-config-flags="--static" --enable-gpl --enable-libass --enable-libfdk-aac --enable-libfreetype --enable-libmp3lame --enable-libopus --enable-libtheora --enable-libvorbis --enable-libvpx --enable-libx264 --enable-libx265 --enable-nonfree --enable-mmal  --enable-decoder=mpeg2_mmal --enable-decoder=mpeg4_mmal --enable-decoder=h264_mmal --enable-decoder=vc1_mmal --enable-omx-rpi --enable-encoder=h264_omx
make -j4
make install
make distclean
cd $HOME
rm -R --interactive=never $HOME/sources/ffmpeg $HOME/ffmpeg_build
hash -r
```

#### Install Comskip (OPTIONAL)
(scans a video and marks commercial breaks)

```
# 7) Install dependencies:
cd $HOME/sources
sudo apt-get install autoconf automake git libargtable2-dev libtool
git clone git://github.com/erikkaashoek/Comskip
cd Comskip
./autogen.sh
./configure
make -j4
sudo make install
cd $HOME
rm -R --interactive=never $HOME/sources/Comskip
```

#### Install Filebot (OPTIONAL)
(renames video files to meet the Kodi standards and move files to your library folder)
8) Go to http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html and download the Java SDK „Linux ARM 32 Hard Float ABI“ (YES, ARM 32 even if you are on an RPi 3!) Take note of the Version no. which you'll need later. In this example it is 8u131 Copy it somewhere to your Pi's home folder (using Samba, FTP, it doesn't matter). In the following I assume you placed it under ~/sources/jdk
```
cd $HOME/sources/jdk
tar -zxf jdk-8u131-linux-arm32-vfp-hflt.tar.gz
sudo mkdir /opt/jre
# 9) We are only interested in the runtime environment JRE, so we're only installing this and dump the rest. Note that the directory contains the version no. You have to adjust this with the one you wrote down.
sudo mv $HOME/sources/jdk/jdk1.8.0_131/jre /opt
sudo update-alternatives --install /usr/bin/java java /opt/jre/bin/java 100
rm -R --interactive=never $HOME/sources/jdk
# 10) Download Filebot
mkdir $HOME/sources/filebot
cd $HOME/sources/filebot
wget https://downloads.sourceforge.net/project/filebot/filebot/FileBot_4.7.9/filebot_4.7.9_armhf.deb
sudo dpkg -i $HOME/sources/filebot/filebot_4.7.9_armhf.deb
cd $HOME
rm -R --interactive=never $HOME/sources/filebot
```

#### Install Autoedit

```
cd /opt
git clone git://github.com/IfThenERROR/autoedit
chmod +x /opt/autoedit/autoedit
sudo update-alternatives --install /usr/bin/autoedit autoedit /opt/autoedit/autoedit 100
cp /opt/autoedit/comskip.ini $HOME/
nano /opt/autoedit/settings.txt
```
Change the settings to your liking. Especially the outfolder and language need to be set.


Set Cron job (OPTIONAL)

11) To run all the magic overnight we want to create a schedule. The following runs autoedit every night on 4am. Change the time as you like.
```
crontab -e
```
Paste in the last line
```
0 4 * * * . $HOME/.profile && PATH=$PATH:/usr/local/bin && /usr/bin/nice -n 19 autoedit
```
Hit Ctrl+o and Ctrl+x to save.



#### Check if everything works

12) Now let's test all the installed parts.
```
java -version
```
Should output `Java(TM) SE Runtime Environment (build 1.8.0_xxx-xxx)`

```
ffmpeg -encoders | grep omx
```
Should output `V..... h264_omx             OpenMAX IL H.264 video encoder (codec h264)`

```
comskip
```
Should output `ComSkip: missing option <file>`

```
autoedit --help
```
Should output `Usage: bash autoedit (options) …`


#### Tadaaaaa! Everything is ready.


### Configure TVHeadend:
Now let's hook up autoedit in TVHeadend. Open a browser and go to the TVHeadend interface (http://your_RPi_IP:9981). Go to Configuration→Recording and in the (Default Profile). In the field for „Post-Processor Command“ enter:
```
/usr/bin/autoedit --input "%f" --title "%t" --episode "%s" --comskip --transcodeexternal "/path/to/list/for/external/transcoder(optional)" --rename --wait --update IP-Address-to-Kodi
```

--input [video file]: This is the file to process, %f makes TVHeadend deliver the full path.

--comskip : Marks commercials for automatic skipping using Comskip.

--rename : Rename the file to a Kodi compatible format using filebot.

--movie : Switch filebot to movie database. (Default is tv series database)

--title [name]: Use this for series title. If not set autoedit will try to read the title from the metadata if present.

--episode [name]: Use this for episode name. If not set autoedit will try to read the episode from the description in the metadata.

--transcode [decoder] [encoder] [bitrate]: Transcodes the video using ffmpeg. The sample above does de- and encoding in RPi's hardware and hence is pretty fast but only medium quality. For SD videos it's near 90 fps. Make sure you use the correct settings here.

--transcodeexternal [file]: Add the resulting video file to the specified list. This list can be read from external scripts to achieve better quality for transcoding.

--wait : Do not start processing immediately, just queue the video.

--update [IP address1] [IP address2] [IP addressN]: Send remote commands to specified Kodi clients to update the video library.

Also create a seccond profile for movies as filebot can't reliably distinguish between movies and series. All settings here are the same as above, but with '--movie' added.
```
/usr/bin/autoedit --input "%f" --title "%t" --episode "%s" --comskip --transcodeexternal "/path/to/list/for/external/transcoder(optional)" --movie --rename --wait --update IP-Address-to-Kodi
```


### Known issues:
- Quite many TV networks suck big time in properly labeling their broadcasts. They add fancy additions to series' titles, don't use the correct fields in the metadata, add additional spaces where they don't belong and generally mess things up a lot. I wrote the postscript as robust as possible, and tried to remove the most common junk. But still sometimes the tags are just too messed up. So it doesn't hurt to check your recording folder and the postscript's log some here and then. If you regularly have problems with a series you can edit the script. The corresponding code is around line 150.
- Your video directory in the settings.txt **should** not contain blank spaces as Comskip is officially not able to handle these. My personal tests seem to indicate it actually can do though, but no guarantees.
- Also if Comskip is set to decode in hardware, the little Raspberry is to busy to play a video smoothly at the same time. So either turn of hardware decoding in the comskip.ini, or set the script to run nightly with cron.
- The standard package in Debian Stretch does not contain the h.265 decoder. If you wish to process videos with this compression, you need to resort to building ffmpeg from source.
- If your system crashed not all is lost. The script is designed to be robust. You can resume an interupted run by entering „autoedit --forcerun“. This will try to pick up where it left and in most cases will work just fine. Be carefull to not start more than one instance of the script though.
