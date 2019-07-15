---
layout: post
title : "Build a cross-platform video player in 7 days"
date: 2019-07-15 15:21:59 +0800
categories: Video Tech
author: aaron
---


It’s been quite a while that I want to write some articles about FFmpeg. Learning and using FF in production is always a painful process for me and I believe the others probably have the same feelings.

FFmpeg is powerful tool and contains rich set of features for video codec and editing. It’s hard to explain all features of it, in this series of articles. I will demonstrate how to write an simple crossplatform player based on FFmpeg, which will demonstrate the major usage of FFmpeg.

I will choose iOS as dev platform just for convince (I really hate to write JNI interface). The code will available for Android after few cross-compile work.



1. [Day 1: Setup workspace (This article)]()



# Prepare FFmpeg for iOS

*Goal: Understand how to build FFmpeg for iOS*

FFmpeg is written in C, so it’s natually available for multiple plarform if we can find suitable cross-compiler, e.g. Xcode for iOS, NDK for Android.

After the suitable tool installed, we still need to do many many configuration before we can cross build the FF. Fortunately there are plenty pre-configure build scripts for both iOS&Android in github, which will save bunch of time for us.

Locate the folder you want to save the scripts. Then run the following commands one by one.

```coffee
git clone git@github.com:kewlbear/FFmpeg-iOS-build-script.git
cd FFmpeg-iOS-build-script/
./build-ffmpeg.sh armv7 x86_64
```

> add x86_64 to enable iPhone simulator debugging.

Scripts will automatically download the necessary component such as FFmpeg sources. After the building process finished, you can find the binary lib file and header file in `./FFmpeg-iOS/`

# Player Project Setup

One thing need to be know for cross-platform project is it’s impossible to erase all platform-specific codes, it is necessary for porting our player to real world mobile project. We should place all platform-specific code together to keep core part of codes clean.

1. Based on previous analysis, we slightly changed the folder to the following version

- NKMPlayer/
  - libs/
    - include/
    - libs/
  - platform/
    - iOS/
      - FFPlayground/
    - Android/
  - core/

> FFPlayground is our test project for test our player library.

2. Create two files `NKMPlayer.cpp` `NKMPlayer.h` under `core/`
3. Add some code for configuration testing. There is only 1 member function currently, used to get ffmpeg build configuration string.


`NKMPlayer.h`

```c
#ifndef _NKMPlayer_h
#define _NKMPlayer_h
#include<string>

class NKMPlayer
{
    public:
    NKMPlayer();
    std::string ffBuildConfiguration(); 

};

#endif

```

`NKMPlayer.cpp`

```c
#include "NKMPlayer.h"
extern "C" {
    #include "libavformat/avformat.h"
}

NKMPlayer::NKMPlayer()
{

}

std::string NKMPlayer::ffBuildConfiguration()
{
    av_register_all();
    char s[1000] = {0};
    sprintf(s, "%s\n", avcodec_configuration());
    std::string result(s);
    return s;
}
```

4. Build it! Firstly we need to create a plain text file named `CMakeLists.txt` under path `src`

```yaml
# 1.
project(NKMPlayer)
cmake_minimum_required(VERSION 3.4.1)

set(CMAKE_CXX_STANDARD 11)
set(MOBILE_PLATFORM ON)
set(DEVICE_FAMILY "1")

# 2. 
FILE(GLOB SRC_FILES ${CMAKE_SOURCE_DIR}/core/*.*)

# 3.
macro(ADD_FRAMEWORK PROJECT_NAME FRAMEWORK_NAME)
    find_library(
        "FRAMEWORK_${FRAMEWORK_NAME}"
        NAMES ${FRAMEWORK_NAME}
        PATHS ${CMAKE_OSX_SYSROOT}/System/Library
        PATH_SUFFIXES Frameworks
        NO_DEFAULT_PATH
    )
    if(${FRAMEWORK_${FRAMEWORK_NAME}} STREQUAL FRAMEWORK_${FRAMEWORK_NAME}-NOTFOUND)
        MESSAGE(ERROR ": Framework ${FRAMEWORK_NAME} not found")
    else()
        TARGET_LINK_LIBRARIES(${PROJECT_NAME} ${FRAMEWORK_${FRAMEWORK_NAME}})
        message(STATUS "find framework: ${FRAMEWORK_${FRAMEWORK_NAME}}")
    endif()
endmacro(ADD_FRAMEWORK)


#ADD_DEFINE(IOS)
set(DEPLOYMENT_TARGET 8.0) 

# 4.
include_directories(
    ${CMAKE_SOURCE_DIR}/../libs/include/
    ${CMAKE_SOURCE_DIR}/core
)

# 5.
link_directories(
    ${CMAKE_SOURCE_DIR}/../libs/lib/
)

# 6. 
add_library(NKMPlayer SHARED ${SRC_FILES})
ADD_FRAMEWORK(NKMPlayer VideoToolBox)
ADD_FRAMEWORK(NKMPlayer CoreMedia)
ADD_FRAMEWORK(NKMPlayer CoreVideo)
ADD_FRAMEWORK(NKMPlayer CoreFoundation)
ADD_FRAMEWORK(NKMPlayer Security)
ADD_FRAMEWORK(NKMPlayer AudioToolBox)

# 7.
target_link_libraries(NKMPlayer
"-Wl" 
avformat avcodec avdevice avfilter avutil swresample swscale
z bz2 iconv
)
```

Explanation:
\#1. Declare our project name 
\#2. Tell cmake where to find our source files.
\#3. Add a "CMake Function" —— `ADD_FRAMEWORK` which we will use it to find&link iOS framework to our project.
\#4. Header search path.
\#5. Library search path.
\#6. Declare our target: a shared Library with name: `NKMPlayer`, and associate dependency framework. 
\#7. Finally, associate static dependency library.

5. Are we cool now? Not yet. We need to tell cmake what toolchain we want.

Generally speaking, The term "toolchain" means a set of build tools in cross-compile field which provided by target operation system company, such as we have android toolchain from NDK and iOS toolchain from Xcode. It usually includes compiler, linker, archiever, etc.

CMake has a option `-DCMAKE\_TOOLCHAIN\_FILE` allow us easily import toolchain configuration from a file. There are many well wrote iOS toolchain file available in github, e.g. <https://github.com/sheldonth/ios-cmake>. We can directly use them in out project.

Create a plain text file with name `ios.cmake` under `src`

`ios.cmake`

```yaml
 set (CMAKE_SYSTEM_NAME Darwin)
set (CMAKE_SYSTEM_VERSION 1)
set (UNIX True)
set (APPLE True)
set (IOS True)

# Required as of cmake 2.8.10
set (CMAKE_OSX_DEPLOYMENT_TARGET "" CACHE STRING "Force unset of the deployment target for iOS" FORCE)

# Determine the cmake host system version so we know where to find the iOS SDKs
find_program (CMAKE_UNAME uname /bin /usr/bin /usr/local/bin)
if (CMAKE_UNAME)
	exec_program(uname ARGS -r OUTPUT_VARIABLE CMAKE_HOST_SYSTEM_VERSION)
	string (REGEX REPLACE "^([0-9]+)\\.([0-9]+).*$" "\\1" DARWIN_MAJOR_VERSION "${CMAKE_HOST_SYSTEM_VERSION}")
endif (CMAKE_UNAME)

# Force the compilers to Clang for iOS
include (CMakeForceCompiler)
#set(CMAKE_C_COMPILER /usr/bin/clang Clang)
#set(CMAKE_CXX_COMPILER /usr/bin/clang++ Clang)
set(CMAKE_AR ar CACHE FILEPATH "" FORCE)

# Skip the platform compiler checks for cross compiling
set (CMAKE_CXX_COMPILER_WORKS TRUE)
set (CMAKE_C_COMPILER_WORKS TRUE)

# All iOS/Darwin specific settings - some may be redundant
set (CMAKE_SHARED_LIBRARY_PREFIX "lib")
set (CMAKE_SHARED_LIBRARY_SUFFIX ".dylib")
set (CMAKE_SHARED_MODULE_PREFIX "lib")
set (CMAKE_SHARED_MODULE_SUFFIX ".so")
set (CMAKE_MODULE_EXISTS 1)
set (CMAKE_DL_LIBS "")

set (CMAKE_C_OSX_COMPATIBILITY_VERSION_FLAG "-compatibility_version ")
set (CMAKE_C_OSX_CURRENT_VERSION_FLAG "-current_version ")
set (CMAKE_CXX_OSX_COMPATIBILITY_VERSION_FLAG "${CMAKE_C_OSX_COMPATIBILITY_VERSION_FLAG}")
set (CMAKE_CXX_OSX_CURRENT_VERSION_FLAG "${CMAKE_C_OSX_CURRENT_VERSION_FLAG}")

set (CMAKE_C_FLAGS_INIT "")
set (CMAKE_CXX_FLAGS_INIT "")

set (CMAKE_C_LINK_FLAGS "-Wl,-search_paths_first ${CMAKE_C_LINK_FLAGS}")
set (CMAKE_CXX_LINK_FLAGS "-Wl,-search_paths_first ${CMAKE_CXX_LINK_FLAGS}")

set (CMAKE_PLATFORM_HAS_INSTALLNAME 1)
set (CMAKE_SHARED_LIBRARY_CREATE_C_FLAGS "-dynamiclib -headerpad_max_install_names")
set (CMAKE_SHARED_MODULE_CREATE_C_FLAGS "-bundle -headerpad_max_install_names")
set (CMAKE_SHARED_MODULE_LOADER_C_FLAG "-Wl,-bundle_loader,")
set (CMAKE_SHARED_MODULE_LOADER_CXX_FLAG "-Wl,-bundle_loader,")
set (CMAKE_FIND_LIBRARY_SUFFIXES ".dylib" ".so" ".a")

# hack: if a new cmake (which uses CMAKE_INSTALL_NAME_TOOL) runs on an old build tree
# (where install_name_tool was hardcoded) and where CMAKE_INSTALL_NAME_TOOL isn't in the cache
# and still cmake didn't fail in CMakeFindBinUtils.cmake (because it isn't rerun)
# hardcode CMAKE_INSTALL_NAME_TOOL here to install_name_tool, so it behaves as it did before, Alex
if (NOT DEFINED CMAKE_INSTALL_NAME_TOOL)
	find_program(CMAKE_INSTALL_NAME_TOOL install_name_tool)
endif (NOT DEFINED CMAKE_INSTALL_NAME_TOOL)

# Setup iOS platform unless specified manually with IOS_PLATFORM
if (NOT DEFINED IOS_PLATFORM)
	set (IOS_PLATFORM "OS")
endif (NOT DEFINED IOS_PLATFORM)
set (IOS_PLATFORM ${IOS_PLATFORM} CACHE STRING "Type of iOS Platform")

# Setup building for arm64 or not
if (NOT DEFINED BUILD_ARM64)
    set (BUILD_ARM64 true)
endif (NOT DEFINED BUILD_ARM64)
set (BUILD_ARM64 ${BUILD_ARM64} CACHE STRING "Build arm64 arch or not")

# Check the platform selection and setup for developer root
if (IOS_PLATFORM STREQUAL "OS")
	set (IOS_PLATFORM_LOCATION "iPhoneOS.platform")

	# This causes the installers to properly locate the output libraries
	set (CMAKE_XCODE_EFFECTIVE_PLATFORMS "-iphoneos")
elseif (IOS_PLATFORM STREQUAL "SIMULATOR")
    set (SIMULATOR true)
	set (IOS_PLATFORM_LOCATION "iPhoneSimulator.platform")

	# This causes the installers to properly locate the output libraries
	set (CMAKE_XCODE_EFFECTIVE_PLATFORMS "-iphonesimulator")
elseif (IOS_PLATFORM STREQUAL "SIMULATOR64")
    set (SIMULATOR true)
	set (IOS_PLATFORM_LOCATION "iPhoneSimulator.platform")

	# This causes the installers to properly locate the output libraries
	set (CMAKE_XCODE_EFFECTIVE_PLATFORMS "-iphonesimulator")
else ()
	message (FATAL_ERROR "Unsupported IOS_PLATFORM value selected. Please choose OS or SIMULATOR")
endif ()

# Setup iOS developer location unless specified manually with CMAKE_IOS_DEVELOPER_ROOT
# Note Xcode 4.3 changed the installation location, choose the most recent one available
set (XCODE_POST_43_ROOT "/Applications/Xcode.app/Contents/Developer/Platforms/${IOS_PLATFORM_LOCATION}/Developer")
set (XCODE_PRE_43_ROOT "/Developer/Platforms/${IOS_PLATFORM_LOCATION}/Developer")
if (NOT DEFINED CMAKE_IOS_DEVELOPER_ROOT)
	if (EXISTS ${XCODE_POST_43_ROOT})
		set (CMAKE_IOS_DEVELOPER_ROOT ${XCODE_POST_43_ROOT})
	elseif(EXISTS ${XCODE_PRE_43_ROOT})
		set (CMAKE_IOS_DEVELOPER_ROOT ${XCODE_PRE_43_ROOT})
	endif (EXISTS ${XCODE_POST_43_ROOT})
endif (NOT DEFINED CMAKE_IOS_DEVELOPER_ROOT)
set (CMAKE_IOS_DEVELOPER_ROOT ${CMAKE_IOS_DEVELOPER_ROOT} CACHE PATH "Location of iOS Platform")

# Find and use the most recent iOS sdk unless specified manually with CMAKE_IOS_SDK_ROOT
if (NOT DEFINED CMAKE_IOS_SDK_ROOT)
	file (GLOB _CMAKE_IOS_SDKS "${CMAKE_IOS_DEVELOPER_ROOT}/SDKs/*")
	if (_CMAKE_IOS_SDKS) 
		list (SORT _CMAKE_IOS_SDKS)
		list (REVERSE _CMAKE_IOS_SDKS)
		list (GET _CMAKE_IOS_SDKS 0 CMAKE_IOS_SDK_ROOT)
	else (_CMAKE_IOS_SDKS)
		message (FATAL_ERROR "No iOS SDK's found in default search path ${CMAKE_IOS_DEVELOPER_ROOT}. Manually set CMAKE_IOS_SDK_ROOT or install the iOS SDK.")
	endif (_CMAKE_IOS_SDKS)
	message (STATUS "Toolchain using default iOS SDK: ${CMAKE_IOS_SDK_ROOT}")
endif (NOT DEFINED CMAKE_IOS_SDK_ROOT)
set (CMAKE_IOS_SDK_ROOT ${CMAKE_IOS_SDK_ROOT} CACHE PATH "Location of the selected iOS SDK")

# Set the sysroot default to the most recent SDK
set (CMAKE_OSX_SYSROOT ${CMAKE_IOS_SDK_ROOT} CACHE PATH "Sysroot used for iOS support")

# set the architecture for iOS 
if (IOS_PLATFORM STREQUAL "OS")
    set (IOS_ARCH armv7 armv7s arm64)
elseif (IOS_PLATFORM STREQUAL "SIMULATOR")
    set (IOS_ARCH i386)
elseif (IOS_PLATFORM STREQUAL "SIMULATOR64")
    set (IOS_ARCH x86_64)
endif ()

set (CMAKE_OSX_ARCHITECTURES ${IOS_ARCH} CACHE string  "Build architecture for iOS")

# Set the find root to the iOS developer roots and to user defined paths
set (CMAKE_FIND_ROOT_PATH ${CMAKE_IOS_DEVELOPER_ROOT} ${CMAKE_IOS_SDK_ROOT} ${CMAKE_PREFIX_PATH} CACHE string  "iOS find search path root")

# default to searching for frameworks first
set (CMAKE_FIND_FRAMEWORK FIRST)

# set up the default search directories for frameworks
set (CMAKE_SYSTEM_FRAMEWORK_PATH
	${CMAKE_IOS_SDK_ROOT}/System/Library/Frameworks
	${CMAKE_IOS_SDK_ROOT}/System/Library/PrivateFrameworks
	${CMAKE_IOS_SDK_ROOT}/Developer/Library/Frameworks
)

# only search the iOS sdks, not the remainder of the host filesystem
set (CMAKE_FIND_ROOT_PATH_MODE_PROGRAM ONLY)
set (CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set (CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)


# This little macro lets you set any XCode specific property
macro (set_xcode_property TARGET XCODE_PROPERTY XCODE_VALUE)
	set_property (TARGET ${TARGET} PROPERTY XCODE_ATTRIBUTE_${XCODE_PROPERTY} ${XCODE_VALUE})
endmacro (set_xcode_property)


# This macro lets you find executable programs on the host system
macro (find_host_package)
	set (CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
	set (CMAKE_FIND_ROOT_PATH_MODE_LIBRARY NEVER)
	set (CMAKE_FIND_ROOT_PATH_MODE_INCLUDE NEVER)
	set (IOS FALSE)

	find_package(${ARGN})

	set (IOS TRUE)
	set (CMAKE_FIND_ROOT_PATH_MODE_PROGRAM ONLY)
	set (CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
	set (CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
endmacro (find_host_package)
```

6. Time to do real build! Create a folder with name `build` under `NKMPlayer`, same level with `src`, then exec the following commands.

```yaml
cd build/
cmake -DCMAKE_TOOLCHAIN_FILE=./ios.cmake -DIOS_PLATFORM=SIMULATOR64 ../src -Wno-dev
```

You'll see the library file `libNKMPlayer.dylib` in `build` if everything goes well.
Wow, our hello-world library is ready to launch !

# Test Library in Simulator

1. Create a Single View App iOS project in Xcode,and save it in `NKMPlayer/platform/iOS/FFPlayground` (FFPlaygruond is the project name)
2. Add `libNKMPlayer.dylib` in build phrase tab.
3. Add some code to `ViewController.m`, make it looks like following

```objc
#import "ViewController.h"
#include "NKMPlayer.h"
#include<iostream>
using namespace std;
@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    NKMPlayer player;
    cout<<player.ffBuildConfiguration()<<endl;
    // Do any additional setup after loading the view.
}
```

4. Press CMD + R to run the program, you'll see the ffmpeg build configuration be printed in console, which will looks like:

```objc
--target-os=darwin --arch=x86_64 --cc='xcrun -sdk iphonesimulator clang' --as='gas-preprocessor.pl -- xcrun -sdk iphonesimulator clang' --enable-cross-compile --disable-debug --disable-programs --disable-doc --enable-pic --extra-cflags='-arch x86_64 -mios-simulator-version-min=8.0' --extra-ldflags='-arch x86_64 -mios-simulator-version-min=8.0' --prefix=/Users/ali/Documents/projects/github/FFmpeg-iOS-build-script/thin/x86_64
```

It’s great right? Looks lilke our player lib works well on the iOS.

# Conclusion
In this article, we finished following tasks:
1. Cross build a ffmpeg binary for iOS
2. Configure cmake-based cross platform library project with FFmpeg
3. Test our library in iPhone simulator.

We made our first step, we are going to implement basic video decode for our player, to explore the magnificent world of FFmpeg in next article.



