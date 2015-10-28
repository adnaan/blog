+++
date = "2015-10-29"
title = "AOSP tidbits"
slug = "aosp-tidbits"

+++

To merge a library manifest with the main package manifest, the flag LOCAL_FULL_LIBS_MANIFEST_FILES can be used.

e.g:

LOCAL_FULL_LIBS_MANIFEST_FILES := $(LOCAL_PATH)/path/to/library/AndroidManifest.xml
