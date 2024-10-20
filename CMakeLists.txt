cmake_minimum_required(VERSION 3.28)
project(VulkanApp)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 设置 Vulkan SDK 路径
if (APPLE)
    set(Vulkan_SDK_PATH $ENV{VULKAN_SDK})
    if (NOT Vulkan_SDK_PATH)
        message(FATAL_ERROR "VULKAN_SDK environment variable is not set")
    endif()
    set(Vulkan_INCLUDE_DIR "${Vulkan_SDK_PATH}/include")
    set(Vulkan_LIBRARY "${Vulkan_SDK_PATH}/lib/libvulkan.dylib")
    set(MOLTENVK_LINK "${Vulkan_SDK_PATH}/lib/libMoltenVK.dylib")
endif()

# 添加包含目录
include_directories(${Vulkan_INCLUDE_DIR})

# 动态查找 GLFW 库及其包含目录
find_path(GLFW_INCLUDE_DIR GLFW/glfw3.h HINTS /opt/homebrew/include /usr/local/include)
find_library(GLFW_LIBRARY glfw HINTS /opt/homebrew/lib /usr/local/lib)

# 检查是否找到了 GLFW
if (NOT GLFW_LIBRARY OR NOT GLFW_INCLUDE_DIR)
    message(FATAL_ERROR "GLFW not found. Please install it via Homebrew or set paths manually.")
endif()

# 添加包含目录
include_directories(${GLFW_INCLUDE_DIR})

# 定义编译着色器的函数
function(compile_shader SHADER)
    get_filename_component(FILENAME ${SHADER} NAME)  # 使用 NAME 获取完整文件名，包括扩展名
    set(SPIRV "${CMAKE_CURRENT_BINARY_DIR}/shaders/${FILENAME}.spv")
    add_custom_command(
            OUTPUT ${SPIRV}
            COMMAND glslangValidator -V ${SHADER} -o ${SPIRV}
            DEPENDS ${SHADER}
            COMMENT "Compiling ${SHADER} to SPIR-V"
    )
    list(APPEND SPIRV_FILES ${SPIRV})
    set(SPIRV_FILES ${SPIRV_FILES} PARENT_SCOPE)
endfunction()

# 收集所有着色器文件
file(GLOB SHADERS
        "${CMAKE_SOURCE_DIR}/shaders/*.frag"
        "${CMAKE_SOURCE_DIR}/shaders/*.vert"
        "${CMAKE_SOURCE_DIR}/shaders/*.glsl"
)

# 编译着色器
set(SPIRV_FILES)
foreach(SHADER ${SHADERS})
    compile_shader(${SHADER})
endforeach()

# 添加 external 文件夹
include_directories(${CMAKE_SOURCE_DIR}/external)

add_library(STB_IMAGE "external/stb_image.cpp")
set(LIBS ${LIBS} STB_IMAGE)

add_library(TINY_OBJ_LOADER "external/tiny_obj_loader.cc")
set(LIBS ${LIBS} TINY_OBJ_LOADER)

add_library(TINYFILEDIALOGS "external/tinyfiledialogs.c")
set(LIBS ${LIBS} TINYFILEDIALOGS)

# 添加可执行文件，源文件在 src 目录下
add_executable(VulkanApp
        src/main.cpp
        # src/VulkanApp.cpp
        # src/VulkanApp.h
        # 不需要将着色器源文件添加到可执行文件中
)

# 将 SPIR-V 文件添加为依赖项
add_custom_target(Shaders DEPENDS ${SPIRV_FILES})
add_dependencies(VulkanApp Shaders)

# 链接库
target_link_libraries(VulkanApp ${Vulkan_LIBRARY} ${GLFW_LIBRARY} ${LIBS})

# 构建后复制 SPIR-V 着色器文件到输出目录
add_custom_command(
        TARGET VulkanApp POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory $<TARGET_FILE_DIR:VulkanApp>/shaders
        COMMAND ${CMAKE_COMMAND} -E copy ${SPIRV_FILES} $<TARGET_FILE_DIR:VulkanApp>/shaders
        COMMENT "Copying compiled shaders to build directory"
)

# 添加包含目录（可选，如果需要）
target_include_directories(VulkanApp PRIVATE ${Vulkan_INCLUDE_DIR} ${GLFW_INCLUDE_DIR})
