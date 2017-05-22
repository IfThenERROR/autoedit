# autoedit
script to postprocess recordings from TVHeadend for Kodi

My goal was to build a fully automated TV recorder. It's supposed to just work without further interaction with the end users (wife and kids). In fact they should not even notice anything is happening in the background. The system should just silently do it's task, record all the favorite series and movies, remove all the advertising junk, convert everything to a decently compressed format and cleanly add every recording to the movie or series library.

For each of these tasks there's good open source software available. But it is not just a simple download and run. Most needs to be compiled from source and the documentation is incomplete and scattered around the web. Nothing for a beginner at least. Also you have to manually run it on each recorded video. That's very inconvenient and absolutely not acceptable for a family box.

So I took notes of all required steps to set up the software and wrote a little script to do all the processing. When it's done, you can just lean back and enjoy your automated DVR.

Requirements:
This tutorial is supposed to be for everybody who is interested in an automated DVR. Even if you are an absolute beginner you should be able to succeed. No linux or programming skills whatsoever required!

All you need is:
• A Raspberry Pi (2 or 3 recommended, but even the first models should work). Everything except for the hardware encoding should also work on any Vero, but I can't test this.
• Sufficient storage attached, either via USB or network.
• OSMC installed and set up. If you need help on this see here.
• TVHeadend server installed. You need to have everything set up to stream and record TV. There are plenty of good articles on that if you haven't (see here).
• Crontab from the OSMC App Store – if you want to run the CPU heavy commercial detection and transcoding on an overnight schedule (OPTIONAL).
• Ca. 4h of free time. Don't be scared, most of the time is compiling so you can do other things in between.

Let's start:

If you are not interested in one certain feature of my script you can just skip the corresponding steps. All optional components are marked so. Otherwise follow the instructions closely. If anything is not working or you miss an essential function, all feedback is welcome!

Install ffmpeg

1) Install dependencies:
sudo apt-get update
sudo apt-get install autoconf automake build-essential libass-dev libfreetype6-dev libsdl1.2-dev libtheora-dev libtool libva-dev libvdpau-dev libvorbis-dev libxcb1-dev libxcb-shm0-dev libxcb-xfixes0-dev pkg-config texinfo zlib1g-dev yasm libx264-dev cmake mercurial libfdk-aac-dev libmp3lame-dev libopus-dev
2) Make temporary directories for sources and building:
mkdir $HOME/sources/ffmpeg
mkdir $HOME/ffmpeg_build
3) Compile h.265 software de-/encoder
cd ~/sources/ffmpeg
hg clone https://bitbucket.org/multicoreware/x265
cd ~/sources/ffmpeg/x265/build/linux
cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="$HOME/ffmpeg_build" -DENABLE_SHARED:bool=off ../../source
make -j4
make install
4) Compile VP8/VP9 de-/encoder
cd ~/sources/ffmpeg
wget http://storage.googleapis.com/downloads.webmproject.org/releases/webm/libvpx-1.5.0.tar.bz2
tar xjvf libvpx-1.5.0.tar.bz2
cd libvpx-1.5.0
./configure --prefix="$HOME/ffmpeg_build" --disable-examples --disable-unit-tests
make -j4
make install
make clean
5) Compile Raspberry Pi hardware de-/encoder (MMAL and OMX)
cd ~/sources/ffmpeg
sudo apt-get install git
git clone git://github.com/raspberrypi/userland
cd userland-master
./buildme
6) ffmpeg itself
cd ~/sources/ffmpeg
wget http://ffmpeg.org/releases/ffmpeg-snapshot.tar.bz2
tar xjvf ffmpeg-snapshot.tar.bz2
cd ffmpeg
PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig" ./configure --prefix="$HOME/ffmpeg_build" --pkg-config-flags="--static" --extra-cflags="-I$HOME/ffmpeg_build/include" --extra-ldflags="-L$HOME/ffmpeg_build/lib" --bindir="$HOME/programs/ffmpeg" --enable-gpl --enable-libass --enable-libfdk-aac --enable-libfreetype --enable-libmp3lame --enable-libopus --enable-libtheora --enable-libvorbis --enable-libvpx --enable-libx264 --enable-libx265 --enable-nonfree --enable-mmal --enable-omx-rpi --disable-debug
make -j4
make install
make distclean
cd $HOME
rm -R ~/sources/ffmpeg ~/ffmpeg_build
hash -r


Install Comskip (OPTIONAL)
(scans a video and marks commercial breaks)

7) Install dependencies:
cd ~/sources
sudo apt-get install libargtable2-dev
git clone git://github.com/erikkaashoek/Comskip
cd Comskip
./autogen.sh
./configure
make -j4
make install
cd $HOME
rm -R ~/sources/Comskip


Install Filebot (OPTIONAL)
(renames video files to meet the Kodi standards and move files to your library folder)
8) Go to http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html and download the Java SDK „Linux ARM 32 Hard Float ABI“ (YES, ARM 32 even if you are on an RPi 3!) Copy it somewhere to your Pi's home folder, in the following I assume you placed it under ~/sources/JDK

??? sudo apt-get install oracle-java8-jdk ???

mkdir ~/sources/jdk
cd ~/sources/jdk
tar -zxf jdk-8u131-linux-arm32-vfp-hflt.tar.gz
mkdir /opt/jre
9) We are only interested in the runtime environment JRE, so we're only installing this and dump the rest
mv ~/sources/jdk/jdk1.8_*/jre /opt/jre
sudo update-alternatives --install /usr/bin/java java /opt/jre/bin/java 100
rm -R ~/sources/jdk
10) Download Filebot
mkdir ~/sources/filebot
cd ~/sources/filebot
wget https://downloads.sourceforge.net/project/filebot/filebot/FileBot_4.7.9/filebot_4.7.9_armhf.deb
sudo dpkg -i ~/sources/filebot/filebot.deb
rm -R ~/sources/filebot


Install Autoedit

cd /opt
git clone git://github.com/IfThenERROR/autoedit
chmod +x /opt/autoedit/autoedit
sudo update-alternatives --install /usr/bin/autoedit autoedit /opt/autoedit/autoedit 100
cp /opt/autopostscript/comskip.ini $HOME/
nano /opt/autoedit/settings.txt
Change the settings to your liking. Especially the outfolder and languale need to be set.


Set Cron job (OPTIONAL)

11) To run all the magic overnight we cant to create a schedule. The following runs autoedit every night on 2am. Change the time as you like.
crontab -e
Paste in the last line
0 2 * * * bash -l -c autoedit
Hit Ctrl+o and Ctrl+x to save.



Check if everything works

11) Now let's test all the installed parts.
java -version
Should output xxxxxxx

ffmpeg --encoders | grep omx
Should output xxxxxxx

comskip
Should output xxxxxxx

autoedit --help
Should output xxxxxxx

Tadaaaaa! Everything is ready.


Installation:
Now let's hook up autoedit in TVHeadend. Open a browser and go to the TVHeadend interface (http://your_RPi_IP:9981). Go to Configuration→Recording→Profiles and in the (Default Profile) set „Tag with metadata“. In the field for „Post-Processor Command“ enter autoedit --input "%f" --comskip --transcode mpeg2_mmal h264_omx 1800k --rename --wait“

--input : This is the file to process, %f makes TVHeadend deliver the full path.
--comskip : Marks commercials for automatic skipping using Comskip.
--transcode [decoder] [encoder] [bitrate]: Transcodes the video using ffmpeg. The sample above does de- and encoding in RPi's hardware and hence is pretty fast. For SD videos it's near 90 fps. Make sure you use the correct settings here.
--rename : Rename the file to a Kodi compatible format using filebot.
--wait : Do not start processing immediately, just queue the video.

Also create a seccond profile for movies as filebot can't reliably distinguish between movies and series. All settings here are the same as above, but „Post-Processor Command“ is „Post-Processor Command“ enter autoedit --input "%f" --comskip --transcode mpeg2_mmal h264_omx 1800k --rename --movie --wait“


Caveats:
Quite many TV networks suck big time in properly labeling their broadcasts. They add fancy additions to series' titles, don't use the correct fields in the metadata and generally mess things up a lot. I wrote the postscript as robust as possible, and tried to remove the most common junk. But still sometimes the tags are just too messed up. So it doesn't hurt to check your recording folder and the postscript's log some here and then. If you regularly have problems with a series you can edit the script. The corresponding code is somewhere around line 100.
Your video directory in the settings.txt may not contain blank spaces as Comskip is not able to handle these.
Also Comskip is currently not capable of decoding in hardware on the little Raspberry. So running on HD videos is kinda slow.
If your system crashed not all is lost. Again the script is designed to be robust. You can resume an interupted run by entering „autoedit --forcerun“. This will try to pick up where it left.
