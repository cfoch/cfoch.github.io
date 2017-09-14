---
layout: post
section-type: post
title: Cheese's pipeline
category: tech
tags: [ 'gnome', 'cheese' ]
---
I have been reading the source code of Cheese. I wanted to get an idea of how it works. So this is what I understand. Below the diagram you will see an explanation of this, if you know about GStreamer you may want to skip the explanation.

![Developer Console I](/assets/cheese-pipeline.png)

Cheese reads some data from the selected webcam device using the bin *camera_source*. In parallel, Cheese uses an *autoaudiosrc* to capture data from the sound card. The data from the webcam is sent to a *tee* which work is to duplicate the output.

Firstly, one of the outputs of this *tee* is passed to an element called *element_bin_preview* (this also uses a *tee* of nine outputs... but that's not showed in the diagram). An *elements_bin_preview* has 9 sink pads. The output that flows out from there is (for each sink pad) passed to a *filter* and finally to a *cluttersink*. Its name on itself is just descriptive, *elements_bin_preview* is used to preview the effects in that 3x3 "grid" used for Cheese when you click on the "Effects" button.

Secondly, the data that flows from the other sink pad of the *tee* is sent to an element called *current_selected_filter* which is the filter that has been currently selected by the user by clicking the grid of effects in Cheese. In case of recording video, the output that flows out from one of the sink pads of the *tee* is sent in parallel with the *autoaudiosrc* to an *encodebin* which (of course) encodes and mixes to pass it to a *filesink* so you have the result (a video file) on your disk. In case of using the "burst mode" in Cheese, the second output of the *tee* is used to be passed to a *filesink*. The *filesink* can then capture some bunch of frames (by default 4 in Cheese) saving the output in the disk. *filesink* usually receives an index in its filename, so when you take a picture with Cheese, you will usually see that base names of file names have a number from 1 to 4 in the sufix when using "Burst mode". Finally, the third output of the sink is used to show the output of the camera with the filter applied in the Cheese main window using a *cluttersink* element.
