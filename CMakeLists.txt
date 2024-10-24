cmake_minimum_required(VERSION 3.28)
project(VulkanApp)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 查找 Vulkan 和 GLFW
find_package(Vulkan REQUIRED)
find_package(glfw3 REQUIRED)

# 设置外部库目录
set(EXTERNAL_DIR ${CMAKE_SOURCE_DIR}/external)

# 添加 ImGui 目录
set(IMGUI_DIR ${EXTERNAL_DIR}/imgui)
set(IMGUI_BACKENDS_DIR ${IMGUI_DIR}/backends)

# 编译着色器函数
function(compile_shader SHADER)
    get_filename_component(FILENAME ${SHADER} NAME)
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
file(GLOB SHADERS "${CMAKE_SOURCE_DIR}/shaders/*.frag" "${CMAKE_SOURCE_DIR}/shaders/*.vert" "${CMAKE_SOURCE_DIR}/shaders/*.comp")
set(SPIRV_FILES)
foreach(SHADER ${SHADERS})
    compile_shader(${SHADER})
endforeach()

# 添加 ImGui 源文件
set(IMGUI_SOURCES
        ${IMGUI_DIR}/imgui.cpp
        ${IMGUI_DIR}/imgui_draw.cpp
        ${IMGUI_DIR}/imgui_demo.cpp
        ${IMGUI_DIR}/imgui_widgets.cpp
        ${IMGUI_DIR}/imgui_tables.cpp   # 确保添加了这个文件
        ${IMGUI_BACKENDS_DIR}/imgui_impl_glfw.cpp
        ${IMGUI_BACKENDS_DIR}/imgui_impl_vulkan.cpp
)

# 添加 external 文件夹下的库
add_library(STB_IMAGE "external/stb_image.cpp")
add_library(TINY_OBJ_LOADER "external/tiny_obj_loader.cc")
add_library(TINYFILEDIALOGS "external/tinyfiledialogs.c")
set(LIBS ${LIBS} STB_IMAGE TINY_OBJ_LOADER TINYFILEDIALOGS)

# 收集 src 文件夹下的源文件
file(GLOB_RECURSE SOURCES "${CMAKE_SOURCE_DIR}/src/*.cpp" "${CMAKE_SOURCE_DIR}/src/*.h")

# 添加可执行文件
add_executable(VulkanApp ${SOURCES} ${IMGUI_SOURCES})

# 添加着色器依赖
add_custom_target(Shaders DEPENDS ${SPIRV_FILES})
add_dependencies(VulkanApp Shaders)

# 链接库
target_link_libraries(VulkanApp Vulkan::Vulkan glfw ${LIBS})

# 构建后复制着色器文件到输出目录
add_custom_command(
        TARGET VulkanApp POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory $<TARGET_FILE_DIR:VulkanApp>/shaders
        COMMAND ${CMAKE_COMMAND} -E copy ${SPIRV_FILES} $<TARGET_FILE_DIR:VulkanApp>/shaders
        COMMENT "Copying compiled shaders to build directory"
)

# 添加 ImGui 和 GLFW 包含目录
target_include_directories(VulkanApp PRIVATE ${IMGUI_DIR} ${IMGUI_BACKENDS_DIR})
