---
layout: post
section-type: post
title: Writing a freesound plugin for Pitivi
category: tech
tags: [ 'pitivi', 'gnome', 'investment', 'value investing', 'blender' ]
---
I always say that my first geeky passion is computer programming. But that is
a passion I developed about 8 years ago. Another geeky passion I have recently
developed has been security analysis. Because of this reason I started a Youtube
channel the previous year called
"[Inversor Moderno](https://www.youtube.com/channel/UCl461K0D0-US_-YBAHm7fvA?view_as=subscriber)"
("Modern investor" in English). Besides the fact that I prefer to follow the
fundamental analysis and to be more specific the "value investing" philosophy,
I started the channel with the purpose of leaning and teaching more about
investments and specifically about quantitative trading. However, it has been
a long time since the last time I uploaded a video.

![Inversor moderno intro](/img/posts/inversormoderno-pitivi-screenshot.png)

The intro of my videos was made using Blender and Pitivi. The bag of money was
made in Blender and the sound of the Wall Street bell was added using Pitivi.
All the rest of each video in my channel are made using Pitivi. I consider that
the best ideas you may have come up when you really start to eat your own dog
food. When I was animating the intro, I remembered that when I started trading,
the platform I used notified me with a Wall Street bell sound when a position
touched "take profit" or "stop loss". So I wanted to add a sound like this one
to my intro. Then I thought: "It would be nice if Pitivi had a sound library".

> **Flashback**: I remember I had an introductory course in the university in which
students had as an assignment task to develop a video game. Nobody knew how to
develop a video game, but the idea was also to research. By that time, I knew
something about Blender and the BGE and I could convince to my friends to use
Blender. We modeled the university in Blender and then put a human in there with
Makehuman. But our game needed sounds. Searching for Creative Commons sounds, I
found the Freesound database.

![preview](/img/posts/freesound-logo.png)

Luckily, <freesound.org> not only has an API to access their sound database, but
it also has a [Python module](https://github.com/MTG/freesound-python).
So I decided to write a plugin that
allows to query for some sounds from Pitivi and importing them to the Pitivi
Media Library. I made the first step the previous year, but I didn't continue
it. This last week, I decided to continue this mini-project. The plugin is
working now, but there are some things to complete and a lot more to review.
However, you can see a preview here.

![preview](/img/posts/freesound-preview.gif)

I need suggestions about design. I am still not sure if I should use a
GtkListBox or if a GtkTreeView would look better. Also I don't know what message
should be shown when no result is found after searching and also what message
to show when the Freesound library window is open for the first time.

You can try this feature in my
[branch](https://gitlab.gnome.org/cfoch1/pitivi/tree/freesound) (yes, it needs
to be rebased).
