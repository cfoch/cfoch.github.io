---
layout: post
section-type: post
title: 'What I learned about Linux device development: Part I'
category: tech
tags: [ 'linux', 'kernel', 'gnome', 'linux driver development', 'linux device development' ]
---


I have been reading the firsts three chapters of the [Linux Device Drivers](https://lwn.net/Kernel/LDD3/) book during this week. I have been reading this book in parallel with other book of 600 pages (related to Peruvian economy history since 1889), so the progress I have made on this week is productive enough to me. I have also put in on practice the implementation of the *scull* driver.

The mentioned book is a little bit outdated of course because there are new ways to do things that were added later in the kernel. So the purpose of this post is to show these outdated parts, what is the new way to do it, and also to extend some things that I did not see covered in the book. 

**Do not use mknod** 
When registering a char region with *[register_chrdev_region](https://www.kernel.org/doc/html/latest/core-api/kernel-api.html?highlight=register_chrdev_region#c.register_chrdev_region)* you have to pass to it a specific major number and a minor number (in the first argument). The book suggests "your drivers should almost certainly be using
*alloc_chrdev_region* rather than *[register_chrdev_region](https://www.kernel.org/doc/html/latest/core-api/kernel-api.html?highlight=register_chrdev_region#c.alloc_chrdev_region)*".

In fact, it's better to let the kernel to pick a free major number for you.  The annoying part with it is that once you created your driver and loaded it up with the *insmod* command, the device is not automatically added to the */dev* directory. The book tells that when distributing a driver you will have to provide a script that creates the device files manually (usually called [MAKEDEV](http://manpages.ubuntu.com/manpages/xenial/man8/MAKEDEV.8.html)), in this case your script should manually create: */dev/scull0*, */dev/scull1*, */dev/scull2* and */dev/scull3*. For that you do:

```
mknod /dev/${device}0 c $major 0
mknod /dev/${device}1 c $major 1
mknod /dev/${device}2 c $major 2
mknod /dev/${device}3 c $major 3
```

However, if you created your device dynamically as the book suggested and **you have to tell mknod the major number**, how can you tell what major number to use? The book does it by looking at `/proc/devices` and parsing the major number from it: **that's annoying** but understandable since that book targeted the 2.6.10 kernel version. 

Instead...

The new method to do this is through [udev](https://en.wikipedia.org/wiki/Udev).  What the kernel has is a pair of functions called *class_create* and *device_create*. 

So do this:
```
static struct class *scull_class = NULL;

/* ... */

scull_class = class_create (THIS_MODULE, "scull");
if (IS_ERR (scull_class)) {
    res = PTR_ERR (scull_class);
    goto error;
}
```
This piece of code will create a folder */sys/class/scull/* in the *sysfs* virtual file system.

Then... for each minor, in this case 0, 1, 2, 3 (4 devices):
```
static struct scull_dev *scull_devices = NULL;

/* ... */

device = device_create (scull_class, NULL, devno, NULL, "scull%d", minor);
if (IS_ERR (device)) {
    res = PTR_ERR (device);
    goto error;
}
```
This will automatically create the folders *scull0*... *scull3* in */sys/class/scull/*. 
```
$ ls /sys/class/scull/
scull0  scull1  scull2  scull3
```
and when that happens the device nodes */dev/scull0*, ..., */dev/scull3* will be created.

And of course do not forget to clean up your device and class :
```
/* For each i in 0, 1, 2, 3 */
device_destroy (scull_class, MKDEV (scull_major, scull_minor + i));
```
```
class_destroy (scull_class);
```

**Device permissions** 

If you have followed the above recommendations you will realize that you need superuser permissions to open the files */dev/scull0*, ..., */dev/scull3*. The script I mentioend also took care of assigning permissions manually. You can specify permissions programatically, too. That is what the [tty device does](https://github.com/torvalds/linux/blob/v4.6/drivers/tty/tty_io.c#L3595-L3603).

So once you have created the class *scull_class* you can do the following

```
scull_class->devnode = scull_devnode;
```

where `scull_devnode` is defined like:
```
static char *
scull_devnode (struct device * dev, umode_t *mode)
{
  if (!mode)
    return NULL;

  *mode = 0666;
  return NULL;
}


```

**Adding support for O_APPEND** 
The scull device example, at least until what I saw did not support append text to the end of the file which is an operation performed with `echo foo >> /dev/scull0` for example. 

In your *.write* virtual function, (wrapped with a mutex) do the following:
```
static ssize_t
scull_write (struct file * filp, const char __user * buf, size_t count,
    loff_t * f_pos)

	/* ... */
	
	if ((filp->f_flags & O_APPEND)
	    *f_pos = dev->size;

```
Now the file position is right at the end, writing should happen from there. 


----

That is all for now. I am not an expert on this area, **yet**.  I plan in the future to continue studying about polling, interruptions, and after that I hope that in the next at most three weeks I can create a simple and stupid mouse driver from scratch. 


EDIT: The book mentions about the existance of *udev* but it does not use it in the *scull* example. I have just realized it, and it is on the page 403.
