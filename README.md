# How to webm (or mp4) for 4chan
If you just care about the webm software:
- [webm-for-4chan (my script)](https://github.com/chameleon-ai/webm-for-4chan)
- [Handbrake](https://github.com/HandBrake/HandBrake)
- [Webm for Lazys](https://github.com/argorar/WebMConverter)

If you want to learn how to webm, read on.\
[ 	![image.jpg](https://i.kym-cdn.com/photos/images/newsfeed/000/770/315/d8b.jpg)](https://i.kym-cdn.com/photos/images/newsfeed/000/770/315/d8b.jpg)


## Codecs and Containers
webm is a container format that is a modified form of [.mkv](https://www.matroska.org/what_is_matroska.html)\
That means there is not just *one* type of webm. It can be any combination of a supported video + supported audio track.\
For the purpose of 4chan, you have [vp8](https://trac.ffmpeg.org/wiki/Encode/VP8) or [vp9](https://trac.ffmpeg.org/wiki/Encode/VP9) for video, [vorbis](https://xiph.org/vorbis/) or [opus](https://opus-codec.org/) for audio.\
Keep in mind that webm can also contain [av1](https://aomedia.org/specifications/av1/) but this isn't allowed on 4chan as of the time of this writing.\
webm also supports embedded [vtt](https://www.w3.org/TR/webvtt1/) subtitles, but this is also not supported by 4chan.\
Metadata is also stripped by 4chan, so don't bother.

If you come across a webm (or any container) and you want to know what's in it, use [ffprobe](https://trac.ffmpeg.org/wiki/FFprobeTips).

**Basically you want to use vp9 + opus in all circumstances. vp8 and vorbis are legacy codecs that perform objectively worse.**

## webm vs mp4
4chan supports only [h264](https://trac.ffmpeg.org/wiki/Encode/H.264) video with [aac](https://trac.ffmpeg.org/wiki/Encode/AAC) audio for mp4, so for the purposes of comparison, I'm talking about vp9+opus webm vs h264+aac mp4.\
Side note: If you mix vp9+aac or h264+opus, you're gonna create an mkv that can't be posted to 4chan.
- **Contrary to popular belief, vp9 is not objectively superior to vp9 in all circumstances.**
- Generally speaking, opus is better than aac. So if you're going for music, like a static image with high quality audio, use webm.
- h264 encoding is *much* faster than vp9
- h264 better preserves film grain and fine detail *at an adequate bitrate*, meaning that mp4 is better for short high quality clips of live action movies and the like
- Since vp9 destroys film grain, it's good for digital animation where there are large blocks of solid color
- vp9 is better at really low bitrates, meaning that webm really shines when you have a long video that is super crunched

With all that said, the strategies for encoding mp4 are substantially the same as webm, so you can apply the same knowledge regardless of which you choose.\
Generally speaking, I recommend webm by default and mp4 if your clip is very short or you know what you're doing.

### Regarding YouTube
- Generally speaking, YouTube videos have h264, vp9, or av1 video streams, and audio that is opus or [aac-lc/mp4a.40.2](https://www.free-codecs.com/guides/understanding_the_differences_between_aac__aac-lc__and_aac-he.htm).
- Livestreams are h264+aac and after the stream ends, they are re-encoded to vp9+opus. Apparently YouTube determined that the space savings from re-encoding to vp9 was worth it for playback, lending to the argument that webm is a good default.
- Short videos are often encoded to av1+opus, so if you download a webm from YouTube and it's under the 4chan size limit, don't be surprised if you can't post it.

## Size and Length Limits
4chan limits:
- /wsg/: 400 seconds, 6MiB
- /gif/: 300 seconds, 4MiB
- All non-sound boards: 120 seconds, 4MiB

### Base 1024 vs Base 1000 Bytes
This is a whole can of worms but important to understand. 4chan's size limits are the traditional 1024 based definition of Megabytes (Sometimes these are called [Mebibytes](https://en.wikipedia.org/wiki/Binary_prefix) with MiB to differentiate from base 1000 MB).

For clarity, I will use `Gi, Mi, Ki` for base 1024 bytes and `G, M, K` for base 1000 bytes because I need to refer to both.

Some systems display local file sizes in MB. So if you see that your file is 6.29 MB, it's fine because the real limit is 6 * 1024 * 1024 = 6291456 bytes. This also explains why a file displayed as 6 MB will show as 5.72 when uploaded to 4chan.

When specifying bitrates in ffmpeg, the prefix `k` means Kilobits, so it's important to keep in mind that you're using the base 1000 definition of the term as well as *bits* so you have to divide by 8 to get bytes.

Side note: [webm-for-4chan](https://github.com/chameleon-ai/webm-for-4chan/tree/main) prints the final webm file size in KiB and you want a file under 6144 or 4096 KiB



