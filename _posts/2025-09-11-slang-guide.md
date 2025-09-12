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
- Extract from VulkanSDK: Valid, but not ideal if you need recent language features not included in the SDK version.
- Add as a Git submodule: Useful if you need access to Slang source files which have different built-in types (like ```slang::List```, ```slang::string```, etc.). Downside: increases repo size significantly because Slang has 20+ dependencies.
- Use a CMake script to download release build from GitHub: Recommended if you only need to initialize the shader language.

In this tutorial, I will focus on the CMake script method, as it downloads only ```.lib```, ```.dll```, and ```.h``` files.
However, it ***does not*** include extra dependencies or source code, so debugging Slang internals in unusual situations requires building from source manually.
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
You have to make all header files visible to your project, so inside of ```target_include_directories specify``` the include directory mentioned above.
Then you can link the library, easiest way to do that would be to call a ```target_link_directories``` and specify aforementioned command with the lib directory path(don't forget to link Slang after that, just by specifying ```target_link_libraries(target access modifier slang)```

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

Slang::ComPtr<slang::IGlobalSession> _globalSession;
SLANG_CHECK(slang::createGlobalSession(&_globalSession));
```
> Where ```SLANG_CHECK``` is a user-defined macro that we will also use later to check whether result code of the function call is true.  
> ```ComPtr``` is just a smart pointer.

### Local session
A local session is where the actual shader compilation setup happens.
This session is configured with compilation targets, matrix layout rules and other language options.
It requires two fields, where the first one is ```slang::TargetDesc  targetDesc{};```
A target basically specifies how the shader will be compiled.
```cpp
targetDesc.format  = SLANG_SPIRV;
targetDesc.profile = _globalSession->findProfile("sm_6_8");
targetDesc.flags = SLANG_TARGET_FLAG_GENERATE_SPIRV_DIRECTLY;
targetDesc.forceGLSLScalarBufferLayout = true;
```
As this tutorial is written for Vulkan I suggest you to use these settings.  
Flag ```SLANG_TARGET_FLAG_GENERATE_SPIRV_DIRECTLY``` will highlight to the slang that it can generate SPIR-V bytecode directly bypassing HLSL stage as it's not necessary with Vulkan.
Profile "sm_6_8"(Shader Model 6.8) will allow you to use every modern slang feature.
> If you utilize Vulkan's __VK_EXT_scalar_block_layout__ you must specify this in target as I did above by setting ```forceGLSLScalarBufferLayout``` to true. And if you don't I recommend you to enable this extension as it's in the core since Vulkan 1.2 and allows you to avoid CPU/GPU struct padding issues.

### The last thing left is to create local session is to describe the session itself
```cpp
slang::SessionDesc sessionDesc{};
sessionDesc.targets = &targetDesc;
sessionDesc.targetCount = 1;
sessionDesc.defaultMatrixLayoutMode = SLANG_MATRIX_LAYOUT_COLUMN_MAJOR; // GLSL-like
```
> By default, Slang uses row-major matrices, GLSL, however, assumes column-major. If you want to keep it like that set ```defaultMatrixLayoutMode``` as I did above

And now we can create a local session by simply calling
```cpp
Slang::ComPtr<slang::ISession> _localSession;
SLANG_CHECK(_globalSession->createSession(sessionDesc, &_localSession));
```
That's it - at this point you should have both a global and a local session created.

# Shader modules
Now we can create Vulkan shader modules, which are part of pipeline creation and I won't describe this whole process but only shader modules creation.
To create it we will need ```slang::IModule*```  which we need to load by providing a path to the shader and it's name.
To not recreate modules in case if you will create a different pipelines with the same shader I suggest to wrap created above sessions and modules in such a class:
```cpp
class VulkanShader
{
private:
	std::unordered_map<std::string, slang::IModule*> _modulesStorage;

	Slang::ComPtr<slang::IGlobalSession> _globalSession;
	Slang::ComPtr<slang::ISession> _localSession;
public:
	slang::IModule* LoadModule(const fs::path& shaderPath);

	slang::IModule* GetModuleByName(const std::string& name);

	VkShaderModule CreateShaderModule(slang::IModule* slangModule, const std::string& entrypoint);
};
```
When creating a pipeline  provide a shader name and pass it to ```LoadModule``` method like this:
```cpp
fs::path shaderPath = _specification.shaderName;

shaderPath += ".slang";

slang::IModule* slangModule = _shaderObject.LoadModule(shaderPath);
```
> fs::path is an  alias for std::filesystem::path

Sadly, ```std::filesystem::path``` doesn’t provide a direct way to locate your project root, so in this method we will need to transform the path somehow.
As an option I decided to implement this with a loop:
```cpp
fs::path currentDir = fs::current_path();
while (!helpers::IsProjectRoot(currentDir))
{
	currentDir = currentDir.parent_path();
}

// Purpose: check if in project's root now
inline bool IsProjectRoot(const fs::path& path)
{
	return fs::exists(path / "headers") && fs::exists(path / "src");
}
```
It starts at the working directory and walks upward until it finds the project root.
I also recommend to create a Slang-specific helper function:
```cpp
void PrintDiagnosticBlob(ComPtr<slang::IBlob> blob)
{
#ifndef NDEBUG
	if (blob != nullptr)
	{
		printf("%s", (const char*)blob->getBufferPointer());
	}
#endif
}
```
Which will print diagnostic messages to the console where error occurs. I __strongly__ suggest you to enable it, especially that we've downloaded slang build without sources and as a result unable to debug it properly without building manually.
A complete ```LoadModule``` method example:
```cpp
slang::IModule* VulkanShader::LoadModule(const fs::path& shaderName)
{
	fs::path currentDir = fs::current_path();
	while (!helpers::IsProjectRoot(currentDir))
	{
		currentDir = currentDir.parent_path();
	}

	fs::path changedShaderPath = currentDir / "resources" / "shaders" / shaderName;
	const std::string shaderPathStr = changedShaderPath.string();

	slang::IModule* slangModule = nullptr;
  	if (_modulesStorage.find(shaderPathStr) == _modulesStorage.end())
	{
		ComPtr<slang::IBlob> diagnosticsBlob;
		slangModule = _localSession->loadModule(shaderPathStr.c_str(), diagnosticsBlob.writeRef());
		PrintDiagnosticBlob(diagnosticsBlob);

		if (!slangModule)
			std::abort();

		_modulesStorage[shaderPathStr] = slangModule;
	}
	else
		slangModule = _modulesStorage[changedShaderPath.string()];

	return slangModule;
}
```
The key ideas are already described above. In my case it searches in the module storage and if the module exists already it will just return it, otherwise create a new one by passing transformed path to the shader and a diagnostic blob as a second parameter to enable validation on errors.

Now in the pipeline creation method you can call ```CreateShaderModule``` which will return ```VkShaderModule``` on success. It accepts Slang's module we've just created and an entrypoint of the shader.

### Entrypoint
An entrypoint is just a name of the shader. For example:  
```cpp
[shader("fragment")]
FragmentOutput FragmentMain(VertexOutput input)
```
> ```FragmentMain``` is the entrypoint

```CreateShaderModule``` first of all will try to find an entrypoint by name we passed to it. Then it will link the program from this entrypoint with every module you will import to this shader(I'll describe it later) and after that it will get the entrypoint code in SPIR-V from which we are finally able to create a ```VkShaderModule```
Full method:
```cpp
VkShaderModule VulkanShader::CreateShaderModule(slang::IModule* slangModule, const std::string& entrypointName)
{
	assert(slangModule && "Can't create shader module, slang module is nullptr");


	ComPtr<slang::IEntryPoint> entryPoint;
	SlangResult result = slangModule->findEntryPointByName(entrypointName.c_str(), entryPoint.writeRef());
	if (result != 0)
	{
		std::cout << "Unable to create shader module by entrypoint: " << entrypointName << '\n';
		assert(false);
	}


	ComPtr<slang::IComponentType> linkedProgram;
	{
		ComPtr<slang::IBlob> diagnosticsBlob;
		SlangResult result = entryPoint->link(linkedProgram.writeRef(), diagnosticsBlob.writeRef());
		PrintDiagnosticBlob(diagnosticsBlob);

		SLANG_CHECK(result);	
	}

	ComPtr<slang::IBlob> spirv;
	{
		ComPtr<slang::IBlob> diagnosticsBlob;
		SlangResult result = linkedProgram->getEntryPointCode(
			0,
			0,
			spirv.writeRef(),
			diagnosticsBlob.writeRef());
		PrintDiagnosticBlob(diagnosticsBlob);

		SLANG_CHECK(result);
	}
	 

	VkShaderModule shaderModule = vkhelpers::ReadShaderFile(static_cast<const u32*>(spirv->getBufferPointer()),
		spirv->getBufferSize(), _deviceObj.GetDevice());


	return shaderModule;
}
```
Where ```ReadShaderFile```:
```cpp
VkShaderModule ReadShaderFile(const u32* data, size_t size, VkDevice device)
{
	VkShaderModuleCreateInfo createInfo{ VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO };
	createInfo.pCode = data;
	createInfo.codeSize = size;


	VkShaderModule shaderModule;
	VK_CHECK(vkCreateShaderModule(device, &createInfo, nullptr, &shaderModule));

	return shaderModule;
}
```
We need to use ```static_cast``` manually because ```getBufferPointer``` returns a void* data.
Now you can use this base to create a modules for every type of the shaders: Vertex, Fragment, Compute, RayGen and others.

# Shaders
I will show some examples of shaders in Slang and highlight the key advantages over GLSL/HLSL.
In my opinion, the main features of Slang is it's modularity and flexibility. Unlike GLSL, it allows you to have different shader stages in one file, allowing you to store vertex and fragment shaders together in one file or even the entire ray tracing pipeline. Which sounds logical, since the feature exists in SPIR-V, which means the problem was in the language itself.
The second is support for many modern things, like buffer pointers if you use Buffer Device Address. In GLSL you have to first create a structure that explains this.

> To find modules in specific folder place the file named ```slangdconfig.json``` to the main directory of the shader with this content
```cpp
{
  "slang.additionalSearchPaths": [
    "./common",
    "./"
  ]
}
```
Which will provide the folder to look for modules to Slang.

Slang's syntax is very C#-like so you can create a structure like that: 
```cpp
public struct Triangle
{
    public Array<float3, 3> pos;
    public Array<float2, 3> uv;
}
```
Which means it will be visible in every file imported it(yes, you need to make fields public as well)
You can do it in GLSL as well, however Slang is giving this possibility for modules out of the box which is the advantage I think.

For example, create some module to abstract some logic and you can import it in the other shader like that ```import common.mesh_common;``` where __common__ is the folder and __.__ is the __/__ for the folder.
In GLSL you have a built-in variable ```gl_NumWorkGroups``` to get the dispatch size, Slang and HLSL, however, doesn't have it. What to do?
Not a problem, if you use Vulkan just use an assembly to get this variable while preserving HLSL-like syntax:
```cpp
uint3 GetWorkgroupCount()
{
    __target_switch
    {
    case glsl: __intrinsic_asm "gl_NumWorkGroups";
    case spirv:
        return spirv_asm {
                result:$$uint3 = OpLoad builtin(NumWorkgroups:uint3);
           };
    }
}
```

Here's the example of the shader:
```cpp
import common.camera;
import common.common;
import common.lights;
import common.PBR_common;


[[vk::push_constant]]
cbuffer PushConstants
{ 
    PointLight *lightsPtr;
    int *lightIndicesPtr;
    ViewData *viewDataPtr;

    uint positionTexIndex;
    uint normalsTexIndex;
    uint albedoTexIndex;
    uint metallicRoughnessTexIndex;

    uint pointLightsCount;
    uint tileSize;
};

static const Array<float3, 6> vertices = 
{
    float3(-1.0f, -1.0f, 0.0f),
    float3(1.0f, -1.0f, 0.0f),
    float3(1.0f, 1.0f, 0.0f),
    float3(1.0f, 1.0f, 0.0f),
    float3(-1.0f, 1.0f, 0.0f),
    float3(-1.0f, -1.0f, 0.0f)
};

static const Array<float2, 6> texCoords =
{
	float2(0.0f, 1.0f),
    float2(1.0f, 1.0f),
    float2(1.0f, 0.0f),
    float2(1.0f, 0.0f),
    float2(0.0f, 0.0f),
    float2(0.0f, 1.0f)
};

struct VertexInput
{
    uint vertexIndex : SV_VertexID;
};

struct VertexOutput
{
    float4 position : SV_Position;
    float2 texCoord;
};

[shader("vertex")]
VertexOutput VertexMain(VertexInput input)
{
    VertexOutput output = (VertexOutput)0;
    output.position = float4(vertices[input.vertexIndex], 1.0);
    output.texCoord = float2(texCoords[input.vertexIndex]);

    return output;
}

[vk::binding(0, 0)]
public Sampler2D textures[];
[vk::binding(1, 0)]
public RWTexture2D<uint2> lightsGrid;

struct FragmentOutput 
{
    float4 color : SV_TARGET0;
};

[shader("fragment")]
FragmentOutput FragmentMain(VertexOutput input)
{
    FragmentOutput output = (FragmentOutput)0;

    float3 positions = float3(0.0);
    if (positionTexIndex > 0)
    {
        positions = textures[positionTexIndex].Sample(input.texCoord).xyz;
    }

    float3 albedoColor = float3(0.5);
    if (albedoTexIndex > 0)
    {
        albedoColor = textures[albedoTexIndex].Sample(input.texCoord).xyz;
    }

    float3 normals = float3(0.0);
    if (normalsTexIndex > 0)
    {
        normals = textures[normalsTexIndex].Sample(input.texCoord).xyz;
    }

    float3 metallicRoughnessColor = float3(0.0);
    if (metallicRoughnessTexIndex > 0)
    {
        metallicRoughnessColor = textures[metallicRoughnessTexIndex].Sample(input.texCoord).xyz;
    }

    int3 fragCoord = int3(input.position.xyz);

    uint2 lightsDataInTile = lightsGrid.Load(int2(fragCoord.x / tileSize, fragCoord.y / tileSize));
    uint startIndex = lightsDataInTile.x;
    uint lightsCount = lightsDataInTile.y;  

    float3 lightingResult = albedoColor * 0.1;

    float3 Lo = float3(0.0);
    for (uint i = 0; i < lightsCount; ++i)
    {
        uint lightIndex = lightIndicesPtr[i + startIndex];
        PointLight pointLight = lightsPtr[lightIndex];

        Lo += CalculateLight(pointLight, albedoColor, metallicRoughnessColor, normals, viewDataPtr.position, positions);
    }

    float3 ambient = float3(0.001) * albedoColor;
    float3 color = ambient + Lo;
    color = color / (color + float3(1.0));
    color = pow(color, float3(1.0 / 2.2));

    output.color = float4(color, 1.0);

    return output;
}
```
If you use a texture then use ```Sampler2D```, if you need a storage image you can use ```RWTexture2D<uint2>```
This is just a simple example: Slang is giving a possibilities to use generics, interfaces which allows you to reuse the methods with multiple data, cross-target compilation. A modularity provides a data-reuse instead of "copying-pasting" source code every time you include the shader, which allows to have a circular-includes.  
I won't describe every possibility it gives to you, as you can read their [user's guide](https://shader-slang.org/slang/user-guide/)  
Investigate [Sascha's Willems examples](https://github.com/SaschaWillems/Vulkan/tree/master/shaders/slang) in Slang.

# Suggestions
One of the core reasons I decided to switch from GLSL was a syntax highlight. If you use Vulkan with such a modern features as Buffer Device Address, GLSL syntax highlighters will show an error, although the code compiles because it has no up-to-date linter which forces you to work without syntax highlight at all or to observe constant errors.
Slang, however, has an official Visual Studio and Visual Studio Code language extensions and they work great.
