---
layout: post
section-type: post
title: Pitivi Developer Console Plugin
category: tech
tags: [ 'pitivi', 'gnome' ]
---

The first part of my project was focused in adding support for creating Python-based plugin managers in libpeas and polishing the Pitivi Plugin Manager. Initially, before the Google Summer of Code started, the Pitivi Plugin Manager was done using the PeasGtkPluginManager. However the design didn't fit pretty well in the Pitivi Preferences Dialog, so I had to implement it again but in Python. I took as reference the GNOME Builder Preferences window. It is worth to say that I could have used libdazzle, but Pitivi doesn't use GSettings and instead it uses ConfigParser.


![alt text](/img/posts/pitivi-plugin-manager.png)

The second part of my project has been to finish to implement the Developer Console Plugin. This plugin allows developers to do some quick experiments at run-time when executing Pitivi. This is very useful if you are a Pitivi developer. I have actually used it while developing the Pitivi Plugin Manager and this same plugin.

When you enable the Developer Console Plugin, a new menu item will be added to the menu.

![alt text](/img/posts/developer-console-menu.png)

When you click on it, a new dialog will be displayed. The plugin is based on the Gedit's one. But new features has been added like autocompletion and shortcuts commands. Also it is integrated with Pitivi so it can show preferences in the Pitivi Preferences Dialog. Adding those preferences has allowed me to find some bugs in Pitivi due to deprecated functions and to implement new functions that are usuful to add and remove pages from the preferences dialog.

![alt text](/img/posts/developer-console-preferences.png)

I think that one of the most challenging parts of this part of my project has been to implement the shortcuts commands. Currently, the console works redirecting stdout and stderr to a GtkTextBuffer. When you hit *return key*, the input command is passed *eval* and if it doesn't work, then it is passed to *exec*. If there is an error, the error is put in the GtkTextBuffer with the error color indicated in the preferences dialog. *eval* and *exec*, both takes as argument a *dictionary*. An example of this *dictionary* may be

<pre>
<code data-trim class="python">
{% raw %}
namespace = {
    "objA": "banana",
    "objB": "some",
    "objC": "pigs",
    "objD": "fly"
}
{% endraw %}
</code>
</pre>

When this *namespace* is passed to *eval or *exec*, actually, you can use those keys *"objA"*, *"objB"*, *"objC"* and *"objD"* as variables. So you for example could do somehing simple like:

<pre>
<code data-trim class="python">
eval("print(objC + ' ' + objD)", namespace)
</code>
</pre>

And you would get as output:

<pre>
pigs fly
</pre>

Okay, nothing complicated with that. But the thing that was a bit complicated to me was that I was adding the internal GES.Timeline object to that *dictionary* to have it as a shortcut command with the name *"timeline"*. So if the user just typed *timeline* in the Pitivi Developer Console plugin, he/she would access quickly to the timeline without going through *app.gui.timeline_ui.ges_timeline*. The problem was that when the Developer Console was loaded, the Pitivi Timeline had not loaded yet, so if you typed *timeline* in the console plugin you would get a *None* value! The same problem occured with other similar objects.

The first idea was to keep it simple and add to the namespace functions instead of variables. So there was a function *_get_timeline()* as a shortcut command. So the user could do something *timeline = _get_timeline()*. But to be honest, it was ugly. I talked to Alex about this problem and I think that he possibly had the same idea, because he asked me that the access to the Pitivi Timeline should just as simple as typing *timeline*. His idea was to load the console when the timeline is loaded. I had the same idea before, but the problem was that if you open a new project in Pitivi a new GES.Timeline is loaded.

I was trying to think in some hack that can lie the user about that the *timeline* object is actually an object and instead execute a function *_get_timeline()* which returns the GES.Timeline. The first idea that came to my mind was Python class properties. So I could do something like:

<pre>
<code data-trim class="python">
class Shortcuts:
    def __init__(self):
        pass

    @property
    def timeline(self):
        return app.gui.timeline_ui.ges_timeline

shortcuts = Shortcuts()

namespace = {
    "timeline": shortcuts.timeline
}
</code>
</pre>

But obviously that idea wouldn't work because as soon the developer console loads, *shortcuts.timeline* would evaluate to *None* and I was facing the same problem described above. I started to get some distraction, like play SuperTuxKary, play on the PlayStation. Then I walked around the park thinking on a solution. Then an idea came to my mind and to be honest I am not sure why it didn't come to my mind before. The solution was to override 'dict'. So I would create a *Namespace* class that inherits from *dict* (overriding *__getitem__*) and a *Shortcuts* class containing all the shortcuts commands as methods. Whenever you requested an item from the *Namespace*, *__getitem__(key)* would scan in Shortcuts if there is a method with that *key* name. If it was there, then it executed the *Shortcuts* class' method. I was changing the behavior of a typical *dict*. The following is the code of the *Namespace* class and the *Shortucts* that I called *PitiviNamespace*.

<pre>
<code data-trim class="python">
{% raw %}
class Namespace(dict):
    class ShortcutReadOnly(Exception):
        pass

    def __init__(self):
        dict.__init__(self)
        for key in self.get_shortcuts():
            dict.__setitem__(self, key, None)

    @staticmethod
    def shortcut(func):
        """Decorator to add methods or properties to the namespace."""
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            return func(*args, **kwargs)
        setattr(wrapper, "__is_shortcut", True)
        return wrapper

    def __getitem__(self, key):
        if key in self.get_shortcuts():
            return getattr(self, key)
        return dict.__getitem__(self, key)

    def __setitem__(self, key, item):
        if key in self.get_shortcuts():
            raise Namespace.ShortcutReadOnly("Not possible to override '%s', "
                                             "because shortcuts commands are"
                                             "read-only." % key)
        dict.__setitem__(self, key, item)

    def __repr__(self):
        return "<%s at %s>" % (self.__class__.__name__, hex(id(self)))

    @classmethod
    def get_shortcuts(cls):
        for attr_name in dir(cls):
            attr = getattr(cls, attr_name)
            is_shortcut = False
            if hasattr(attr, "__is_shortcut"):
                is_shortcut = getattr(attr, "__is_shortcut")
            elif isinstance(attr, property):
                if hasattr(attr.fget, "__is_shortcut"):
                    is_shortcut = getattr(attr.fget, "__is_shortcut")
            if is_shortcut:
                yield attr_name
{% endraw %}
</code>
</pre>

<pre>
<code data-trim class="python">
{% raw %}
class PitiviNamespace(Namespace):
    """A class to define public objects in the namespace."""

    def __init__(self, app):
        Namespace.__init__(self)
        self._app = app

    @property
    @Namespace.shortcut
    def app(self):
        """Gets the internal Pitivi Main Application object."""
        return self._app

    # ...
    @property
    @Namespace.shortcut
    def timeline(self):
        """Gets the internal GES.Timeline."""
        return self._app.gui.timeline_ui.timeline.ges_timeline

    # ...

    @property
    @Namespace.shortcut
    def shortcuts(self):
        """Gets the available methods in the namespace."""
        print("These are the available methods or attributes.")
        print()
        for attr in self.get_shortcuts():
            print(" - %s" % attr)
        print()

    # ...
{% endraw %}
</code>
</pre>

These has been the better ideas I had for the moment. The Developer Console Plugin can be refactored using the Python's module called *"code"*, althought using it doesn't skip the problem mentioned above because this module uses *eval* and *exec* calls under-the-hood. So it also needs a *dict* (namespace).

When you start the console you will see the following window with a banner:

![alt text](/img/posts/console1.png)

When you type *shortcuts*, actually this object does not exist! But you are calling to the property *shortcuts*, so it will executed the decorated function.

![alt text](/img/posts/console2.png)

Also, you cannot override *shortcuts* nor the *shortcuts* commands (methods decorated with *@Namespace.shortcut*) because it will raise an exception *ShortcutReadOnly* wich is handled internally by the console.

![alt text](/img/posts/console3.png)

And then you can access whichever of the shortcuts, like the *timeline*:

![alt text](/img/posts/console4.png)

That's it. I hope you enjoy this post and I hope you can soon try this plugin!
