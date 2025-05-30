cmake_minimum_required(VERSION 3.21)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_VISIBILITY_PRESET hidden)

if ("${CMAKE_SYSTEM_NAME}" STREQUAL "iOS" OR IOS)
    set(GLOBED_IOS ON)
endif()

if (GLOBED_IOS)
    set(CMAKE_OSX_ARCHITECTURES "arm64")
else()
    set(CMAKE_OSX_ARCHITECTURES "arm64;x86_64")
endif()

project(globed2 VERSION 1.0.0)

option(GLOBED_OSS_BUILD "Open source build that does not require closed-source dependencies" OFF)
option(GLOBED_NATIVE_ARCH "Native architecture build" OFF)
option(GLOBED_DEBUG "Debug mode" OFF)
option(GLOBED_RELEASE "Release build" OFF)
option(GLOBED_DISABLE_VOICE_SUPPORT "Disable voice chat support" OFF)
option(GLOBED_DEBUG_INTERPOLATION "Dump interpolation logs" OFF)
option(GLOBED_DEBUG_PACKETS "Log all incoming & outgoing packets and bandwidth" OFF)
option(GLOBED_DEBUG_PACKETS_PRINT "Print every incoming/outgoing packet" OFF)
option(GLOBED_LESS_BINDINGS "Disable extra hooks & calls to some GD functions, useful when porting to a new GD version" OFF)
option(GLOBED_GP_CHANGES "GP changes" OFF)
option(GLOBED_LINK_TO_FMOD "Whether to link to FMOD, disables voice chat if off" ON)
option(GLOBED_ENABLE_STACKTRACE "Whether to enable stacktrace library" OFF)

if (WIN32)
    option(GLOBED_COMPILE_LIBS "Don't use precompiled libraries and compile manually" OFF)
else()
    option(GLOBED_COMPILE_LIBS "Don't use precompiled libraries and compile manually" ON)
endif()

include(cmake/environment.cmake)

if (ANDROID)
    file(GLOB_RECURSE OS_SOURCES CONFIGURE_DEPENDS "src/platform/os/android/*.cpp" "src/platform/arch/arm/*.cpp")
elseif (APPLE AND GLOBED_IOS)
    file(GLOB_RECURSE OS_SOURCES CONFIGURE_DEPENDS "src/platform/os/ios/*.cpp" "src/platform/arch/arm/*.cpp" "src/platform/os/ios/*.mm")
elseif (APPLE)
    file(GLOB_RECURSE OS_SOURCES CONFIGURE_DEPENDS "src/platform/os/mac/*.cpp" "src/platform/arch/x86/*.cpp" "src/platform/arch/arm/*.cpp" "src/platform/os/mac/*.mm")
elseif (WIN32)
    file(GLOB_RECURSE OS_SOURCES CONFIGURE_DEPENDS
        "src/platform/os/windows/*.cpp"
        "src/platform/arch/x86/*.cpp"
    )
else()
    message(FATAL_ERROR "Unsupported operating system")
endif()

set_source_files_properties(src/platform/os/ios/objc.mm PROPERTIES SKIP_PRECOMPILE_HEADERS ON)
set_source_files_properties(src/platform/os/mac/objc.mm PROPERTIES SKIP_PRECOMPILE_HEADERS ON)

file(GLOB_RECURSE SOURCES CONFIGURE_DEPENDS
    src/audio/*.cpp
    src/crypto/*.cpp
    src/data/*.cpp
    src/globed/*.cpp
    src/game/*.cpp
    src/hooks/*.cpp
    src/managers/*.cpp
    src/net/*.cpp
    src/ui/*.cpp
    src/util/*.cpp
    src/main.cpp
    ${OS_SOURCES}
)

add_library(${PROJECT_NAME} SHARED ${SOURCES})

foreach(option IN ITEMS
    GLOBED_DISABLE_VOICE_SUPPORT
    GLOBED_DEBUG_INTERPOLATION
    GLOBED_DEBUG_PACKETS
    GLOBED_DEBUG_PACKETS_PRINT
    GLOBED_LESS_BINDINGS
    GLOBED_GP_CHANGES
    GLOBED_LINK_TO_FMOD
)
    if (${option})
        target_compile_definitions(${PROJECT_NAME} PRIVATE ${option}=1)
    endif()
endforeach()

if (GLOBED_LINK_TO_FMOD AND NOT GLOBED_DISABLE_VOICE_SUPPORT)
    target_compile_definitions(${PROJECT_NAME} PRIVATE GLOBED_VOICE_SUPPORT=1)
    if (WIN32)
        target_compile_definitions(${PROJECT_NAME} PRIVATE GLOBED_VOICE_CAN_TALK=1)
    endif()
endif()

if (GLOBED_IS_DEBUG)
    add_compile_definitions(GLOBED_DEBUG=1 GEODE_DEBUG=1 ASP_ENABLE_DEBUG=1)
elseif (GLOBED_RELEASE)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
    target_compile_definitions(${PROJECT_NAME} PRIVATE GLOBED_RELEASE=1)
endif()

if (CMAKE_SYSTEM_NAME STREQUAL "Windows")
    if (NOT CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
        message(FATAL_ERROR "MSVC is not supported, please use clang to compile Globed!")
    endif()
    add_compile_definitions(WIN32_LEAN_AND_MEAN=1)
endif()

target_compile_definitions(${PROJECT_NAME} PRIVATE GLOBED_EXPORTING)
target_include_directories(${PROJECT_NAME} PRIVATE src/ libs/ include)

if (NOT DEFINED ENV{GEODE_SDK})
    message(FATAL_ERROR "Unable to find Geode SDK! Please define GEODE_SDK environment variable to point to Geode")
else()
    message(STATUS "Found Geode: $ENV{GEODE_SDK}")
endif()
add_subdirectory($ENV{GEODE_SDK} ${CMAKE_CURRENT_BINARY_DIR}/geode)

CPMAddPackage("gh:blu3berryys/uibuilder#24956b0")
CPMAddPackage(
    NAME Boost
    VERSION 1.88.0
    GITHUB_REPOSITORY boostorg/boost
    GIT_TAG boost-1.88.0
    OPTIONS "BOOST_ENABLE_CMAKE ON" "BOOST_INCLUDE_LIBRARIES describe"
)
CPMAddPackage("gh:dankmeme01/asp2#5b0bae3")
CPMAddPackage("gh:GlobedGD/argon@1.1.3")

if (WIN32)
    target_compile_definitions(asp PRIVATE ASP_ENABLE_FORMAT=1)
    target_compile_definitions(${PROJECT_NAME} PRIVATE ASP_ENABLE_FORMAT=1)
endif()

file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/globed-codegen")
target_include_directories(${PROJECT_NAME} PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/globed-codegen")
include(cmake/baked_resources_gen.cmake)
generate_baked_resources_header(
    "${CMAKE_CURRENT_SOURCE_DIR}/embedded-resources.json"
    "${CMAKE_CURRENT_BINARY_DIR}/globed-codegen/embedded_resources.hpp"
)

if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    # Example compile options
    # target_compile_options(${PROJECT_NAME} PRIVATE "-ftime-trace")
    # target_compile_options(asp PRIVATE "-ftime-trace")
endif()

target_link_libraries(${PROJECT_NAME} UIBuilder Boost::describe asp argon)

if (GLOBED_ENABLE_STACKTRACE)
    CPMAddPackage("gh:jeremy-rifkin/cpptrace#v0.8.3")
    target_link_libraries(${PROJECT_NAME} cpptrace::cpptrace)
endif()

if (GLOBED_COMPILE_LIBS)
    CPMAddPackage("gh:blu3berryys/libsodium-cmake#715fe19")
    CPMAddPackage("gh:xiph/opus#v1.5.2")

    if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        target_compile_options(sodium PRIVATE "-Wno-inaccessible-base" "-Wno-pointer-sign" "-Wno-user-defined-warnings")
    endif()

    target_link_libraries(${PROJECT_NAME} sodium opus)
else()
    CPMAddPackage(NAME sodium GIT_REPOSITORY "https://github.com/jedisct1/libsodium.git" GIT_TAG "1.0.20-RELEASE")
    CPMAddPackage(NAME opus GIT_REPOSITORY "https://github.com/xiph/opus.git" GIT_TAG "v1.5.2" DOWNLOAD_ONLY YES)

    add_library(sodium STATIC IMPORTED)
    set_target_properties(sodium PROPERTIES IMPORTED_LOCATION "${CMAKE_CURRENT_SOURCE_DIR}/libs/sodium/libsodium.lib")

    add_library(opus STATIC IMPORTED)
    set_target_properties(opus PROPERTIES IMPORTED_LOCATION "${CMAKE_CURRENT_SOURCE_DIR}/libs/opus/opus.lib")

    target_compile_definitions(${PROJECT_NAME} PRIVATE SODIUM_STATIC=1)
    target_include_directories(${PROJECT_NAME} PRIVATE "${sodium_SOURCE_DIR}/src/libsodium/include")
    target_include_directories(${PROJECT_NAME} PRIVATE "${opus_SOURCE_DIR}/include")

    set(SODIUM_VERSION_IN "${CMAKE_CURRENT_SOURCE_DIR}/libs/sodium/version.h")
    set(SODIUM_VERSION_OUT "${sodium_SOURCE_DIR}/src/libsodium/include/sodium/")

    if (NOT EXISTS "${SODIUM_VERSION_OUT}/version.h")
        file(COPY "${SODIUM_VERSION_IN}" DESTINATION "${SODIUM_VERSION_OUT}")
    else()
        file(READ "${SODIUM_VERSION_IN}" CONTENT_IN)
        file(READ "${SODIUM_VERSION_OUT}/version.h" CONTENT_OUT)
        if (NOT "${CONTENT_IN}" STREQUAL "${CONTENT_OUT}")
            file(COPY "${SODIUM_VERSION_IN}" DESTINATION "${SODIUM_VERSION_OUT}")
        endif()
    endif()

    target_link_libraries(${PROJECT_NAME} sodium opus)
endif()

if (WIN32)
    target_link_libraries(${PROJECT_NAME} ws2_32)
elseif (APPLE)
    find_library(IOKIT_LIBRARY IOKit)
    target_link_libraries(${PROJECT_NAME} ${IOKIT_LIBRARY})
endif()

CPMAddPackage(
    NAME curl
    GIT_REPOSITORY "https://github.com/curl/curl.git"
    GIT_TAG "curl-8_13_0"
    DOWNLOAD_ONLY YES
)

add_library(curl STATIC IMPORTED)

if (ANDROID)
    set(LIB_PLATFORM "android64")
elseif (WIN32)
    set(LIB_PLATFORM "win64")
elseif (GLOBED_IOS)
    set(LIB_PLATFORM "ios")
else()
    set(LIB_PLATFORM "macos")
endif()

target_compile_definitions(${PROJECT_NAME} PRIVATE CURL_STATICLIB=1)

if (WIN32)
    if (GLOBED_IS_DEBUG)
        set_target_properties(curl PROPERTIES IMPORTED_LOCATION "$ENV{GEODE_SDK}/loader/include/link/${LIB_PLATFORM}/gd-libcurl.lib")
    else()
        set_target_properties(curl PROPERTIES IMPORTED_LOCATION "${CMAKE_CURRENT_SOURCE_DIR}/libs/libcurl.lib") # TODO: replace this back to the geode path
    endif()
else()
    set_target_properties(curl PROPERTIES IMPORTED_LOCATION "$ENV{GEODE_SDK}/loader/include/link/${LIB_PLATFORM}/libcurl.a")
endif()

target_include_directories(${PROJECT_NAME} PRIVATE "${curl_SOURCE_DIR}/include")
target_link_libraries(${PROJECT_NAME} curl)

if (APPLE)
    target_link_libraries(${PROJECT_NAME} z)
elseif (WIN32)
    target_link_libraries(${PROJECT_NAME} crypt32)
elseif (ANDROID)
    target_link_libraries(${PROJECT_NAME} android)
endif()

if (GLOBED_OSS_BUILD OR GLOBED_IOS)
    message(STATUS "Building open-source version, not linking to bb")
    target_compile_definitions(${PROJECT_NAME} PRIVATE GLOBED_OSS_BUILD=1)
else()
    if (WIN32)
        target_link_libraries(${PROJECT_NAME}
            "${CMAKE_CURRENT_SOURCE_DIR}/libs/bb/bb.lib"
            ntdll.lib userenv.lib runtimeobject.lib Iphlpapi.lib bcrypt.lib
        )
    else()
        target_link_libraries(${PROJECT_NAME} "${CMAKE_CURRENT_SOURCE_DIR}/libs/bb/bb-${LIB_PLATFORM}.a")
    endif()
endif()

setup_geode_mod(${PROJECT_NAME})
