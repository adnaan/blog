+++
date = "2015-11-05"
title = "AOSP snippets"
slug = "aosp-snippets"
+++

#### List of AOSP related tips, links and snippets in no particular order

+ Code search : For fast indexed code search a CLI tool goes a long way. Switching context between Opengrok and the terminal gets frustrating. Using https://github.com/google/codesearch cuts down the search time from minutes (ag, grep) to milliseconds.

+ Local repo init: I found the flag "--reference" in the repo tool quite useful. Easy way to sync the whole tree on a new system.

```
repo init --reference=~/android/cm -u git://url/to/manifest.git -b marshmallow
```

[More repo tricks and tips at xda-university](http://xda-university.com/as-a-developer/repo-tips-tricks)

<!--more-->

+ Repo sync only current branch. Normally rep would fetch all the branches for a base manifest.

```
$ repo sync -c
```

+ repo forall executes a command for all repos. For e.g it can be used to find a certain word in commit log of all repos.

```
$ repo forall -vpc "git log --all --grep='bugfix'" -j10
```
+ [IDEGen](https://android.googlesource.com/platform/development/+/master/tools/idegen/README) : Viewing AOSP sources in Android Studio.

```
$ source build/envsetup.sh
$ lunch aosp_arm-eng # your target
$ make -j10
$ mmm development/tools/idegen/
$ development/tools/idegen/idegen.sh
```

In AOSP root `android.ipr` and `android.iml` are produced. Now the whole source can be imported in android studio.

+ To merge a library manifest with the main package manifest, the flag `LOCAL_FULL_LIBS_MANIFEST_FILES` can be used.

e.g:

```
LOCAL_FULL_LIBS_MANIFEST_FILES := $(LOCAL_PATH)/path/to/mylibrary/AndroidManifest.xml
```
[androidxref-source](http://androidxref.com/6.0.0_r1/xref/build/core/android_manifest.mk)

+ Build using build.sh. It's a wrapper script over make and provides several useful utilities:

```
$ cd root-android
$ ./build.sh product_name -d -jN
```

Options to build only images and modules are also available. A full list of options can be seen in the usage of the script.

```
$ ./build.sh -h
```

```
Usage:
    bash ./build.sh <TARGET_PRODUCT> [OPTIONS]

Description:
    Builds Android tree for given TARGET_PRODUCT

OPTIONS:
    -c, --clean_build
        Clean build - build from scratch by removing entire out dir

    -d, --debug
        Enable debugging - captures all commands while doing the build

    -h, --help
        Display this help message

    -i, --image
        Specify image to be build/re-build (bootimg/sysimg/usrimg)

    -j, --jobs
        Specifies the number of jobs to run simultaneously (Default: 8)

    -k, --kernel_defconf
        Specify defconf file to be used for compiling Kernel

    -l, --log_file
        Log file to store build logs (Default: <TARGET_PRODUCT>.log)

    -m, --module
        Module to be build

    -p, --project
        Project to be build

    -s, --setup_ccache
        Set CCACHE for faster incremental builds (true/false - Default: true)

    -u, --update-api
        Update APIs

    -v, --build_variant
        Build variant (Default: userdebug)
```

+ See all build environment commands:

```
$ source build/envsetup.sh
$ hmm
```

```
Invoke ". build/envsetup.sh" from your shell to add the following functions to your environment:
- lunch:   lunch <product_name>-<build_variant>
- tapas:   tapas [<App1> <App2> ...] [arm|x86|mips|armv5|arm64|x86_64|mips64] [eng|userdebug|user]
- croot:   Changes directory to the top of the tree.
- m:       Makes from the top of the tree.
- mm:      Builds all of the modules in the current directory, but not their dependencies.
- mmm:     Builds all of the modules in the supplied directories, but not their dependencies.
           To limit the modules being built use the syntax: mmm dir/:target1,target2.
- mma:     Builds all of the modules in the current directory, and their dependencies.
- mmma:    Builds all of the modules in the supplied directories, and their dependencies.
- cgrep:   Greps on all local C/C++ files.
- ggrep:   Greps on all local Gradle files.
- jgrep:   Greps on all local Java files.
- resgrep: Greps on all local res/*.xml files.
- mangrep: Greps on all local AndroidManifest.xml files.
- sepgrep: Greps on all local sepolicy files.
- sgrep:   Greps on all local source files.
- godir:   Go to the directory containing a file.

Environemnt options:
- SANITIZE_HOST: Set to 'true' to use ASAN for all host modules. Note that
                 ASAN_OPTIONS=detect_leaks=0 will be set by default until the
                 build is leak-check clean.

Look at the source to view more functions. The complete list is:
addcompletions add_lunch_combo cgrep check_product check_variant choosecombo chooseproduct choosetype choosevariant core coredump_enable coredump_setup cproj croot findmakefile get_abs_build_var getbugreports get_build_var getdriver getlastscreenshot get_make_command getprebuilt getscreenshotpath getsdcardpath gettargetarch gettop ggrep godir hmm is isviewserverstarted jgrep key_back key_home key_menu lunch _lunch m make mangrep mgrep mm mma mmm mmma pez pid printconfig print_lunch_menu qpid rcgrep resgrep runhat runtest sepgrep set_java_home setpaths set_sequence_number set_stuff_for_environment settitle sgrep smoketest stacks startviewserver stopviewserver systemstack tapas tracedmdump treegrep
```
...
WIP
