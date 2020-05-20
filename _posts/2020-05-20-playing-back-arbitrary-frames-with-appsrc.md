---
layout: post
section-type: post
title: Playing back arbitrary frames with appsrc
category: Category
tags: [ 'gnome', 'gstreamer', 'pitivi' ]
---
If you have used GStreamer you may have used source elements like **filesrc** or **v4l2src**. Both of them use an existing source to play back a video, for example, the former takes as an input a video file from the source and the latter takes input from the camera. But, imagine you want to create a video by hand, something like. For example, [videotestsrc](https://github.com/GStreamer/gst-plugins-base/blob/master/gst/videotestsrc/videotestsrc.c), the element that displays a [test (card) pattern](https://en.wikipedia.org/wiki/Test_card), creates this pattern by filling a buffer by hand.

I will not talk about creating an element similar to **videotestsrc**. I will show an example program in which you tell programatically to GStreamer what to display at a given timestamp or frame.

------

The following example, will display

![appsrc-example-colors.gif](/img/posts/appsrc-example-colors.gif)

- **from second 0 to 1**: a blue frame
- **from second 1 to 2**: a green frame
- **from second 2 to 3**: a red frame

and repeat it for the next frames.

{% gist cfc28980244c117e631e729c177d15ac main.c %}



------

First, in line 104, we use the **appsrc** element, this element will allow us to push buffers by hand. We need to specify the caps we will use: `caps=video/x-raw,format=RGBx,width=640,height=460`:

Now in line 108, we connect to the signal "need-data", which will be emitted as soon the appsrc internal queue starts running out of data. Because initially there is no data in the queue, it will be emitted as soon as possible. When this signal gets emited, we will start pushing data (in this case the frames of solid blue, green and red colors), and this is what is done in the function `push_data` of line 24. But before that, see that if we would not have connected to the "need-data" signal, we would have seen a black screen.

In line 109, we connect to the signal "enough-data". This signal will be emitted when the appsrc inetrnal queue size exceeds the maximum allowed queue size (given by the property "max-bytes").

For this example, I should haved probably increased the "max-bytes" property since the queue maximum size will be always exceeded. I will left that for you. The queue maximum size will be exceeded because we are pushing frames of *640 * 460 * 4 (width * height * 4 [RGBx] channels) = 1177600*, and the default queue "max-bytes" property is 200000.

Ok, let's go to the signal handler of "need-data", line 24, the function `push_data`. Here is the interesting part. In line 35, firstly a buffer is allocated with the size of *width * height * 4*. Then, in lines 36-37, I have to timestamp the buffers. The first frame will have a timestamp of 0; the second frame, of 1 second; the third one, of 2 seconds... and so on. The duration assigned, in this example, is of 1 second for all the frames. This will make to play back 1 frame for 1 second, then the next second, the other frame and so on. Try playing with timestamp and duration and you will change the rate at which these frames are displayed. 

Now the part were we start to write data to the buffer starts. We have to map the buffer in GST_MAP_WRITE mode for that (line 40). Then we start to write data. I picked RGBx to make it easier to write data, so I can cast the data as guint32 array and assign each item in the array to a color (lines 44-46). See that guint32 has 4 bytes, but we are just using 3 colors: in RGBx, [the x "channel" is ignored](https://gstreamer.freedesktop.org/documentation/additional/design/mediatype-video-raw.html?gi-language=c).

Once the buffer is filled, we need to emit the "push-buffer" so the buffer is taken by the appsrc element. If we do not do that, we will see a black screen.

------

That is all for now. Sorry if there are things not used in the code, this is an old example and I did not have too much time to change it.

Now as task, try this with RGB or other colorspaces from here [here](https://gstreamer.freedesktop.org/documentation/additional/design/mediatype-video-raw.html?gi-language=c).

*I made this very quick and small example [when working on a MR for gst-plugins-base](https://gitlab.freedesktop.org/gstreamer/gst-plugins-base/-/merge_requests/626) to be able to apply whatever effect just in a region of a given frame (GstVideoRegionOfInterestMeta). I needed to see if what I did worked in videos without GstVideoMeta like in this example.*
