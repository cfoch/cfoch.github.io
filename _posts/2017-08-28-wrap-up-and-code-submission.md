---
layout: post
section-type: post
title: GSoC 2017 &#58; wrap-up and code submission
category: tech
tags: [ 'gnome', 'pitivi' ]
---

This post pretends to summarize what has been done during my project in the [Google Summer of Code](https://developers.google.com/open-source/gsoc/). This is also my Work Product Submission. The project has consisted on implementing a plugin manager for Pitivi and adding a plugin called the Developer Console.

# Libpeas

The first part of my project I worked on the [bug 780685](https://bugzilla.gnome.org/show_bug.cgi?id=780685#c11). Currently it is not possible to implement a plugin system in Python based applications using [Libpeas](https://wiki.gnome.org/Projects/Libpeas). I have wrote two patches (attachments [354018](https://bugzilla.gnome.org/attachment.cgi?id=354018&action=diff) and [354019](https://bugzilla.gnome.org/attachment.cgi?id=354019&action=diff)) that fixes the problem. Get these patches merged was depending on Libpeas mantainer. Unfortunately, although he has already reviewed my code, he is currently disabled to review my patches. Lately, the GNOME Builder mantainer has offered to review my patches in [bug 660014](https://bugzilla.gnome.org/show_bug.cgi?id=660014). As response to his review, these patches has been proposed: [attachment 357644](https://bugzilla.gnome.org/attachment.cgi?id=357644&action=diff) and [attachment 357645](https://bugzilla.gnome.org/attachment.cgi?id=357645&action=diff). In my opinion, the two set of patches proposed to the Libpeas mantainer and to the GNOME Builder mantainer are ready to merge, but it seems it will take time. This is not a a big obstacle for Pitivi, though, because I have added my patch as a source of Libpeas in the Pitivi's Flatpak manifest. This has just been merged in Pitivi (see [D1746](https://phabricator.freedesktop.org/D1746)).

# Plugin Manager

The development of the Plugin manager consisted firstly on creating a module PluginManager that handles the plugins in Pitivi (see [D1748](https://phabricator.freedesktop.org/D1748)), which also provides the API that exposes the application instance to the plugins. The plugin manager class is configuration-aware (see [D1812](https://phabricator.freedesktop.org/D1812)) so it remembers which plugins were activated or deactivated on the last execution of the program. Then I have created an user interface for it (see [D1789](https://phabricator.freedesktop.org/D1789). The *Preferences Dialog* has been extended to list the available plugins and allow activating and deactivating them. It also handles plugins with dependencies as Libpeas does. Also, the *Pitivi Preferences Dialog* has been modified (see [D1853](https://phabricator.freedesktop.org/D1853)) so plugins can add custom preferences widgets to the *Pitivi Preferences Dialog*. All the patches linked have just been merged in Pitivi. Other feature that has not been merged yet is one that classifies plugin in system and user plugins.

![Plugin Manager UI](/assets/plugin-manager.png)

# Developer Console Plugin

The developer console plugin is based on a Gedit's plugin calle "[pythonconsole](https://git.gnome.org/browse/gedit/tree/plugins/pythonconsole/pythonconsole)". Initially my project was reusing most of the "pythonconsole" plugin, but lately through many iterations of reviews it has been improved a lot. Gedit's "pythonconsole" works trying [to *eval* the input command and if it fails then it tries *exec*](https://git.gnome.org/browse/gedit/tree/plugins/pythonconsole/pythonconsole/console.py#n371). This approach (see [D1793](https://phabricator.freedesktop.org/D1793)) has been improved using the [*code*](https://docs.python.org/3/library/code.html) module. I have added to it autocompletion support (see [D1857](https://phabricator.freedesktop.org/D1857)). Also I have integrated color preferences with the *Pitivi Settings* and I have added the option to customize the font of the emulated console (see [D1790](https://phabricator.freedesktop.org/D1790), [D1792](https://phabricator.freedesktop.org/D1792) and [D1828](https://phabricator.freedesktop.org/D1828)).

![Developer Console I](/assets/developer-console-1.png)

The developer console (that uses *code.InteractiveConsole*) has access to internal objects on Pitivi. That behavior is achieved by adding a dictionary of locals variables, for example it receives as locals an argument like `{"app": app}`. The problem with this was that the user can (accidentally) override `app` to `None` for example and thus losing the access to the Pitivi's internal objects. That has been solved by adding a custom class called *Namespace* that inherits from a *dict* overriding *__setitem__* so whenever the user wants to change the value of `app` (for example), the console writes an error message to *stderr*. Also, some "shorcuts" commands has been added for a fast access to objects that my mentor and I think will be frequently used. See [D1793](https://phabricator.freedesktop.org/D1793) or this [old post](https://cfoch.github.io/tech/2017/07/16/pitivi-developer-console.html) for an extended explanation.

![Developer Console I](/assets/developer-console-1.gif)

*This plugin is not merged yet, but some of the patches here has been accepted.*

# Documentation

Two patches that documents a basic behavior of the console has been proposed. The first patch (see [D1858](https://phabricator.freedesktop.org/D1858)) provides plugin developers a very basic information about how to create a plugin. The second patch is a tutorial that explains how to create a plugin that does something useful that is to remove gaps between clips (see [D1859](https://phabricator.freedesktop.org/D1858)). *This patches has not been reviewed yet but my mentor told me that it would be helpful to have documentation, so I did this*.

Yes... you can code in Pitivi :)
