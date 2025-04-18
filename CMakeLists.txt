# FreeRDP: A Remote Desktop Protocol Implementation
# FreeRDP cmake build script
#
# Copyright 2011 O.S. Systems Software Ltda.
# Copyright 2011 Otavio Salvador <otavio@ossystems.com.br>
# Copyright 2011 Marc-Andre Moreau <marcandre.moreau@gmail.com>
# Copyright 2012 HP Development Company, LLC
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

cmake_minimum_required(VERSION 3.13)

if(POLICY CMP0091)
	cmake_policy(SET CMP0091 NEW)
endif()
project(FreeRDP
	LANGUAGES C
)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS ON)

add_custom_target(fuzzers
	COMMENT "Build fuzzers"
)

if(NOT DEFINED VENDOR)
	set(VENDOR "FreeRDP" CACHE STRING "FreeRDP package vendor")
endif()

if(NOT DEFINED PRODUCT)
	set(PRODUCT "FreeRDP" CACHE STRING "FreeRDP package name")
endif()

if(NOT DEFINED FREERDP_VENDOR)
	set(FREERDP_VENDOR 1)
endif()

if (NOT WIN32 AND NOT ANDROID)
	if (APPLE)
		set(OPT_DEFAULT_VAL OFF)
	else()
		set(OPT_DEFAULT_VAL ON)
	endif()
	option(WITH_X11 "build X11 client/server" ${OPT_DEFAULT_VAL})
endif()

# Include our extra modules
list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/)
include(CommonConfigOptions)

if((CMAKE_SYSTEM_NAME MATCHES "WindowsStore") AND (CMAKE_SYSTEM_VERSION MATCHES "10.0"))
	set(UWP 1)
	add_definitions("-D_UWP")
	set(CMAKE_WINDOWS_VERSION "WIN10")
endif()

# Check for cmake compatibility (enable/disable features)
include(CheckCmakeCompat)

# Include cmake modules
if(WITH_CLANG_FORMAT)
	include(ClangFormat)
endif()

include(CompilerFlags)
include(DetectBSD)
include(CheckIncludeFiles)
include(CheckLibraryExists)
include(CheckSymbolExists)
include(CheckStructHasMember)
include(TestBigEndian)

include(FindFeature)
include(ShowCMakeVars)
include(ConfigOptions)
include(FeatureSummary)
include(CheckCCompilerFlag)
include(CMakePackageConfigHelpers)
include(InstallFreeRDPMan)
include(GetGitRevisionDescription)
include(SetFreeRDPCMakeInstallDir)

# Soname versioning
set(BUILD_NUMBER 0)
if ($ENV{BUILD_NUMBER})
	set(BUILD_NUMBER $ENV{BUILD_NUMBER})
endif()

set(VERSION_REGEX "^(.*)([0-9]+)\\.([0-9]+)\\.([0-9]+)-?(.*)")
set(RAW_VERSION_STRING "3.3.1-dev0")
if(EXISTS "${PROJECT_SOURCE_DIR}/.source_tag")
	file(READ ${PROJECT_SOURCE_DIR}/.source_tag RAW_VERSION_STRING)
elseif(USE_VERSION_FROM_GIT_TAG)
	git_get_exact_tag(_GIT_TAG --tags --always)
	if (NOT ${_GIT_TAG} STREQUAL "n/a")
		string(REGEX MATCH ${VERSION_REGEX} FOUND_TAG "${_GIT_TAG}")
		if (FOUND_TAG)
			set(RAW_VERSION_STRING ${_GIT_TAG})
		endif()
	endif()
endif()
string(STRIP ${RAW_VERSION_STRING} RAW_VERSION_STRING)

string(REGEX REPLACE "${VERSION_REGEX}" "\\2" FREERDP_VERSION_MAJOR "${RAW_VERSION_STRING}")
string(REGEX REPLACE "${VERSION_REGEX}" "\\3" FREERDP_VERSION_MINOR "${RAW_VERSION_STRING}")
string(REGEX REPLACE "${VERSION_REGEX}" "\\4" FREERDP_VERSION_REVISION "${RAW_VERSION_STRING}")
string(REGEX REPLACE "${VERSION_REGEX}" "\\5" FREERDP_VERSION_SUFFIX "${RAW_VERSION_STRING}")

set(FREERDP_API_VERSION "${FREERDP_VERSION_MAJOR}")
set(FREERDP_VERSION "${FREERDP_VERSION_MAJOR}.${FREERDP_VERSION_MINOR}.${FREERDP_VERSION_REVISION}")
if (FREERDP_VERSION_SUFFIX)
	set(FREERDP_VERSION_FULL "${FREERDP_VERSION}-${FREERDP_VERSION_SUFFIX}")
else()
	set(FREERDP_VERSION_FULL "${FREERDP_VERSION}")
endif()
message("FREERDP_VERSION=${FREERDP_VERSION_FULL}")

if(EXISTS "${PROJECT_SOURCE_DIR}/.source_version" )
	file(READ ${PROJECT_SOURCE_DIR}/.source_version GIT_REVISION)

  string(STRIP ${GIT_REVISION} GIT_REVISION)
elseif(USE_VERSION_FROM_GIT_TAG)
	git_get_exact_tag(GIT_REVISION --tags --always)

	if (${GIT_REVISION} STREQUAL "n/a")
		git_rev_parse (GIT_REVISION --short)
	endif()
endif()

if (NOT GIT_REVISION)
	set(GIT_REVISION ${FREERDP_VERSION})
endif()

message(STATUS "Git Revision ${GIT_REVISION}")

# Make the detected version available as default version for all subprojects
set(FREERDP_DEFAULT_PROJECT_VERSION ${FREERDP_VERSION} CACHE STRING INTERNAL)

set(FREERDP_MAJOR_DIR "freerdp${FREERDP_VERSION_MAJOR}")
set(FREERDP_INCLUDE_DIR "include/${FREERDP_MAJOR_DIR}/")

option(WITH_SMARTCARD_EMULATE "Emulate smartcards instead of redirecting readers" ON)
if (WITH_SMARTCARD_EMULATE)
	add_definitions(-DWITH_SMARTCARD_EMULATE)
	find_package(ZLIB REQUIRED)
endif()

option(WITH_FREERDP_DEPRECATED "Build FreeRDP deprecated symbols" OFF)
if (WITH_FREERDP_DEPRECATED)
	add_definitions(-DWITH_FREERDP_DEPRECATED)
endif()

option(WITH_FREERDP_DEPRECATED_COMMANDLINE "Build FreeRDP deprecated command line options" OFF)
if (WITH_FREERDP_DEPRECATED_COMMANDLINE)
	add_definitions(-DWITH_FREERDP_DEPRECATED_COMMANDLINE)
endif()

# Make paths absolute
if (CMAKE_INSTALL_PREFIX)
	get_filename_component(CMAKE_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}" ABSOLUTE)
endif()
if (FREERDP_EXTERNAL_PATH)
	get_filename_component (FREERDP_EXTERNAL_PATH "${FREERDP_EXTERNAL_PATH}" ABSOLUTE)
endif()

# Allow to search the host machine for git/ccache
if(CMAKE_CROSSCOMPILING)
	SET(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM BOTH)
endif(CMAKE_CROSSCOMPILING)

find_program(CCACHE ccache)
if(CCACHE AND WITH_CCACHE)
	if(NOT DEFINED CMAKE_C_COMPILER_LAUNCHER)
		SET(CMAKE_C_COMPILER_LAUNCHER ${CCACHE})
	endif(NOT DEFINED CMAKE_C_COMPILER_LAUNCHER)
endif(CCACHE AND WITH_CCACHE)

if(CMAKE_CROSSCOMPILING)
	SET (CMAKE_FIND_ROOT_PATH_MODE_PROGRAM ONLY)
endif(CMAKE_CROSSCOMPILING)
# /Allow to search the host machine for git/ccache

# Turn on solution folders (2.8.4+)
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

option(EXPORT_ALL_SYMBOLS "Export all symbols form library" OFF)

if(BUILD_TESTING)
	set(EXPORT_ALL_SYMBOLS TRUE)
	set(CTEST_OUTPUT_ON_FAILURE TRUE)
	add_definitions(-DBUILD_TESTING)
elseif(NOT DEFINED EXPORT_ALL_SYMBOLS)
	set(EXPORT_ALL_SYMBOLS FALSE)
endif()

if (EXPORT_ALL_SYMBOLS)
	#	set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
	add_definitions(-DEXPORT_ALL_SYMBOLS)
endif(EXPORT_ALL_SYMBOLS)

if(FREEBSD)
	find_path(EPOLLSHIM_INCLUDE_DIR NAMES sys/epoll.h sys/timerfd.h HINTS /usr/local/include/libepoll-shim)
	find_library(EPOLLSHIM_LIBS NAMES epoll-shim libepoll-shim HINTS /usr/local/lib)
endif()

# Enable 64bit file support on linux and FreeBSD.
if("${CMAKE_SYSTEM_NAME}" MATCHES "Linux" OR FREEBSD)
	add_definitions("-D_FILE_OFFSET_BITS=64")
endif()

# Use Standard conforming getpwnam_r() on Solaris.
if("${CMAKE_SYSTEM_NAME}" MATCHES "SunOS")
	add_definitions("-D_POSIX_PTHREAD_SEMANTICS")
endif()

# Compiler-specific flags
if(CMAKE_COMPILER_IS_GNUCC)
	if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64" OR CMAKE_SYSTEM_PROCESSOR MATCHES "i686")
		CHECK_SYMBOL_EXISTS(__x86_64__ "" IS_X86_64)
		if(IS_X86_64)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")
		else()
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=i686")
		endif()
	endif()

	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall")

	if(NOT EXPORT_ALL_SYMBOLS)
		message(STATUS "GCC default symbol visibility: hidden")
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fvisibility=hidden")
	endif()
	CHECK_C_COMPILER_FLAG (-Wimplicit-function-declaration Wimplicit-function-declaration)
	if(Wimplicit-function-declaration)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wimplicit-function-declaration")
	endif()

	if (NOT OPENBSD)
		CHECK_C_COMPILER_FLAG (-Wredundant-decls Wredundant-decls)
		if(Wredundant-decls)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wredundant-decls")
		endif()
	endif()
	if(CMAKE_BUILD_TYPE STREQUAL "Release")
		add_definitions(-DNDEBUG)
	else()
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g")
	endif()
endif()

# When building with Unix Makefiles and doing any release builds
# try to set __FILE__ to relative paths via a make specific macro
if (CMAKE_GENERATOR MATCHES "Unix Makefile*")
	if(CMAKE_BUILD_TYPE STREQUAL "Release" OR CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
		string(TOUPPER ${CMAKE_BUILD_TYPE} UPPER_BUILD_TYPE)
		CHECK_C_COMPILER_FLAG (-Wno-builtin-macro-redefined Wno-builtin-macro-redefined)
		if(Wno-builtin-macro-redefined)
			set(CMAKE_C_FLAGS_${UPPER_BUILD_TYPE} "${CMAKE_C_FLAGS_${UPPER_BUILD_TYPE}} -Wno-builtin-macro-redefined -D__FILE__='\"$(subst ${PROJECT_BINARY_DIR}/,,$(subst ${PROJECT_SOURCE_DIR}/,,$(abspath $<)))\"'")
		endif()
	endif()
endif()

if(${CMAKE_C_COMPILER_ID} STREQUAL "Clang")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-c11-extensions -Wno-gnu")
endif()

set(THREAD_PREFER_PTHREAD_FLAG TRUE)

if(NOT IOS)
	find_package(Threads REQUIRED)
endif()

# Enable address sanitizer, where supported and when required
if(${CMAKE_C_COMPILER_ID} STREQUAL "Clang" OR CMAKE_COMPILER_IS_GNUCC)
	CHECK_C_COMPILER_FLAG ("-fno-omit-frame-pointer" fno-omit-frame-pointer)

	if (fno-omit-frame-pointer)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fno-omit-frame-pointer")
	endif()

	set(CMAKE_REQUIRED_LINK_OPTIONS_SAVED ${CMAKE_REQUIRED_LINK_OPTIONS})
	file(WRITE ${PROJECT_BINARY_DIR}/foo.txt "")
	if(WITH_SANITIZE_ADDRESS)
		list(APPEND CMAKE_REQUIRED_LINK_OPTIONS "-fsanitize=address")
		CHECK_C_COMPILER_FLAG ("-fsanitize=address" fsanitize-address)
		CHECK_C_COMPILER_FLAG ("-fsanitize-blacklist=${PROJECT_BINARY_DIR}/foo.txt" fsanitize-blacklist)
		CHECK_C_COMPILER_FLAG ("-fsanitize-address-use-after-scope" fsanitize-address-use-after-scope)

		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address")
		set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fsanitize=address")
		set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address")

		if(fsanitize-blacklist)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize-blacklist=${PROJECT_SOURCE_DIR}/scripts/blacklist-address-sanitizer.txt")
		endif(fsanitize-blacklist)

		if(fsanitize-address-use-after-scope)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize-address-use-after-scope")
		endif(fsanitize-address-use-after-scope)
	elseif(WITH_SANITIZE_MEMORY)
		list(APPEND CMAKE_REQUIRED_LINK_OPTIONS "-fsanitize=memory")
		CHECK_C_COMPILER_FLAG ("-fsanitize=memory" fsanitize-memory)
		CHECK_C_COMPILER_FLAG ("-fsanitize-blacklist=${PROJECT_BINARY_DIR}/foo.txt" fsanitize-blacklist)
		CHECK_C_COMPILER_FLAG ("-fsanitize-memory-use-after-dtor" fsanitize-memory-use-after-dtor)
		CHECK_C_COMPILER_FLAG ("-fsanitize-memory-track-origins" fsanitize-memory-track-origins)

		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=memory")
		set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fsanitize=memory")
		set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=memory")

		if(fsanitize-blacklist)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize-blacklist=${PROJECT_SOURCE_DIR}/scripts/blacklist-memory-sanitizer.txt")
		endif(fsanitize-blacklist)

		if (fsanitize-memory-use-after-dtor)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize-memory-use-after-dtor")
		endif(fsanitize-memory-use-after-dtor)

		if (fsanitize-memory-track-origins)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize-memory-track-origins")
		endif(fsanitize-memory-track-origins)
	elseif(WITH_SANITIZE_THREAD)
		list(APPEND CMAKE_REQUIRED_LINK_OPTIONS "-fsanitize=thread")
		CHECK_C_COMPILER_FLAG ("-fsanitize=thread" fsanitize-thread)
		CHECK_C_COMPILER_FLAG ("-fsanitize-blacklist=${PROJECT_BINARY_DIR}/foo.txt" fsanitize-blacklist)

		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=thread")
		if(fsanitize-blacklist)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize-blacklist=${PROJECT_SOURCE_DIR}/scripts/blacklist-thread-sanitizer.txt")
		endif(fsanitize-blacklist)
		set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fsanitize=thread")
		set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=thread")
	endif()

	file(REMOVE ${PROJECT_BINARY_DIR}/foo.txt)
	set(CMAKE_REQUIRED_LINK_OPTIONS ${CMAKE_REQUIRED_LINK_OPTIONS_SAVED})

	if (WITH_NO_UNDEFINED)
		CHECK_C_COMPILER_FLAG (-Wl,--no-undefined no-undefined)

		if(no-undefined)
			SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--no-undefined" )
			SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--no-undefined" )
		endif()
	endif()
endif()

if(MSVC)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /Gd")

	set(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR})
	set(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR})

	if(CMAKE_BUILD_TYPE STREQUAL "Release")
	else()
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /Zi")
	endif()

	add_definitions(-D_CRT_NONSTDC_NO_DEPRECATE)
endif()

if(ANDROID)
	# workaround for https://github.com/android-ndk/ndk/issues/243
	string(REPLACE "-g " "" CMAKE_C_FLAGS ${CMAKE_C_FLAGS})
	set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -g")
endif()

if(WIN32)
	add_definitions(-DUNICODE -D_UNICODE)
	add_definitions(-D_CRT_SECURE_NO_WARNINGS)
	add_definitions(-DWIN32_LEAN_AND_MEAN)
	add_definitions(-D_WINSOCK_DEPRECATED_NO_WARNINGS)

	set(CMAKE_DL_LIBS "")
	set(CMAKE_USE_RELATIVE_PATH ON)
	if (${CMAKE_GENERATOR} MATCHES "NMake Makefile*" OR ${CMAKE_GENERATOR} MATCHES "Ninja*" OR ${CMAKE_GENERATOR} MATCHES "Unix Makefiles")
		set(CMAKE_PDB_BINARY_DIR ${PROJECT_BINARY_DIR})
	elseif (${CMAKE_GENERATOR} MATCHES "Visual Studio*")
		set(CMAKE_PDB_BINARY_DIR "${PROJECT_BINARY_DIR}/\${CMAKE_INSTALL_CONFIG_NAME}")
	else()
		message(FATAL_ERROR "Unknown generator ${CMAKE_GENERATOR}")
	endif()

	string(TIMESTAMP RC_VERSION_YEAR "%Y")

	if(NOT DEFINED CMAKE_WINDOWS_VERSION)
		set(CMAKE_WINDOWS_VERSION "WIN7")
	endif()

	if(CMAKE_WINDOWS_VERSION STREQUAL "WINXP")
		add_definitions(-DWINVER=0x0501 -D_WIN32_WINNT=0x0501)
	elseif(CMAKE_WINDOWS_VERSION STREQUAL "WIN7")
		add_definitions(-DWINVER=0x0601 -D_WIN32_WINNT=0x0601)
	elseif(CMAKE_WINDOWS_VERSION STREQUAL "WIN8")
		add_definitions(-DWINVER=0x0602 -D_WIN32_WINNT=0x0602)
	elseif(CMAKE_WINDOWS_VERSION STREQUAL "WIN10")
		add_definitions(-DWINVER=0x0A00 -D_WIN32_WINNT=0x0A00)
	endif()

	# Set product and vendor for dll and exe version information.
	set(RC_VERSION_VENDOR ${VENDOR})
	set(RC_VERSION_PRODUCT ${PRODUCT})
	set(RC_VERSION_PATCH ${BUILD_NUMBER})
		set(RC_VERSION_DESCRIPTION "${FREERDP_VERSION_FULL} ${GIT_REVISION} ${CMAKE_WINDOWS_VERSION} ${CMAKE_SYSTEM_PROCESSOR}")

	if (FREERDP_EXTERNAL_SSL_PATH)
		set(OPENSSL_ROOT_DIR ${FREERDP_EXTERNAL_SSL_PATH})
	endif()
endif()

add_definitions(-DFREERDP_EXPORTS)

# Mac OS X
if(APPLE)
	if(IOS)
		if (NOT FREERDP_IOS_EXTERNAL_SSL_PATH)
			message(STATUS "FREERDP_IOS_EXTERNAL_SSL_PATH not set! Required if openssl is not found in the iOS SDK (which usually isn't")
		endif()
		set(CMAKE_FIND_ROOT_PATH ${CMAKE_FIND_ROOT_PATH} ${FREERDP_IOS_EXTERNAL_SSL_PATH})
		set_property(GLOBAL PROPERTY XCODE_ATTRIBUTE_SKIP_INSTALL YES)
	endif(IOS)

# Temporarily disabled, causes the cmake script to be reexecuted, causing the compilation to fail.
# Workaround: specify the parameter in the command-line
#	if(WITH_CLANG)
#		set(CMAKE_C_COMPILER "clang")
#	endif()

	if (WITH_VERBOSE)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -v")
	endif()
endif(APPLE)

# Android
if(ANDROID)
	set_property( GLOBAL PROPERTY FIND_LIBRARY_USE_LIB64_PATHS ${ANDROID_LIBRARY_USE_LIB64_PATHS} )

	if (${ANDROID_ABI} STREQUAL "armeabi")
		set (WITH_NEON OFF)
	endif()

	if(ANDROID_ABI STREQUAL arm64-v8a)
		# https://github.com/android/ndk/issues/910
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfloat-abi=softfp")
	endif()

	if("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
		add_definitions(-DNDK_DEBUG=1)

		# NOTE: Manually add -gdwarf-3, as newer toolchains default to -gdwarf-4,
		#       which is not supported by the gdbserver binary shipped with
		#       the android NDK (tested with r9b)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${CMAKE_C_FLAGS_DEBUG} -gdwarf-3")
	endif()
	set(CMAKE_C_LINK_FLAGS "${CMAKE_C_LINK_FLAGS} -llog")

	# CMAKE_PREFIX_PATH detection is broken in most Android toolchain files
	# Append it to CMAKE_FIND_ROOT_PATH and avoid potential duplicates
	list(APPEND CMAKE_FIND_ROOT_PATH ${CMAKE_PREFIX_PATH})
	list(REMOVE_DUPLICATES CMAKE_FIND_ROOT_PATH)

	if (NOT FREERDP_EXTERNAL_PATH)
		if (IS_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/external/")
			set (FREERDP_EXTERNAL_PATH "${CMAKE_CURRENT_SOURCE_DIR}/external/")
		else()
			message(STATUS "FREERDP_EXTERNAL_PATH not set!")
		endif()
	endif()

	list (APPEND CMAKE_INCLUDE_PATH ${FREERDP_EXTERNAL_PATH}/${ANDROID_ABI}/include)
	list (APPEND CMAKE_LIBRARY_PATH ${FREERDP_EXTERNAL_PATH}/${ANDROID_ABI}/ )
	set( CMAKE_FIND_ROOT_PATH_MODE_LIBRARY BOTH )
	set( CMAKE_FIND_ROOT_PATH_MODE_INCLUDE BOTH )

	if (WITH_GPROF)
		CONFIGURE_FILE(${PROJECT_SOURCE_DIR}/scripts/gprof_generate.sh.cmake
			${PROJECT_BINARY_DIR}/scripts/gprof_generate.sh @ONLY)
	endif(WITH_GPROF)
endif()

if(WITH_VALGRIND_MEMCHECK)
	check_include_files(valgrind/memcheck.h FREERDP_HAVE_VALGRIND_MEMCHECK_H)
else()
	unset(FREERDP_HAVE_VALGRIND_MEMCHECK_H CACHE)
endif()

if(UNIX OR CYGWIN)
	set(WAYLAND_FEATURE_TYPE "RECOMMENDED")
else()
	set(WAYLAND_FEATURE_TYPE "DISABLED")
endif()

if(WITH_PCSC_WINPR)
	find_package(PCSCWinPR)
endif()

set(WAYLAND_FEATURE_PURPOSE "Wayland")
set(WAYLAND_FEATURE_DESCRIPTION "Wayland client")

set(OPENSSL_FEATURE_TYPE "REQUIRED")
set(OPENSSL_FEATURE_PURPOSE "cryptography")
set(OPENSSL_FEATURE_DESCRIPTION "encryption, certificate validation, hashing functions")

set(MBEDTLS_FEATURE_TYPE "OPTIONAL")
set(MBEDTLS_FEATURE_PURPOSE "cryptography")
set(MBEDTLS_FEATURE_DESCRIPTION "encryption, certificate validation, hashing functions")

set(PCSC_FEATURE_TYPE "RECOMMENDED")
set(PCSC_FEATURE_PURPOSE "smart card")
set(PCSC_FEATURE_DESCRIPTION "smart card device redirection")

set(FFMPEG_FEATURE_TYPE "RECOMMENDED")
set(FFMPEG_FEATURE_PURPOSE "multimedia")
set(FFMPEG_FEATURE_DESCRIPTION "multimedia redirection, audio and video playback")

set(VAAPI_FEATURE_TYPE "OPTIONAL")
set(VAAPI_FEATURE_PURPOSE "multimedia")
set(VAAPI_FEATURE_DESCRIPTION "VA-API hardware acceleration for video playback")

set(IPP_FEATURE_TYPE "OPTIONAL")
set(IPP_FEATURE_PURPOSE "performance")
set(IPP_FEATURE_DESCRIPTION "Intel Integrated Performance Primitives library")

set(OPENH264_FEATURE_TYPE "OPTIONAL")
set(OPENH264_FEATURE_PURPOSE "codec")
set(OPENH264_FEATURE_DESCRIPTION "use OpenH264 library")

set(OPENCL_FEATURE_TYPE "OPTIONAL")
set(OPENCL_FEATURE_PURPOSE "codec")
set(OPENCL_FEATURE_DESCRIPTION "use OpenCL library")

set(GSM_FEATURE_TYPE "OPTIONAL")
set(GSM_FEATURE_PURPOSE "codec")
set(GSM_FEATURE_DESCRIPTION "GSM audio codec library")

set(LAME_FEATURE_TYPE "OPTIONAL")
set(LAME_FEATURE_PURPOSE "codec")
set(LAME_FEATURE_DESCRIPTION "lame MP3 audio codec library")

set(FAAD2_FEATURE_TYPE "OPTIONAL")
set(FAAD2_FEATURE_PURPOSE "codec")
set(FAAD2_FEATURE_DESCRIPTION "FAAD2 AAC audio codec library")

set(FAAC_FEATURE_TYPE "OPTIONAL")
set(FAAC_FEATURE_PURPOSE "codec")
set(FAAC_FEATURE_DESCRIPTION "FAAC AAC audio codec library")

set(SOXR_FEATURE_TYPE "OPTIONAL")
set(SOXR_FEATURE_PURPOSE "codec")
set(SOXR_FEATURE_DESCRIPTION "SOX audio resample library")

if(WIN32)
	set(WAYLAND_FEATURE_TYPE "DISABLED")
	set(PCSC_FEATURE_TYPE "DISABLED")
	set(FFMPEG_FEATURE_TYPE "OPTIONAL")
	set(VAAPI_FEATURE_TYPE "DISABLED")
endif()

if(APPLE)
	set(FFMPEG_FEATURE_TYPE "OPTIONAL")
	set(VAAPI_FEATURE_TYPE "DISABLED")
	set(WAYLAND_FEATURE_TYPE "DISABLED")
	if(IOS)
		set(PCSC_FEATURE_TYPE "DISABLED")
	endif()
endif()

if(ANDROID)
	set(WAYLAND_FEATURE_TYPE "DISABLED")
	set(PCSC_FEATURE_TYPE "DISABLED")
	set(VAAPI_FEATURE_TYPE "DISABLED")
endif()

find_feature(Wayland ${WAYLAND_FEATURE_TYPE} ${WAYLAND_FEATURE_PURPOSE} ${WAYLAND_FEATURE_DESCRIPTION})

option(WITH_LIBRESSL "build with LibreSSL" OFF)
if (WITH_LIBRESSL)
	find_package(LibreSSL REQUIRED)
	include_directories(${LibreSSL_INCLUDE_DIRS})
	set(OPENSSL_INCLUDE_DIR ${LIBRESSL_INCLUDE_DIR})
	set(OPENSSL_LIBRARIES ${LIBRESSL_LIBRARIES})
	set(OPENSSL_CRYPTO_LIBRARIES ${LIBRESSL_LIBRARIES})
	set(WITH_OPENSSL ON)
	set(OPENSSL_FOUND ON)
	add_definitions("-DWITH_LIBRESSL")
	add_definitions("-DWITH_OPENSSL")
else()
	find_feature(OpenSSL ${OPENSSL_FEATURE_TYPE} ${OPENSSL_FEATURE_PURPOSE} ${OPENSSL_FEATURE_DESCRIPTION})
	find_feature(MbedTLS ${MBEDTLS_FEATURE_TYPE} ${MBEDTLS_FEATURE_PURPOSE} ${MBEDTLS_FEATURE_DESCRIPTION})
endif()

find_feature(PCSC ${PCSC_FEATURE_TYPE} ${PCSC_FEATURE_PURPOSE} ${PCSC_FEATURE_DESCRIPTION})

find_package(cJSON)
option(WITH_AAD "Compile with support for Azure AD authentication" ${cJSON_FOUND})

if (WITH_DSP_FFMPEG OR WITH_VIDEO_FFMPEG OR WITH_FFMPEG)
	set(FFMPEG_FEATURE_TYPE "REQUIRED" )
endif()

find_feature(FFmpeg ${FFMPEG_FEATURE_TYPE} ${FFMPEG_FEATURE_PURPOSE} ${FFMPEG_FEATURE_DESCRIPTION})

find_feature(OpenH264 ${OPENH264_FEATURE_TYPE} ${OPENH264_FEATURE_PURPOSE} ${OPENH264_FEATURE_DESCRIPTION})
find_feature(OpenCL ${OPENCL_FEATURE_TYPE} ${OPENCL_FEATURE_PURPOSE} ${OPENCL_FEATURE_DESCRIPTION})
find_feature(GSM ${GSM_FEATURE_TYPE} ${GSM_FEATURE_PURPOSE} ${GSM_FEATURE_DESCRIPTION})
find_feature(LAME ${LAME_FEATURE_TYPE} ${LAME_FEATURE_PURPOSE} ${LAME_FEATURE_DESCRIPTION})
find_feature(FAAD2 ${FAAD2_FEATURE_TYPE} ${FAAD2_FEATURE_PURPOSE} ${FAAD2_FEATURE_DESCRIPTION})
find_feature(FAAC ${FAAC_FEATURE_TYPE} ${FAAC_FEATURE_PURPOSE} ${FAAC_FEATURE_DESCRIPTION})
find_feature(soxr ${SOXR_FEATURE_TYPE} ${SOXR_FEATURE_PURPOSE} ${SOXR_FEATURE_DESCRIPTION})

if (WITH_OPENH264 AND NOT WITH_OPENH264_LOADING)
	option(WITH_OPENH264_LOADING "Use LoadLibrary to load openh264 at runtime" OFF)
endif (WITH_OPENH264 AND NOT WITH_OPENH264_LOADING)

# Version check, if we have detected FFMPEG but the version is too old
# deactivate it as sound backend.
if (WITH_DSP_FFMPEG)
	# Deactivate FFmpeg backend for sound, if the version is too old.
	# See libfreerdp/codec/dsp_ffmpeg.h
	file(STRINGS "${AVCODEC_INCLUDE_DIR}/libavcodec/version.h" AV_VERSION_FILE REGEX "LIBAVCODEC_VERSION_M[A-Z]+[\t ]*[0-9]+")
	if (EXISTS "${AVCODEC_INCLUDE_DIR}/libavcodec/version_major.h")
		file(STRINGS "${AVCODEC_INCLUDE_DIR}/libavcodec/version_major.h" AV_VERSION_FILE2 REGEX "LIBAVCODEC_VERSION_M[A-Z]+[\t ]*[0-9]+")
		list(APPEND AV_VERSION_FILE ${AV_VERSION_FILE2})
	endif()

	FOREACH(item ${AV_VERSION_FILE})
	  STRING(REGEX MATCH "LIBAVCODEC_VERSION_M[A-Z]+[\t ]*[0-9]+" litem ${item})
		IF(litem)
				string(REGEX REPLACE "[ \t]+" ";" VSPLIT_LINE ${litem})
				list(LENGTH VSPLIT_LINE VSPLIT_LINE_LEN)
				if (NOT "${VSPLIT_LINE_LEN}" EQUAL "2")
					message(ERROR "invalid entry in libavcodec version header ${item}")
				endif(NOT "${VSPLIT_LINE_LEN}" EQUAL "2")
				list(GET VSPLIT_LINE 0 VNAME)
				list(GET VSPLIT_LINE 1 VVALUE)
				set(${VNAME} ${VVALUE})
		ENDIF(litem)
	ENDFOREACH(item ${AV_VERSION_FILE})

	set(AVCODEC_VERSION "${LIBAVCODEC_VERSION_MAJOR}.${LIBAVCODEC_VERSION_MINOR}.${LIBAVCODEC_VERSION_MICRO}")
	if (AVCODEC_VERSION VERSION_LESS "57.48.101")
		message(WARNING "FFmpeg version detected (${AVCODEC_VERSION}) is too old. (Require at least 57.48.101 for sound). Deactivating")
		set(WITH_DSP_FFMPEG OFF)
	endif()
endif (WITH_DSP_FFMPEG)

if (WITH_OPENH264 AND NOT OPENH264_FOUND)
	message(FATAL_ERROR "OpenH264 support requested but not detected")
endif()
set(WITH_OPENH264 ${OPENH264_FOUND})

if(TARGET_ARCH MATCHES "x86|x64")
	if (NOT APPLE)
		# Intel Performance Primitives
		find_feature(IPP ${IPP_FEATURE_TYPE} ${IPP_FEATURE_PURPOSE} ${IPP_FEATURE_DESCRIPTION})
	endif()
endif()

if(OPENSSL_FOUND)
	add_definitions("-DWITH_OPENSSL")
	message(STATUS "Using OpenSSL Version: ${OPENSSL_VERSION}")
	include_directories(${OPENSSL_INCLUDE_DIR})
endif()

if(MBEDTLS_FOUND)
	add_definitions("-DWITH_MBEDTLS")
endif()

if (WITH_OPENH264 OR WITH_MEDIA_FOUNDATION OR WITH_VIDEO_FFMPEG OR WITH_MEDIACODEC)
	set(WITH_GFX_H264 ON)
else()
	set(WITH_GFX_H264 OFF)
endif()

# Android expects all libraries to be loadable
# without paths.
if (ANDROID OR WIN32 OR MAC_BUNDLE)
	set(PLUGIN_ABS_PATHS_DEFAULT OFF)
else()
	set(PLUGIN_ABS_PATHS_DEFAULT ON)
endif()
option(WITH_ABSOLUTE_PLUGIN_LOAD_PATHS "Load plugins with absolute paths" ${PLUGIN_ABS_PATHS_DEFAULT})

if (NOT WITH_ABSOLUTE_PLUGIN_LOAD_PATHS)
  set(FREERDP_DATA_PATH "share")
  if (NOT FREERDP_INSTALL_PREFIX)
	  set(FREERDP_INSTALL_PREFIX ".")
  endif()
  set(FREERDP_LIBRARY_PATH ".")
  set(FREERDP_PLUGIN_PATH ".")
else()
	set(FREERDP_DATA_PATH "${CMAKE_INSTALL_PREFIX}/share/${FREERDP_MAJOR_DIR}")
	if (NOT FREERDP_INSTALL_PREFIX)
		set(FREERDP_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}")
	endif()
	set(FREERDP_LIBRARY_PATH "${CMAKE_INSTALL_LIBDIR}")
	if (WIN32)
		set(FREERDP_PLUGIN_PATH "${CMAKE_INSTALL_BINDIR}/${FREERDP_MAJOR_DIR}")
	else()
		set(FREERDP_PLUGIN_PATH "${CMAKE_INSTALL_LIBDIR}/${FREERDP_MAJOR_DIR}")
	endif()
endif()
set(FREERDP_ADDIN_PATH "${FREERDP_PLUGIN_PATH}")

# Path to put extensions
set(FREERDP_EXTENSION_POSTFIX "${FREERDP_MAJOR_DIR}/extensions")
set(FREERDP_EXTENSION_REL_PATH "${CMAKE_INSTALL_LIBDIR}/${FREERDP_EXTENSION_POSTFIX}")
set(FREERDP_EXTENSION_PATH "${CMAKE_INSTALL_FULL_LIBDIR}/${FREERDP_EXTENSION_POSTFIX}")

# Proxy plugins path
if(NOT DEFINED PROXY_PLUGINDIR)
	message("using default plugins location")
	set(FREERDP_PROXY_PLUGINDIR "${FREERDP_PLUGIN_PATH}/proxy/")
else()
	set(FREERDP_PROXY_PLUGINDIR "${PROXY_PLUGINDIR}")
endif()

if (BUILD_SHARED_LIBS)
	set(CMAKE_MACOSX_RPATH ON)
endif()

# Android profiling
if(ANDROID)
	if(WITH_GPROF)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pg")
		set(PROFILER_LIBRARIES
			"${FREERDP_EXTERNAL_PROFILER_PATH}/obj/local/${ANDROID_ABI}/libandroid-ndk-profiler.a")
		include_directories("${FREERDP_EXTERNAL_PROFILER_PATH}")
	endif()
endif()

# Unit Tests

include(CTest)

if(BUILD_TESTING)
	enable_testing()

	if(MSVC)
		set(TESTING_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
	else()
		set(TESTING_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/Testing")
	endif()
endif()

# WinPR
# We want to control the winpr assert for the whole project
option(WITH_VERBOSE_WINPR_ASSERT "Compile with verbose WINPR_ASSERT." ON)
if (WITH_VERBOSE_WINPR_ASSERT)
	add_definitions(-DWITH_VERBOSE_WINPR_ASSERT)
endif()

if (FREERDP_UNIFIED_BUILD)
	add_subdirectory(winpr)
	if (WITH_WAYLAND)
		add_subdirectory(uwac)
	endif()
	if (WITH_SERVER)
		option(WITH_RDTK "build rdtk toolkit" ON)
		if (WITH_RDTK)
			add_subdirectory(rdtk)
		endif()
	endif()

	include_directories(${PROJECT_SOURCE_DIR}/winpr/include)
	include_directories(${PROJECT_BINARY_DIR}/winpr/include)
else()
	find_package(WinPR 3 REQUIRED)
	include_directories(${WinPR_INCLUDE_DIR})
endif()

# Include directories
include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(${CMAKE_CURRENT_BINARY_DIR}/include)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)

# Sub-directories

if(WITH_THIRD_PARTY)
	add_subdirectory(third-party)
	if (NOT "${THIRD_PARTY_INCLUDES}" STREQUAL "")
		include_directories(${THIRD_PARTY_INCLUDES})
	endif()
endif()


add_subdirectory(libfreerdp)

if(BSD)
	if(IS_DIRECTORY /usr/local/include)
		include_directories(/usr/local/include)
		link_directories(/usr/local/lib)
	endif()
	if(OPENBSD)
		if(IS_DIRECTORY /usr/X11R6/include)
			include_directories(/usr/X11R6/include)
		endif()
	endif()
endif()

if(WITH_CHANNELS)
	add_subdirectory(channels)
endif()

if(WITH_CLIENT_COMMON OR WITH_CLIENT)
add_subdirectory(client)
endif()

if(WITH_SERVER)
	add_subdirectory(server)
endif()

# Packaging

set(CMAKE_CPACK_INCLUDE_FILE "CMakeCPack.cmake")

if(NOT (VENDOR MATCHES "FreeRDP"))
	if(DEFINED CLIENT_VENDOR_PATH)
		if(EXISTS "${PROJECT_SOURCE_DIR}/${CLIENT_VENDOR_PATH}/CMakeCPack.cmake")
			set(CMAKE_CPACK_INCLUDE_FILE "${CLIENT_VENDOR_PATH}/CMakeCPack.cmake")
		endif()
	endif()
endif()

#message("VENDOR: ${VENDOR} CLIENT_VENDOR_PATH: ${CLIENT_VENDOR_PATH} CMAKE_CPACK_INCLUDE_FILE: ${CMAKE_CPACK_INCLUDE_FILE}")

set(FREERDP_BUILD_CONFIG_LIST "")
GET_CMAKE_PROPERTY(res VARIABLES)
FOREACH(var ${res})
	IF (var MATCHES "^WITH_*|^BUILD_TESTING|^WINPR_HAVE_*")
		LIST(APPEND FREERDP_BUILD_CONFIG_LIST "${var}=${${var}}")
	ENDIF()
ENDFOREACH()
string(REPLACE ";" " " FREERDP_BUILD_CONFIG "${FREERDP_BUILD_CONFIG_LIST}")

add_subdirectory(include)

include(${CMAKE_CPACK_INCLUDE_FILE})

message(STATUS "Intrinsic path configuration:")
#ShowCMakeVars("^CMAKE_INSTALL_PREFIX")
#ShowCMakeVars("^CMAKE_INSTALL_LIBDIR")
ShowCMakeVars("^FREERDP_INSTALL_PREFIX|^FREERDP_LIBRARY_PATH|^FREERDP_PLUGIN_PATH")
ShowCMakeVars("^FREERDP_ADDIN_PATH|^FREERDP_EXTENSION_PATH|^FREERDP_PROXY_PLUGINDIR")



