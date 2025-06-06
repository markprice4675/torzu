# SPDX-FileCopyrightText: 2018 yuzu Emulator Project and 2024 torzu project
# SPDX-License-Identifier: GPL-2.0-or-later

cmake_minimum_required(VERSION 3.22)

project(yuzu)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/CMakeModules")
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/externals/cmake-modules")

include(DownloadExternals)
include(CMakeDependentOption)
include(CTest)

option(YUZU_USE_BUNDLED_VCPKG "Use vcpkg for yuzu dependencies" "${MSVC}")
if (NOT YUZU_USE_BUNDLED_VCPKG)
    option(YUZU_USE_CPM "Use CPM.cmake for yuzu dependencies" OFF)
else()
    set(YUZU_USE_CPM OFF)
endif()

# Set bundled sdl2/qt as dependent options.
# OFF by default, but if ENABLE_SDL2 and MSVC are true then ON
option(ENABLE_SDL2 "Enable the SDL2 frontend" ON)
CMAKE_DEPENDENT_OPTION(YUZU_USE_BUNDLED_SDL2 "Download bundled SDL2 binaries" ON "ENABLE_SDL2;MSVC" OFF)
# On Linux system SDL2 is likely to be lacking HIDAPI support which have drawbacks but is needed for SDL motion
CMAKE_DEPENDENT_OPTION(YUZU_USE_EXTERNAL_SDL2 "Compile external SDL2" ON "ENABLE_SDL2;NOT MSVC" OFF)

cmake_dependent_option(ENABLE_LIBUSB "Enable the use of LibUSB" ON "NOT ANDROID" OFF)

option(ENABLE_OPENGL "Enable OpenGL" ON)
mark_as_advanced(FORCE ENABLE_OPENGL)
option(ENABLE_QT "Enable the Qt frontend" ON)
option(ENABLE_QT6 "Allow usage of Qt6 to be attempted" OFF)
set(QT6_LOCATION "" CACHE PATH "Additional Location to search for Qt6 libraries like C:/Qt/6.3.1/msvc2019_64/")

option(ENABLE_QT_TRANSLATION "Enable translations for the Qt frontend" ON)
CMAKE_DEPENDENT_OPTION(YUZU_USE_BUNDLED_QT "Download bundled Qt binaries" "${MSVC}" "ENABLE_QT" OFF)

option(ENABLE_WEB_SERVICE "Enable web services (telemetry, etc.)" ON)

option(YUZU_USE_BUNDLED_FFMPEG "Download/Build bundled FFmpeg" "${WIN32}")

if (NOT YUZU_USE_CPM)
    option(YUZU_USE_EXTERNAL_VULKAN_HEADERS "Use Vulkan-Headers from externals" ON)

    option(YUZU_USE_EXTERNAL_VULKAN_UTILITY_LIBRARIES "Use Vulkan-Utility-Libraries from externals" ON)

    option(YUZU_USE_EXTERNAL_VULKAN_SPIRV_TOOLS "Use SPIRV-Tools from externals" ON)
else()
    set(YUZU_USE_EXTERNAL_VULKAN_HEADERS OFF)

    set(YUZU_USE_EXTERNAL_VULKAN_UTILITY_LIBRARIES OFF)

    set(YUZU_USE_EXTERNAL_VULKAN_SPIRV_TOOLS OFF)
endif()

option(YUZU_USE_QT_MULTIMEDIA "Use QtMultimedia for Camera" OFF)

option(YUZU_USE_QT_WEB_ENGINE "Use QtWebEngine for web applet implementation" OFF)

option(ENABLE_CUBEB "Enables the cubeb audio backend" ON)

option(USE_DISCORD_PRESENCE "Enables Discord Rich Presence" OFF)

option(YUZU_NO_CPU_DEBUGGER "Do not build debugging code" OFF)

option(YUZU_TESTS "Compile tests" "${BUILD_TESTING}")

option(YUZU_USE_PRECOMPILED_HEADERS "Use precompiled headers" ON)

option(YUZU_DOWNLOAD_ANDROID_VVL "Download validation layer binary for android" ON)

CMAKE_DEPENDENT_OPTION(YUZU_ROOM "Compile LDN room server" ON "NOT ANDROID" OFF)

option(YUZU_CMD "Compile the -cmd executable (can disable if -cmd is unused)" ON)

CMAKE_DEPENDENT_OPTION(YUZU_CRASH_DUMPS "Compile crash dump (Minidump) support" OFF "WIN32 OR LINUX" OFF)

option(YUZU_CHECK_SUBMODULES "Check if submodules are present" ON)

option(YUZU_ENABLE_LTO "Enable link-time optimization" OFF)

option(YUZU_DOWNLOAD_TIME_ZONE_DATA "Always download time zone binaries" OFF)

option(YUZU_ENABLE_PORTABLE "Allow yuzu to enable portable mode if a user folder is found in the CWD" ON)

option(YUZU_USE_LLVM_DEMANGLE "Use LLVM Demangle" ON)

CMAKE_DEPENDENT_OPTION(YUZU_USE_FASTER_LD "Check if a faster linker is available" ON "NOT WIN32" OFF)

CMAKE_DEPENDENT_OPTION(USE_SYSTEM_MOLTENVK "Use the system MoltenVK lib (instead of the bundled one)" OFF "APPLE" OFF)

# Detect current compilation architecture and create standard definitions
# =======================================================================

include(CheckSymbolExists)
function(detect_architecture symbol arch)
    if (NOT DEFINED ARCHITECTURE)
        set(CMAKE_REQUIRED_QUIET 1)
        check_symbol_exists("${symbol}" "" ARCHITECTURE_${arch})
        unset(CMAKE_REQUIRED_QUIET)

        # The output variable needs to be unique across invocations otherwise
        # CMake's crazy scope rules will keep it defined
        if (ARCHITECTURE_${arch})
            set(ARCHITECTURE "${arch}" PARENT_SCOPE)
            set(ARCHITECTURE_${arch} 1 PARENT_SCOPE)
            add_definitions(-DARCHITECTURE_${arch}=1)
        endif()
    endif()
endfunction()

if (NOT ENABLE_GENERIC)
    if (MSVC)
        detect_architecture("_M_AMD64" x86_64)
        detect_architecture("_M_IX86" x86)
        detect_architecture("_M_ARM" arm)
        detect_architecture("_M_ARM64" arm64)
    else()
        detect_architecture("__x86_64__" x86_64)
        detect_architecture("__i386__" x86)
        detect_architecture("__arm__" arm)
        detect_architecture("__aarch64__" arm64)
    endif()
endif()

if (NOT DEFINED ARCHITECTURE)
    set(ARCHITECTURE "GENERIC")
    set(ARCHITECTURE_GENERIC 1)
    add_definitions(-DARCHITECTURE_GENERIC=1)
endif()
message(STATUS "Target architecture: ${ARCHITECTURE} (${YUZU_MARCH})")

if (MSVC AND NOT CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    set(YUZU_MARCH "SSE4.2" CACHE STRING "Compile for specified x86 arch (AVX, AVX2, etc.)")
    add_compile_options(/arch:${YUZU_MARCH})
else()
    if (ARCHITECTURE STREQUAL "x86_64")
        set(YUZU_MARCH "x86-64-v2" CACHE STRING "Compile for specified x86 microarchitecture level (x86-64-v3, native, etc.)")
        add_compile_options(-march=${YUZU_MARCH})
    elseif (ARCHITECTURE STREQUAL "arm64")
        set(YUZU_MARCH "armv8-a" CACHE STRING "Compile for specified ARM architecture (armv8.1-a, native, etc.)")
        add_compile_options(-march=${YUZU_MARCH})
    else()
        message(WARNING "Architecture ${ARCHITECTURE} unknown, EXPECT THINGS TO GO WRONG.")
    endif()
endif()

if (MSVC)
    add_definitions(-DWIN32)
endif()

set(DEFAULT_ENABLE_OPENSSL ON)
if (ANDROID OR WIN32 OR APPLE)
    # - Windows defaults to the Schannel backend.
    # - macOS defaults to the SecureTransport backend.
    # - Android currently has no SSL backend as the NDK doesn't include any SSL
    #   library; a proper 'native' backend would have to go through Java.
    # But you can force builds for those platforms to use OpenSSL if you have
    # your own copy of it.
    set(DEFAULT_ENABLE_OPENSSL OFF)
endif()
option(ENABLE_OPENSSL "Enable OpenSSL backend for ISslConnection" ${DEFAULT_ENABLE_OPENSSL})

if (ANDROID AND YUZU_DOWNLOAD_ANDROID_VVL)
    set(vvl_version "1.4.304.1")
    set(vvl_zip_file "${CMAKE_BINARY_DIR}/externals/vvl-android.zip")

    set(vvl_lib_path "${CMAKE_CURRENT_SOURCE_DIR}/src/android/app/src/main/jniLibs/arm64-v8a/")
    set(vvl_final_lib "${vvl_lib_path}/libVkLayer_khronos_validation.so")

    if (NOT EXISTS "${vvl_final_lib}")
        # Download and extract validation layer release to externals directory
        if (NOT EXISTS "${vvl_zip_file}")
            set(vvl_base_url "https://github.com/KhronosGroup/Vulkan-ValidationLayers/releases/download")
            file(DOWNLOAD "${vvl_base_url}/vulkan-sdk-${vvl_version}/android-binaries-${vvl_version}.zip"
                "${vvl_zip_file}" SHOW_PROGRESS)
            execute_process(COMMAND ${CMAKE_COMMAND} -E tar xf "${vvl_zip_file}"
                            WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/externals")
        endif()

        # Copy the arm64 binary to src/android/app/main/jniLibs
        file(COPY "${CMAKE_BINARY_DIR}/externals/android-binaries-${vvl_version}/arm64-v8a/libVkLayer_khronos_validation.so"
                DESTINATION "${vvl_lib_path}")
    endif()
endif()

if (ANDROID)
    set(CMAKE_SKIP_INSTALL_RULES ON)
endif()

if (YUZU_USE_BUNDLED_VCPKG)
    if (ANDROID)
        set(ENV{ANDROID_NDK_HOME} "${ANDROID_NDK}")
        list(APPEND VCPKG_MANIFEST_FEATURES "android")

        if (CMAKE_ANDROID_ARCH_ABI STREQUAL "arm64-v8a")
            set(VCPKG_TARGET_TRIPLET "arm64-android")
            # this is to avoid CMake using the host pkg-config to find the host
            # libraries when building for Android targets
            set(PKG_CONFIG_EXECUTABLE "aarch64-none-linux-android-pkg-config" CACHE FILEPATH "" FORCE)
        elseif (CMAKE_ANDROID_ARCH_ABI STREQUAL "x86_64")
            set(VCPKG_TARGET_TRIPLET "x64-android")
            set(PKG_CONFIG_EXECUTABLE "x86_64-none-linux-android-pkg-config" CACHE FILEPATH "" FORCE)
        else()
            message(FATAL_ERROR "Unsupported Android architecture ${CMAKE_ANDROID_ARCH_ABI}")
        endif()
    endif()

    if (MSVC)
        set(VCPKG_DOWNLOADS_PATH ${PROJECT_SOURCE_DIR}/externals/vcpkg/downloads)
        set(NASM_VERSION "2.16.01")
        set(NASM_DESTINATION_PATH ${VCPKG_DOWNLOADS_PATH}/nasm-${NASM_VERSION}-win64.zip)
        set(NASM_DOWNLOAD_URL "https://github.com/litucks/ext-windows-bin/raw/master/nasm/nasm-${NASM_VERSION}-win64.zip")

        if (NOT EXISTS ${NASM_DESTINATION_PATH})
            file(DOWNLOAD ${NASM_DOWNLOAD_URL} ${NASM_DESTINATION_PATH} SHOW_PROGRESS STATUS NASM_STATUS)

            if (NOT NASM_STATUS EQUAL 0)
                # Warn and not fail since vcpkg is supposed to download this package for us in the first place
                message(WARNING "External nasm vcpkg package download from ${NASM_DOWNLOAD_URL} failed with status ${NASM_STATUS}")
            endif()
        endif()
    endif()

    if (YUZU_TESTS)
        list(APPEND VCPKG_MANIFEST_FEATURES "yuzu-tests")
    endif()
    if (ENABLE_WEB_SERVICE)
        list(APPEND VCPKG_MANIFEST_FEATURES "web-service")
    endif()
    if (ANDROID)
        list(APPEND VCPKG_MANIFEST_FEATURES "android")
    endif()

    include(${CMAKE_SOURCE_DIR}/externals/vcpkg/scripts/buildsystems/vcpkg.cmake)
elseif(NOT "$ENV{VCPKG_TOOLCHAIN_FILE}" STREQUAL "")
    # Disable manifest mode (use vcpkg classic mode) when using a custom vcpkg installation
    option(VCPKG_MANIFEST_MODE "")
    include("$ENV{VCPKG_TOOLCHAIN_FILE}")
endif()

if (YUZU_USE_PRECOMPILED_HEADERS)
    if (MSVC AND CCACHE)
        # buildcache does not properly cache PCH files, leading to compilation errors.
        # See https://github.com/mbitsnbites/buildcache/discussions/230
        message(WARNING "buildcache does not properly support Precompiled Headers. Disabling PCH")
        set(DYNARMIC_USE_PRECOMPILED_HEADERS OFF CACHE BOOL "" FORCE)
        set(YUZU_USE_PRECOMPILED_HEADERS OFF CACHE BOOL "" FORCE)
    endif()
endif()
if (YUZU_USE_PRECOMPILED_HEADERS)
    message(STATUS "Using Precompiled Headers.")
    set(CMAKE_PCH_INSTANTIATE_TEMPLATES ON)
endif()


# Default to a Release build
get_property(IS_MULTI_CONFIG GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)
if (NOT IS_MULTI_CONFIG AND NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel." FORCE)
    message(STATUS "Defaulting to a Release build")
endif()

if(EXISTS ${PROJECT_SOURCE_DIR}/hooks/pre-commit AND NOT EXISTS ${PROJECT_SOURCE_DIR}/.git/hooks/pre-commit)
    if (EXISTS ${PROJECT_SOURCE_DIR}/.git/)
        message(STATUS "Copying pre-commit hook")
        file(COPY hooks/pre-commit DESTINATION ${PROJECT_SOURCE_DIR}/.git/hooks)
    endif()
endif()

# Sanity check : Check that all submodules are present
# =======================================================================

function(check_submodules_present)
    file(READ "${PROJECT_SOURCE_DIR}/.gitmodules" gitmodules)
    string(REGEX MATCHALL "path *= *[^ \t\r\n]*" gitmodules ${gitmodules})
    foreach(module ${gitmodules})
        string(REGEX REPLACE "path *= *" "" module ${module})
        if (NOT EXISTS "${PROJECT_SOURCE_DIR}/${module}/.git")
            message(FATAL_ERROR "Git submodule ${module} not found. "
                    "Please run: \ngit submodule update --init --recursive")
        endif()
    endforeach()
endfunction()

if(EXISTS ${PROJECT_SOURCE_DIR}/.gitmodules AND YUZU_CHECK_SUBMODULES)
    check_submodules_present()
endif()
configure_file(${PROJECT_SOURCE_DIR}/dist/compatibility_list/compatibility_list.qrc
               ${PROJECT_BINARY_DIR}/dist/compatibility_list/compatibility_list.qrc
               COPYONLY)
if (EXISTS ${PROJECT_SOURCE_DIR}/dist/compatibility_list/compatibility_list.json)
    configure_file("${PROJECT_SOURCE_DIR}/dist/compatibility_list/compatibility_list.json"
                   "${PROJECT_BINARY_DIR}/dist/compatibility_list/compatibility_list.json"
                   COPYONLY)
endif()
if (ENABLE_COMPATIBILITY_LIST_DOWNLOAD AND NOT EXISTS ${PROJECT_BINARY_DIR}/dist/compatibility_list/compatibility_list.json)
    message(STATUS "Downloading compatibility list for yuzu...")
    file(DOWNLOAD
        https://api.yuzu-emu.org/gamedb/
        "${PROJECT_BINARY_DIR}/dist/compatibility_list/compatibility_list.json" SHOW_PROGRESS)
endif()
if (NOT EXISTS ${PROJECT_BINARY_DIR}/dist/compatibility_list/compatibility_list.json)
    file(WRITE ${PROJECT_BINARY_DIR}/dist/compatibility_list/compatibility_list.json "")
endif()


if (UNIX)
    add_definitions(-DYUZU_UNIX=1)
endif()

if (ARCHITECTURE_arm64 AND (ANDROID OR ${CMAKE_SYSTEM_NAME} STREQUAL "Linux"))
    set(HAS_NCE 1)
    add_definitions(-DHAS_NCE=1)
endif()

# Configure C++ standard
# ===========================

# boost asio's concept usage doesn't play nicely with some compilers yet.
add_definitions(-DBOOST_ASIO_DISABLE_CONCEPTS)
if (MSVC)
    add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/std:c++20>)

    # boost still makes use of deprecated result_of.
    add_definitions(-D_HAS_DEPRECATED_RESULT_OF)
else()
    set(CMAKE_CXX_STANDARD 20)
    set(CMAKE_CXX_STANDARD_REQUIRED ON)
endif()

# Output binaries to bin/
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)

# System imported libraries
# =======================================================================

# Enforce the search mode of non-required packages for better and shorter failure messages
if (NOT YUZU_USE_CPM)
    find_package(Boost 1.86.0 REQUIRED headers context system fiber)
    find_package(enet 1.3 MODULE)
    find_package(fmt REQUIRED)
    if (YUZU_USE_LLVM_DEMANGLE)
        find_package(LLVM 17.0.2 MODULE COMPONENTS Demangle)
    endif()
    find_package(lz4 REQUIRED)
    find_package(nlohmann_json 3.8 REQUIRED)
    find_package(Opus 1.3 MODULE)
    find_package(RenderDoc MODULE)
    find_package(SimpleIni MODULE)
    find_package(stb MODULE)
    find_package(VulkanMemoryAllocator CONFIG)
    find_package(ZLIB 1.2 REQUIRED)
    find_package(zstd 1.5 REQUIRED)
else()
    include(CMakeModules/CPM.cmake)

    # Disable tests in all externals supporting the standard option name
    set(BUILD_TESTING OFF)

    # Some externals take these options instead
    set(BUILD_TESTS OFF)
    set(BUILD_TOOLS OFF)

    # Build only static externals
    set(BUILD_SHARED_LIBS OFF)

    # Do not attempt to use Brotli in httplib since we're not downloading it
    set(HTTPLIB_USE_BROTLI_IF_AVAILABLE OFF)

    # Configure Opus to not generate as much overhead
    set(OPUS_BUILD_TESTING OFF)
    set(OPUS_BUILD_PROGRAMS OFF)
    set(OPUS_INSTALL_PKG_CONFIG_MODULE OFF)
    set(OPUS_INSTALL_CMAKE_CONFIG_MODULE OFF)

    message(STATUS "Downloading and extracting boost library sources. This will take some time...")
    CPMAddPackage(
        NAME boost
        URL "https://github.com/boostorg/boost/releases/download/boost-1.88.0/boost-1.88.0-cmake.7z"
        PATCHES boost-1.88.0-fix.patch
        VERSION 1.88.0
    )
    CPMAddPackage("gh:lsalzman/enet@1.3.18")
    CPMAddPackage("gh:fmtlib/fmt#11.1.4")
    CPMAddPackage(
        NAME lz4
        GIT_REPOSITORY https://github.com/lz4/lz4.git
        VERSION 1.10.0
        DOWNLOAD_ONLY YES
    )
    add_subdirectory(${lz4_SOURCE_DIR}/build/cmake lz4)
    CPMAddPackage("gh:nlohmann/json@3.8.0")
    CPMAddPackage("gh:xiph/opus@1.3.1")
    CPMAddPackage("gh:brofield/simpleini@4.22")
    CPMAddPackage("gh:GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator@3.2.1")
    CPMAddPackage("gh:madler/zlib@1.2.13")
    CPMAddPackage(
        NAME zstd
        GIT_REPOSITORY https://github.com/facebook/zstd.git
        VERSION 1.5.7
        DOWNLOAD_ONLY YES
    )
    add_subdirectory(${zstd_SOURCE_DIR}/build/cmake zstd)
    CPMAddPackage("gh:KhronosGroup/SPIRV-Headers#vulkan-sdk-1.3.280.0")
    CPMAddPackage("gh:yhirose/cpp-httplib@0.20.0")

    # Set up required aliases
    add_library(Opus::opus ALIAS opus)
    add_library(lz4::lz4 ALIAS lz4)
    add_library(zstd::zstd ALIAS libzstd)
    add_library(zstd::libzstd ALIAS libzstd)
    add_library(nlohmann::json ALIAS nlohmann_json)

    # Enet specific setup to add missing include dir
    add_library(enet_fixed INTERFACE)
    target_link_libraries(enet_fixed INTERFACE enet)
    target_include_directories(enet_fixed INTERFACE ${enet_SOURCE_DIR}/include)
    add_library(enet::enet ALIAS enet_fixed)
endif()

if (NOT YUZU_USE_EXTERNAL_VULKAN_HEADERS)
    if (NOT YUZU_USE_CPM)
        find_package(VulkanHeaders 1.3.274 REQUIRED)
    else()
        CPMAddPackage("gh:KhronosGroup/Vulkan-Headers@1.3.274")
    endif()
endif()

if (NOT YUZU_USE_EXTERNAL_VULKAN_UTILITY_LIBRARIES)
    if (NOT YUZU_USE_CPM)
        find_package(VulkanUtilityLibraries REQUIRED)
    else()
        CPMAddPackage("gh:KhronosGroup/Vulkan-Utility-Libraries#fce11d52fee0344bb10a098b0a398dff42cb5d51")
    endif()
endif()

if (NOT YUZU_USE_EXTERNAL_VULKAN_SPIRV_TOOLS)
    if (NOT YUZU_USE_CPM)
        find_package(PkgConfig REQUIRED)
        pkg_check_modules(SPIRV-Tools REQUIRED SPIRV-Tools)
    else()
        CPMAddPackage("gh:KhronosGroup/SPIRV-Tools#dd4b663e13c07fea4fbb3f70c1c91c86731099f7") # This is tag v2024.2, but some weird but is preventing us from using it here
    endif()
endif()

if (ENABLE_LIBUSB)
    find_package(libusb 1.0.24 MODULE)
endif()

if (ARCHITECTURE_x86 OR ARCHITECTURE_x86_64)
    find_package(xbyak 7 CONFIG)
endif()

if (ARCHITECTURE_arm64)
    find_package(oaknut 2.0.1 CONFIG)
endif()

if (ARCHITECTURE_x86_64 OR ARCHITECTURE_arm64)
    find_package(dynarmic 6.4.0 CONFIG)
endif()

if (ENABLE_CUBEB)
    find_package(cubeb CONFIG)
endif()

if (USE_DISCORD_PRESENCE)
    find_package(DiscordRPC MODULE)
endif()

if (ENABLE_WEB_SERVICE)
    find_package(cpp-jwt 1.4 CONFIG)
endif()

if (YUZU_TESTS)
    find_package(Catch2 3.0.1 REQUIRED)
endif()

# boost:asio has functions that require AcceptEx et al
if (MINGW)
    find_library(MSWSOCK_LIBRARY mswsock REQUIRED)
endif()

if(ENABLE_OPENSSL)
    find_package(OpenSSL 1.1.1 REQUIRED)
endif()

if (UNIX AND NOT APPLE)
    find_package(gamemode 1.7 MODULE)
endif()

# Please consider this as a stub
if(ENABLE_QT6 AND Qt6_LOCATION)
    list(APPEND CMAKE_PREFIX_PATH "${Qt6_LOCATION}")
endif()

# QT6 searches for Vulkan::Headers, too, so we have to define it before QT6 has a chance to do it.
add_subdirectory(externals)

function(set_yuzu_qt_components)
    # Best practice is to ask for all components at once, so they are from the same version
    set(YUZU_QT_COMPONENTS2 Core Widgets Concurrent)
    if (${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
        list(APPEND YUZU_QT_COMPONENTS2 DBus)
    endif()
    if (YUZU_USE_QT_MULTIMEDIA)
        list(APPEND YUZU_QT_COMPONENTS2 Multimedia)
    endif()
    if (YUZU_USE_QT_WEB_ENGINE)
        list(APPEND YUZU_QT_COMPONENTS2 WebEngineCore WebEngineWidgets)
    endif()
    if (ENABLE_QT_TRANSLATION)
        list(APPEND YUZU_QT_COMPONENTS2 LinguistTools)
    endif()
    if (USE_DISCORD_PRESENCE)
        list(APPEND YUZU_QT_COMPONENTS2 Network)
    endif()
    set(YUZU_QT_COMPONENTS ${YUZU_QT_COMPONENTS2} PARENT_SCOPE)
endfunction(set_yuzu_qt_components)

# Qt5 requires that we find components, so it doesn't fit our pretty little find package function
if(ENABLE_QT)
    set(QT_VERSION 5.15)
    # These are used to specify minimum versions
    set(QT5_VERSION 5.15)
    set(QT6_VERSION 6.3.1)

    set_yuzu_qt_components()
    if (ENABLE_QT6)
        find_package(Qt6 ${QT6_VERSION} COMPONENTS ${YUZU_QT_COMPONENTS})
    endif()
    if (Qt6_FOUND)
        message(STATUS "yuzu/CMakeLists.txt: Qt6Widgets_VERSION ${Qt6Widgets_VERSION}, setting QT_VERSION")
        set(QT_VERSION ${Qt6Widgets_VERSION})
        set(QT_MAJOR_VERSION 6)
        # Qt6 sets cxx_std_17 and we need to undo that
        set_target_properties(Qt6::Platform PROPERTIES INTERFACE_COMPILE_FEATURES "")
    else()
        message(STATUS "yuzu/CMakeLists.txt: Qt6 not found/not selected, trying for Qt5")
        # When Qt6 partially found, need this set to use Qt5 when not specifying version
        set(QT_DEFAULT_MAJOR_VERSION 5)
        set(QT_MAJOR_VERSION 5)

        set(YUZU_USE_QT_MULTIMEDIA ON)
        # Check for system Qt on Linux, fallback to bundled Qt
        if (UNIX AND NOT APPLE)
            if (NOT YUZU_USE_BUNDLED_QT)
                find_package(Qt5 ${QT5_VERSION} COMPONENTS Widgets DBus Multimedia)
            endif()
            if (${CMAKE_SYSTEM_NAME} STREQUAL "Linux" AND (NOT Qt5_FOUND OR YUZU_USE_BUNDLED_QT))
                # Check for dependencies, then enable bundled Qt download

                # Check that the system GLIBCXX version is compatible
                find_program(OBJDUMP objdump)
                if (NOT OBJDUMP)
                    message(FATAL_ERROR "Required program `objdump` not found.")
                endif()
                find_library(LIBSTDCXX libstdc++.so.6)
                execute_process(
                    COMMAND
                        ${OBJDUMP} -T ${LIBSTDCXX}
                    COMMAND
                        grep GLIBCXX_3.4.28
                    COMMAND
                        sed "s/[0-9a-f]*.*  //"
                    COMMAND
                        sed "s/ .*//"
                    COMMAND
                        sort -u
                    OUTPUT_VARIABLE
                        GLIBCXX_MET
                )
                if (NOT GLIBCXX_MET)
                    message(FATAL_ERROR "Qt too old or not found, and bundled Qt package is not \
                        compatible with this system. Either install Qt ${QT_VERSION}, or provide the path \
                        to Qt by setting the variable Qt5_ROOT.")
                endif()

                # Check for headers
                find_package(PkgConfig REQUIRED)
                pkg_check_modules(QT_DEP_GLU QUIET glu>=9.0.0)
                if (NOT QT_DEP_GLU_FOUND)
                    message(FATAL_ERROR "Qt bundled package dependency `glu` not found. \
                        Perhaps `libglu1-mesa-dev` needs to be installed?")
                endif()
                pkg_check_modules(QT_DEP_MESA QUIET dri>=20.0.8)
                if (NOT QT_DEP_MESA_FOUND)
                    message(FATAL_ERROR "Qt bundled package dependency `dri` not found. \
                        Perhaps `mesa-common-dev` needs to be installed?")
                endif()

                # Check for X libraries
                set(BUNDLED_QT_REQUIREMENTS
                    libxcb-icccm.so.4
                    libxcb-image.so.0
                    libxcb-keysyms.so.1
                    libxcb-randr.so.0
                    libxcb-render-util.so.0
                    libxcb-render.so.0
                    libxcb-shape.so.0
                    libxcb-shm.so.0
                    libxcb-sync.so.1
                    libxcb-xfixes.so.0
                    libxcb-xinerama.so.0
                    libxcb-xkb.so.1
                    libxcb.so.1
                    libxkbcommon-x11.so.0
                    libxkbcommon.so.0
                )
                set(UNRESOLVED_QT_DEPS "")
                foreach (REQUIREMENT ${BUNDLED_QT_REQUIREMENTS})
                    find_library(BUNDLED_QT_${REQUIREMENT} ${REQUIREMENT})
                    if (NOT BUNDLED_QT_${REQUIREMENT})
                        set(UNRESOLVED_QT_DEPS ${UNRESOLVED_QT_DEPS} ${REQUIREMENT})
                    endif()
                    unset(BUNDLED_QT_${REQUIREMENT})
                endforeach()
                unset(BUNDLED_QT_REQUIREMENTS)

                if (NOT "${UNRESOLVED_QT_DEPS}" STREQUAL "")
                    message(FATAL_ERROR "Bundled Qt package missing required dependencies: ${UNRESOLVED_QT_DEPS}")
                endif()

                set(YUZU_USE_BUNDLED_QT ON CACHE BOOL "Download bundled Qt" FORCE)
            endif()
            if (YUZU_USE_BUNDLED_QT)
                # Binary package currently does not support Qt webengine, so make sure it's disabled
                set(YUZU_USE_QT_WEB_ENGINE OFF CACHE BOOL "Use Qt Webengine" FORCE)
            endif()
        endif()

        set(YUZU_QT_NO_CMAKE_SYSTEM_PATH)

        if(YUZU_USE_BUNDLED_QT)
            ## if ((MSVC_VERSION GREATER_EQUAL 1920 AND MSVC_VERSION LESS 1940) AND ARCHITECTURE_x86_64)
            if ((MSVC_VERSION GREATER_EQUAL 1920) AND ARCHITECTURE_x86_64)
                set(QT_BUILD qt-5.15.2-msvc2019_64)
            elseif ((${CMAKE_SYSTEM_NAME} STREQUAL "Linux") AND NOT MINGW AND ARCHITECTURE_x86_64)
                set(QT_BUILD qt5_5_15_2)
            else()
                message(FATAL_ERROR "No bundled Qt binaries for your toolchain. Disable YUZU_USE_BUNDLED_QT and provide your own.")
            endif()

            if (DEFINED QT_BUILD)
                download_bundled_external("qt/" ${QT_BUILD} QT_PREFIX)
                execute_process(COMMAND chmod +x ${QT_PREFIX}/bin/moc.exe ERROR_QUIET)
                execute_process(COMMAND chmod +x ${QT_PREFIX}/bin/uic.exe ERROR_QUIET)
                execute_process(COMMAND chmod +x ${QT_PREFIX}/bin/rcc.exe ERROR_QUIET)
                execute_process(COMMAND chmod +x ${QT_PREFIX}/bin/lrelease.exe ERROR_QUIET)
            endif()

            set(QT_PREFIX_HINT HINTS "${QT_PREFIX}")

            set(YUZU_QT_NO_CMAKE_SYSTEM_PATH "NO_CMAKE_SYSTEM_PATH")
            # Binary package for Qt5 has Qt Multimedia
            set(YUZU_USE_QT_MULTIMEDIA ON CACHE BOOL "Use Qt Multimedia" FORCE)
        endif()

        set_yuzu_qt_components()
        find_package(Qt5 ${QT5_VERSION} COMPONENTS ${YUZU_QT_COMPONENTS} ${QT_PREFIX_HINT} ${YUZU_QT_NO_CMAKE_SYSTEM_PATH})
    endif()

endif()

# find SDL2 exports a bunch of variables that are needed, so its easier to do this outside of the yuzu_find_package
if (ENABLE_SDL2)
    if (YUZU_USE_BUNDLED_SDL2)
        # Detect toolchain and platform
        ## if ((MSVC_VERSION GREATER_EQUAL 1920 AND MSVC_VERSION LESS 1940) AND ARCHITECTURE_x86_64)
        if ((MSVC_VERSION GREATER_EQUAL 1920) AND ARCHITECTURE_x86_64)
            set(SDL2_VER "SDL2-2.28.2")
        else()
            message(FATAL_ERROR "No bundled SDL2 binaries for your toolchain. Disable YUZU_USE_BUNDLED_SDL2 and provide your own.")
        endif()

        if (DEFINED SDL2_VER)
            download_bundled_external("sdl2/" ${SDL2_VER} SDL2_PREFIX)
        endif()

        set(SDL2_FOUND YES)
        set(SDL2_INCLUDE_DIR "${SDL2_PREFIX}/include" CACHE PATH "Path to SDL2 headers")
        set(SDL2_LIBRARY "${SDL2_PREFIX}/lib/x64/SDL2.lib" CACHE PATH "Path to SDL2 library")
        set(SDL2_DLL_DIR "${SDL2_PREFIX}/lib/x64/" CACHE PATH "Path to SDL2.dll")

        add_library(SDL2::SDL2 INTERFACE IMPORTED)
        target_link_libraries(SDL2::SDL2 INTERFACE "${SDL2_LIBRARY}")
        target_include_directories(SDL2::SDL2 INTERFACE "${SDL2_INCLUDE_DIR}")
    elseif (YUZU_USE_EXTERNAL_SDL2)
        message(STATUS "Using SDL2 from externals.")
    else()
        find_package(SDL2 2.26.4 REQUIRED)
    endif()
endif()

# List of all FFmpeg components required
set(FFmpeg_COMPONENTS
    avcodec
    avfilter
    avutil
    swscale)

if (UNIX AND NOT APPLE AND NOT ANDROID)
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(LIBVA libva)
endif()
if (NOT YUZU_USE_BUNDLED_FFMPEG)
    # Use system installed FFmpeg
    find_package(FFmpeg 4.3 REQUIRED QUIET COMPONENTS ${FFmpeg_COMPONENTS})
endif()

if (WIN32 AND YUZU_CRASH_DUMPS)
    set(BREAKPAD_VER "breakpad-c89f9dd")
    download_bundled_external("breakpad/" ${BREAKPAD_VER} BREAKPAD_PREFIX)

    set(BREAKPAD_CLIENT_INCLUDE_DIR "${BREAKPAD_PREFIX}/include")
    set(BREAKPAD_CLIENT_LIBRARY "${BREAKPAD_PREFIX}/lib/libbreakpad_client.lib")

    add_library(libbreakpad_client INTERFACE IMPORTED)
    target_link_libraries(libbreakpad_client INTERFACE "${BREAKPAD_CLIENT_LIBRARY}")
    target_include_directories(libbreakpad_client INTERFACE "${BREAKPAD_CLIENT_INCLUDE_DIR}")
endif()

# Prefer the -pthread flag on Linux.
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)

# Platform-specific library requirements
# ======================================

if (APPLE)
    # Umbrella framework for everything GUI-related
    find_library(COCOA_LIBRARY Cocoa)
    set(PLATFORM_LIBRARIES ${COCOA_LIBRARY} ${IOKIT_LIBRARY} ${COREVIDEO_LIBRARY})
elseif (WIN32)
    # Target Windows 10
    add_definitions(-D_WIN32_WINNT=0x0A00 -DWINVER=0x0A00)
    set(PLATFORM_LIBRARIES winmm ws2_32 iphlpapi)
    if (MINGW)
        # PSAPI is the Process Status API
        set(PLATFORM_LIBRARIES ${PLATFORM_LIBRARIES} psapi imm32 version)
    endif()
elseif (CMAKE_SYSTEM_NAME MATCHES "^(Linux|kFreeBSD|GNU|SunOS)$")
    set(PLATFORM_LIBRARIES rt)
endif()

# Setup a custom clang-format target (if clang-format can be found) that will run
# against all the src files. This should be used before making a pull request.
# =======================================================================

set(CLANG_FORMAT_POSTFIX "-15")
find_program(CLANG_FORMAT
    NAMES clang-format${CLANG_FORMAT_POSTFIX}
          clang-format
    PATHS ${PROJECT_BINARY_DIR}/externals)
# if find_program doesn't find it, try to download from externals
if (NOT CLANG_FORMAT)
    if (WIN32 AND NOT CMAKE_CROSSCOMPILING)
        message(STATUS "Clang format not found! Downloading...")
        set(CLANG_FORMAT "${PROJECT_BINARY_DIR}/externals/clang-format${CLANG_FORMAT_POSTFIX}.exe")
        file(DOWNLOAD
            https://github.com/litucks/ext-windows-bin/raw/master/clang-format${CLANG_FORMAT_POSTFIX}.exe
            "${CLANG_FORMAT}" SHOW_PROGRESS
            STATUS DOWNLOAD_SUCCESS)
        if (NOT DOWNLOAD_SUCCESS EQUAL 0)
            message(WARNING "Could not download clang format! Disabling the clang format target")
            file(REMOVE ${CLANG_FORMAT})
            unset(CLANG_FORMAT)
        endif()
    else()
        message(WARNING "Clang format not found! Disabling the clang format target")
    endif()
endif()

if (CLANG_FORMAT)
    set(SRCS ${PROJECT_SOURCE_DIR}/src)
    set(CCOMMENT "Running clang format against all the .h and .cpp files in src/")
    if (WIN32)
        add_custom_target(clang-format
            COMMAND powershell.exe -Command "Get-ChildItem '${SRCS}/*' -Include *.cpp,*.h -Recurse | Foreach {&'${CLANG_FORMAT}' -i $_.fullname}"
            COMMENT ${CCOMMENT})
    elseif(MINGW)
        add_custom_target(clang-format
            COMMAND find `cygpath -u ${SRCS}` -iname *.h -o -iname *.cpp | xargs `cygpath -u ${CLANG_FORMAT}` -i
            COMMENT ${CCOMMENT})
    else()
        add_custom_target(clang-format
            COMMAND find ${SRCS} -iname *.h -o -iname *.cpp | xargs ${CLANG_FORMAT} -i
            COMMENT ${CCOMMENT})
    endif()
    unset(SRCS)
    unset(CCOMMENT)
endif()

# Include source code
# ===================

# This function should be passed a list of all files in a target. It will automatically generate
# file groups following the directory hierarchy, so that the layout of the files in IDEs matches the
# one in the filesystem.
function(create_target_directory_groups target_name)
    # Place any files that aren't in the source list in a separate group so that they don't get in
    # the way.
    source_group("Other Files" REGULAR_EXPRESSION ".")

    get_target_property(target_sources "${target_name}" SOURCES)

    foreach(file_name IN LISTS target_sources)
        get_filename_component(dir_name "${file_name}" PATH)
        # Group names use '\' as a separator even though the entire rest of CMake uses '/'...
        string(REPLACE "/" "\\" group_name "${dir_name}")
        source_group("${group_name}" FILES "${file_name}")
    endforeach()
endfunction()

# Prevent boost from linking against libs when building
if (NOT TARGET Boost::headers)
    target_link_libraries(Boost::headers INTERFACE Boost::disable_autolinking)
endif()

# Adjustments for MSVC + Ninja
if (MSVC AND CMAKE_GENERATOR STREQUAL "Ninja")
    add_compile_options(
        /wd4464 # relative include path contains '..'
        /wd4711 # function 'function' selected for automatic inline expansion
        /wd4820 # 'bytes' bytes padding added after construct 'member_name'
    )
endif()

# Adjustments for Clang-cl
if (MSVC AND CMAKE_CXX_COMPILER_ID STREQUAL "Clang" AND ARCHITECTURE STREQUAL "x86_64")
    set(LLVM_MINGW_VERSION 20250402)

    # Set download URL and library path within the ZIP
    set(ZIP_URL "https://github.com/mstorsjo/llvm-mingw/releases/download/${LLVM_MINGW_VERSION}/llvm-mingw-${LLVM_MINGW_VERSION}-ucrt-x86_64.zip")
    set(LIB_PATH "llvm-mingw-${LLVM_MINGW_VERSION}-ucrt-x86_64/lib/clang/20/lib/windows/libclang_rt.builtins-x86_64.a")

    # Set paths for download and extraction
    set(DOWNLOAD_DIR "${CMAKE_BINARY_DIR}/llvm-mingw-download")
    set(ZIP_FILE "${DOWNLOAD_DIR}/llvm-mingw.zip")
    set(EXTRACTED_LIB "${DOWNLOAD_DIR}/${LIB_PATH}")

    # Create download directory if it doesn't exist
    file(MAKE_DIRECTORY "${DOWNLOAD_DIR}")

    # Download and extract if the library doesn't exist
    if(NOT EXISTS "${EXTRACTED_LIB}")
        message(STATUS "Downloading llvm-mingw runtime libraries...")

        # Download the ZIP file
        file(DOWNLOAD
            ${ZIP_URL}
            ${ZIP_FILE}
            SHOW_PROGRESS
            # Uncomment and add EXPECTED_HASH if you know the SHA256 checksum
            EXPECTED_HASH SHA256=4edc13d878b4ec49c2f1a6e9161abb093bbaefc8b7d129f3b3f57a22a4a41d38
        )

        message(STATUS "Extracting compiler-rt builtins library...")

        # Extract the specific file from the ZIP
        execute_process(
            COMMAND ${CMAKE_COMMAND} -E tar xvf "${ZIP_FILE}" "${LIB_PATH}"
            WORKING_DIRECTORY "${DOWNLOAD_DIR}"
            RESULT_VARIABLE extraction_result
        )

        if(NOT extraction_result EQUAL 0)
            message(FATAL_ERROR "Failed to extract library: ${extraction_result}")
        endif()
    endif()

    # Create imported target for the library
    add_library(llvm-mingw-runtime STATIC IMPORTED)
    set_target_properties(llvm-mingw-runtime PROPERTIES
        IMPORTED_LOCATION "${EXTRACTED_LIB}"
    )

    # Link the library to all executables in the project
    link_libraries(llvm-mingw-runtime)
endif()

if (YUZU_USE_FASTER_LD AND CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    # We will assume that if the compiler is GCC, it will attempt to use ld.bfd by default.
    # Try to pick a faster linker.
    find_program(LLD lld)
    find_program(MOLD mold)

    if (MOLD AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL "12.1")
        message(STATUS "Selecting mold as linker")
        add_link_options("-fuse-ld=mold")
    elseif (LLD)
        message(STATUS "Selecting lld as linker")
        add_link_options("-fuse-ld=lld")
    endif()
endif()

add_subdirectory(src)

# Set yuzu project or yuzu-cmd project as default StartUp Project in Visual Studio depending on whether QT is enabled or not
if(ENABLE_QT)
    set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT yuzu)
else()
    set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT yuzu-cmd)
endif()


# Installation instructions
# =========================

# Install freedesktop.org metadata files, following those specifications:
# https://specifications.freedesktop.org/desktop-entry-spec/desktop-entry-spec-latest.html
# https://specifications.freedesktop.org/icon-theme-spec/icon-theme-spec-latest.html
# https://specifications.freedesktop.org/shared-mime-info-spec/shared-mime-info-spec-latest.html
# https://www.freedesktop.org/software/appstream/docs/
if(ENABLE_QT AND UNIX AND NOT APPLE)
    install(FILES "dist/onion.torzu_emu.torzu.desktop"
            DESTINATION "share/applications")
    install(FILES "dist/yuzu.svg"
            DESTINATION "share/icons/hicolor/scalable/apps"
            RENAME "onion.torzu_emu.torzu.svg")
    install(FILES "dist/onion.torzu_emu.torzu.xml"
            DESTINATION "share/mime/packages")
    install(FILES "dist/onion.torzu_emu.torzu.metainfo.xml"
            DESTINATION "share/metainfo")
endif()
