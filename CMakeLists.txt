# Copyright 2014 Google Inc. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
if(fpl_ios)
  cmake_minimum_required(VERSION 3.3.1)
else()
  cmake_minimum_required(VERSION 2.8.12)
endif()

project(pindrop)

set(pindrop_standalone_mode OFF)
if("${CMAKE_CURRENT_LIST_DIR}" STREQUAL "${CMAKE_SOURCE_DIR}")
  set(pindrop_standalone_mode ON)
  message(STATUS "Pindrop standalone: building library and samples")
endif()
option(pindrop_build_sample "Build the pindrop sample"
       ${pindrop_standalone_mode})
option(pindrop_build_tests "Build tests for this project."
       ${pindrop_standalone_mode})

# By default file load operations are blocking. FPLBase offers an async loader
# class that we can optionally use. To enable async loading set
# pindrop_async_loading=ON as well as the path to fplbase with the variable
# dependencies_fplbase_dir.
option(pindrop_async_loading "Support async loading with fplbase" OFF)

option(pindrop_multistream "Support multiple channels of streaming audio" OFF)
if(pindrop_multistream)
  add_definitions(-DPINDROP_MULTISTREAM)
endif()

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
if(NOT fpl_ios)
  # This needs to be default for iOS as output dirs are of the form
  # Debug-iphoneos, Debug-iphonesimulator to distinguish iphone and simulator
  # builds, otherwise each build overwrites the other if the output dir is the
  # same
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/bin)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/bin)
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/bin)
endif()

# Temporary files (like object files) created while compiling projects.
set(tmp_dir ${CMAKE_CURRENT_BINARY_DIR}/obj)

# Directory which contains the source for 3rd party libraries.
if(NOT DEFINED third_party_root)
  get_filename_component(
    third_party_root "${CMAKE_CURRENT_SOURCE_DIR}/../../../../external"
    REALPATH)
endif()

# Directory which contains source for FPL libraries.
if(NOT DEFINED fpl_root)
  get_filename_component(
      fpl_root "${CMAKE_CURRENT_SOURCE_DIR}/../../libs" REALPATH)
endif()

# If the dependencies directory exists, assume this is the root directory for
# all libraries required by this project.
if(NOT DEFINED dependencies_root)
  set(dependencies_root "${CMAKE_CURRENT_SOURCE_DIR}/dependencies")
  if(EXISTS "${dependencies_root}")
    set(third_party_root "${dependencies_root}")
    set(fpl_root "${dependencies_root}")
  endif()
endif()

# Configurable locations of dependencies of this project.
set(dependencies_gtest_dir "${fpl_root}/googletest"
    CACHE PATH "Directory containing the GoogleTest library.")
set(dependencies_flatbuffers_dir "${fpl_root}/flatbuffers"
    CACHE PATH "Directory containing the Flatbuffers library.")
set(dependencies_fplbase_dir "${fpl_root}/fplbase"
    CACHE PATH "Directory containing the FplBase library.")
set(dependencies_mathfu_dir "${fpl_root}/mathfu"
    CACHE PATH "Directory containing the MathFu library.")
set(dependencies_sdl_dir "${third_party_root}/sdl"
    CACHE PATH "Directory containing the SDL library.")
set(dependencies_libogg_cmake_dir "${CMAKE_MODULE_PATH}/libogg"
    CACHE PATH "Directory containing the libogg cmake project.")
set(dependencies_libogg_distr_dir "${third_party_root}/libogg"
    CACHE PATH "Directory containing the libogg library.")
set(dependencies_libvorbis_cmake_dir "${CMAKE_MODULE_PATH}/libvorbis"
    CACHE PATH "Directory containing the libvorbis cmake project.")
set(dependencies_libvorbis_distr_dir "${third_party_root}/libvorbis"
    CACHE PATH "Directory containing the libvorbis library.")
set(dependencies_sdl_mixer_cmake_dir "${CMAKE_MODULE_PATH}/sdl_mixer"
    CACHE PATH "Directory containing the SDL Mixer cmake project.")
set(dependencies_sdl_mixer_distr_dir "${third_party_root}/sdl_mixer"
    CACHE PATH "Directory containing the SDL Mixer library.")

# Detect clang
if(CMAKE_CXX_COMPILER_ID MATCHES ".*Clang" OR
   CMAKE_CXX_COMPILER_ID MATCHES ".*clang")
  set(CMAKE_COMPILER_IS_CLANGXX 1)
endif()

# External code should be compiled with these compiler options.
# Call this function before add_subdirectory([external project]), and then
# call restore_comiler_flags() after.
# TODO: This is duplicated from fplbase. We should find a central location
#       for it, or better yet, create proxy projects for external libraries.
function(pindrop_set_compiler_flags_for_external_libraries)
  # Save current compiler flags so that we can restore them in
  # pindrop_restore_compiler_flags()
  set(PINDROP_SAVED_CXX_FLAGS "${CMAKE_CXX_FLAGS}" PARENT_SCOPE)
  set(PINDROP_SAVED_C_FLAGS "${CMAKE_C_FLAGS}" PARENT_SCOPE)

  # Suppress all warnings.
  if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX OR
     CMAKE_COMPILER_IS_CLANGXX)
    set(CMAKE_CXX_FLAGS "-std=c++0x -w" PARENT_SCOPE)
    set(CMAKE_C_FLAGS "-w" PARENT_SCOPE)
  endif()
endfunction()

# Restore the compiler flags to the values they had before
# pindrop_set_compiler_flags_for_external_libraries() was called.
function(pindrop_restore_compiler_flags)
  set(CMAKE_CXX_FLAGS "${PINDROP_SAVED_CXX_FLAGS}" PARENT_SCOPE)
  set(CMAKE_C_FLAGS "${PINDROP_SAVED_C_FLAGS}" PARENT_SCOPE)
endfunction()

# Include SDL in this project while working around SDL requirement of out of
# tree builds.
set(SDL_SHARED OFF CACHE BOOL "Build a shared version of the library")
# (hack) We redefine install to stop SDL's CMakeLists.txt from referring to SDL2 lib
function(install)
endfunction(install)
# For some reason SDL defaults to these not being defined, and then defines their own,
# causing linker errors.
add_definitions(-DHAVE_LIBC)
add_definitions(-DHAVE_STDIO_H)
set(BIN_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(CMAKE_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/.)
if(APPLE)
  set(SDL_AUDIO_DRIVER_COREAUDIO 1)
endif(APPLE)
if(NOT TARGET SDL2-static AND NOT fpl_ios)
  add_subdirectory(${dependencies_sdl_dir} ${tmp_dir}/sdl)
endif()
set(CMAKE_BINARY_DIR ${BIN_DIR})

# Include libogg.
pindrop_set_compiler_flags_for_external_libraries()
if(NOT TARGET libogg AND NOT fpl_ios)
  add_subdirectory("${dependencies_libogg_cmake_dir}" ${tmp_dir}/libogg)
endif()

# Include libvorbis.
if(NOT TARGET libvorbis AND NOT fpl_ios)
  add_subdirectory("${dependencies_libvorbis_cmake_dir}" ${tmp_dir}/libvorbis)
endif()

# Include SDL Mixer.
set(SDL_MIXER_OGG_SUPPORT ON CACHE BOOL "Compile in Ogg Vorbis support")
if(NOT TARGET sdl_mixer AND NOT fpl_ios)
  add_subdirectory("${dependencies_sdl_mixer_cmake_dir}" ${tmp_dir}/sdl_mixer)
endif()
pindrop_restore_compiler_flags()

# Include MathFu in this project with test and benchmark builds disabled.
set(mathfu_build_benchmarks OFF CACHE BOOL "")
set(mathfu_build_tests OFF CACHE BOOL "")
add_subdirectory(${dependencies_mathfu_dir} ${tmp_dir}/mathfu)

# Include FlatBuffers in this project.
set(FLATBUFFERS_BUILD_TESTS OFF CACHE BOOL "")
if(NOT TARGET flatc)
  add_subdirectory("${dependencies_flatbuffers_dir}" ${tmp_dir}/flatbuffers)
endif()

# Include fplbase in this project.
set(pindrop_async_loading and FLATBUFFERS_BUILD_TESTS OFF CACHE BOOL "")
if(NOT TARGET flatc)
  add_subdirectory("${dependencies_fplbase_dir}" ${tmp_dir}/fplbase)
endif()

# Generate source files for all FlatBuffers schema files under the src
# directory.
set(PINDROP_FLATBUFFERS_GENERATED_INCLUDES_DIR
    ${CMAKE_SOURCE_DIR}/${CMAKE_FILES_DIRECTORY}/include/pindrop)
file(GLOB_RECURSE FLATBUFFERS_SCHEMAS ${CMAKE_CURRENT_SOURCE_DIR}/schemas/*.fbs)

# Generate rules to build the set of output files from the set of input
# schema files.
build_flatbuffers("${FLATBUFFERS_SCHEMAS}"
                  ""
                  pindrop_generated_includes
                  ""
                  ${PINDROP_FLATBUFFERS_GENERATED_INCLUDES_DIR}
                  ""
                  "")

if(pindrop_async_loading)
  set(FILE_LOADER_DIR src/asynchronous_loader)
else()
  set(FILE_LOADER_DIR src/synchronous_loader)
endif()

# AudioEngine source files.
set(pindrop_SRCS
    include/pindrop/pindrop.h
    src/audio_engine.cpp
    src/audio_engine_internal_state.h
    src/backend.cpp
    src/backend.h
    src/bus.cpp
    src/bus_internal_state.cpp
    src/bus_internal_state.h
    src/channel.cpp
    src/channel_internal_state.cpp
    src/channel_internal_state.h
    src/intrusive_list.h
    src/listener.cpp
    src/listener_internal_state.h
    src/ref_counter.cpp
    src/ref_counter.h
    src/sound.cpp
    src/sound.h
    src/sound_bank.cpp
    src/sound_bank.h
    src/sound_collection.cpp
    src/sound_collection.h
    ${FILE_LOADER_DIR}/file_loader.cpp
    ${FILE_LOADER_DIR}/file_loader.h)

# Includes for this project.
include_directories(src include)
if(pindrop_async_loading)
  include_directories(src/asynchronous_loader)
else()
  include_directories(src/synchronous_loader)
endif()
if(WIN32)
  include_directories(external/include/windows)
endif()
if (pindrop_async_loading)
  include_directories(${dependencies_fplbase_dir}/include)
endif()
include_directories(${dependencies_flatbuffers_dir}/include)
include_directories(${dependencies_webp_distr_dir}/include)
include_directories(${dependencies_webp_distr_dir}/src)

# SDL includes.
set(SDL2_SOURCE_DIR "${dependencies_sdl_dir}")
include_directories(${SDL2_BINARY_DIR}/include ${SDL2_SOURCE_DIR}/include)

# libogg includes.
include_directories(${dependencies_libogg_distr_dir}/include)

# libvorbis includes.
include_directories(${dependencies_libvorbis_distr_dir}/include)

# SDL Mixer includes.
include_directories(${dependencies_sdl_mixer_distr_dir})

# Compiler flags.
set(C_FLAGS_WARNINGS "")
if(MSVC)
  set(C_FLAGS_WARNINGS "/W4 /WX")
elseif(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX OR
       CMAKE_COMPILER_IS_CLANGXX)
  add_definitions(-g)
  set(CMAKE_CXX_FLAGS
    "${CMAKE_CXX_FLAGS} -std=c++0x -Wall -pedantic -Werror -Wextra -Wno-unused-function")
  # -Wno-unused-function is because of webp encode.h
endif()
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${C_FLAGS_WARNINGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${C_FLAGS_WARNINGS}")

if(NOT fpl_ios)
  set(SDL_LIBRARIES SDL2-static)
else()
  set(SDL_LIBRARIES "")
endif()
if (pindrop_async_loading)
  set(FPLBASE_LIBRARY fplbase)
else()
  set(FPL_LIBRARY "")
endif()
if(WIN32)
  add_definitions(-D_USE_MATH_DEFINES)
  set(SDL_LIBRARIES SDL2main ${SDL_LIBRARIES})
  link_directories("$ENV{DXSDK_DIR}/Lib/$ENV{PROCESSOR_ARCHITECTURE}")
endif()

# Executable target.
add_library(pindrop ${pindrop_SRCS})
mathfu_set_ios_attributes(pindrop)
mathfu_configure_flags(pindrop)
add_dependencies(pindrop pindrop_generated_includes)
if(fpl_ios)
  target_link_libraries(pindrop ${FPLBASE_LIBRARY})
else()
  target_link_libraries(pindrop
    ${SDL_LIBRARIES}
    ${FPLBASE_LIBRARY}
    sdl_mixer
    libvorbis
    libogg)
endif()

if(NOT fpl_ios AND pindrop_build_sample)
  add_subdirectory(samples)
endif()

# gtest seems to prefer the non-DLL runtime on Windows, which conflicts with
# everything else.
option(
  gtest_force_shared_crt
  "Use shared (DLL) run-time lib even when Google Test is built as static lib."
  ON)

# Tests.
if(pindrop_build_tests AND NOT fpl_ios)
  set(GUNIT_INCDIR "${dependencies_gtest_dir}/include")
  set(GTEST_LIBDIR "${dependencies_gtest_dir}")
  add_subdirectory("${dependencies_gtest_dir}" googletest)
  if (pindrop_async_loading)
    add_subdirectory("${dependencies_fplbase_dir}" ${tmp_dir}/fplbase)
  endif()
  include(${GTEST_LIBDIR}/cmake/internal_utils.cmake)
  config_compiler_and_linker()
  string(REPLACE "-W4" "-W3" cxx_default "${cxx_default}")
  string(REPLACE "-Wshadow" "" cxx_default "${cxx_default}")
  string(REPLACE "-Wextra" "" cxx_default "${cxx_default}")

  include_directories(${GUNIT_INCDIR}
                      ${CMAKE_CURRENT_SOURCE_DIR}
                      ${dependencies_flatbuffers_dir}/include)

  # Common libraries for tests.
  if(NOT MSVC)
    find_package(Threads)
  endif()

  set (SDL_LIBRARIES SDL2-static)
  if(WIN32)
    set(SDL_LIBRARIES SDL2main;${SDL_LIBRARIES})
  endif()

  function(test_executable name libs)
    cxx_executable_with_flags(${name}_test "${cxx_default}"
        "${libs}" ${CMAKE_CURRENT_SOURCE_DIR}/unit_tests/${name}_test.cpp
        ${ARGN})
    mathfu_configure_flags(${name}_test)
  endfunction()

  test_executable(audio_engine "gtest;pindrop;${SDL_LIBRARIES}")
  test_executable(intrusive_list "gtest")
endif()

