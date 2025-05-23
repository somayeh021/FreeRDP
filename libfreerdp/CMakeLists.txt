# FreeRDP: A Remote Desktop Protocol Implementation
# libfreerdp cmake build script
#
# Copyright 2012 Marc-Andre Moreau <marcandre.moreau@gmail.com>
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

set(MODULE_NAME "freerdp")
set(MODULE_PREFIX "FREERDP")

# CMake modules includes
include(FindCairo)

# Create imported targets for Intel IPP libraries

if(IPP_FOUND)
    foreach(ipp_lib ${IPP_LIBRARIES})
        add_library("${ipp_lib}_imported" STATIC IMPORTED)
        set_property(TARGET "${ipp_lib}_imported" PROPERTY IMPORTED_LOCATION "${IPP_LIBRARY_DIRS}/${ipp_lib}")
    endforeach()
endif()

set(LIBFREERDP_DIR ${CMAKE_CURRENT_SOURCE_DIR})
set(LIBFREERDP_SRCS "")
set(LIBFREERDP_LIBS "")
set(LIBFREERDP_INCLUDES "")
set(LIBFREERDP_DEFINITIONS "")

macro (freerdp_module_add)
        file (RELATIVE_PATH _relPath "${LIBFREERDP_DIR}" "${CMAKE_CURRENT_SOURCE_DIR}")
        foreach (_src ${ARGN})
                if (_relPath)
                        list (APPEND LIBFREERDP_SRCS "${_relPath}/${_src}")
                else()
                        list (APPEND LIBFREERDP_SRCS "${_src}")
                endif()
        endforeach()
        if (_relPath)
                set (LIBFREERDP_SRCS ${LIBFREERDP_SRCS} PARENT_SCOPE)
        endif()
endmacro()

macro (freerdp_include_directory_add)
        file (RELATIVE_PATH _relPath "${LIBFREERDP_DIR}" "${CMAKE_CURRENT_SOURCE_DIR}")
        foreach (_inc ${ARGN})
                if (IS_ABSOLUTE ${_inc})
                            list (APPEND LIBFREERDP_INCLUDES "${_inc}")
                else()
                    if (_relPath)
                            list (APPEND LIBFREERDP_INCLUDES "${_relPath}/${_inc}")
                    else()
                            list (APPEND LIBFREERDP_INCLUDES "${_inc}")
                    endif()
                endif()
        endforeach()
        if (_relPath)
                set (LIBFREERDP_INCLUDES ${LIBFREERDP_INCLUDES} PARENT_SCOPE)
        endif()
endmacro()

macro (freerdp_library_add_public)
    foreach (_lib ${ARGN})
        list (APPEND LIBFREERDP_PUB_LIBS "${_lib}")
    endforeach()
    set (LIBFREERDP_PUB_LIBS ${LIBFREERDP_PUB_LIBS} PARENT_SCOPE)
endmacro()

macro (freerdp_library_add)
    foreach (_lib ${ARGN})
        list (APPEND LIBFREERDP_LIBS "${_lib}")
    endforeach()
    set (LIBFREERDP_LIBS ${LIBFREERDP_LIBS} PARENT_SCOPE)
endmacro()

macro (freerdp_definition_add)
        foreach (_define ${ARGN})
                        list (APPEND LIBFREERDP_DEFINITIONS "${_define}")
        endforeach()
         set (LIBFREERDP_DEFINITIONS ${LIBFREERDP_DEFINITIONS} PARENT_SCOPE)
endmacro()

if (WITH_SWSCALE)
    find_package(SWScale REQUIRED)
endif(WITH_SWSCALE)
if (WITH_CAIRO)
    find_package(Cairo REQUIRED)
endif(WITH_CAIRO)

# Prefer SWScale over Cairo, both at the same time are not possible.
if (WITH_SWSCALE)
    include_directories(${SWScale_INCLUDE_DIR})
    freerdp_library_add(${SWScale_LIBRARY})
endif()
if (WITH_CAIRO)
    include_directories(${CAIRO_INCLUDE_DIR})
    freerdp_library_add(${CAIRO_LIBRARY})
endif()
if (NOT WITH_SWSCALE AND NOT WITH_CAIRO)
    message(WARNING "-DWITH_SWSCALE=OFF and -DWITH_CAIRO=OFF, compiling without image scaling support!")
endif()

set(${MODULE_PREFIX}_SUBMODULES
    emu
    utils
    common
    gdi
    cache
    crypto
    locale
    core)

foreach(${MODULE_PREFIX}_SUBMODULE ${${MODULE_PREFIX}_SUBMODULES})
    add_subdirectory(${${MODULE_PREFIX}_SUBMODULE})
endforeach()

if (NOT WITH_DSP_FFMPEG AND NOT WITH_FAAC)
    message(WARNING "Compiling without WITH_DSP_FFMPEG and WITH_FAAC, AAC encoder support disabled")
endif ()

## cmake source properties are only seen by targets in the same CMakeLists.txt
## therefore primitives and codecs need to be defined here

# codec
set(CODEC_SRCS
    codec/bulk.c
    codec/bulk.h
    codec/dsp.c
    codec/color.c
    codec/audio.c
    codec/planar.c
    codec/bitmap.c
    codec/interleaved.c
    codec/progressive.c
    codec/rfx_bitstream.h
    codec/rfx_constants.h
    codec/rfx_decode.c
    codec/rfx_decode.h
    codec/rfx_differential.h
    codec/rfx_dwt.c
    codec/rfx_dwt.h
    codec/rfx_encode.c
    codec/rfx_encode.h
    codec/rfx_quantization.c
    codec/rfx_quantization.h
    codec/rfx_rlgr.c
    codec/rfx_rlgr.h
    codec/rfx_types.h
    codec/rfx.c
    codec/region.c
    codec/nsc.c
    codec/nsc_encode.c
    codec/nsc_encode.h
    codec/nsc_types.h
    codec/ncrush.c
    codec/xcrush.c
    codec/mppc.c
    codec/zgfx.c
    codec/clear.c
    codec/jpeg.c
    codec/h264.c
    codec/yuv.c)

set(CODEC_SSE2_SRCS
    codec/rfx_sse2.c
    codec/rfx_sse2.h
    codec/nsc_sse2.c
    codec/nsc_sse2.h)

set(CODEC_NEON_SRCS
    codec/rfx_neon.c
    codec/rfx_neon.h)

if(WITH_SSE2)
    set(CODEC_SRCS ${CODEC_SRCS} ${CODEC_SSE2_SRCS})

    if(CMAKE_COMPILER_IS_GNUCC OR ${CMAKE_C_COMPILER_ID} STREQUAL "Clang")
        set_source_files_properties(${CODEC_SSE2_SRCS} PROPERTIES COMPILE_FLAGS "-msse2" )
    endif()

    if(MSVC)
        set_source_files_properties(${CODEC_SSE2_SRCS} PROPERTIES COMPILE_FLAGS "/arch:SSE2" )
    endif()
endif()

if (WITH_DSP_FFMPEG)
    set(CODEC_SRCS
        ${CODEC_SRCS}
        codec/dsp_ffmpeg.c
        codec/dsp_ffmpeg.h)
        freerdp_include_directory_add(${FFMPEG_INCLUDE_DIRS})
        freerdp_library_add(${FFMPEG_LIBRARIES})
endif (WITH_DSP_FFMPEG)

if (WITH_SOXR)
    freerdp_library_add(${SOXR_LIBRARIES})
    include_directories(${SOXR_INCLUDE_DIR})
endif(WITH_SOXR)

if(GSM_FOUND)
    freerdp_library_add(${GSM_LIBRARIES})
    include_directories(${GSM_INCLUDE_DIRS})
endif()

if(LAME_FOUND)
    freerdp_library_add(${LAME_LIBRARIES})
    include_directories(${LAME_INCLUDE_DIRS})
endif()

set(OPUS_DEFAULT OFF)
if (NOT WITH_DSP_FFMPEG)
    find_package(Opus)
    if (Opus_FOUND)
            set(OPUS_DEFAULT ${OPUS_FOUND})
    else()
        find_package(PkgConfig)
        if (PkgConfig_FOUND)
            pkg_check_modules(OPUS opus)
            set(OPUS_DEFAULT ${OPUS_FOUND})
        endif()
    endif()
endif()

option(WITH_OPUS "compile with opus codec support" ${OPUS_DEFAULT})
if (WITH_OPUS)
    find_package(Opus)
    if (Opus_FOUND)
        freerdp_library_add(Opus::opus)
    else()
        find_package(PkgConfig REQUIRED)
        pkg_check_modules(OPUS REQUIRED opus)
        if(OPUS_FOUND)
            freerdp_library_add(${OPUS_LIBRARIES})
            include_directories(${OPUS_INCLUDE_DIRS})
	    link_directories(${OPUS_LIBRARY_DIRS})
        endif()
    endif()
endif()

if(FAAD2_FOUND)
    freerdp_library_add(${FAAD2_LIBRARIES})
    include_directories(${FAAD2_INCLUDE_DIRS})
endif()

if(FAAC_FOUND)
    freerdp_library_add(${FAAC_LIBRARIES})
    include_directories(${FAAC_INCLUDE_DIRS})
endif()

if (WITH_AAD)
    if (NOT cJSON_FOUND)
        find_package(PkgConfig REQUIRED)
        pkg_check_modules(CJSON libcjson)
    endif()
    if (NOT CJSON_LIBRARIES OR NOT CJSON_INCLUDE_DIRS)
        find_path(CJSON_INCLUDE_DIRS
            NAMES cjson/cJSON.h
            REQUIRED
        )
        find_library(CJSON_LIBRARIES
            NAMES cjson
            REQUIRED
        )
    endif()

    freerdp_library_add(${CJSON_LIBRARIES})
    include_directories(${CJSON_INCLUDE_DIRS})
endif()

if(WITH_NEON)
    check_symbol_exists("_M_AMD64"     ""  MSVC_ARM64)
    check_symbol_exists("__aarch64__"  ""  ARCH_ARM64)

    if (NOT MSVC_ARM64 AND NOT ARCH_ARM64)
        set_source_files_properties(${CODEC_NEON_SRCS} PROPERTIES COMPILE_FLAGS "-mfpu=neon" )
    endif()

    set(CODEC_SRCS ${CODEC_SRCS} ${CODEC_NEON_SRCS})
endif()

if(WITH_OPENH264)
    set(CODEC_SRCS ${CODEC_SRCS} codec/h264_openh264.c)
    freerdp_include_directory_add(${OPENH264_INCLUDE_DIR})
    if (NOT WITH_OPENH264_LOADING)
        freerdp_library_add(${OPENH264_LIBRARIES})
    endif (NOT WITH_OPENH264_LOADING)
endif()

if(WITH_VIDEO_FFMPEG)
    set(CODEC_SRCS ${CODEC_SRCS} codec/h264_ffmpeg.c)
    freerdp_include_directory_add(${FFMPEG_INCLUDE_DIRS})
    freerdp_library_add(${FFMPEG_LIBRARIES})
endif()

if(WIN32 AND WITH_MEDIA_FOUNDATION)
    set(CODEC_SRCS ${CODEC_SRCS} codec/h264_mf.c)
endif()

if(ANDROID AND WITH_MEDIACODEC)
    list(APPEND CODEC_SRCS codec/h264_mediacodec.c)

    find_library(MEDIACODEC mediandk REQUIRED)
    freerdp_library_add(${MEDIACODEC})
endif()

freerdp_module_add(${CODEC_SRCS})

if(BUILD_TESTING)
    add_subdirectory(codec/test)
endif()

# /codec

# primitives

set(PRIMITIVES_SRCS
    primitives/prim_add.c
    primitives/prim_andor.c
    primitives/prim_alphaComp.c
    primitives/prim_colors.c
    primitives/prim_copy.c
    primitives/prim_set.c
    primitives/prim_shift.c
    primitives/prim_sign.c
    primitives/prim_YUV.c
    primitives/prim_YCoCg.c
    primitives/primitives.c
    primitives/prim_internal.h)

if (WITH_SSE2 OR WITH_NEON)
    set(PRIMITIVES_SSE2_SRCS
        primitives/prim_colors_opt.c
        primitives/prim_set_opt.c)

    set(PRIMITIVES_SSE3_SRCS
        primitives/prim_add_opt.c
        primitives/prim_alphaComp_opt.c
        primitives/prim_andor_opt.c
        primitives/prim_shift_opt.c)

    set(PRIMITIVES_SSSE3_SRCS
        primitives/prim_sign_opt.c
        primitives/prim_YCoCg_opt.c)

    if (WITH_SSE2)
        set(PRIMITIVES_SSSE3_SRCS ${PRIMITIVES_SSSE3_SRCS}
            primitives/prim_YUV_ssse3.c)
    endif()

    if (WITH_NEON)
        set(PRIMITIVES_SSSE3_SRCS ${PRIMITIVES_SSSE3_SRCS}
            primitives/prim_YUV_neon.c)
    endif()
endif()

if (WITH_OPENCL)
    set(PRIMITIVES_OPENCL_SRCS primitives/prim_YUV_opencl.c)

        freerdp_include_directory_add(${OpenCL_INCLUDE_DIRS})
        freerdp_library_add(OpenCL::OpenCL)

endif()

set(PRIMITIVES_OPT_SRCS
    ${PRIMITIVES_SSE2_SRCS}
    ${PRIMITIVES_SSE3_SRCS}
    ${PRIMITIVES_SSSE3_SRCS}
    ${PRIMITIVES_OPENCL_SRCS})

### IPP Variable debugging
if(WITH_IPP)
    if(CMAKE_COMPILER_IS_GNUCC OR ${CMAKE_C_COMPILER_ID} STREQUAL "Clang")
        foreach(INCLDIR ${IPP_INCLUDE_DIRS})
            set(OPTIMIZATION "${OPTIMIZATION} -I${INCLDIR}")
        endforeach(INCLDIR)
    endif()
endif()

if(WITH_SSE2)
    if(CMAKE_COMPILER_IS_GNUCC OR ${CMAKE_C_COMPILER_ID} STREQUAL "Clang")
        set_source_files_properties(${PRIMITIVES_SSE2_SRCS}
            PROPERTIES COMPILE_FLAGS "${OPTIMIZATION} -msse2")
        set_source_files_properties(${PRIMITIVES_SSE3_SRCS}
            PROPERTIES COMPILE_FLAGS "${OPTIMIZATION} -msse3")
        set_source_files_properties(${PRIMITIVES_SSSE3_SRCS}
            PROPERTIES COMPILE_FLAGS "${OPTIMIZATION} -mssse3")
    endif()

    if(MSVC)
        set_source_files_properties(${PRIMITIVES_OPT_SRCS}
            PROPERTIES COMPILE_FLAGS "${OPTIMIZATION} /arch:SSE2")
    endif()
elseif(WITH_NEON)
    if(CMAKE_COMPILER_IS_GNUCC OR ${CMAKE_C_COMPILER_ID} STREQUAL "Clang")
        if (NOT MSVC_ARM64 AND NOT ARCH_ARM64)
            set_source_files_properties(${PRIMITIVES_OPT_SRCS}
                PROPERTIES COMPILE_FLAGS "${OPTIMIZATION} -mfpu=neon")
        endif()
    endif()
    # TODO: Add MSVC equivalent
endif()

set(PRIMITIVES_SRCS ${PRIMITIVES_SRCS} ${PRIMITIVES_OPT_SRCS})

freerdp_module_add(${PRIMITIVES_SRCS})

if(IPP_FOUND)
    freerdp_include_directory_add(${IPP_INCLUDE_DIRS})
        foreach(ipp_lib ${IPP_LIBRARIES})
        freerdp_library_add("${ipp_lib}_imported")
    endforeach()
endif()

if(BUILD_TESTING AND NOT WIN32 AND NOT APPLE)
    add_subdirectory(primitives/test)
endif()


# /primitives

list(APPEND LIBFREERDP_PUB_LIBS winpr)

list(REMOVE_DUPLICATES LIBFREERDP_DEFINITIONS)
list(REMOVE_DUPLICATES LIBFREERDP_LIBS)
list(REMOVE_DUPLICATES LIBFREERDP_PUB_LIBS)
list(REMOVE_DUPLICATES LIBFREERDP_INCLUDES)
include_directories(${LIBFREERDP_INCLUDES})

# On windows create dll version information.
# Vendor, product and year are already set in top level CMakeLists.txt
if (WIN32)
  set (RC_VERSION_MAJOR ${FREERDP_VERSION_MAJOR})
  set (RC_VERSION_MINOR ${FREERDP_VERSION_MINOR})
  set (RC_VERSION_BUILD ${FREERDP_VERSION_REVISION})
  set (RC_VERSION_FILE "${CMAKE_SHARED_LIBRARY_PREFIX}${MODULE_NAME}${FREERDP_API_VERSION}${CMAKE_SHARED_LIBRARY_SUFFIX}" )

  configure_file(
    ${PROJECT_SOURCE_DIR}/cmake/WindowsDLLVersion.rc.in
    ${CMAKE_CURRENT_BINARY_DIR}/version.rc
    @ONLY)

  set (LIBFREERDP_SRCS ${LIBFREERDP_SRCS} ${CMAKE_CURRENT_BINARY_DIR}/version.rc)
endif()

add_library(${MODULE_NAME} ${LIBFREERDP_SRCS})
add_definitions(${LIBFREERDP_DEFINITIONS})

set_target_properties(${MODULE_NAME} PROPERTIES LINKER_LANGUAGE C)
set_target_properties(${MODULE_NAME} PROPERTIES OUTPUT_NAME ${MODULE_NAME}${FREERDP_VERSION_MAJOR})

if (WITH_LIBRARY_VERSIONING)
    set_target_properties(${MODULE_NAME} PROPERTIES VERSION ${FREERDP_VERSION} SOVERSION ${FREERDP_API_VERSION})
endif()

target_include_directories(${MODULE_NAME} INTERFACE $<INSTALL_INTERFACE:include>)
target_link_libraries(${MODULE_NAME} PRIVATE ${LIBFREERDP_LIBS})
target_link_libraries(${MODULE_NAME} PUBLIC ${LIBFREERDP_PUB_LIBS})
install(TARGETS ${MODULE_NAME} COMPONENT libraries EXPORT FreeRDPTargets
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

if (WITH_DEBUG_SYMBOLS AND MSVC AND BUILD_SHARED_LIBS)
    get_target_property(OUTPUT_FILENAME ${MODULE_NAME} OUTPUT_NAME)
    install(FILES ${CMAKE_PDB_BINARY_DIR}/${OUTPUT_FILENAME}.pdb DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT symbols)
endif()

set_property(TARGET ${MODULE_NAME} PROPERTY FOLDER "FreeRDP/libfreerdp")

include(pkg-config-install-prefix)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/freerdp.pc.in ${CMAKE_CURRENT_BINARY_DIR}/freerdp${FREERDP_VERSION_MAJOR}.pc @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/freerdp${FREERDP_VERSION_MAJOR}.pc DESTINATION ${PKG_CONFIG_PC_INSTALL_DIR})

## cmake project
export(PACKAGE freerdp)

SetFreeRDPCMakeInstallDir(FREERDP_CMAKE_INSTALL_DIR "FreeRDP${FREERDP_VERSION_MAJOR}")

configure_package_config_file(FreeRDPConfig.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/FreeRDPConfig.cmake
INSTALL_DESTINATION ${FREERDP_CMAKE_INSTALL_DIR}
PATH_VARS FREERDP_INCLUDE_DIR FREERDP_PLUGIN_PATH)

write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/FreeRDPConfigVersion.cmake
VERSION ${FREERDP_VERSION} COMPATIBILITY SameMajorVersion)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/FreeRDPConfig.cmake ${CMAKE_CURRENT_BINARY_DIR}/FreeRDPConfigVersion.cmake
DESTINATION ${FREERDP_CMAKE_INSTALL_DIR})

install(EXPORT FreeRDPTargets DESTINATION ${FREERDP_CMAKE_INSTALL_DIR})
