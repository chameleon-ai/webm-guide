# How to webm (or mp4) for 4chan
If you just care about the webm software:
- [webm-for-4chan (my script)](https://github.com/chameleon-ai/webm-for-4chan)
- [Handbrake](https://github.com/HandBrake/HandBrake)
- [Webm for Lazys](https://github.com/argorar/WebMConverter)
- [Boram](https://github.com/Kagami/boram)

If you want to learn how to webm, read on.\
[ 	![image.jpg](https://i.kym-cdn.com/photos/images/newsfeed/000/770/315/d8b.jpg)](https://i.kym-cdn.com/photos/images/newsfeed/000/770/315/d8b.jpg)

Contents:
- [Quick Reference](#quick-reference)
- [Codecs and Containers](#codecs-and-containers)
- [webm vs mp4](#webm-vs-mp4)
- [Size and Length Limits](#size-and-length-limits)
- [yt-dlp](#yt-dlp)
- [ffmpeg](#ffmpeg)
  - [CRF vs Average Bitrate](#crf-vs-average-bitrate)
    - [Calculating Video Size](#calculating-video-size)
    - [Preventing Size Overshoot](#preventing-size-overshoot)
  - [Deadline and Multithreading](#deadline-and-multithreading)
  - [Clipping](#clipping)
  - [Filters](#filters)
    - [Complex Filters](#complex-filters)
  - [Multiple Audio Tracks](#multiple-audio-tracks)
  - [Audio and Video Sync](#audio-and-video-sync)
- [Audio](#audio)
  - [Audio Bitrates](#audio-bitrates)
  - [Stereo and Mono Mixdown](#stereo-and-mono-mixdown)
  - [Music webms](#music-webms)
- [Making Precisely Sized webms](#making-precisely-sized-webms)
- [Resolution](#resolution)
- [Subtitles](#subtitles)
- [webm for 4chan](#webm-for-4chan)

# Quick Reference
tl;dr here's what you do up front
- Use vp9 video
- Use opus audio
  - 192k for music
  - 96k for everything else
  - 56k mono if you need to save space
- Pick a resolution, use lookup table for ballpark
- Hopefully you're using a tool that can calculate target size, but if not:
  - Calculate video bitrate using [this](#making-precisely-sized-webms) method
  - Or use the lookup table:

| Duration | Resolution | Bitrate | Bitrate | Bitrate |
| | | (6 MiB) | (4 MiB w/ audio) | (4 MiB no audio) |
| ----------- | ----------- | ----------- | ----------- | ----------- |
| < 30 seconds | 1920x1080 | 1500k | 990k | 1090k |
| 0:30 - 0:45 | 1600x900 | 1000k | 630k | 725k |
| 0:45 - 1:15 | 1440x810 | 550k | 335k | 435k |
| 1:15 - 2:00 | 1280x720 | 310k | 175k | 270k |
| 2:00 - 2:30 | 1024x576 | 230k | 120k | - |
| 2:30 - 3:00 | 960x540 | 175k | 80k | - |
| 3:00 - 4:00 | 854x480 | 100k | 35k | - |
| 4:00 - 5:00 | 736x414 | 60k | 8k | - |
| 5:00 - 6:00 | 640x360 | 35k | - | - |
| 6:00 - 6:40 | 480x270 | 20k | - | - |

# Codecs and Containers
webm is a container format that is a modified form of [.mkv](https://www.matroska.org/what_is_matroska.html)\
That means there is not just *one* type of webm. It can be any combination of a supported video + supported audio track.\
For the purpose of 4chan, you have [vp8](https://trac.ffmpeg.org/wiki/Encode/VP8) or [vp9](https://trac.ffmpeg.org/wiki/Encode/VP9) for video, [vorbis](https://xiph.org/vorbis/) or [opus](https://opus-codec.org/) for audio.\
Keep in mind that webm can also contain [av1](https://aomedia.org/specifications/av1/) but this isn't allowed on 4chan as of the time of this writing.\
webm also supports embedded [vtt](https://www.w3.org/TR/webvtt1/) subtitles, but this is also not supported by 4chan.\
Metadata is also stripped by 4chan, so don't bother.

If you come across a webm (or any container) and you want to know what's in it, use [ffprobe](https://trac.ffmpeg.org/wiki/FFprobeTips).

**Basically you want to use vp9 + opus in all circumstances. vp8 and vorbis are legacy codecs that perform objectively worse.**

# webm vs mp4
4chan supports only [h264](https://trac.ffmpeg.org/wiki/Encode/H.264) video with [aac](https://trac.ffmpeg.org/wiki/Encode/AAC) audio for mp4, so for the purposes of comparison, I'm talking about vp9+opus webm vs h264+aac mp4.\
Side note: If you mix vp9+aac or h264+opus, you're gonna create an mkv that can't be posted to 4chan.
- **Contrary to popular belief, vp9 is not objectively superior to h264 in all circumstances.**
- Generally speaking, opus is better than aac. So if you're going for music, like a static image with high quality audio, use webm.
- h264 encoding is *much* faster than vp9
- h264 better preserves film grain and fine detail *at an adequate bitrate*, meaning that mp4 is better for short high quality clips of live action movies and the like
- Since vp9 destroys film grain, it's good for digital animation where there are large blocks of solid color
- vp9 is better at really low bitrates, meaning that webm really shines when you have a long video that is super crunched

With all that said, the strategies for encoding mp4 are substantially the same as webm, so you can apply the same knowledge regardless of which you choose.\
Generally speaking, I recommend webm by default and mp4 if your clip is very short or you know what you're doing.

## Regarding YouTube
- Generally speaking, YouTube videos have h264, vp9, or av1 video streams, and audio that is opus or [aac-lc/mp4a.40.2](https://www.free-codecs.com/guides/understanding_the_differences_between_aac__aac-lc__and_aac-he.htm).
- Livestreams are h264+aac and after the stream ends, they are re-encoded to vp9+opus. Apparently YouTube determined that the space savings from re-encoding to vp9 was worth it for playback, lending to the argument that webm is a good default.
- Short videos are often encoded to av1+opus, so if you download a webm from YouTube and it's under the 4chan size limit, don't be surprised if you can't post it.

## Size and Length Limits
4chan limits:
- /wsg/: 400 seconds, 6MiB
- /gif/: 300 seconds, 4MiB
- All non-sound boards: 120 seconds, 4MiB

## Base 1024 vs Base 1000 Bytes
This is a whole can of worms but important to understand. 4chan's size limits are the traditional 1024 based definition of Megabytes (Sometimes these are called [Mebibytes](https://en.wikipedia.org/wiki/Binary_prefix) with MiB to differentiate from base 1000 MB).

For clarity, I will use `Gi, Mi, Ki` for base 1024 bytes and `G, M, K` for base 1000 bytes because I need to refer to both.

Some systems display local file sizes in MB. So if you see that your file is 6.29 MB, it's fine because the real limit is 6 * 1024 * 1024 = 6291456 bytes. This also explains why a file displayed as 6 MB will show as 5.72 when uploaded to 4chan.

When specifying bitrates in ffmpeg, the prefix `k` means Kilobits, so it's important to keep in mind that you're using the base 1000 definition of the term as well as *bits* so you have to divide by 8 to get bytes.

Side note: [webm-for-4chan](https://github.com/chameleon-ai/webm-for-4chan/tree/main) prints the final webm file size in KB and you want a file under 6144 or 4096

# yt-dlp
[yt-dlp](https://github.com/yt-dlp/yt-dlp) is pretty much the best way to download any video. It supports thousands of sites, not just YouTube.

Most of the time, you just need to provide the URL and it will download the best available video by default. There are a couple handy things to know:
- Download an upcoming or ongoing livestream: `yt-dlp --wait-for-video 30 --live-from-start`
- Download the auto subtitles: `yt-dlp --write-auto-sub --skip-download`
- Download only the section from 10:00 to the end of the video: `yt-dlp --download-sections "*10:00-inf"`

# ffmpeg
[ 	![image.jpg](https://files.catbox.moe/rklvkb.png)](https://files.catbox.moe/rklvkb.png)

Every solution out there is some kind of ffmpeg wrapper, so even if you use a GUI, the concepts discussed below still apply. The raw commands are just presented differently.

## CRF vs Average Bitrate
For [vp9](https://trac.ffmpeg.org/wiki/Encode/VP9) (and [h.264](https://trac.ffmpeg.org/wiki/Encode/H.264)) encoding you have 2 methods: constrained quality (CRF) and average bitrate. For your typical non-4chan use-case you probably want CRF, but this produces a variable bit-rate that makes the file size unpredictable. You will get a *consistent* quality, but not the *best* possible quality, because at the end of the day, it all boils down to video bitrate. CRF is just a way of telling ffmpeg that you don't care about the specific bitrate or size, just make the perceived quality consistent. If you use CRF you'll just end up making webms too big or too small.

The way to go is average bitrate, specifically [two-pass](https://trac.ffmpeg.org/wiki/Encode/VP9#twopass) encoding. On the first pass, ffmpeg builds a profile of the whole video that allows it to maximize compression on the next pass. Using average video bitrate (`-b:v`), the encoder will produce a file that has variable bitrate at any one point in time, but it averages out to the specified rate over the whole length of the clip. Basically it "saves" bits during sections that are highly compressible (black screens, no motion) and "spends" the saved bits when needed. This isn't unique to vp9, this is how two-pass encoding works for any codec.

### Calculating Video Size
Ignoring audio, your output video size is easily calculated:\
`bitrate = size_limit / duration` where duration is in seconds.\
Make sure your bitrate and size limit are the same units. For instance if you want to target a `6MiB` file, that's `6 * 1024 * 1024 = 6291456` bytes.\
Represented as kilobits, that's `6291456 / 1024 * 8 = 49152`.\
Then, divide that by the duration and there's your target bitrate in kbps.

So for example, a `30 second` video is `49152 / 30 = 1638.4`

Then run ffmpeg two-pass encoding:\
`ffmpeg -i in.mp4 -c:v libvpx-vp9 -b:v 1638k -pass 1 -an -f null /dev/null`\
`ffmpeg -i in.mp4 -c:v libvpx-vp9 -b:v 1638k -pass 2 -an output.webm`
- `-i` specifies the input file
- `-c:v` specifies the video codec
- `-b:v` specifies the video bitrate
- `-an` means no audio
- `-f null /dev/null` on pass 1 means no output video
  - Use `NUL` instead of `/dev/null` if you're using Windows

### Preventing Size Overshoot
Even though we can calculate the bitrate down to the exact byte, it's a *target* and ffmpeg can produce a file that's a little bigger or smaller than the target.\
The longer the video, the more ffmpeg is likely to overshoot on size.\
In general, round down to the nearest integer in kbps, and for clips over 2 or 3 minutes, reduce the target by another 2-4 kbps.

It also appears that the h264 encoder is much worse about overshooting than vp9, so you may want to build in additional margin when making mp4.

## Deadline and Multithreading
[libvpx-vp9](https://trac.ffmpeg.org/wiki/Encode/VP9) has a couple options that affect quality and enconding speed.

- Speed up encoding a little by enabling [row-based multithreading](https://trac.ffmpeg.org/wiki/Encode/VP9#rowmt) (`-row-mt 1`)
- The [deadline](https://trac.ffmpeg.org/wiki/Encode/VP9#DeadlineQuality) option changes the encode quality
  - The default deadline is `good`, and I recommend using this most of the time
  - If you don't care about waiting, use `-deadline best`. It will take significantly longer to encode, but the quality will be somewhat higher.
  - Get a significant speed-up at the cost of quality with `-deadline realtime`

## Clipping
You can directly make a webm of any time slice using [seeking](https://trac.ffmpeg.org/wiki/Seeking):\
`ffmpeg -ss 1:00:04.25 -t 45 -i input.mp4 output.webm`
- `-ss` will seek to 1 hour, 4 seconds and 250 ms.
- `-t` will take 45 seconds starting from the ss time.
- Use `-to` instead of `-t` to specify an absolute timestamp

This is precise when explicitly re-encoding, but keep in mind that if you just copy the streams (`-c:v copy -c:a copy`), that can result in a broken video at the edges due to missing keyframes.

## Filters
[filters](https://ffmpeg.org/ffmpeg-filters.html) are really handy. You can add filters using `-vf` (video filter) or `-af` (audio filter).\
See the official [Filter Guide](https://trac.ffmpeg.org/wiki/FilteringGuide) for more info.\
A few useful video filters to know:
- [blackframe](https://ffmpeg.org/ffmpeg-filters.html#blackframe) Use this to find the timestamp where the real video begins if it starts with black frames or has a fade in from black.
- [crop](https://ffmpeg.org/ffmpeg-filters.html#crop) crops the video to whatever you want.
- [cropdetect](https://ffmpeg.org/ffmpeg-filters.html#cropdetect) automatically finds the edges of a letterboxed video. Run this as a first pass and use the result with crop.
- [minterpolate](https://ffmpeg.org/ffmpeg-filters.html#minterpolate) intelligently reduces the framerate of the video using motion interpolation.
- [scale](https://ffmpeg.org/ffmpeg-filters.html#scale-1) resizes the video.
- [subtitles](https://ffmpeg.org/ffmpeg-filters.html#subtitles-1) will burn-in subtitles from file.

### Complex Filters
You can get crazy with `-filter_complex` by building an entire filter graph.\
One use case for this is to take multiple segments of a video and concatenate them together in one operation. For more information, see [this](https://github.com/sriramcu/ffmpeg_video_editing) project.

I'm not going to cover complex filters here, but read [this](https://trac.ffmpeg.org/wiki/FilteringGuide) and [this](https://medium.com/craftsmenltd/ffmpeg-basic-filter-graphs-74f287dc104e) and [this](https://lav.io/notes/ffmpeg-explorer/) if you're interested.

## Multiple Audio Tracks
How do we handle containers with multiple audio tracks? First, you have to figure out the index of the track by either inspecting it in a video player or with ffprobe:\
`ffprobe -v error -show_entries stream=index:stream_tags=language -select_streams a -of csv=p=0 input.mkv`\
This will show you something like this:
````
1,eng
2,jpn
````

One minor quirk is that the index shown is not the index you need to specify in ffmpeg. To get the real index, start with 0 and count up from there.\
Then, in ffmpeg you can use `-map` to specify the audio index.
- `-map 0:a:0` would select the first track (english)
- `-map 0:a:1` would select the second track (japanese)

## Audio and Video Sync
Sometimes an encode can get out of sync due to variable frame rate ([vfr](https://gist.github.com/rlaphoenix/2cadb7cc4050a33f1a6f08519530980c)) especially when [changing the framerate](https://trac.ffmpeg.org/wiki/ChangingFrameRate). In general, you want vfr because it efficiently decides how many frames are needed to represent a section of video.

Use `-async 1 -vsync 2` when encoding.
- `-async 1` will [prevent audio desync](https://lzone.de/blog/Easily-fix-async-video-with-ffmpeg)
- `-vsync 2` will specify [vfr](https://ffmpeg.org/ffmpeg.html) video sync method

# Audio
Calculating the video bitrate is not all there is to it. You have to factor in the size of the audio as well.\
For webm the audio codec is [libopus](https://ffmpeg.org/ffmpeg-codecs.html#libopus).

If you want to encode only the audio portion, the ffmpeg command is:\
`ffmpeg -i input.mp4 -vn -c:a libopus -b:a 128k output.ogg`
- `-i` specifies the input file
- `-vn` specifies no video
- `-c:a` specifies the audio codec
- `-b:a` specifies the audio bitrate

Generally, you can estimate the audio size using the audio bitrate and duration. However, the actual size can vary depending on how well it's compressed. In order to get the most accurate size, render the audio using the command above and get the size of that file. Then, subtract that from the total file size limit to get the video size limit. Rendering audio is extremely fast, so this step doesn't take long at all compared to video encoding.

### Audio Bitrates
Choosing the right audio bitrate isn't as straightforward as video. In general you can go with 96k by default, but there are other considerations.

Since video is usually more important than audio for perception of quality, you'll want to use the lowest audio bitrate you can get away with. This table will give you a good idea of how much space your audio takes up:

| Bitrate | Size at 3 minutes | Comments |
| ----------- | ----------- | ----------- |
| 320k | 7 MiB | Not practical except for music under 2:30
| 256k | 5.6 MiB | Upper practical limit for music webms
| 192k | 4.2 MiB | Good for most high quality music
| 160k | 3.5 MiB | Good for 5.1 surround sound
| 128k | 2.8 MiB | Good for high quality stereo tracks
| 112k | 2.5 MiB | If you need a little more than 96k
| 96k | 2.1 MiB | Good default
| 80k | 1.8 MiB | You can probably get away with this most of the time
| 64k | 1.4 MiB | High quality mono track
| 56k | 1.3 MiB | Decent quality for most mono applications
| 48k | 1.1 MiB | Lower quality but you can usually get away with it
| 32k | 721 KiB | Noticably degraded, don't recommend

Of course for short webms this doesn't matter as much, but the bitrate adds up for long duration webms.
Keep in mind that if you're getting your stuff from YouTube, that's usually at 128k so don't bother with higher bitrate.

## Stereo and Mono Mixdown
It's important to understand that the bitrate is the total bitrate *across all channels*, meaning that you can save on total bitrate by reducing the number of channels. This is especially pronounced for 5.1 surround sound, so if you're converting a movie clip or something, it's a good idea to mixdown to stereo.\
In ffmpeg, this is `-ac 2` (2 audio channels) or `-ac 1` (1 audio channel)

We can take this even further by analyzing the similarity of the 2 stereo tracks.\
In python, this can be done with the [cosine similarity](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.pairwise.cosine_similarity.html) function of the [scikit-learn](https://pypi.org/project/scikit-learn/#description) package.
````python
import scipy.io.wavfile as wavfile
from sklearn.metrics.pairwise import cosine_similarity

rate, data = wavfile.read('temp.wav')
num_channels = data.shape[1]
if num_channels == 1:
    print('Mono audio detected.')
    cosim = 1
elif num_channels == 2:
    left = data[:, 0]
    right = data[:, 1]
    cosim = cosine_similarity(left.reshape(1, -1),right.reshape(1, -1))[0][0]
print('Channel cosine similarity: {:.4f}%'.format(cosim * 100))
````

**Basically if both tracks are highly similar, we can go ahead and mixdown to mono and cut the bitrate in half.** For short clips it's usually not worth it to do this, but as you can see from the table, a 56k mono track has significant space savings over a 96k stereo track at 3 minutes, granting about 1 MB to be used toward the video bitrate.

## Music webms
Making a music webm is one of the easier things to do with ffmpeg. By music webm, I mean a webm with a static image and a song.\
`ffmpeg -i cover.jpg -i input.mp3 out.webm`

However, this will use the default audio bitrate of 96k. We can be more explicit by setting the `-b:a` audio bitrate:\
`ffmpeg -i cover.jpg -i input.mp3 -b:a 192k out.webm`

We can take this one step further by messing with the keyframes:\
`ffmpeg -framerate 1 -loop 1 -i cover.jpg -i input.mp3 -c:a libopus -b:a 128k -c:v libvpx-vp9 -g 212 -b:v 100k -t 0:03:32 out.webm`
- `-framerate` sets the frame rate to 1 fps. You don't need a 24fps static image, do you?
- `-g` sets the [group-of-pictures](https://www.veneratech.com/understanding-gop-what-is-group-of-pictures-and-why-is-it-important/) interval to match the duration of the song, effectively making a video that only has one keyframe.
- Note that 212 seconds = 3:32, the duration of the song
- Even though the video bitrate `-b:v` is set to the high value of 100k, this is an upper limit. The output video stream size will be basically negligible.

Doing it with an animated gif is a little more tricky:\
`ffmpeg -ignore_loop 0 -i dancing_baby.gif -i input.mp3 -c:a libopus -b:a 128k -c:v libvpx-vp9 -b:v 108k -t 0:03:32 out.webm`
- `-ignore_loop 0` will cause the gif to continually loop
- `-t 3:32` limits the video duration. You want this to match the song duration, otherwise you'll keep looping the gif after the song ends.

# Making Precisely Sized webms
Using all the above knowledge, we now know how to make a webm using 3 passes:
- Render the audio: `ffmpeg -i input.mp4 -vn -c:a libopus -b:a 96k output.ogg`
  - Get the size of the audio file in bytes
  - Lookup the size limit in bytes (`6 * 1024 * 1024` or `4 * 1024 * 1024`)
  - `video_size = size_limit - audio_size`
  - Get the total `duration` of the video in seconds
  - `bitrate = video_size / duration` that's your video bitrate in Bytes per second
  - `bitrate_kbps = bitrate * 8 / 1024` that's your video bitrate in kbps. For this example let's say it's 715.
- Run ffmpeg 1st pass
  - `ffmpeg -i input.mp4 -c:v libvpx-vp9 -b:v 715k -async 1 -vsync 2 -pass 1 -an -f null /dev/null`
  - or `NUL` instead of `/dev/null` if you're using Windows
  - Note that on the first pass we do `-an` because ffmpeg is doing video stuff only
- Run ffmpeg 2nd pass
  - `ffmpeg -i input.mp4 -c:v libvpx-vp9 -b:v 715k -async 1 -vsync 2 -pass 2 -c:a libopus -b:a 96k out.webm`

That'll produce a webm that's very close to the size you expect.

# Resolution
In ffmpeg, changing the resolution is done using the [scale](https://ffmpeg.org/ffmpeg-filters.html#scale-1) video filter.\
`ffmpeg -i input.mp4 -vf scale=1280:-1 output.webm`
- `-vf` applies a video filter, `scale` in this case
- `-1` scales the height dimension automatically

But we can generalize this statement to work for any sized input:\
`-vf scale='min(1280,iw)':'min(1280,ih):force_original_aspect_ratio=decrease'`
- `min(1280,iw)` and `min(1280,ih)` limit the size in each dimension to 1280 or the input size (`iw` = image width, `ih` = image height), whichever is smaller.
- So for an input below 1280p this does nothing. And it applies equally to horizontally and vertically oriented videos.

Obviously you don't want to keep the original resolution most of the time, but the trick is figuring out the right resolution.
The table below is a good start:

| Duration | Size |
| ----------- | ----------- |
| < 30 seconds | 1920 x 1080 |
| 0:30 - 0:45 | 1600 x 900 |
| 0:45 - 1:15 | 1440 x 810 |
| 1:15 - 2:00 | 1280 x 720 |
| 2:00 - 2:30 | 1024 x 576 |
| 2:30 - 3:00 | 960 x 540 |
| 3:00 - 4:00 | 854 x 480 |
| 4:00 - 4:45 | 736 x 414 |
| 4:00 - 5:30 | 640 x 360 |
| 5:30 - 6:40 | 480 x 270 |

Let's use math to do better than a lookup table. Resolution is better determined as a function of file size and duration. And resolution is better defined by the *total number of pixels* rather than assuming that the input is 16:9 1080p. Because we need a solution that scales well for 3:4 or square videos or whatever.

So really, you need to account for the size of the audio just like when determining the video bitrate. Conveniently, the target video bitrate is exactly what we need because the bitrate itself was determined as a function of file size and duration.

````python
total_pixels = width * height
# Factor in the total resolution of the image and the bit rate
x = target_bitrate * total_pixels
# Calculate the ideal resolution using logarithmic curve: y = a * ln(x/b)
a = 2.311e-01
b = 3.547e+01
scale_factor = a * math.log(target_bitrate/b)
scaled_pixels = total_pixels * scale_factor
scaled_height = scaled_pixels / width
scaled_width = scaled_pixels / height
calculated_resolution = max(scaled_height, scaled_width)
````
This is a [curve fit](https://curve.fit) that lines up with the table above. The difference between this and the lookup table is that now this scales with the target bitrate, so the audio size is a factor and can change the target resolution.

But we still have a problem. The original table has a baked-in assumption: The input is 1080p. To correct for this, we have to scale the input to 1080p so that the curve gives us a sane scale factor.
````python
# Scales resolution sources to 1080p to match the calibrated resolution curve
def scale_to_1080(width, height):
    min_dimension = min(width, height)
    scale_factor = 1080 / min_dimension
    return [width * scale_factor, height * scale_factor]

width, height = scale_to_1080(raw_width, raw_height)
````
And yes, this is still valid for any resolution because what's being calculated is a *resolution limit*. All `scale_to_1080` does is align the resolution with the baked in assumption in the curve fit, which make sure that smaller inputs don't get reduced in size too much.

**This method is far from perfect, but it gets the resolution in the right ballpark most of the time.** In practice, there are other considerations, like the amount of motion in the clip, the number of colors, number of scene changes, and more. Getting the resolution right is more of a subjective process, so it's difficult to come up with a purely mathematical solution that works unless we result to complex solutions like image analysis.

# Subtitles
Subtitle burn-in is pretty easy in ffmpeg using the [subtitles](https://ffmpeg.org/ffmpeg-filters.html#subtitles-1) filter. You can specify external or embedded subs.
- For internal subs: `-vf subtitles=input.mkv:si=1` where si is the subtitle index.
- For external subs: `-vf subtitles=subs.ass`

You can identify embedded subtitles using ffprobe:\
`ffprobe -v error -select_streams s -of csv=p=0 -show_entries stream=index:stream_tags=language input.mkv`\
Which will output something like this:
````
3,jpn
4,eng
````
And just like the audio index, the number isn't the index you want to specify in ffmpeg. Count from 0 from the top.

If you want to export the embedded subs to file:
`ffmpeg -i input.mkv -map 0:s:1 subs.ass`
- where `0:s:1` would specify the english subtitles and `0:s:0` would specify  the japanese subtitles as listed above.

You can download YouTube subtitles with yt-dlp:\
`yt-dlp --write-sub --sub-lang en --sub-format ttml`\
But ttml doesn't really work for ffmpeg so you'll have to convert to ass. This can be done with [ttml2ssa](https://github.com/Paco8/ttml2ssa).\
Alternatively, use vtt subtitles:\
`yt-dlp --write-sub --sub-lang en --sub-format vtt`

Sometimes, embedded subtitles aren't usable by ffmpeg. In this case I recommend [handbrake](https://github.com/HandBrake/HandBrake) which is very good at recognizing obscure sub formats.

# webm for 4chan
[ 	![image.jpg](https://i.imgflip.com/5vzx2b.jpg)](https://i.imgflip.com/5vzx2b.jpg)

Everything I discussed above is implemented in my [webm-for-4chan](https://github.com/chameleon-ai/webm-for-4chan) python script. So if you want to see it in action, use the script. It prints out the calculations and ffmpeg commands so that you can understand exactly what it's doing. Use the `--dry_run` option if you just want to see the calculations and commands without rendering the webm.

For the most part, the script does everything by default. You only have to specify options if you need to do something special like modify the audio bitrate or burn-in subtitles, etc.
