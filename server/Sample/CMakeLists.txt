# FreeRDP: A Remote Desktop Protocol Implementation
# FreeRDP Sample Server cmake build script
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

set(MODULE_NAME "sfreerdp-server")
set(MODULE_PREFIX "FREERDP_SERVER_SAMPLE")

set(SRCS
	sfreerdp.c
	sfreerdp.h
	sf_audin.c
	sf_audin.h
	sf_rdpsnd.c
	sf_rdpsnd.h
	sf_encomsp.c
	sf_encomsp.h)

if (CHANNEL_AINPUT_SERVER)
    list(APPEND SRCS sf_ainput.c sf_ainput.h)
endif()

	# On windows create dll version information.
# Vendor, product and year are already set in top level CMakeLists.txt
if (WIN32)
  set (RC_VERSION_MAJOR ${FREERDP_VERSION_MAJOR})
  set (RC_VERSION_MINOR ${FREERDP_VERSION_MINOR})
  set (RC_VERSION_BUILD ${FREERDP_VERSION_REVISION})
  set (RC_VERSION_FILE "${MODULE_NAME}${CMAKE_EXECUTABLE_SUFFIX}" )

  configure_file(
    ${PROJECT_SOURCE_DIR}/cmake/WindowsDLLVersion.rc.in
    ${CMAKE_CURRENT_BINARY_DIR}/version.rc
    @ONLY)

  set ( SRCS ${SRCS} ${CMAKE_CURRENT_BINARY_DIR}/version.rc)
endif()

if (WITH_BINARY_VERSIONING)
  set(SAMPLE_RESOURCE_ROOT ${CMAKE_INSTALL_FULL_DATAROOTDIR}/FreeRDP${FREERDP_VERSION_MAJOR}/images)
else()
  set(SAMPLE_RESOURCE_ROOT ${CMAKE_INSTALL_FULL_DATAROOTDIR}/FreeRDP/images)
endif()

set(SAMPLE_ICONS
    test_icon.bmp
    test_icon.png
    test_icon.jpg
    test_icon.webp
)
install(FILES ${SAMPLE_ICONS} DESTINATION ${SAMPLE_RESOURCE_ROOT})

# We need this in runtime path for TestConnect
file(COPY test_icon.bmp DESTINATION ${CMAKE_CURRENT_BINARY_DIR})

add_executable(${MODULE_NAME} ${SRCS})
if (WITH_BINARY_VERSIONING)
	set_target_properties(${MODULE_NAME} PROPERTIES OUTPUT_NAME "${MODULE_NAME}${FREERDP_API_VERSION}")
endif()

target_compile_definitions(${MODULE_NAME}
  PRIVATE
    -DSAMPLE_RESOURCE_ROOT="${SAMPLE_RESOURCE_ROOT}"
)
list(APPEND LIBS freerdp-server)
list(APPEND LIBS winpr freerdp)

target_link_libraries(${MODULE_NAME} ${LIBS})
install(TARGETS ${MODULE_NAME} DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT server)
if (WITH_DEBUG_SYMBOLS AND MSVC)
    install(FILES ${CMAKE_PDB_BINARY_DIR}/${MODULE_NAME}.pdb DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT symbols)
endif()

set_property(TARGET ${MODULE_NAME} PROPERTY FOLDER "Server/Sample")
