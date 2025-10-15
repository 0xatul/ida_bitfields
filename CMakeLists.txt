cmake_minimum_required(VERSION 3.14)

project(ida_bitfields)

include(FetchContent)

FetchContent_Declare(
  HexSuite
  GIT_REPOSITORY https://github.com/can1357/hexsuite
)

FetchContent_MakeAvailable(HexSuite)

add_library(bitfields SHARED ${CMAKE_CURRENT_SOURCE_DIR}/plugin.cpp)

target_compile_features(bitfields PRIVATE cxx_std_20)

# Allow specifying IDA SDK directory via command line
if (DEFINED IDA_SDK_DIR)
    set(IDASDK ${IDA_SDK_DIR}/src)
elseif (DEFINED ENV{IDA_PATH})
    set(IDASDK $ENV{IDA_PATH}/src)
elseif (DEFINED ENV{IDA_DIR})
    set(IDASDK $ENV{IDA_DIR}/src)
else()
    message(FATAL_ERROR "IDA_SDK_DIR, IDA_PATH or IDA_DIR environment variable must be set")
endif()

message(STATUS "Using IDA SDK at: ${IDASDK}")

# Verify IDA SDK is properly available
if(NOT EXISTS ${IDASDK}/include/ida.hpp)
    message(FATAL_ERROR "Missing header file in IDA SDK: ${IDASDK}/include/ida.hpp")
endif()

# Set up platform-specific library paths and defines
set(IMPORTED_PROPERTY "IMPORTED_LOCATION")
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    if(NOT CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
        message(FATAL_ERROR "Unsupported architecture ${CMAKE_SYSTEM_PROCESSOR} for platform ${CMAKE_SYSTEM_NAME}")
    endif()
    set(PLATFORM_DEFINE -D__LINUX__=1)
    set(IDA_KERNEL_LIB "${IDASDK}/lib/x64_linux_gcc_64/libida.so")
    set(IDA_LIB_LIB "${IDASDK}/lib/x64_linux_gcc_64/libidalib.so")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    if(NOT CMAKE_SYSTEM_PROCESSOR STREQUAL "AMD64")
        message(FATAL_ERROR "Unsupported architecture ${CMAKE_SYSTEM_PROCESSOR} for platform ${CMAKE_SYSTEM_NAME}")
    endif()
    set(PLATFORM_DEFINE -D__NT__=1)
    set(IDA_KERNEL_LIB "${IDASDK}/lib/x64_win_vc_64/ida.lib")
    set(IDA_LIB_LIB "${IDASDK}/lib/x64_win_vc_64/idalib.lib")
    set(IMPORTED_PROPERTY "IMPORTED_IMPLIB")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set(PLATFORM_DEFINE -D__MAC__=1)
    if(CMAKE_SYSTEM_PROCESSOR STREQUAL "arm64")
        set(IDA_KERNEL_LIB "${IDASDK}/lib/arm64_mac_clang_64/libida.dylib")
        set(IDA_LIB_LIB "${IDASDK}/lib/arm64_mac_clang_64/libidalib.dylib")
    elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
        set(IDA_KERNEL_LIB "${IDASDK}/lib/x64_mac_clang_64/libida.dylib")
        set(IDA_LIB_LIB "${IDASDK}/lib/x64_mac_clang_64/libidalib.dylib")
    else()
        message(FATAL_ERROR "Unsupported architecture ${CMAKE_SYSTEM_PROCESSOR} for platform ${CMAKE_SYSTEM_NAME}")
    endif()
else()
    message(FATAL_ERROR "Unsupported platform: ${CMAKE_SYSTEM_NAME}")
endif()

# Create imported targets for IDA SDK
add_library(idasdk::headers INTERFACE IMPORTED)
target_include_directories(idasdk::headers INTERFACE ${IDASDK}/include)
target_compile_definitions(idasdk::headers INTERFACE __EA64__=1 ${PLATFORM_DEFINE})

add_library(idasdk::kernel SHARED IMPORTED)
set_target_properties(idasdk::kernel PROPERTIES
    ${IMPORTED_PROPERTY} ${IDA_KERNEL_LIB}
    INTERFACE_LINK_LIBRARIES idasdk::headers
)

add_library(idasdk::lib SHARED IMPORTED)
set_target_properties(idasdk::lib PROPERTIES
    ${IMPORTED_PROPERTY} ${IDA_LIB_LIB}
    INTERFACE_LINK_LIBRARIES idasdk::kernel
)

# Set output name for 64-bit builds
if("${CMAKE_SIZEOF_VOID_P}" STREQUAL "8")
    set_target_properties(bitfields PROPERTIES OUTPUT_NAME "bitfields64")
endif()

# Link our plugin against HexSuite and IDA SDK
target_include_directories(bitfields SYSTEM PRIVATE ${hexsuite_SOURCE_DIR})
target_link_libraries(bitfields PRIVATE idasdk::lib)

# For Unix systems, set the appropriate file extensions
if(UNIX)
    if(APPLE)
        set_target_properties(bitfields PROPERTIES SUFFIX ".dylib")
    else()
        set_target_properties(bitfields PROPERTIES SUFFIX ".so")
    endif()
endif()
