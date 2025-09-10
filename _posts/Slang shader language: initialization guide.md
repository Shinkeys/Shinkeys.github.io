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
