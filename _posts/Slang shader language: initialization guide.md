---
layout: post
title: "Slang: Basic Initialization Guide"
---
<h1>Slang: Basic Initialization Guide</h1>

<p>This article explains how to get started with <strong>Slang</strong> in a C++ project. I’ll cover:</p>
<ul>
    <li>Installation options and recommended setup with CMake.</li>
    <li>Common pitfalls and errors you may encounter.</li>
    <li>A minimal initialization example with Vulkan.</li>
</ul>

<div class="note">
Note: This tutorial assumes you already know the basics of C++ and CMake.
</div>

<hr>

<h2>Installing Slang</h2>

<p>There are several ways to add Slang to your project, and I'll describe all of them:</p>

<ul>
    <li><strong>Extract from VulkanSDK:</strong> Valid, but not ideal if you need recent language features not included in the SDK version.</li>
    <li><strong>Add as a Git submodule:</strong> Useful if you need access to Slang source files which have different built-in types (like <code>slang::List</code>, <code>slang::string</code>, etc.). Downside: increases repo size significantly because Slang has 20+ dependencies.</li>
    <li><strong>Use a CMake script to download release build from GitHub:</strong> <em>Recommended</em> if you only need to initialize the shader language.  
        <ul>
            <li>Downloads only <code>.lib</code>, <code>.dll</code>, and <code>.h</code> files.</li>
            <li>Does not include extra dependencies or source code, so debugging Slang internals in unusual situations requires building from source manually.</li>
        </ul>
    </li>
</ul>

<div class="note">
In this tutorial, I will focus on the CMake script method, as it is more scalable and keeps your project lightweight.
</div>
<hr>
<h2>CMake Setup</h2>
INSERT CODE !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!<br>
To be short and precise: this script would download and build slang from source if it's already not installed in the project.

You can copy paste this script as CMake is sometimes really painful to work with and it would save some time for you.
All you left to do now is to include this file in your main CMake script like this: <code>include(${CMAKE_SOURCE_DIR}/CMakeScripts/slang-setup.cmake)</code>
The next step would be to call the function <code>download_slang()</code> somewhere below in this file, it would call aforementioned script and if slang library is not found in your project it would proceed to download.

By executing download slang script now you should be able to call 3 user defined CMake variables:
- ${SPIRV_LIBRARY_DIR} - path to the directory with the library files
- ${SLANG_INCLUDE_DIR} - path to the include directory
- ${SLANG_BINARY_DIR}  - path to the directory with slang binary files

Now you would need only the first two.
You need to make all header files visible to your project, so inside of target_include_directories specify the include dir mentioned above.
Then you can link the library, easiest way to do that would be to make a target_link_directories and specify aforementioned command with the lib directory path(don't forget to link slang after that, just by specifying target_link_libraries(target access modifier slang))

Tiny example of the code:
<code>
include(${CMAKE_SOURCE_DIR}/CMakeScripts/slang-setup.cmake)
download_slang()
target_include_directories(MyProject PRIVATE ${SLANG_INCLUDE_DIR})
target_link_directories(MyProject PRIVATE ${SLANG_LIBRARY_DIR})
target_link_libraries(MyProject PRIVATE slang)
</code>

