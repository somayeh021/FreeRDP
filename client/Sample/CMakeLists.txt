# FreeRDP: A Remote Desktop Protocol Implementation
# FreeRDP Sample UI cmake build script
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
cmake_minimum_required(VERSION 3.13)

if(POLICY CMP0091)
	cmake_policy(SET CMP0091 NEW)
endif()
if (NOT FREERDP_DEFAULT_PROJECT_VERSION)
	set(FREERDP_DEFAULT_PROJECT_VERSION "1.0.0.0")
endif()

project(sfreerdp
	LANGUAGES C
	VERSION ${FREERDP_DEFAULT_PROJECT_VERSION}
)

message("project ${PROJECT_NAME} is using version ${PROJECT_VERSION}")

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS ON)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/../../cmake/)
include(CommonConfigOptions)

include(ConfigureFreeRDP)

set(SRCS
	tf_channels.c
	tf_channels.h
	tf_freerdp.h
	tf_freerdp.c)

# On windows create dll version information.
# Vendor, product and year are already set in top level CMakeLists.txt
if (WIN32)
  set (RC_VERSION_MAJOR ${FREERDP_VERSION_MAJOR})
  set (RC_VERSION_MINOR ${FREERDP_VERSION_MINOR})
  set (RC_VERSION_BUILD ${FREERDP_VERSION_REVISION})
  set (RC_VERSION_FILE "${PROJECT_NAME}${CMAKE_EXECUTABLE_SUFFIX}" )

  configure_file(
    ${PROJECT_SOURCE_DIR}/../../cmake/WindowsDLLVersion.rc.in
    ${CMAKE_CURRENT_BINARY_DIR}/version.rc
    @ONLY)

list (APPEND SRCS ${CMAKE_CURRENT_BINARY_DIR}/version.rc)
endif()

add_executable(${PROJECT_NAME} ${SRCS})

set(LIBS
    freerdp-client
    freerdp
    winpr
)
target_link_libraries(${PROJECT_NAME} PRIVATE ${LIBS})

if (WITH_BINARY_VERSIONING)
	set_target_properties(${PROJECT_NAME} PROPERTIES OUTPUT_NAME "${PROJECT_NAME}${PROJECT_VERSION_MAJOR}")
endif()
set_property(TARGET ${PROJECT_NAME} PROPERTY FOLDER "Client/Sample")
install(TARGETS ${PROJECT_NAME} DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT client)
