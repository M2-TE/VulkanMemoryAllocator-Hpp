cmake_minimum_required(VERSION 3.6...3.30 FATAL_ERROR)

project(
    VulkanMemoryAllocator-Hpp
    VERSION 3.2.0
    DESCRIPTION "C++ bindings for the VulkanMemoryAllocator library."
    LANGUAGES C CXX)

if (CMAKE_VERSION VERSION_LESS "3.21")
    # https://cmake.org/cmake/help/latest/variable/PROJECT_IS_TOP_LEVEL.html
    string(COMPARE EQUAL ${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_SOURCE_DIR} PROJECT_IS_TOP_LEVEL)
endif()

# all options for VulkanMemoryAllocator-Hpp
option(VMA_HPP_ENABLE_INSTALL "Install VulkanMemoryAllocator-Hpp" ${PROJECT_IS_TOP_LEVEL})
option(VMA_HPP_ENABLE_MODULE "Enables building of the VMA C++20 module; requires minimum CMake version 3.28" OFF)
option(VMA_HPP_ENABLE_MODULE_STD "Enables import std for the VMA C++20 module; requires minimum CMake version 3.30" OFF)
option(VMA_HPP_ENABLE_VULKAN_HPP_MODULE "Enables import vulkan_hpp" ON)
option(VMA_HPP_ENABLE_VULKAN_HPP_DYNAMIC "Enables dynamic dispatch" OFF)

# create interface target for VMA-Hpp
add_library(VulkanMemoryAllocator-Hpp INTERFACE)
add_library(VulkanMemoryAllocator-Hpp::VulkanMemoryAllocator-Hpp ALIAS VulkanMemoryAllocator-Hpp)
add_library(VulkanMemoryAllocator::Hpp ALIAS VulkanMemoryAllocator-Hpp) # more compact alias target
target_include_directories(VulkanMemoryAllocator-Hpp INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>)

# set up compile definitions
if (VMA_HPP_ENABLE_VULKAN_HPP_DYNAMIC)
    target_compile_definitions(VulkanMemoryAllocator-Hpp INTERFACE "VK_NO_PROTOTYPES=1")
endif()
if (VMA_HPP_ENABLE_VULKAN_HPP_MODULE)
    target_compile_definitions(VulkanMemoryAllocator-Hpp INTERFACE "VMA_ENABLE_VULKAN_HPP_MODULE")
endif()

# check if VulkanMemoryAllocator target is missing
if (NOT TARGET GPUOpen::VulkanMemoryAllocator)
    if (EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/VulkanMemoryAllocator/include")
        # add via submodule if pulled
        add_subdirectory("${CMAKE_CURRENT_SOURCE_DIR}/VulkanMemoryAllocator")
    else()
        # use FindPackage as a last resort
        find_package(VulkanMemoryAllocator CONFIG REQUIRED)
    endif()
    # TODO: could employ FetchContent as another fallback
endif()
# link to VulkanMemoryAllocator
target_link_libraries(VulkanMemoryAllocator-Hpp INTERFACE GPUOpen::VulkanMemoryAllocator)

# create c++20 module target
if (VMA_HPP_ENABLE_MODULE)
    # check for compiler support
    if ((CMAKE_CXX_COMPILER_ID STREQUAL "MSVC" AND (MSVC_VERSION GREATER_EQUAL "1941")) OR
        # clang-cl doesn't currently support modules
        (CMAKE_CXX_COMPILER_ID STREQUAL "Clang"
            AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL "16.0"
            AND (NOT CMAKE_C_COMPILER_FRONTEND_VARIANT MATCHES "MSVC")
            AND (NOT CMAKE_CXX_COMPILER_CLANG_SCAN_DEPS STREQUAL CMAKE_CXX_COMPILER_CLANG_SCAN_DEPS-NOTFOUND)) OR
        (CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL "14.0")
    )
        # check for CMake support
        if(VMA_HPP_ENABLE_MODULE_STD AND CMAKE_VERSION VERSION_LESS "3.30")
            message(FATAL_ERROR "Vulkan-Hpp: C++20 module with import std requires CMake 3.30 or later")
        elseif (CMAKE_VERSION VERSION_LESS "3.28")
            message(FATAL_ERROR "Vulkan-Hpp: C++20 module requires CMake 3.28 or later")
        endif()
        
        # set up Vulkan-HppModule
        add_library(VulkanMemoryAllocator-HppModule)
        add_library(VulkanMemoryAllocator::HppModule ALIAS VulkanMemoryAllocator-HppModule)
        target_sources(VulkanMemoryAllocator-HppModule PUBLIC 
            FILE_SET module
            TYPE CXX_MODULES
            FILES "${CMAKE_CURRENT_SOURCE_DIR}/src/vk_mem_alloc.cppm")
        set_target_properties(VulkanMemoryAllocator-HppModule PROPERTIES
            CXX_STANDARD_REQUIRED ON # CXX version is a hard requirement for modules
            CXX_EXTENSIONS OFF) # Clang's module support can be broken with extensions enabled
        target_link_libraries(VulkanMemoryAllocator-HppModule PUBLIC VulkanMemoryAllocator::Hpp)

        # set up CXX standard
        if (VMA_HPP_ENABLE_MODULE_STD)
            target_compile_features(VulkanMemoryAllocator-HppModule
                PRIVATE   cxx_std_23
                INTERFACE cxx_std_20) # only C++20 is required to consume this module
            set_target_properties(VulkanMemoryAllocator-HppModule PROPERTIES CXX_MODULE_STD ON)
        else()
            target_compile_features(VulkanMemoryAllocator-HppModule PUBLIC cxx_std_20)
        endif()
    else()
        message(FATAL_ERROR "Vulkan-Hpp: C++20 module support is requested but was disabled due to lacking compiler support on this platform")
    endif()
endif()

# install
if (VMA_HPP_ENABLE_INSTALL)
    include(GNUInstallDirs)
    include(CMakePackageConfigHelpers)

    install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/include/" DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
    install(TARGETS VulkanMemoryAllocator-Hpp EXPORT VulkanMemoryAllocator-HppConfig INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
    install(EXPORT VulkanMemoryAllocator-HppConfig NAMESPACE "VulkanMemoryAllocator-Hpp::" DESTINATION "share/cmake/VulkanMemoryAllocator-Hpp")

    write_basic_package_version_file(VulkanMemoryAllocator-HppConfigVersion.cmake COMPATIBILITY SameMajorVersion ARCH_INDEPENDENT)
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/VulkanMemoryAllocator-HppConfigVersion.cmake" DESTINATION "share/cmake/VulkanMemoryAllocator-Hpp")
endif()
