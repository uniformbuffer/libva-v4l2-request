Hi,
this is a fork that i'm trying to get working on Pinephone, but should work on every Allwinner device that support cedrus.
I'm absolutly not an expert in low level stuff so do not take my words (and my works) as reliable.
I got a lot of information on https://xnux.eu regarding Pinephone, especially https://xnux.eu/devices/feature/cedrus-pp.html where there are some instruction to hardware decode.

Compile using
```
git clone https://github.com/uniformbuffer/libva-v4l2-request
cd ./libva-v4l2-request
meson build
cd ./build
meson configure -Dbuildtype=release -Dprefix=*path to the install folder, if none /usr/local/lib is used*
ninja
ninja install
```

To try vaapi hardware decoding.
`export LIBVA_DRIVER_NAME=v4l2_request`

On Pinephone the video device for cedrus is /dev/video1.
On other device you must look for "cedrus" at /sys/class/video4linux/*/name
`export LIBVA_V4L2_REQUEST_VIDEO_PATH=/dev/video1`

On Pinephone the media device for cedrus is /dev/media0 that is used by default, so there is no need to specify.
On other device you must look for "cedrus" at /sys/bus/media/devices/*/model
`export LIBVA_V4L2_REQUEST_MEDIA_PATH=/dev/media0`

Use the install folder specified on meson before
`export LIBVA_DRIVERS_PATH=/*path to the install folder*/lib/dri`

Then on the same terminal run
`mpv --hwdec=vaapi-copy video.mp4`

You can also use 
`mpv --hwdec=yes video.mp4`
but this cause mpv probing for hardware devices, enabling the camera for a sec (you can hear the camera shutter opening).
I don't like to trigger mechanical parts if there is no need.

# v4l2-request libVA Backend

## About

This libVA backend is designed to work with the Linux Video4Linux2
Request API that is used by a number of video codecs drivers,
including the Video Engine found in most Allwinner SoCs.

## Status

The v4l2-request libVA backend currently supports the following formats:
* MPEG2 (Simple and Main profiles)
* H264 (Baseline, Main and High profiles)
* H265 (Main profile)

## Instructions

In order to use this libVA backend, the `v4l2_request` driver has to
be specified through the `LIBVA_DRIVER_NAME` environment variable, as
such:

	export LIBVA_DRIVER_NAME=v4l2_request

A media player that supports VAAPI (such as VLC) can then be used to decode a
video in a supported format:

	vlc path/to/video.mpg

Sample media files can be obtained from:

	http://samplemedia.linaro.org/MPEG2/
	http://samplemedia.linaro.org/MPEG4/SVT/

## Technical Notes

### Surface

A Surface is an internal data structure never handled by the VA's user
containing the output of a rendering. Usualy, a bunch of surfaces are created
at the begining of decoding and they are then used alternatively. When
created, a surface is assigned a corresponding v4l capture buffer and it is
kept until the end of decoding. Syncing a surface waits for the v4l buffer to
be available and then dequeue it.

Note: since a Surface is kept private from the VA's user, it can ask to
directly render a Surface on screen in an X Drawable. Some kind of
implementation is available in PutSurface but this is only for development
purpose.

### Context

A Context is a global data structure used for rendering a video of a certain
format. When a context is created, input buffers are created and v4l's output
(which is the compressed data input queue, since capture is the real output)
format is set.

### Picture

A Picture is an encoded input frame made of several buffers. A single input
can contain slice data, headers and IQ matrix. Each Picture is assigned a
request ID when created and each corresponding buffer might be turned into a
v4l buffers or extended control when rendered. Finally they are submitted to
kernel space when reaching EndPicture.

The real rendering is done in EndPicture instead of RenderPicture
because the v4l2 driver expects to have the full corresponding
extended control when a buffer is queued and we don't know in which
order the different RenderPicture will be called.

### Image

An Image is a standard data structure containing rendered frames in a usable
pixel format. Here we only use NV12 buffers which are converted from sunxi's
proprietary tiled pixel format with tiled_yuv when deriving an Image from a
Surface.
