# Stop Complaining About YouTube

# Work in progress.  Don't use this guide yet. 

## Preamble

YoutTube compresses video.  It's what they do.  It makes sense when they push petabytes of data around the planet every month to an audience that is probably 1/10th as sensitive to compression artifacting as you and I.

If you upload eye-wateringly huge videos to YouTube, they WILL be compressed before people can see them. Compression takes a LONG time and YouTube's settings destroy your content quality.  Everyone knows this. Let's stop complaining about it and do what we can to prevent it.  

So, how exactly do we prevent this?

PLAY BY YOUTUBE'S RULES!

This guide is not a "best way to transcode" guide.  This guide is not a definitive FFmpeg guide. This guide has one purpose: how to beat YouTube's video inspection algorithms and ensure your videos avoid re-transcoding by YouTube so that (a) they're available to view quickly (no hours-long delays at peak time), and (b) they're not re-compressed to horrendous visual quality. 

## YouTube provided guide

YouTube/Google have provided documentation.  Read it.  For example, their guides on recommended bitrates are listed here:

https://support.google.com/youtube/answer/1722171

If you upload video with these guides in mind, two things are likely to happen:
1) Your video will be viewable immediately.  No transcode required for people who want to watch it at the resolution you uploaded it at.
2) Your video at native resolution and codec will likely not be re-transcoded, so the quality will remain consistent. 
3) Your video will transcode to other formats/resolutions faster (and more tips below on how to improve that further).

My additional guidance is as follows:

1) Never, ever exceed the bitrate YouTube specify. That's a fast way to getting your video compressed using their ugly transcode methods.  This WHOLE GUIDE is about using free tools to do exactly this - get a video to the best possible visual quality that's UNDER YouTube's bitrate limit, and stop them applying their own transcoding to ruin your video. 
2) Learn FFmpeg. Use FFmpeg. Love FFmpeg. Yes it's command line, and difficult, and has complex syntax. But humans can learn new things.  I'll write scripts and put them in this repo to help.
3) Don't use hardware/GPU transcoding.  The visual quality is worse and the bitrate is higher.  Hardware/GPU transcoding is EXCELLENT for realtime streaming. You definitely want this for streaming video live to services like YouTube/Twitch/etc.  But the tradeoff is zero configurability around quality-to-bitrate ratios.  FFmpeg's software transcoders are far better for final output quality at the cost of being much slower to transcode.  The upside is when you upload a video, you have a decent expectation in advance of what it will look like.
4) YouTube much prefer MP4 containers, however they have the downside of putting critical metadata at the end of the file by default.  This makes YouTube unable to process them as they upload.  I'll show you some tricks with FFmpeg to put the metadata at the beginning of the file, which allows YouTube the ability to process large files as soon as the first few bytes hit their server, while your video uploads. 

Reading the link above, here's the YouTube-specified critical components for a 1080p video. Note that you can upload all sorts of weird stuff to YouTube, but again this buide and repo is about ensuring your best chances for a video that is available quickly and doesn't get re-compressed:
* Container: MP4. Note that this is not the codec. Learn the difference. 
* Codec: H.264.  Note that this is not the container. Learn the difference.
  * Progressive scan. Interlaced is dead, get over it. (I love CRTs, but it's the 21st century now. Stop using interlaced video if you're not professional avpres/digipres) 
  * High profile. This is the standard 8 bit colour, 4:2:0 chroma subsampling (more on that later) profile. Other profiles include things like high10 (10bit colour), high422 and high444, etc. Yes, these look way nicer. No, YouTube won't accept them. Deal with it. 
  * Frame rates:  24, 25, 30, 48, 50, 60.  More notes on this below.
  * Consecutive B-frames: 2. "B-Frames" are frames that refer to data in frames before or after the current frame, to do with how lossy codecs like H264 describe differences between frames (saving data by not having to redraw 100% of a frame, just the stuff that's changed). This value can be as high as 16, however YouTube want 2. 
  * Closed GOP at half framerate. "Group Of Pictures", or how far apart keyframes can be. This needs to be hard set (non-variable) and exactly half the frame rate. 
  * CABAC (Context-adaptive binary arithmetic coding). The default encoding for H.264 lossy encoding. Disabling this is only done typically for lossless or very high bitrate H.264, which can look spectacular. A reminder again that this is YouTube. 
  * Variable bitrate (more on bitrate below)
  * Chroma Subsampling 4:2:0.  A common method for the ratio of sampling of luma (light information on a black-and-white scale) versus chroma (colour information). 4:2:0 uses full resolution luma and half resolution chroma.  As the human eye is less sensitive to colour than brightness/lightness, this ends up saving a lot of space for not too much detail loss.  Again, 4:2:2 and of course fully uncompressed 4:4:4 look far better. But once again, this is YouTube, deal with it.
  * SDR (Standard Dynamic Range). YouTube recently added support for HDR and some time later finally offered tone mapping that didn't suck.  However that's outside the scope of this repo for now (might be something I look at later). 
  * Colour space: BT.709. This implies Rec.709 with BT.1886 EOTF ("gamma"), and limited range (aka "TV range") output.
* Bitrates.  YouTube have a few guides, but I'm going to insist you stick to two specific resolutions (more below).  For those, the bitrate ranges/caps suggested are:
  * 1080p30 : 8 Mbps
  * 1080p60 : 12 Mbps
  * 2160p30 : 35 - 45 Mbps
  * 2160p60 : 53 - 68 Mbps
* Audio codec: AAC-LC, Stereo or Stereo+5.1, Sample rate 96KHz or 48KHz

## Extra notes

Some extra notes from me outside of YouTube's guide:
* Stick with exact 30 or 60 FPS.  I don't care if you want to be all 24 FPS film auteur or 29.97 broadcast expert.  I'm sorry if you live in a PAL territory and want 25 FPS (I live in Australia, so that includes me).  If you want your videos to work cleanly, quickly, and smoothly on YouTube to that audience, stick with 30 and 60.  Firstly the sheer bulk of screens on planet Earth are 60.00FPS.  Secondly the YouTube app on almost any phone/tablet/PC is going to be hard stuck at 60.00FPS, and your 24/29.97 video is just going to be vsynced, time stretched or frame delayed to fit 30.00FPS anyway (at best changing the timing subtly, at worst causing judder for the viewer). 
* Stick with exact 1920x1080 (HD) or 3840x2160 (UHD) pixel resolutions and 16:9 aspect ratio.  Don't get all arty with anamorphic resolutions or DCI 4K. YouTube will accept it, but again you're back to them doing extra processing and scaling. Re-read the title of this repo if in doubt. 
* Encode as slow as you can bear and only use software encoding. As above: hardware/GPU encoding is fast, but the trade off is a worse picture and higher bitrate. ffmpeg's software encoding, specifically their use of libx264, offers several speed presets at which you encode. The slower you encode, the more CPU time libx264 can spend trying to make a picture that looks good for a lower bitrate.  The entire game to getting your video on YouTube under their bitrate for the best possible quality.  You take the encode time offline so that when you upload your video, you're not left waiting hours for YouTube to tell you it's ready. 
* Stick to uploading at H.264.  Yes, H.265 is way better than H.264.  Yes, AV1 is way better than H.265. But the whole point of this guide is getting videos up on YouTube as quickly as possible, in the best quality possible, and not having YouTube take hours to transcode things or decimate your hard work with their transcode quality.  That means gaming their transcode algorithm and doing what you need to in order to deal with their crummy service.  Also, like it or not, H.264 is the most universally compatible codec on planet Earth right now.  H.265, VP8 and VP9 are gaining adoption, but it's slow. I genuinely hope AV1 dominates the world in 5 years because it's a truly open source, royalty-free, licensing-unencumbered format. But it takes a LONG time for codec support to hit hardware inside commercial devices like TVs and phones. YouTube's goal is to be as compatible as possible with as many devices as possible, and that means using a codec that might not be "the best" overall. 
* Stick to 2 channel audio 48KHz.  As above - comaptibility is key. 96KHz and 5.1 is great, but every playback device can do stereo 48KHz. Compatibility is key. 

## General ffmpeg commands

I'll eventually fill this repo with a bunch of different scripts to help you do a range of things, such as batch-processing a folder full of videos, automatically choosing outputs based on the probed framerates/resolutions of the input videos, powershell scripts for Windows users and BASH scripts for Linux/Mac users, etc.  But for now, here's what I recommend. 

Firstly, create your video as normal, but constrain yourself to EXACTLY the following (re-read the above if you don't understand why):
* 30.00 or 60.00 FPS. 
* 1920x1080 (HD) or 3840x2160 (UHD) pixels
* 16:9 display aspect ratio (yes, this can be different to your pixel aspect ratio.  Ensure it's not)

The command below will enforce these (and several other constraints).  However I recommend the above to save yourself painful and costly re-shoots/re-edits.  The more you can do at capture/edit time, the better these transcodes will look.  Audio is less of an issue because ffmpeg's downmixing and re-sampling isn't too bad.  Likewise ffmpeg and libx264's colour depth, chroma subsampling conversion, bit depth, video range, etc at transcode time do a good job of converting from your high quality video to what YouTube expects.

Despite all my ranting here, you should absolutely create and store videos as high quality as you desire.  Keep your "masters" at high quality on your own private storage for as long as you can, and invistigate high quality codecs like AV1 for long term storage.  Use these scripts to make YouTube compatible versions for output.  You can always re-create lower quality videos from your high quality masters over and over again, should you need to (or want to upload the same video to different providers). But again, the resolutions, aspect ratios and framerates above should be your hard constraints if you want to minimise your YouTube frustrations. 

The basic ffmpeg commands are:

HD 1920x1080 30FPS video:
```
ffmpeg -nostdin -hide_banner -n -i "Your Video Name.ext" -vf "scale=1920:1080,zscale=transfer=709:matrix=709:primaries=709:range=limited,setsar=sar=1/1,setdar=dar=16/9,format=yuv420p" -r 30 -c:v libx264 -coder ac -b:v 5M -flags +cgop -g 15 -bf 2 -preset slow -c:a aac -ac 2 -b:a 256K -movflags +faststart "Outpt video name.mp4"
```

HD 1920x1080 60FPS video:
```
ffmpeg -nostdin -hide_banner -n -i "Your Video Name.ext" -vf "scale=1920:1080,zscale=transfer=709:matrix=709:primaries=709:range=limited,setsar=sar=1/1,setdar=dar=16/9,format=yuv420p" -r 60 -c:v libx264 -coder ac b:v 10M -flags +cgop -g 30 -bf 2 -preset slow -c:a aac -ac 2 -b:a 256K -movflags +faststart "Outpt video name.mp4"
```

UHD 3840x2160 30FPS video:
```
ffmpeg -nostdin -hide_banner -n -i "Your Video Name.ext" -vf "scale=3840:2160,zscale=transfer=709:matrix=709:primaries=709:range=limited,setsar=sar=1/1,setdar=dar=16/9,format=yuv420p" -r 30 -c:v libx264 -coder ac b:v 20M -flags +cgop -g 15 -bf 2 -preset slow -c:a aac -ac 2 -b:a 256K -movflags +faststart "Outpt video name.mp4"
```

UHD 3840x2160 60FPS video:
```
ffmpeg -nostdin -hide_banner -n -i "Your Video Name.ext" -vf "scale=3840:2160,zscale=transfer=709:matrix=709:primaries=709:range=limited,setsar=sar=1/1,setdar=dar=16/9,format=yuv420p" -r 60 -c:v libx264 -coder ac b:v 30M -flags +cgop -g 30 -bf 2 -preset slow -c:a aac -ac 2 -b:a 256K -movflags +faststart "Outpt video name.mp4"
```

What the commands do:
* -nostdin : stops ffmpeg taking "stdin" input (piping from other tools, or weird input during processing loops)
* -hide_banner : hides the noisy ffmpeg command line banner with build/version information
* -n : Never overwrite an existing video.  Fails immediately if a filename exists that could be overwritten. 
* -i "Your Video Name.ext" : the input video filename and extension.  Can be anything ffmpeg supports (which is almost anything you can imagine).
* -vf : ffmpeg's complex filter format. Breaking down the sub-commands:
  * scale=1920:1080 : Use ffmpeg's excellent quality software video scaler to scale the video to exact pixels. No action taken if your video is already this scale. This is here as a "just in case you didn't follow the advice" thing.
  * zscale : use libzimg to set/scale colour values. Values are changed only if not already in the required format.
  * transfer=bt709 : Set/scale the EOTF (electro-optical transfer function, sometimes called "gamma") to BT.709/BT.1886
  * matrix=bt709 : Set/scale the colour matrix to BT.709
  * primaries=bt709 : Set/scale the RGB primaries to BT.709
  * range=tv : Set/scale the colour range (aka "RGB range") to TV specs (aka "RGB Limited Range")
  * setsar=sar=1/1 : Set/scale the SAR (Sample Aspect Ratio, i.e.: pixel aspect ratio) metadata. 1:1 is square pixels, or non-anamorphic.
  * setdar=dar=16/9 : Set/scale the DAR (Display Aspect Ratio) metadata to 16:9. 
  * format=yuv420p : Set/scale the pixel format to yuv420p. YUV colour space, 4:2:0 chroma subsampling, 8 bit per pixel colour.
  * range=limited : Set limited range (aka TV video standard)
* -r 30 : Hard set 30FPS (or 60FPS) frame rate.  Won't change the duration of the video, but will convert frame rates if they don't match exactly already. Again, here to ensure you followed the instructions. 
* -c:v libx264 : Use the libx264 software H.264 video codec encoder
* -coder ac : use CABAC (Context-adaptive binary arithmetic coding)
* -b:v 5M : Set the video-only bitrate (audio adds to the overall size). "5M" is 5 Mbps (i.e.: 5000 Kbit/s) in this case. Rate isn't perfectly exact at every moment, as it can vary depending on what's happening frame by frame.  This is averaged over time.  Rates are a fair bit under the YouTube upper limits, chosen on purpose. 
* -flags +cgop : Set consistent GOP (Group Of Frames)
* -g 15 : Set the GOP size to 15 (half the frame rate. -g 30 when framerate is 60)
* -bf 2 : 2 consecutive b-frames
* -preset slow : Encode slightly slower than the standard.  This produces slightly better quality video for the same bitrate at the expense of CPU time.
* -c:a aac : Use the AAC audio codec
* -ac 2 : Set 2 audio channels (up/downmixing if necessary)
* -b:a 256k : Set the audio-only bitrate (video adds to the overall size). In this case 256Kbit/s, which with stereo (2 channel) is 128kbit/s per channel
* -movflags +faststart : Add the "faststart" flag to the MP4 container creation. This will create a normal MP4 container on one pass, and then on a second pass move all of the MP4 metadata from the end of the MP4 (standard) to the beginning. This assists YouTube to begin analysing/processing your video once the first few KB of data are uploaded.  Without this, your entire file needs to be uploaded before YouTube can analyse/process it. 
* "Outpt video name.mp4" : Output video name, and essentially making the extensions ".mp4" to generate an MP4 container. 
