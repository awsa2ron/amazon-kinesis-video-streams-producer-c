cmake_minimum_required(VERSION 2.8)
project(KinesisVideoProducerC LANGUAGES C)

# User Flags
option(ADD_MUCLIBC "Add -muclibc c flag" OFF)
option(BUILD_DEPENDENCIES "Whether or not to build depending libraries from source" ON)
option(BUILD_MBEDTLS "If building dependencies, whether or not building mebedtls from source" OFF)
OPTION(BUILD_COMMON_CURL "Whether or not to build ProducerC curl common library" ON)

# Developer Flags
option(BUILD_TEST "Build the testing tree." OFF)
option(CODE_COVERAGE "Enable coverage reporting" OFF)
option(COMPILER_WARNINGS "Enable all compiler warnings." OFF)
option(ADDRESS_SANITIZER "Build with AddressSanitizer." OFF)
option(MEMORY_SANITIZER "Build with MemorySanitizer." OFF)
option(THREAD_SANITIZER "Build with ThreadSanitizer." OFF)
option(UNDEFINED_BEHAVIOR_SANITIZER "Build with UndefinedBehaviorSanitizer." OFF)

set(CMAKE_MACOSX_RPATH TRUE)

get_filename_component(ROOT "${CMAKE_CURRENT_SOURCE_DIR}" ABSOLUTE)

if(NOT KINESIS_VIDEO_PIC_SRC)
  if(DEFINED ENV{KINESIS_VIDEO_PIC_SRC})
    set(KINESIS_VIDEO_PIC_SRC $ENV{KINESIS_VIDEO_PIC_SRC})
  else()
    set(KINESIS_VIDEO_PIC_SRC "${CMAKE_CURRENT_SOURCE_DIR}/open-source/amazon-kinesis-video-streams-pic/")
  endif()
endif()

if(NOT KINESIS_VIDEO_PRODUCER_C_SRC)
    if(DEFINED ENV{KINESIS_VIDEO_PRODUCER_C_SRC})
        set(KINESIS_VIDEO_PRODUCER_C_SRC $ENV{KINESIS_VIDEO_PRODUCER_C_SRC})
    else()
        set(KINESIS_VIDEO_PRODUCER_C_SRC "${CMAKE_CURRENT_SOURCE_DIR}")
    endif()
endif()

if(NOT KINESIS_VIDEO_OPEN_SOURCE_SRC)
  if(DEFINED ENV{KINESIS_VIDEO_OPEN_SOURCE_SRC})
    set(KINESIS_VIDEO_OPEN_SOURCE_SRC $ENV{KINESIS_VIDEO_OPEN_SOURCE_SRC})
  elseif(BUILD_DEPENDENCIES)
    set(KINESIS_VIDEO_OPEN_SOURCE_SRC ${CMAKE_SOURCE_DIR}/open-source/local)
  else()
    set(KINESIS_VIDEO_OPEN_SOURCE_SRC "${CMAKE_CURRENT_SOURCE_DIR}/open-source/downloads/local" )
  endif()
endif()

find_package(PkgConfig REQUIRED)
set(ENV{PKG_CONFIG_PATH} "$ENV{PKG_CONFIG_PATH}:${KINESIS_VIDEO_OPEN_SOURCE_SRC}/lib/pkgconfig")
set(CMAKE_PREFIX_PATH ${KINESIS_VIDEO_OPEN_SOURCE_SRC})

message(STATUS "Kinesis Video PIC path is ${KINESIS_VIDEO_PIC_SRC}")
message(STATUS "Kinesis Video Producer path is ${KINESIS_VIDEO_PRODUCER_C_SRC}")
message(STATUS "Kinesis Video Open Source path is ${KINESIS_VIDEO_OPEN_SOURCE_SRC}")

if(BUILD_DEPENDENCIES)
  set(OPEN_SOURCE_DIR ${CMAKE_SOURCE_DIR}/open-source)

  if(NOT EXISTS ${OPEN_SOURCE_DIR})
    file(MAKE_DIRECTORY ${OPEN_SOURCE_DIR}/local)
  endif()

  set(OPEN_SRC_INSTALL_PREFIX ${CMAKE_SOURCE_DIR}/open-source/local)

  if(BUILD_MBEDTLS)
    if(NOT EXISTS ${OPEN_SOURCE_DIR}/libmbedtls)
      # build mbedtls
      configure_file(
              ${CMAKE_SOURCE_DIR}/cmake-scripts/libmbedtls-CMakeLists.txt
              ${OPEN_SOURCE_DIR}/libmbedtls/CMakeLists.txt COPYONLY)
      execute_process(
              COMMAND ${CMAKE_COMMAND}
                -G ${CMAKE_GENERATOR} .
              RESULT_VARIABLE result
              WORKING_DIRECTORY ${OPEN_SOURCE_DIR}/libmbedtls)
      if(result)
        message(FATAL_ERROR "CMake step for libmbedtls failed: ${result}")
      endif()
      execute_process(
              COMMAND ${CMAKE_COMMAND} --build .
              RESULT_VARIABLE result
              WORKING_DIRECTORY ${OPEN_SOURCE_DIR}/libmbedtls)
      if(result)
        message(FATAL_ERROR "CMake step for libmbedtls failed: ${result}")
      endif()
    endif()
  endif()


  if(NOT EXISTS ${OPEN_SOURCE_DIR}/libcurl AND BUILD_COMMON_CURL)
    configure_file(
      ${CMAKE_SOURCE_DIR}/cmake-scripts/libcurl-CMakeLists.txt
      ${OPEN_SOURCE_DIR}/libcurl/CMakeLists.txt COPYONLY)
    execute_process(
      COMMAND ${CMAKE_COMMAND}
            -DBUILD_MBEDTLS=${BUILD_MBEDTLS}
            -G ${CMAKE_GENERATOR} .
      RESULT_VARIABLE result
      WORKING_DIRECTORY ${OPEN_SOURCE_DIR}/libcurl)
    if(result)
      message(FATAL_ERROR "CMake step for libcurl failed: ${result}")
    endif()
    execute_process(
      COMMAND ${CMAKE_COMMAND} --build .
      RESULT_VARIABLE result
      WORKING_DIRECTORY ${OPEN_SOURCE_DIR}/libcurl)
    if(result)
      message(FATAL_ERROR "CMake step for libcurl failed: ${result}")
    endif()
  endif()

  if(NOT EXISTS ${OPEN_SOURCE_DIR}/libjsmn)
    # build libjsmn
    configure_file(${CMAKE_SOURCE_DIR}/cmake-scripts/libjsmn-CMakeLists.txt
                   ${OPEN_SOURCE_DIR}/libjsmn/CMakeLists.txt COPYONLY)
    execute_process(
      COMMAND ${CMAKE_COMMAND} -G ${CMAKE_GENERATOR} .
      RESULT_VARIABLE result
      WORKING_DIRECTORY ${OPEN_SOURCE_DIR}/libjsmn)
    if(result)
      message(FATAL_ERROR "CMake step for libjsmn failed: ${result}")
    endif()
    execute_process(
      COMMAND ${CMAKE_COMMAND} --build .
      RESULT_VARIABLE result
      WORKING_DIRECTORY ${OPEN_SOURCE_DIR}/libjsmn)
    if(result)
      message(FATAL_ERROR "CMake step for libjsmn failed: ${result}")
    endif()
  endif()

  if(NOT EXISTS ${OPEN_SOURCE_DIR}/libgtest)
    # build gtest
    configure_file(${CMAKE_SOURCE_DIR}/cmake-scripts/libgtest-CMakeLists.txt
                   ${OPEN_SOURCE_DIR}/libgtest/CMakeLists.txt COPYONLY)
    execute_process(
      COMMAND ${CMAKE_COMMAND} -G ${CMAKE_GENERATOR} .
      RESULT_VARIABLE result
      WORKING_DIRECTORY ${OPEN_SOURCE_DIR}/libgtest)
    if(result)
      message(FATAL_ERROR "CMake step for libgtest failed: ${result}")
    endif()
    execute_process(
      COMMAND ${CMAKE_COMMAND} --build .
      RESULT_VARIABLE result
      WORKING_DIRECTORY ${OPEN_SOURCE_DIR}/libgtest)
    if(result)
      message(FATAL_ERROR "CMake step for libgtest failed: ${result}")
    endif()
  endif()
endif()

if(CMAKE_SIZEOF_VOID_P STREQUAL 4)
    message(STATUS "Bitness 32 bits")
    set(KINESIS_VIDEO_BUILD_BITNESS "x86")
    set(KINESIS_VIDEO_BUILD_BITNESS_TYPE "Win32")
    set(KINESIS_VIDEO_BUILD_BITNESS_NAME "x86")
elseif(CMAKE_SIZEOF_VOID_P STREQUAL 8)
    message(STATUS "Bitness 64 bits")
    set(KINESIS_VIDEO_BUILD_BITNESS "x64")
    set(KINESIS_VIDEO_BUILD_BITNESS_TYPE "x64")
    set(KINESIS_VIDEO_BUILD_BITNESS_NAME "x86_64")
else()
    message(FATAL_ERROR "Unknown bitness")
endif()

if("${CMAKE_C_COMPILER_ID}" MATCHES "GNU|Clang")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")

    if(CODE_COVERAGE)
      set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O0 -g -fprofile-arcs -ftest-coverage")
      set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
    endif()

    if(ADDRESS_SANITIZER)
        enableSanitizer("address")
    endif()
    if(MEMORY_SANITIZER)
        enableSanitizer("memory")
    endif()
    if(THREAD_SANITIZER)
        enableSanitizer("thread")
    endif()
    if(UNDEFINED_BEHAVIOR_SANITIZER)
        enableSanitizer("undefined")
    endif()


    if(BUILD_COMMON_CURL)
        pkg_check_modules(LIBCURL REQUIRED libcurl)

        set(OPEN_SRC_INCLUDE_DIRS ${OPEN_SRC_INCLUDE_DIRS} ${LIBCURL_INCLUDE_DIRS})
        link_directories(${LIBCURL_LIBRARY_DIRS})
    endif()

endif()

find_library(Jsmn
    NAMES jsmn
    REQUIRED
    PATHS   ${KINESIS_VIDEO_OPEN_SOURCE_SRC}/lib
    NO_DEFAULT_PATH)

file(GLOB KVS_COMMON_SOURCE_FILES_BASE "src/source/Common/*.c")
file(GLOB KVS_COMMON_SOURCE_FILES_LWS "src/source/Common/Lws/*.c")
file(GLOB KVS_COMMON_SOURCE_FILES_CURL "src/source/Common/Curl/*.c")
file(GLOB PRODUCER_C_SOURCE_FILES "src/source/*.c")
file(GLOB PIC_HEADERS "${KINESIS_VIDEO_PIC_SRC}/src/*/include")

include_directories(${PIC_HEADERS})
include_directories(${KINESIS_VIDEO_PRODUCER_C_SRC}/src/include)
include_directories(${OPEN_SRC_INCLUDE_DIRS})
include_directories(${KINESIS_VIDEO_OPEN_SOURCE_SRC}/include)

add_subdirectory(${KINESIS_VIDEO_PIC_SRC} ./kvspic EXCLUDE_FROM_ALL)

if (BUILD_MBEDTLS)
    set(CPRODUCER_COMMON_TLS_OPTION KVS_USE_MBEDTLS)
endif()

find_library(Mbedtls
        NAMES mbedtls
        REQUIRED)
find_library(Mbedcrypto
        NAMES mbedcrypto
        REQUIRED)
# producer only uses curl right now
add_library(kvsCommonCurl STATIC ${KVS_COMMON_SOURCE_FILES_BASE} ${KVS_COMMON_SOURCE_FILES_CURL})
target_compile_definitions(kvsCommonCurl PRIVATE KVS_BUILD_WITH_CURL ${CPRODUCER_COMMON_TLS_OPTION})
target_link_libraries(kvsCommonCurl
        ${Jsmn}
        ${Mbedtls}
        ${Mbedcrypto}
        ${LIBCURL_LIBRARIES})

add_library(cproducer SHARED ${PRODUCER_C_SOURCE_FILES})

if (MINGW)
# Use PRIVATE otherwise anything linking against cproducer will also link with pic again,
# causing duplicate symbols
target_link_libraries(cproducer PRIVATE
    kvspic
    kvsCommonCurl)
else()
target_link_libraries(cproducer
        kvspic
        kvsCommonCurl)
endif()

add_executable(kvsVideoOnlyStreamingSample ${KINESIS_VIDEO_PRODUCER_C_SRC}/samples/KvsVideoOnlyStreamingSample.c)
target_link_libraries(kvsVideoOnlyStreamingSample
    cproducer)

add_executable(kvsAacAudioVideoStreamingSample ${KINESIS_VIDEO_PRODUCER_C_SRC}/samples/KvsAacAudioVideoStreamingSample.c)
target_link_libraries(kvsAacAudioVideoStreamingSample
        cproducer)

if (BUILD_TEST)
    add_subdirectory(tst)
endif()
