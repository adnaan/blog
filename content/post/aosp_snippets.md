+++
date = "2015-10-29"
title = "AOSP snippets"
slug = "aosp-snippets"
+++

## List of AOSP tricks, tips and snippets in no particular order

+ To merge a library manifest with the main package manifest, the flag `LOCAL_FULL_LIBS_MANIFEST_FILES` can be used.
<!--more-->
e.g:

```
LOCAL_FULL_LIBS_MANIFEST_FILES := $(LOCAL_PATH)/path/to/mylibrary/AndroidManifest.xml
```
[androidxref-source](http://androidxref.com/6.0.0_r1/xref/build/core/android_manifest.mk)

WIP
