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

# Check if we're in CI environment
if(DEFINED ENV{CI} OR DEFINED CI)
  message(STATUS "CI environment detected, applying additional compiler definitions")
  # Add definitions to handle IDA SDK in CI environment
  target_compile_definitions(bitfields PRIVATE 
    USE_STANDARD_FILE_FUNCTIONS
    NO_OBSOLETE_FUNCS
    __EA64__
    __X64__
  )
endif()

# Allow specifying IDA SDK directory via command line
if (DEFINED IDA_SDK_DIR)
    set(IDA_PATH ${IDA_SDK_DIR})
elseif (DEFINED ENV{IDA_PATH})
    set(IDA_PATH $ENV{IDA_PATH})
elseif (DEFINED ENV{IDA_DIR})
    set(IDA_PATH $ENV{IDA_DIR})
else()
    message(FATAL_ERROR "IDA_SDK_DIR, IDA_PATH or IDA_DIR environment variable must be set")
endif()

message(STATUS "Using IDA SDK at: ${IDA_PATH}")

# Set include directories based on IDA SDK structure
target_include_directories(bitfields SYSTEM PRIVATE 
    ${hexsuite_SOURCE_DIR}
)

# Add SDK include paths for the IDA SDK repository
target_include_directories(bitfields SYSTEM PRIVATE 
    ${IDA_PATH}/include
    ${IDA_PATH}/sdk/include
    ${IDA_PATH}/src/include
)

# Detect platform
if (${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    set(IDA_SDK_LIB_OS "win_vc")
elseif (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(IDA_SDK_LIB_OS "mac_clang")
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    set(IDA_SDK_LIB_OS "linux_gcc")
else()
    message(FATAL_ERROR "unrecognized system: ${CMAKE_SYSTEM_NAME} (Windows/Darwin/Linux expected)")
endif()

# Detect architecture
if(${CMAKE_SYSTEM_PROCESSOR} MATCHES "arm") 
    set(IDA_SDK_LIB_ARCH "arm64")
else()
    set(IDA_SDK_LIB_ARCH "x64")
endif()

# Set 32/64 bit mode
if("${CMAKE_SIZEOF_VOID_P}" STREQUAL "8")
    set(IDA_SDK_LIB_BITS "64")
    set_target_properties(bitfields PROPERTIES OUTPUT_NAME "bitfields64")
else()
    set(IDA_SDK_LIB_BITS "32")
endif()

# Different platforms have different library names and extensions
if(WIN32)
    set(IDA_LIB_NAME "ida.lib")
else()
    if(APPLE)
        set(IDA_LIB_NAME "libida64.dylib")
    else() # Linux
        set(IDA_LIB_NAME "libida64.so")
    endif()
endif()

# In CI environment, we might need to skip actual linking against ida libraries
if(DEFINED ENV{CI} OR DEFINED CI)
    message(STATUS "CI environment detected, skipping IDA library linkage")
else()
    target_link_libraries(bitfields PRIVATE ${IDA_LIB_NAME})
endif()

# Add library directories - check both possible SDK structures
if (EXISTS "${IDA_PATH}/lib")
    target_link_directories(
        bitfields PRIVATE 
            ${IDA_PATH}/lib/${IDA_SDK_LIB_ARCH}_${IDA_SDK_LIB_OS}_${IDA_SDK_LIB_BITS}_pro
            ${IDA_PATH}/lib/${IDA_SDK_LIB_ARCH}_${IDA_SDK_LIB_OS}_${IDA_SDK_LIB_BITS}
    )
else()
    target_link_directories(
        bitfields PRIVATE 
            ${IDA_PATH}/sdk/lib/${IDA_SDK_LIB_ARCH}_${IDA_SDK_LIB_OS}_${IDA_SDK_LIB_BITS}_pro
            ${IDA_PATH}/sdk/lib/${IDA_SDK_LIB_ARCH}_${IDA_SDK_LIB_OS}_${IDA_SDK_LIB_BITS}
    )
endif()

# For Unix systems, set the appropriate file extensions
if(UNIX)
    if(APPLE)
        set_target_properties(bitfields PROPERTIES SUFFIX ".dylib")
    else()
        set_target_properties(bitfields PROPERTIES SUFFIX ".so")
    endif()
endif()