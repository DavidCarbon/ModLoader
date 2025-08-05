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
# Define a directory for extracted dependencies
set(DEPENDENCY_ROOT_DIR "${CMAKE_CURRENT_BINARY_DIR}/extracted_deps")
# Define the paths for the archive and extracted content
set(BIT7Z_ARCHIVE_FILENAME "bit7z-v2.1.0-msvc2015_mt_x86.7z")
set(BIT7Z_EXTRACT_DIR "${DEPENDENCY_ROOT_DIR}/bit7z")
set(BIT7Z_ARCHIVE_PATH "${BIT7Z_EXTRACT_DIR}/${BIT7Z_ARCHIVE_FILENAME}")
set(BIT7Z_EXTRACTED_SUBDIR "${BIT7Z_EXTRACT_DIR}/bit7z-v2.1.0-msvc2015_mt_x86")
# Check if the extracted files already exist
if(NOT EXISTS "${BIT7Z_EXTRACTED_SUBDIR}")
    message(STATUS "Downloading bit7z archive...")
	# Create the directory for the download and extraction
    file(MAKE_DIRECTORY "${BIT7Z_EXTRACT_DIR}")
    file(DOWNLOAD
        https://github.com/rikyoz/bit7z/releases/download/v2.1.0/${BIT7Z_ARCHIVE_FILENAME}
        "${BIT7Z_ARCHIVE_PATH}"
        STATUS download_status
        EXPECTED_HASH SHA256=410ac77d059f8d9b47b25ed196601d3e5392070d57846dbc1ed981c7d78b4099
    )
    if(NOT download_status EQUAL "0")
        message(FATAL_ERROR "Failed to download bit7z archive.")
    endif()

    message(STATUS "Extracting bit7z archive...")
    # Use 7z.exe to extract the archive to a specific directory.
    # The -o flag is the issue, so we are passing the path with proper quoting and no trailing slash.
    execute_process(
        COMMAND 7z x "${BIT7Z_ARCHIVE_PATH}"
        WORKING_DIRECTORY "${BIT7Z_EXTRACT_DIR}"
        RESULT_VARIABLE _result
    )
    if(NOT _result EQUAL "0")
        message(FATAL_ERROR "Failed to extract bit7z archive.")
    endif()
endif()

set(BIT7Z_INCLUDE_DIR "${BIT7Z_EXTRACTED_SUBDIR}/include")
set(BIT7Z_LIBRARY_PATH "${BIT7Z_EXTRACTED_SUBDIR}/lib/bit7z.lib")
# Define an IMPORTED library target for bit7z
add_library(bit7z STATIC IMPORTED)
set_target_properties(bit7z PROPERTIES
    IMPORTED_LOCATION "${BIT7Z_LIBRARY_PATH}"
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

target_include_directories(ModLoader PRIVATE "${BIT7Z_INCLUDE_DIR}")

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