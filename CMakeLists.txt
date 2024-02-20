cmake_minimum_required(VERSION 2.8.12)
project(SeriousProton)
set(CMAKE_MODULE_PATH
	${CMAKE_SOURCE_DIR}/cmake/
	${CMAKE_MODULE_PATH}
)

# User-settings
option(SERIOUSPROTON_WITH_JSON "Use json library." OFF)

if(WIN32)
    option(ENABLE_CRASH_LOGGER "Enable the Dr. MinGW crash logging facilities" OFF)
    set(DRMINGW_ROOT DRMINGW_ROOT-NOTFOUND CACHE PATH "Path to Dr. MinGW")
endif()

# Preflight checks.
message(STATUS "Crash Logger is " ${ENABLE_CRASH_LOGGER})

if(ENABLE_CRASH_LOGGER)
    if(NOT DRMINGW_ROOT)
        message("Downloading Dr. MinGW")

        set(DRMINGW_ARCH "64")
        if(${CMAKE_SIZEOF_VOID_P} EQUAL 4)
            set(DRMINGW_ARCH "32")
        endif()

        # 0.9.x seems to give a hard time to people on Win7.
        # Sticking with 0.8 for that reason.
        set(DRMINGW_VERSION "0.8.2")
        set(DRMINGW_BASENAME "drmingw-${DRMINGW_VERSION}-win${DRMINGW_ARCH}")
        set(DRMINGW_ROOT "${CMAKE_CURRENT_BINARY_DIR}/${DRMINGW_BASENAME}" CACHE PATH "Path to Dr. MinGW" FORCE)

        if(NOT EXISTS "${DRMINGW_ROOT}/bin/exchndl.dll")
            set(DRMINGW_ZIP "${CMAKE_CURRENT_BINARY_DIR}/${DRMINGW_BASENAME}.7z")
            
            file(DOWNLOAD "https://github.com/jrfonseca/drmingw/releases/download/${DRMINGW_VERSION}/${DRMINGW_BASENAME}.7z" "${DRMINGW_ZIP}" TIMEOUT 60 TLS_VERIFY ON)
            execute_process(COMMAND ${CMAKE_COMMAND} -E tar -xf "${DRMINGW_ZIP}" WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")
        endif()
    endif()
endif()

#--------------------------------Dependencies----------------------------------
find_package(SFML 2.5 COMPONENTS system audio network window graphics)
if(NOT ${SFML_FOUND})
    message(STATUS "Couldn't find SFML. Building it from scratch.")
    
    set(SFML_ROOT "${CMAKE_CURRENT_BINARY_DIR}/SFML-prefix")

    include(ExternalProject)

    ExternalProject_Add(SFML
        GIT_REPOSITORY git@github.com:SFML/SFML.git
        GIT_TAG 2.5.1 #Always build with this tag, so we're sure it is stable.
        GIT_SHALLOW 1 # Don't get the entire history
        CMAKE_ARGS
            -DBUILD_SHARED_LIBS:BOOL=$<BOOL:${WIN32}>
            -DCMAKE_INSTALL_PREFIX:PATH=${SFML_ROOT}
            -DCMAKE_MAKE_PROGRAM:FILEPATH=${CMAKE_MAKE_PROGRAM}
            -DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}
    )

    set(SFML_INCLUDE_DIR "${SFML_ROOT}/include") #Otherwise this is filled by FindSFML.cmake.

    set(_SFML_LIB_PREFIX STATIC)
    set(_SFML_LIB_SUFFIX STATIC)

    if(WIN32)
        set(_SFML_LIB_PREFIX SHARED)
        set(_SFML_LIB_SUFFIX LINK)
    endif()

    set(SFML_LIBRARIES
        "-L${SFML_ROOT}/lib"
        "${CMAKE_${_SFML_LIB_TYPE}_LIBRARY_PREFIX}sfml-audio${CMAKE_${_SFML_LIB_SUFFIX}_LIBRARY_SUFFIX}"
        "${CMAKE_${_SFML_LIB_TYPE}_LIBRARY_PREFIX}sfml-graphics${CMAKE_${_SFML_LIB_SUFFIX}_LIBRARY_SUFFIX}"
        "${CMAKE_${_SFML_LIB_TYPE}_LIBRARY_PREFIX}sfml-network${CMAKE_${_SFML_LIB_SUFFIX}_LIBRARY_SUFFIX}"
        "${CMAKE_${_SFML_LIB_TYPE}_LIBRARY_PREFIX}sfml-system${CMAKE_${_SFML_LIB_SUFFIX}_LIBRARY_SUFFIX}"
        "${CMAKE_${_SFML_LIB_TYPE}_LIBRARY_PREFIX}sfml-window${CMAKE_${_SFML_LIB_SUFFIX}_LIBRARY_SUFFIX}"
    )
endif()

add_subdirectory(src/Box2D)
add_subdirectory(src/lua)
add_subdirectory(src/GL)
add_subdirectory(src/libopus)

if(SERIOUSPROTON_WITH_JSON)
    add_subdirectory(src/json11)
endif()

#---------------------------------File lists-----------------------------------
set(source_files #All SeriousProton's objects to compile
    src/clipboard.cpp
    src/collisionable.cpp
    src/engine.cpp
    src/event.cpp
    src/fixedSocket.cpp
    src/gameEntity.cpp
    src/httpServer.cpp
    src/i18n.cpp
    src/input.cpp
    src/logging.cpp
    src/multiplayer.cpp
    src/multiplayer_client.cpp
    src/multiplayer_proxy.cpp
    src/multiplayer_server.cpp
    src/multiplayer_server_scanner.cpp
    src/networkAudioStream.cpp
    src/networkRecorder.cpp
    src/P.cpp
    src/PlayerController.cpp
    src/postProcessManager.cpp
    src/random.cpp
    src/Renderable.cpp
    src/resources.cpp
    src/scriptInterface.cpp
    src/scriptInterfaceMagic.cpp
    src/shaderManager.cpp
    src/soundManager.cpp
    src/stringImproved.cpp
    src/textureManager.cpp
    src/tween.cpp
    src/Updatable.cpp
    src/windowManager.cpp
    src/io/network/address.cpp
    src/io/network/selector.cpp
    src/io/network/socketBase.cpp
    src/io/network/tcpListener.cpp
    src/io/network/tcpSocket.cpp
    src/io/network/udpSocket.cpp
)
set(source_files "")
foreach(source_file_relative ${source_files_relative})
	list(APPEND source_files ${CMAKE_CURRENT_SOURCE_DIR}/${source_file_relative})
endforeach()

set(seriousproton_include_dir ${CMAKE_CURRENT_SOURCE_DIR}/src/)

include_directories(
	${CMAKE_CURRENT_BINARY_DIR}
	${CMAKE_CURRENT_SOURCE_DIR}/src
	${CMAKE_CURRENT_SOURCE_DIR}/src/libopus/include
)

#----------------------------------Compiling-----------------------------------
if(CMAKE_VERSION VERSION_LESS "3.1")
	if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
		set(CMAKE_CXX_FLAGS "-std=gnu++11 ${CMAKE_CXX_FLAGS}")
	endif()
else()
	set(CMAKE_CXX_STANDARD 11)
endif()

add_library(seriousproton STATIC ${source_files})

#--------------------------------Dependencies----------------------------------
include(ExternalProject)

target_link_libraries(seriousproton_deps
    INTERFACE 
        box2d lua glew ${SFML_LIBRARIES}
        $<$<BOOL:${WIN32}>:wsock32 ws2_32 crypt32>
        # LTO flag must be on the linker's list as well.
        "$<$<AND:$<BOOL:${CMAKE_COMPILER_IS_GNUCC}>,$<OR:$<CONFIG:RelWithDebInfo>,$<CONFIG:Release>>>:-flto>"
        "$<BUILD_INTERFACE:opus;$<$<BOOL:${SERIOUSPROTON_WITH_JSON}>:json11>;$<$<NOT:$<BOOL:${ANDROID}>>:${CMAKE_DL_LIBS}>>"
)

add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/src/lua)
add_dependencies(seriousproton lua)

add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/src/GL)
add_dependencies(seriousproton glew)

find_package(SFML 2.3 COMPONENTS system audio network window graphics)
if(NOT ${SFML_FOUND})
	message(STATUS "Couldn't find SFML. Building it from scratch. Installing to ${CMAKE_INSTALL_PREFIX}")
	ExternalProject_Add(SFML
		GIT_REPOSITORY https://github.com/SFML/SFML.git
		GIT_TAG 2.4.2 #Always build with this tag, so we're sure it is stable.
		CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} -DBUILD_SHARED_LIBS=OFF
	)
	set(SFML_INCLUDE_DIR ${CMAKE_INSTALL_PREFIX}/include) #Otherwise this is filled by FindSFML.cmake.
	set(SFML_LIBRARIES
		${CMAKE_INSTALL_PREFIX}/lib/${CMAKE_STATIC_LIBRARY_PREFIX}sfml-audio${CMAKE_STATIC_LIBRARY_SUFFIX}
		${CMAKE_INSTALL_PREFIX}/lib/${CMAKE_STATIC_LIBRARY_PREFIX}sfml-graphics${CMAKE_STATIC_LIBRARY_SUFFIX}
		${CMAKE_INSTALL_PREFIX}/lib/${CMAKE_STATIC_LIBRARY_PREFIX}sfml-network${CMAKE_STATIC_LIBRARY_SUFFIX}
		${CMAKE_INSTALL_PREFIX}/lib/${CMAKE_STATIC_LIBRARY_PREFIX}sfml-system${CMAKE_STATIC_LIBRARY_SUFFIX}
		${CMAKE_INSTALL_PREFIX}/lib/${CMAKE_STATIC_LIBRARY_PREFIX}sfml-window${CMAKE_STATIC_LIBRARY_SUFFIX}
	)
	add_dependencies(seriousproton SFML)
endif()
include_directories(${SFML_INCLUDE_DIR})
target_link_libraries(seriousproton ${SFML_LIBRARIES})

#--------------------------------Installation----------------------------------
install(
	TARGETS seriousproton
	EXPORT seriousproton
	LIBRARY DESTINATION lib
	ARCHIVE DESTINATION lib
)
install(
	DIRECTORY ${seriousproton_include_dir}
	DESTINATION include/seriousproton
	FILES_MATCHING PATTERN "*.h"
)
install(
	DIRECTORY ${seriousproton_include_dir}
	DESTINATION include/seriousproton
	FILES_MATCHING PATTERN "*.hpp"
)
install(
	EXPORT seriousproton
	DESTINATION share/seriousproton
)
