---
layout: post
title: "Slang: Basic Initialization Guide"
---

This article explains how to get started with Slang in a C++ project. 
    I’ll cover installation options and recommended setup with CMake and common pitfalls and errors you may encounter.
    At the end of a guide you will have a minimal shader initialization example with Vulkan.

Note: This tutorial assumes you already know the basics of C++ and CMake.

# Installing Slang
### There are several ways to add Slang to your project, and I'll describe all of them:
    Extract from VulkanSDK: Valid, but not ideal if you need recent language features not included in the SDK version.
    Add as a Git submodule: Useful if you need access to Slang source files which have different built-in types (like ```slang::List```, ```slang::string```, etc.). Downside: increases repo size significantly because Slang has 20+ dependencies.
    Use a CMake script to download release build from GitHub: Recommended if you only need to initialize the shader language.  
        Downloads only ```.lib```, ```.dll```, and ```.h``` files.
        Does not include extra dependencies or source code, so debugging Slang internals in unusual situations requires building from source manually.

In this tutorial, I will focus on the CMake script method, as it is more scalable and keeps your project lightweight.
## CMake Setup
```cpp
# Script to install slang. As it is incovenient to install slang via submodules, because you would need to 
# install every dependency

include(FetchContent)

function(download_slang)
    set(SLANG_TARGET_DIR "${CMAKE_SOURCE_DIR}/vendor/slang")
    set(SLANG_BIN "${SLANG_TARGET_DIR}/bin")

    if(EXISTS "${SLANG_BIN}/slangc" OR EXISTS "${SLANG_BIN}/slang.dll")
        message(STATUS "Slang already installed in ${SLANG_TARGET_DIR}")
        setup_slang_variables(${SLANG_TARGET_DIR})
        return()
    endif()

    set(TEMP_DIR "${CMAKE_BINARY_DIR}/slang_tmp")
    file(MAKE_DIRECTORY "${TEMP_DIR}")

    set(RELEASE_API "https://api.github.com/repos/shader-slang/slang/releases/latest")
    file(DOWNLOAD "${RELEASE_API}" "${TEMP_DIR}/release.json" STATUS STATUS_LIST)
    list(GET STATUS_LIST 0 STATUS_CODE)
    if(NOT STATUS_CODE EQUAL 0)
        message(FATAL_ERROR "Failed to fetch Slang release info")
    endif()

    file(READ "${TEMP_DIR}/release.json" RELEASE_JSON)
    string(REGEX MATCH "\"tag_name\"[ \t]*:[ \t]*\"([^\"]+)\"" _ "${RELEASE_JSON}")
    set(VERSION "${CMAKE_MATCH_1}")
    string(REGEX REPLACE "^v" "" VERSION_NUM "${VERSION}")

    if(WIN32)
        set(ARCHIVE "slang-${VERSION_NUM}-windows-x86_64.zip")
    elseif(UNIX AND NOT APPLE)
        set(ARCHIVE "slang-${VERSION_NUM}-linux-x86_64.zip")
    else()
        message(FATAL_ERROR "Unsupported platform")
    endif()

    set(URL "https://github.com/shader-slang/slang/releases/download/${VERSION}/${ARCHIVE}")
    set(ZIP_PATH "${TEMP_DIR}/${ARCHIVE}")

    message(STATUS "Downloading Slang from ${URL}")
    file(DOWNLOAD "${URL}" "${ZIP_PATH}" SHOW_PROGRESS STATUS STATUS_LIST)
    list(GET STATUS_LIST 0 STATUS_CODE)
    if(NOT STATUS_CODE EQUAL 0)
        message(FATAL_ERROR "Failed to download Slang archive")
    endif()

    file(MAKE_DIRECTORY "${SLANG_TARGET_DIR}")
    execute_process(COMMAND ${CMAKE_COMMAND} -E tar xzf "${ZIP_PATH}" WORKING_DIRECTORY "${SLANG_TARGET_DIR}" RESULT_VARIABLE RES)
    if(NOT RES EQUAL 0)
        # fallback for Windows zip
        execute_process(COMMAND ${CMAKE_COMMAND} -E tar xf "${ZIP_PATH}" WORKING_DIRECTORY "${SLANG_TARGET_DIR}" RESULT_VARIABLE RES_ZIP)
        if(NOT RES_ZIP EQUAL 0)
            message(FATAL_ERROR "Failed to extract Slang archive")
        endif()
    endif()

    # Flatten one extra subdir if needed
    file(GLOB SLANG_SUBDIR "${SLANG_TARGET_DIR}/slang-*")
    list(LENGTH SLANG_SUBDIR COUNT)
    if(COUNT EQUAL 1)
        list(GET SLANG_SUBDIR 0 ACTUAL_DIR)
        file(GLOB CONTENTS "${ACTUAL_DIR}/*")
        foreach(ITEM ${CONTENTS})
            get_filename_component(NAME ${ITEM} NAME)
            file(RENAME "${ITEM}" "${SLANG_TARGET_DIR}/${NAME}")
        endforeach()
        file(REMOVE_RECURSE "${ACTUAL_DIR}")
    endif()

    file(REMOVE_RECURSE "${TEMP_DIR}")
    setup_slang_variables(${SLANG_TARGET_DIR})
endfunction()

function(setup_slang_variables ROOT)
    set(SLANG_INCLUDE_DIR "${ROOT}/include" CACHE PATH "")
    set(SLANG_LIBRARY_DIR "${ROOT}/lib" CACHE PATH "")
    set(SLANG_BINARY_DIR "${ROOT}/bin" CACHE PATH "")

    set(SLANG_INCLUDE_DIR "${SLANG_INCLUDE_DIR}" PARENT_SCOPE)
    set(SLANG_LIBRARY_DIR "${SLANG_LIBRARY_DIR}" PARENT_SCOPE)
    set(SLANG_BINARY_DIR "${SLANG_BINARY_DIR}" PARENT_SCOPE)

    message(STATUS "Slang paths:")
    message(STATUS "  Include: ${SLANG_INCLUDE_DIR}")
    message(STATUS "  Lib:     ${SLANG_LIBRARY_DIR}")
    message(STATUS "  Bin:     ${SLANG_BINARY_DIR}")
endfunction()

```
### To be short and precise: this script would download and build slang from source if it's already not installed in the project.

You can copy paste this script as CMake is sometimes really painful to work with and it would save some time for you.
All you left to do now is to include this file in your main CMake script like this: ```include(${CMAKE_SOURCE_DIR}/CMakeScripts/slang-setup.cmake)```
The next step would be to place the function ```download_slang()``` somewhere below in this file, it would call aforementioned script and if slang library is not found in your project it would proceed to download it.

By executing download slang script now you should be able to call 3 user defined CMake variables:
- ${SPIRV_LIBRARY_DIR} - path to the directory with the library files
- ${SLANG_INCLUDE_DIR} - path to the include directory
- ${SLANG_BINARY_DIR}  - path to the directory with slang binary files

Now you would need only the first two.
You have to make all header files visible to your project, so inside of target_include_directories specify the include dir mentioned above.
Then you can link the library, easiest way to do that would be to make a target_link_directories and specify aforementioned command with the lib directory path(don't forget to link slang after that, just by specifying target_link_libraries(target access modifier slang)

Tiny example of the code:
```cpp
include(${CMAKE_SOURCE_DIR}/CMakeScripts/slang-setup.cmake)
download_slang()
target_include_directories(MyProject PRIVATE ${SLANG_INCLUDE_DIR})
target_link_directories(MyProject PRIVATE ${SLANG_LIBRARY_DIR})
target_link_libraries(MyProject PRIVATE slang)
```
And that's it. Now when you will compile your project it should download the slang if it doesn't exist already and link after.

# Creating a Global and Local Slang Session
## This section explains how to create the two main objects required to use the Slang API:
-   A global session which just represents a connection from an application to an implementation of the Slang API
-   A local session which represents a shader compilation context with specific target settings

### Global session
The first thing you need is a global session.
It acts as a top-level connection between your application and the Slang runtime. ***It must be created only once per thread, and cannot be shared between threads.***
If your application is multithreaded, create a separate global session per thread.

And it can be created as simple as:
```cpp
#define SLANG_CHECK(x)                 \
{                                      \
    auto _res = x;                     \
    if(_res != 0)                      \
    {                                  \
        assert(false);                 \
    }                                  \
}

slang::IGlobalSession* _globalSession = nullptr;
SLANG_CHECK(slang::createGlobalSession(&_globalSession));
```
Where SLANG_CHECK is a macro that we will also use later to check whether result code of function call is true.


