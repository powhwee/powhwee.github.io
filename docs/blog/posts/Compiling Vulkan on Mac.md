---
date:
  created: 2025-10-27
  updated: 2025-10-27
title: "Compiling a Vulkan code developed on Windows on a Mac"
categories:
  - Technology
tags:
  - Vulkan

draft: false
---
<!-- more -->

#Compiling and Running Vulkan program on Mac

I was trying to compile and run a Vulkan program written on Windows.  The program does not intentionally use any VisualC++ nor Windows features, 
but there are minor adjustments required.  This entry seek to capture the learnings from this exercise.

##  Pre-requisites
- vcpkg.   clone from here 'https://github.com/microsoft/vcpkg'
- Vulkan SDK.  For Mac, download from here:  https://vulkan.lunarg.com/sdk/home

### vcpkg

You can do a 'brew install vcpkg', but you will still need to git clone the repository (see above link).  
This is because homebrew only install executable, but vcpkg has other files that it needs to do its job.
For example, when compiling with Clang (or gcc), you would need to pass a flag eg.
`cmake -Dcmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=/<path where vcpkg is installed/vcpkg/scripts/buildsystems/vcpkg.cmake`

You will also need to set VCPKG_ROOT.

### Vulkan SDK

The detail instruction to install Vulkan SDK on mac is here https://vulkan.lunarg.com/doc/view/latest/mac/getting_started.html

**Options when installing**
The installation, whether via GUI or command line, has the option to install header and libraries in system path eg. /usr/local/bin; or paths
that are usually searched by compilers and linkers.

My suggestion is *not* to include installation into the system path.  Reasons being:
- Flexibility to run different versions of SDK when needed; having libraries in system path will easily cause conflicts with different versions
- It is not difficult to set the include and library paths when you need to run the executable





# Debugging log in compiling a vulkan program

# NJin Debugging Log

This document outlines the step-by-step process of debugging and fixing the `njin` Vulkan application.

---

## 1. Vulkan Portability Error on macOS

#### Error Message

```
vkCreateInstance: Found drivers that contain devices which support the portability subset, but the instance does not enumerate portability drivers! Applications that wish to enumerate portability drivers must set the VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR bit in the VkInstanceCreateInfo flags.
vkCreateInstance: Found no drivers!
libc++abi: terminating due to uncaught exception of type std::runtime_error: failed to create instance!
```

#### Problem

The application failed to create a Vulkan instance on macOS because it didn't enable the required portability extension needed for MoltenVK (the Vulkan-to-Metal translation layer).

#### Fixes

1.  **Enable Portability Instance Extension**

    *   **File:** `njin/vulkan/src/Instance.cpp`
    *   **Change:** Added the `VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR` flag and the corresponding enumeration extension name.

    ```diff
+        extensions.push_back(VK_KHR_PORTABILITY_ENUMERATION_EXTENSION_NAME);
+
+        info.flags |= VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR;
+
       // extensions
       info.enabledExtensionCount = static_cast<uint32_t>(extensions.size());
       info.ppEnabledExtensionNames = extensions.data();
   
       if (vkCreateInstance(&info, nullptr, &instance_) != VK_SUCCESS) {
           throw std::runtime_error("failed to create instance!");
       }
  ```

2.  **Enable Portability Device Extension**

    *   **File:** `njin/vulkan/src/LogicalDevice.cpp`
    *   **Change:** Added the `VK_KHR_portability_subset` device extension.

    ```diff
const std::vector<const char*> physical_device_extensions = {
VK_KHR_SWAPCHAIN_EXTENSION_NAME,
VK_KHR_SEPARATE_DEPTH_STENCIL_LAYOUTS_EXTENSION_NAME,
VK_KHR_CREATE_RENDERPASS_2_EXTENSION_NAME,
+    "VK_KHR_portability_subset",
     };
     ```

---

## 2. Descriptor Pool Allocation Failure

#### Error Message

```
vkAllocateDescriptorSets(): Trying to allocate 100 of VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER descriptors from VkDescriptorPool 0xf000000000f, but this pool only has a total of 1 descriptors for this type...
libc++abi: terminating due to uncaught exception of type std::runtime_error: failed to allocate descriptor sets
```

#### Problem

The `VkDescriptorPool` was created with an incorrect size. The code was counting the number of descriptor *bindings* (which was 1 for each type) instead of summing the total *count* of descriptors required by each binding (which was 100).

#### Fix

*   **File:** `njin/vulkan/src/DescriptorPool.cpp`
*   **Change:** Correctly summed the `descriptor_count` from each binding.

    ```diff
-                descriptor_type_to_count[binding_info.descriptor_type]++;
+                descriptor_type_to_count[binding_info.descriptor_type] += binding_info.descriptor_count;
    ```

---

## 3. Invalid Image Layouts

This issue involved a two-step fix.

### Part A: Image Layout Not Set

#### Error Message

```
Validation Error: [ VUID-vkCmdDraw-None-09600 ] ... expects VkImage ... to be in layout VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL--instead, current layout is VK_IMAGE_LAYOUT_UNDEFINED.
```

#### Problem

Dummy placeholder images were created in `VK_IMAGE_LAYOUT_UNDEFINED` but were being used in shaders without being transitioned to a shader-readable layout first. The validation layers correctly identified that the image layout did not match its usage.

#### Attempted Fix

A layout transition was added to `njin/vulkan/src/DescriptorSets.cpp` where the dummy images were created.

```diff
// In DescriptorSets::create_resources
                     for (int i{ 0 }; i < binding_info.descriptor_count; ++i) {
                         Image image{ *device_,
                                      *physical_device_,
                                      image_create_info };
                         ImageView image_view{ *device_,
                                               image,
                                               { .width = 1, .height = 1 },
                                               VK_IMAGE_ASPECT_COLOR_BIT };
+                        image.transition_layout(
+                        VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL);
                         images.push_back(std::make_pair(std::move(image),
                                                         std::move(image_view)));
                     }
```

### Part B: Unsupported Layout Transition

#### Error Message

The attempted fix above led to a new error:
```
libc++abi: terminating due to uncaught exception of type std::invalid_argument: unsupported layout transition
```

#### Problem

The `Image::transition_layout` function was too specific and did not have a case to handle a direct transition from `VK_IMAGE_LAYOUT_UNDEFINED` to `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`, causing it to throw an exception.

#### Final Fix

*   **File:** `njin/vulkan/src/Image.cpp`
*   **Change:** Added a new case to the `transition_layout` function to correctly handle the `UNDEFINED` to `SHADER_READ_ONLY_OPTIMAL` transition.

    ```diff
             src_stage_mask = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
             dst_stage_mask = VK_PIPELINE_STAGE_TRANSFER_BIT;
+        } else if (current_layout_ == VK_IMAGE_LAYOUT_UNDEFINED &&
+                   new_layout == VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL) {
+            src_access_mask = 0;
+            dst_access_mask = VK_ACCESS_SHADER_READ_BIT;
+            src_stage_mask = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
+            dst_stage_mask = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
         } else if (current_layout_ == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL &&
                    new_layout == VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL) {
             src_access_mask = VK_ACCESS_TRANSFER_WRITE_BIT;
    ```

---

## 4. Exceeding GPU Descriptor Limits

#### Error Message

```
Validation Error: [ VUID-VkPipelineLayoutCreateInfo-descriptorType-03016 ] ... max per-stage sampler bindings count (100) exceeds device maxPerStageDescriptorSamplers limit (16).
Validation Error: [ VUID-VkPipelineLayoutCreateInfo-descriptorType-03017 ] ... max per-stage uniform buffer bindings count (100) exceeds device maxPerStageDescriptorUniformBuffers limit (31).
Validation Error: [ VUID-VkPipelineLayoutCreateInfo-descriptorType-03018 ] ... max per-stage storage buffer bindings count (100) exceeds device maxPerStageDescriptorStorageBuffers limit (31).
```

#### Problem

The application was hardcoded to request 100 descriptors for various types, which exceeded the user's specific GPU hardware limits.

#### Fix

*   **File:** `njin/vulkan/include/vulkan/config.h`
*   **Change:** Reduced the `MAX_OBJECTS` constant and all related `descriptor_count` values from 100 to 16 to conform to the hardware's most restrictive limit.

    ```diff
-    constexpr int MAX_OBJECTS = 100;
+    constexpr int MAX_OBJECTS = 16;
     ...
-        .descriptor_count = 100,
+        .descriptor_count = 16,
...
-        .descriptor_count = 100,
+        .descriptor_count = 16,
...
-        .descriptor_count = 100,
+        .descriptor_count = 16,
    ```

---

## 5. Shader and Application Mismatch

#### Error Message

```
Validation Error: [ VUID-VkGraphicsPipelineCreateInfo-layout-07991 ] ... pCreateInfos[0].pStages[0] SPIR-V ... uses descriptor ... with a VkDescriptorSetLayoutBinding::descriptorCount of 16, but requires at least 50 in the SPIR-V.
```

#### Problem

The descriptor counts were lowered to 16 in the C++ code, but the GLSL shaders still had a hardcoded array size of 50 for the `models` buffer.

#### Fix

*   **Files:** `shader/shader.vert` and `shader/iso.vert`
*   **Change:** Changed the array size from 50 to 16 in both shaders.

    ```diff
- } models[50];
+ } models[16];
    ```

---

## 6. Final Logic and Build Fixes

#### Error Message

```
/Users/powhweee/coding/njin/njin/ecs/src/njRenderSystem.cpp:204:32: error: no member named 'vulkan' in namespace 'njin'
```

#### Problem

1.  The application logic in `njRenderSystem` did not respect the new `MAX_OBJECTS` limit, creating a risk of out-of-bounds GPU access.
2.  Fixing this revealed a build system issue where the `ecs` module could not see the `vulkan` module's code, leading to the compile error.

#### Fixes

1.  **Build System Dependency**
    *   **File:** `njin/ecs/CMakeLists.txt`
    *   **Change:** The `ecs` library was properly linked against the `vulkan` library.

    ```diff
- target_link_libraries(ecs PUBLIC SDL3::SDL3 math core physics_system)
+ target_link_libraries(ecs PUBLIC SDL3::SDL3 math core physics_system vulkan)
    ```

2.  **Include Header**
    *   **File:** `njin/ecs/src/njRenderSystem.cpp`
    *   **Change:** Added the necessary header to include the `MAX_OBJECTS` definition.

    ```diff
#include "ecs/Components.h"
+#include "vulkan/config.h"

namespace njin::ecs {
```

3.  **Enforce Logic Limit**
    *   **File:** `njin/ecs/src/njRenderSystem.cpp`
    *   **Change:** Added a check to the object gathering loop to ensure it does not exceed `MAX_OBJECTS`.

    ```diff
+        int count = 0;
         for (const auto& [entity, view] : meshes_no_parents) {
+            if (count >= njin::vulkan::MAX_OBJECTS) {
+                break;
+            }
             auto mesh{ std::get<njMeshComponent*>(view) };
             auto transform{ std::get<njTransformComponent*>(view) };
             // global transform = local transform for entities with no parent
@@ -220,6 +224,7 @@
.data = data };
renderables.push_back(renderable);
+            count++;
         }

         buffer_->replace(renderables);
    ```


