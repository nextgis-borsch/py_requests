################################################################################
# Project:  Lib Psycopg
# Purpose:  CMake build scripts
# Author:   Dmitry Baryshnikov, dmitry.baryshnikov@nexgis.com
################################################################################
# Copyright (C) 2017, NextGIS <info@nextgis.com>
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
# OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.
################################################################################
cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)

project(requests)

# some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

if(OSX_FRAMEWORK)
  set(INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/Library" CACHE INTERNAL "Installation directory for libraries" FORCE)
  set(CMAKE_MACOSX_RPATH ON)
else()
    include(GNUInstallDirs)
    set(INSTALL_LIB_DIR ${CMAKE_INSTALL_FULL_LIBDIR} CACHE INTERNAL "Installation directory for libraries" FORCE)
endif()

if(NOT WITH_PYTHON2 AND NOT WITH_PYTHON3)
    set(WITH_PYTHON2 ON)
endif()

if(WITH_PYTHON2)
    find_package(PythonInterp 2)
    if(PYTHONINTERP_FOUND)
        set(PYTHON2_EXECUTABLE ${PYTHON_EXECUTABLE})
        set(PYTHON2_VERSION_MAJOR ${PYTHON_VERSION_MAJOR})
        set(PYTHON2_VERSION_MINOR ${PYTHON_VERSION_MINOR})
    else()
        message(WARNING "Cannot find python 2")
    endif()
endif()

unset(PYTHONINTERP_FOUND)
unset(PYTHON_EXECUTABLE)
unset(PYTHON_VERSION_MAJOR)
unset(PYTHON_VERSION_MINOR)

if(WITH_PYTHON3)
    # Hack, not find python3 withouut this
    find_program(PYTHON3_EXECUTABLE python3)
    set(PYTHON_EXECUTABLE ${PYTHON3_EXECUTABLE})

    find_package(PythonInterp 3)
    if(PYTHONINTERP_FOUND)
        set(PYTHON3_EXECUTABLE ${PYTHON_EXECUTABLE})
        set(PYTHON3_VERSION_MAJOR ${PYTHON_VERSION_MAJOR})
        set(PYTHON3_VERSION_MINOR ${PYTHON_VERSION_MINOR})
    else()
        message(WARNING "Cannot find python 3")
    endif()
endif()

if(NOT PYTHON2_EXECUTABLE AND NOT PYTHON3_EXECUTABLE)
    message(FATAL_ERROR "Cannot find python")
endif()

if(PYTHON2_EXECUTABLE)
    set(PYTHON_EXECUTABLE ${PYTHON2_EXECUTABLE})
else()
    set(PYTHON_EXECUTABLE ${PYTHON3_EXECUTABLE})
endif()

file(DOWNLOAD "https://pypi.python.org/pypi/${PROJECT_NAME}/json" "${CMAKE_CURRENT_BINARY_DIR}/meta.json")

# get version and download url
execute_process(COMMAND ${PYTHON_EXECUTABLE}
    ${CMAKE_SOURCE_DIR}/cmake/pypi_process.py ${PLUGIN} ${CMAKE_CURRENT_BINARY_DIR}/meta.json
    OUTPUT_VARIABLE PACKAGE_DETAILS
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

list(GET PACKAGE_DETAILS 0 DOWNLOAD_URL)
list(GET PACKAGE_DETAILS 1 VERSION)

include(util)
report_version(${PROJECT_NAME} ${VERSION})

# Download tarbal
set(OUT_DIR_NAME ${PROJECT_NAME}-${VERSION})
set(OUT_NAME "${OUT_DIR_NAME}.tar.gz")

if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    status_message("Download ${DOWNLOAD_URL}")
    file(DOWNLOAD "${DOWNLOAD_URL}" "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}"
        SHOW_PROGRESS
    )
endif()

# Extract archive
if(NOT EXISTS ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME})
    status_message("Extract ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    execute_process(COMMAND ${CMAKE_COMMAND} -E tar xzf ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME})
endif()

macro(build_py ver)
    add_custom_target(build_py${ver} ALL
        COMMAND ${PYTHON${ver}_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/setup.py build_ext -b ${CMAKE_CURRENT_BINARY_DIR}/${PYTHON${ver}_VERSION_MAJOR}.${PYTHON${ver}_VERSION_MINOR}/site-packages
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}
        VERBATIM
    )
endmacro()

# python setup.py install --prefix=/usr/local
status_message("Install")

if(PYTHON2_EXECUTABLE)
    build_py(2)
endif()

if(PYTHON3_EXECUTABLE)
    build_py(3)
endif()

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL )
if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(SETUP_OPTS "${SETUP_OPTS} --single-version-externally-managed --record=rekord.txt --prefix=${CMAKE_INSTALL_PREFIX}")

    macro(install_mac_py ver)
        install(CODE "execute_process(
            COMMAND ${PYTHON${ver}_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/setup.py install ${SETUP_OPTS}
            WORKING_DIRECTORY  ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}
            )")
    endmacro()

    if(PYTHON2_EXECUTABLE)
        install_mac_py(2)
    endif()

    if(PYTHON3_EXECUTABLE)
        install_mac_py(3)
    endif()
elseif(UNIX)
    macro(install_py ver)
        execute_process ( COMMAND ${PYTHON${ver}_EXECUTABLE} -c "from distutils.sysconfig import get_python_lib; print(get_python_lib())" OUTPUT_VARIABLE PY_SITE_PACKAGES OUTPUT_STRIP_TRAILING_WHITESPACE)

        set(TEST_CMD "try:
    import setuptools
    print(1)
except ImportError:
    pass")

        execute_process ( COMMAND ${PYTHON${ver}_EXECUTABLE} -c ${TEST_CMD} OUTPUT_VARIABLE PY_HAVE_SETUPTOOLS OUTPUT_STRIP_TRAILING_WHITESPACE)

        set(SETUP_OPTS)

        if(PY_HAVE_SETUPTOOLS)
            set(SETUP_OPTS "${SETUP_OPTS} --single-version-externally-managed --record=rekord.txt")
        endif()

        string( LENGTH "$ENV{DESTDIR}" DESTDIRLEN )
        if ( ${DESTDIRLEN} GREATER 0 )
            set(SETUP_OPTS "${SETUP_OPTS} --root=$ENV{DESTDIR}")
        elseif(PYTHON_DESTDIR)
            set(SETUP_OPTS "${SETUP_OPTS} --root=${PYTHON_DESTDIR}")
        endif()

        # check debian
        string(FIND ${PY_SITE_PACKAGES} "dist-packages" PY_DIST_POS)
        if(${PY_DIST_POS} GREATER -1)
            set(SETUP_OPTS "${SETUP_OPTS} --install-layout=deb")
        endif()

        # TODO: do we need --install-lib=$(PYTHON_INSTALL_LIB) ?

        set(SETUP_OPTS "${SETUP_OPTS} --prefix=${CMAKE_INSTALL_PREFIX}")

        if(NOT SUPPRESS_VERBOSE_OUTPUT)
            message(STATUS "setup tools options: ${SETUP_OPTS}")
        endif()

        install(CODE "execute_process(
                COMMAND ${PYTHON${ver}_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/setup.py install ${SETUP_OPTS}
                WORKING_DIRECTORY  ${CMAKE_CURRENT_BINARY_DIR}
                )")
    endmacro()

    if(PYTHON2_EXECUTABLE)
        install_py(2)
    endif()

    if(PYTHON3_EXECUTABLE)
        install_py(3)
    endif()
else()
    macro(install_win_py ver)
        set(BINDING_PATH ${CMAKE_CURRENT_BINARY_DIR}/${PYTHON${ver}_VERSION_MAJOR}.${PYTHON${ver}_VERSION_MINOR}/site-packages)
        install(DIRECTORY "${BINDING_PATH}/"
                DESTINATION ${INSTALL_LIB_DIR}/Python${PYTHON${ver}_VERSION_MAJOR}${PYTHON${ver}_VERSION_MINOR}/site-packages
                COMPONENT libraries)
    endmacro()

    if(PYTHON2_EXECUTABLE)
        install_win_py(2)
    endif()

    if(PYTHON3_EXECUTABLE)
        install_win_py(3)
    endif()
endif()
endif()
