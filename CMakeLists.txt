cmake_minimum_required(VERSION 3.14 FATAL_ERROR) # Set to a modern version for better features
project(ModLoader VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 17)
add_compile_definitions(UNICODE _UNICODE)

# --- Include CMake Core dependencies ---

# Download CPM.cmake
if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/cmake/CPM.cmake")
    message(STATUS "Downloading CPM.cmake...")
    file(
        DOWNLOAD
        https://github.com/cpm-cmake/CPM.cmake/releases/download/v0.42.0/CPM.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/cmake/CPM.cmake
        EXPECTED_HASH SHA256=2020b4fc42dba44817983e06342e682ecfc3d2f484a581f11cc5731fbe4dce8a
    )
endif()
include(${CMAKE_CURRENT_BINARY_DIR}/cmake/CPM.cmake)

# --- Handle External System Libraries ---
find_package(ZLIB REQUIRED)
find_package(OpenSSL REQUIRED COMPONENTS Crypto SSL)
# Json library
CPMAddPackage("gh:nlohmann/json@3.12.0")
# fmt library 
CPMAddPackage("gh:fmtlib/fmt#11.2.0")
# cpr library 
CPMAddPackage("gh:libcpr/cpr#1.12.0")
# bit7z library
CPMAddPackage(
    NAME bit7z
    VERSION 2.1.0
    GITHUB_REPOSITORY rikyoz/bit7z
    OPTIONS "BUILD_SHARED_LIBS ON"
    # This ensures the include directory is correctly exposed
    STEP_BUILD "target_include_directories(bit7z PUBLIC \${bit7z_SOURCE_DIR}/include)"
)
# Detours library
# Manually configure Detours as a pre-built library, since CPM cannot build it properly.
add_library(detours STATIC IMPORTED)
set_target_properties(detours PROPERTIES
    IMPORTED_LOCATION "${CMAKE_CURRENT_SOURCE_DIR}/libraries/detours/lib.x86/detours.lib"
    INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_CURRENT_SOURCE_DIR}/libraries/detours/src"
)

# --- The Main Executable/Library ---
add_library(ModLoader SHARED
    # A cleaner way to list source files using file(GLOB)
    src/Memory.hpp src/Memory.cpp
    src/library.cpp src/stdafx.hpp
    src/utils.hpp src/utils.cpp
    src/md5.hpp src/md5.cpp
    src/exception.hpp
    src/httplib.hpp
    src/url.hpp src/url.cpp
    
    src/server/modding_info.hpp src/server/modding_info.cpp
    src/server/servertalk.cpp src/server/servertalk.hpp src/server/servertalkexception.hpp
    
    src/loader/mod_package_loader.cpp src/loader/mod_package_loader.hpp
    src/loader/mod_package.cpp src/loader/mod_package.hpp
    src/loader/server_mod_loader.cpp src/loader/server_mod_loader.hpp
    src/loader/mod_loader_exception.hpp
    src/loader/mod_package_header.hpp
    src/loader/mod_package_item.cpp src/loader/mod_package_item.hpp
    
    src/linker/mod_file_linker.cpp src/linker/mod_file_linker.hpp
    src/linker/mod_file_link_info.hpp src/linker/mod_file_linker_exception.hpp
    
    src/patches/bypass_patches.hpp
    src/patches/custom_crash_handler.hpp src/patches/custom_crash_handler.cpp
)

# --- Link all targets using modern, consistent names ---
target_link_libraries(ModLoader PRIVATE
    # System Libraries (Manually Installed)
    ZLIB::ZLIB
    OpenSSL::SSL
    OpenSSL::Crypto
    # Dependencies downloaded by CPMAddPackage
    nlohmann_json::nlohmann_json
    fmt::fmt
    cpr::cpr
	bit7z
	# Local Dependencies (Manually Installed or Build from Source)
    detours
)

# --- Set Target Properties ---
set_target_properties(
    ModLoader
    PROPERTIES
    OUTPUT_NAME "ModLoader"
    SUFFIX ".asi"
)