# @author Lukas Heiligenbrunner
# main CMake file
#
# Build lib dependencies:
## libcurl (with sources)
## libconfig (with sources)
#
# for documenation build doxygen needs to be installed.
# for test build gtest needs to be installed.

cmake_minimum_required(VERSION 3.13)
project(dynuiprefresher DESCRIPTION "Dynu ip refresher" LANGUAGES)
SET(PROJECT_VERSION 1.3.4)

# CONFIGURATION
SET(CMAKE_BUILD_TYPE Release) # manually SET build type (Release / Debug)
SET(LIB_METHOD STATIC) #SHARED / STATIC

option(BUILD_DOC "Build documentation" ON) # additional dependency for Doxygen
option(PACKAGING "Allow Packaging to <exe>, <deb> or <rpm>" ON) # additional dependencies for RPMbuild,dpkg  or NSIS
option(TESTS "Build Tests" ON) # additional dependencies for GTEST - to build tests
option(GUI "Build GUI elements" ON) # additional dependencies to QT libraries needed
option(WinBuild "cross compile for Windows Platform" ON)

# helper variables
SET(CMAKE_CXX_STANDARD 17)
string(TIMESTAMP TIMESTAMP_NOW "%d.%m.%Y")
SET(Application_Name "dynuiprefresher")

SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# setup winbuild compilers
if (${WinBuild})
    message(STATUS "setup Mingw Toolchain for cross compile.")
    set(LIBSUFFIX .dll)
    set(SUFFIX .exe)

    set(CMAKE_SYSTEM_NAME WindowsStore)
    set(CMAKE_SYSTEM_VERSION 10.0)
    set(TOOLCHAIN_PREFIX x86_64-w64-mingw32) # x64 build toolchain
    #set(TOOLCHAIN_PREFIX i686-w64-mingw32)

    # cross compilers to use for C and C++
    set(CMAKE_C_COMPILER /usr/bin/${TOOLCHAIN_PREFIX}-gcc)
    set(CMAKE_CXX_COMPILER /usr/bin/${TOOLCHAIN_PREFIX}-g++)
    set(CMAKE_RC_COMPILER /usr/bin/${TOOLCHAIN_PREFIX}-windres)

    # target environment on the build host system
    set(CMAKE_FIND_ROOT_PATH /usr/${TOOLCHAIN_PREFIX})


    # modify default behavior of FIND_XXX() commands to
    # search for headers/libs in the target environment and
    # search for programs in the build host environment
    set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM ONLY)
    set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
    set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
    set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)

    set(CMAKE_CXX_STANDARD_LIBRARIES "${CMAKE_CSS_STANDARD_LIBRARIES}")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS}")
    # currently libs are dynamically linked eg. dlls have to be copied to same folder
    # todo link dynamically.
    # maybe so: set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static-libgcc -static-libstdc++ -static")
    # or set(CMAKE_CXX_STANDARD_LIBRARIES -lcurl -lpthread -static-libgcc -static-libstdc++ -lcrypto -lssl -lws2_32 -static -DCURL_STATICLIB)
    # or add_definitions(-DCURL_STATICLIB)

    # windows config path is in %appdata% folder of user
    set(CONFIG_PATH "std::string(std::getenv(\"USERPROFILE\")) + \"\\\\AppData\\\\Roaming\\\\DynuIPrefresher\\\\\"")
    # temp file is also stored in appdata folder
    set(TempFilePath "std::string(std::getenv(\"USERPROFILE\")) + \"\\\\AppData\\\\Roaming\\\\DynuIPrefresher\\\\\"")
else ()
    message(STATUS "using nativ gcc toolchain.")
    set(LIBSUFFIX .so)
    set(SUFFIX "")

    # set linux config path
    set(CONFIG_PATH "\"/etc/\"")
    # set path of temp file
    set(TempFilePath "\"/var/tmp/\"")
endif ()

# test compiler settings and enable languages here
message("")
message(STATUS "Testing the C++ compiler!")
enable_language(CXX)
message("")
message(STATUS "Testing the C compiler!")
enable_language(C)

# config libs
message("")
message(STATUS "Config of Libraries")
# libcurl
if (${WinBuild})
    SET(CMAKE_FIND_LIBRARY_SUFFIXES ".lib" ".dll")
    # configure libcurl
    FIND_PATH(CURL_INCLUDE_DIRS curl/curl.h /usr/${TOOLCHAIN_PREFIX}/sys-root/mingw/include/) # search for libconfig include headers
    FIND_LIBRARY(CURL_LIBRARIES NAMES libcurl.dll.a HINTS /usr/${TOOLCHAIN_PREFIX}/sys-root/mingw/lib) # search for actual lib

    message(STATUS "Using CURL include dir(s): ${CURL_INCLUDE_DIRS}")
    message(STATUS "Using CURL lib(s): ${CURL_LIBRARIES}")
    message(STATUS "")

    include_directories(${CURL_INCLUDE_DIRS})

    # configure libconfig
    FIND_PATH(LIBCONFIG++_INCLUDE_DIRS libconfig.h++ /usr/${TOOLCHAIN_PREFIX}/sys-root/mingw/include/) # search for libconfig include headers
    FIND_LIBRARY(LIBCONFIG++_LIBRARIES NAMES libconfig++.dll.a HINTS /usr/${TOOLCHAIN_PREFIX}/sys-root/mingw/lib) # search for actual lib

    message(STATUS "Using LIBCONFIG++ include dir(s): ${LIBCONFIG++_INCLUDE_DIRS}")
    message(STATUS "Using LIBCONFIG++ lib(s): ${LIBCONFIG++_LIBRARIES}")

    if (${GUI})
        set(CMAKE_PREFIX_PATH "/usr/${TOOLCHAIN_PREFIX}/sys-root/mingw/lib/cmake")
    endif ()

    include_directories(${LIBCONFIG++_INCLUDE_DIRS})
else ()
    find_package(CURL REQUIRED)
    if (CURL_INCLUDE_DIRS AND CURL_LIBRARIES)
        message(STATUS "Found CURL version: ${CURL_VERSION_STRING}")
        message(STATUS "Using CURL include dir(s): ${CURL_INCLUDE_DIRS}")
        message(STATUS "Using CURL lib(s): ${CURL_LIBRARIES}")
    else ()
        message(FATAL_ERROR "Could not find CURL")
    endif ()
    include_directories(${CURL_INCLUDE_DIR})

    message("")
    # libconfig
    FIND_PATH(LIBCONFIG++_INCLUDE_DIRS
            NAMES libconfig.h++
            PATHS /usr/include /usr/local/include) # search for libconfig include headers
    FIND_LIBRARY(LIBCONFIG++_LIBRARIES
            NAMES config++
            PATHS /usr/lib /usr/local/lib) # search for actual lib

    if (LIBCONFIG++_INCLUDE_DIRS AND LIBCONFIG++_LIBRARIES)
        message(STATUS "Fount libconfig!")
        message(STATUS "Using libconfig include dir(s): ${LIBCONFIG++_INCLUDE_DIRS}")
        message(STATUS "Using libconfig lib(s): ${LIBCONFIG++_LIBRARIES}")
    else ()
        message(FATAL_ERROR "Could not find LIBCONFIG")
    endif ()

    include_directories(${LIBCONFIG_INCLUDE_DIRS})
endif ()


if (${GUI})
    set(CMAKE_AUTOMOC ON)
    set(CMAKE_INCLUDE_CURRENT_DIR ON)

    find_package(Qt5Widgets REQUIRED)
    find_package(Qt5PrintSupport REQUIRED)
    find_package(Qt5Sql REQUIRED)
endif ()

include_directories(inc)

message("")

#read sample config
FILE(READ ${CMAKE_SOURCE_DIR}/config/dynuiprefresher.cfg SAMPLECONFIG)
#add StaticData header
FILE(WRITE ${CMAKE_SOURCE_DIR}/inc/StaticData.h
        "/**
 * StaticData header to store Version, Config dir and a Sample config
 * Do not edit this file manually, it is generated by the cmake script!
 *
 * @author Lukas Heiligenbrunner
 * @date ${TIMESTAMP_NOW}
 */

#pragma once

#include <string>

namespace StaticData {
  const std::string VERSION = \"${PROJECT_VERSION}\";
  const std::string TempFilePath = ${TempFilePath};
  const std::string ConfigDir = ${CONFIG_PATH};
  const std::string ConfName = \"${Application_Name}.cfg\";
  const std::string SAMPLECONFIG = R\"(${SAMPLECONFIG})\";
}"
        )

add_library(api ${LIB_METHOD}
        src/api/API.cpp
        src/api/TelegramAPI.cpp
        src/api/DynuAPI.cpp
        src/api/IPAPI.cpp)

add_library(libdynuiprefresher ${LIB_METHOD}
        src/IPRefresher.cpp
        src/Config.cpp
        src/IpHelper.cpp
        src/FileLogger.cpp
        src/Logger.cpp
        )

add_executable(${Application_Name} src/main.cpp)

# LINK generated LIBS #
target_link_libraries(${Application_Name} libdynuiprefresher api ${CURL_LIBRARIES} ${LIBCONFIG++_LIBRARIES})

if (${GUI})
    set(QT5_LIBRARIES Qt5::Widgets Qt5::PrintSupport Qt5::Sql)

    set(UI_SOURCES
            src/gui/mainwindow.ui
            )

    #
    # Generate necessary headers from .ui files. (qmake lets `uic` do this job.)
    # hint from [Cross-platform Qt5 project using cmake](http://stackoverflow.com/questions/21174586/cross-platform-qt5-project-using-cmake)
    #
    qt5_wrap_ui(UI_GENERATED_HEADERS ${UI_SOURCES})

    add_executable(${Application_Name}-gui
            src/maingui.cpp
            src/gui/MainWindow.cpp
            inc/gui/MainWindow.h
            ${UI_GENERATED_HEADERS})

    if (${WinBuild})
        # hide console window when starting ui on windows
        set_target_properties(${Application_Name}-gui PROPERTIES LINK_FLAGS "-mwindows")
    endif ()

    # LINK generated LIBS #
    target_link_libraries(${Application_Name}-gui -lpthread libdynuiprefresher api ${CURL_LIBRARIES} ${LIBCONFIG++_LIBRARIES} ${QT5_LIBRARIES})
endif ()

# setting install targets
IF (NOT ${WinBuild})
    # INSTALL to Linux SYSTEM #

    # install binaries
    install(
            TARGETS ${Application_Name}
            DESTINATION usr/bin
            COMPONENT ${Application_Name})
    # install systemd service and enable it
    install(
            FILES service/${Application_Name}.service
            DESTINATION lib/systemd/system
            COMPONENT ${Application_Name})

    if (${GUI})
        # install binaries
        install(
                TARGETS ${Application_Name}-gui
                DESTINATION usr/bin
                COMPONENT ${Application_Name}gui)
    endif ()
ELSE ()
    # INSTALL to Windows SYSTEM #

    # install binary to current folder
    set_target_properties(${Application_Name} PROPERTIES SUFFIX ".exe")
    install(
            TARGETS ${Application_Name}
            DESTINATION .
            COMPONENT ${Application_Name})

    # install .dll dependencies
    # todo check if files exist...
    SET(LIBBINARYPATH "/usr/${TOOLCHAIN_PREFIX}/sys-root/mingw/bin")
    install(FILES /usr/${TOOLCHAIN_PREFIX}/sys-root/mingw/bin/libcurl-4.dll
            ${LIBBINARYPATH}/libssh2-1.dll
            ${LIBBINARYPATH}/libstdc++-6.dll
            ${LIBBINARYPATH}/libgcc_s_seh-1.dll
            ${LIBBINARYPATH}/libcrypto-1_1-x64.dll
            ${LIBBINARYPATH}/libssl-1_1-x64.dll
            ${LIBBINARYPATH}/libwinpthread-1.dll
            ${LIBBINARYPATH}/zlib1.dll
            ${LIBBINARYPATH}/libidn2-0.dll
            ${LIBBINARYPATH}/libconfig++-11.dll
            DESTINATION .
            COMPONENT ${Application_Name})

    if (${GUI})
        # install binaries
        set_target_properties(${Application_Name}-gui PROPERTIES SUFFIX ".exe")
        install(
                TARGETS ${Application_Name}-gui
                DESTINATION .
                COMPONENT ${Application_Name}gui)

        # install .dll dependencies
        install(FILES
                ${LIBBINARYPATH}/Qt5Widgets.dll
                ${LIBBINARYPATH}/Qt5Core.dll
                ${LIBBINARYPATH}/Qt5Gui.dll
                ${LIBBINARYPATH}/iconv.dll
                ${LIBBINARYPATH}/libpcre2-16-0.dll
                ${LIBBINARYPATH}/libpng16-16.dll
                ${LIBBINARYPATH}/libharfbuzz-0.dll
                ${LIBBINARYPATH}/libglib-2.0-0.dll
                ${LIBBINARYPATH}/libintl-8.dll
                ${LIBBINARYPATH}/libpcre-1.dll
                ${LIBBINARYPATH}/libbz2-1.dll
                ${LIBBINARYPATH}/libfreetype-6.dll
                DESTINATION .
                COMPONENT ${Application_Name}gui)

        install(FILES
                /usr/${TOOLCHAIN_PREFIX}/sys-root/mingw/lib/qt5/plugins/platforms/qwindows.dll
                DESTINATION ./platforms
                COMPONENT ${Application_Name}gui)
    endif ()
ENDIF ()

if (${PACKAGING})
    # General Packaging options:
    message(STATUS "config of Package build")

    set(CPACK_PACKAGE_NAME "DynuIpRefresher")
    SET(CPACK_PACKAGE_DESCRIPTION "IPrefresher to refresh Dynu ip address and notify user via Telegram")
    SET(CPACK_PACKAGE_DESCRIPTION_SUMMARY "IPrefresher to refresh Dynu ip address and notify user via Telegram")
    SET(CPACK_PACKAGE_VENDOR "Lukas Heilgienbrunner")
    SET(CPACK_PACKAGE_VERSION "${PROJECT_VERSION}")
    SET(CPACK_SOURCE_PACKAGE_FILE_NAME "${CMAKE_PROJECT_NAME}-${PROJECT_VERSION}")
    SET(CPACK_PACKAGE_CONTACT "Lukas Heiligenbrunner <lukas.heiligenbrunner@gmail.com>")

    SET(CMAKE_INSTALL_PREFIX "/")

    IF (NOT ${WinBuild})
        # generate post script for checking if configuration already exists
        FILE(WRITE ${CMAKE_SOURCE_DIR}/postinst
                "#!/bin/bash
# Post installation script for linux packages
# do not edit this file manually, it is generated by the cmake script
if [ ! -f ${CONFIG_PATH} ]; then
    cat > ${CONFIG_PATH} <<- EOM
${SAMPLECONFIG}EOM
fi

systemctl enable ${Application_Name}.service
systemctl start ${Application_Name}.service")


        SET(CPACK_DEB_COMPONENT_INSTALL 1)
        SET(CPACK_OUTPUT_FILE_PREFIX packages)
        SET(CPACK_PACKAGING_INSTALL_PREFIX "/") # no prefix for package structure

        FIND_PROGRAM(RPMBUILD_EXECUTABLE rpmbuild)
        FIND_PROGRAM(DEB_EXECUTABLE dpkg)

        SET(CPACK_GENERATOR "TGZ;TBZ2")

        #check if rpm build is possible
        if (NOT ${RPMBUILD_EXECUTABLE} STREQUAL "RPMBUILD_EXECUTABLE-NOTFOUND")
            message(STATUS "found rpm build executeable --> able to build rpm")
            SET(CPACK_GENERATOR "${CPACK_GENERATOR};RPM")
            SET(CPACK_RPM_POST_INSTALL_SCRIPT_FILE "${CMAKE_CURRENT_SOURCE_DIR}/postinst")
            SET(CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION
                    "/lib/systemd/system"
                    "/lib/systemd"
                    "/lib"
                    "/usr/local/bin"
                    "/usr/local") # --> not override existing folders

            # dependency management tested with fedora!
            set(CPACK_RPM_PACKAGE_REQUIRES "libcurl,libconfig")
            if (${GUI})
                # append rpm GUI dependencies (qt5)
                set(CPACK_RPM_PACKAGE_REQUIRES "${CPACK_RPM_PACKAGE_REQUIRES},qt5-qtbase")
            endif ()
        else (NOT ${RPMBUILD_EXECUTABLE} STREQUAL "RPMBUILD_EXECUTABLE-NOTFOUND")
            message(STATUS "not found rpm build tools --> not building rpm")
        endif (NOT ${RPMBUILD_EXECUTABLE} STREQUAL "RPMBUILD_EXECUTABLE-NOTFOUND")

        #check if deb build is possible
        if (NOT ${DEB_EXECUTABLE} STREQUAL "DEB_EXECUTABLE-NOTFOUND")
            message(STATUS "found deb build tools --> able to build deb")
            SET(CPACK_GENERATOR "${CPACK_GENERATOR};DEB") # add deb generator
            set(CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA "${CMAKE_CURRENT_SOURCE_DIR}/postinst") # set post inst file

            # dependency management tested with fedora!
            set(CPACK_DEBIAN_PACKAGE_DEPENDS "libcurl4,libconfig++9v5") # add debian dependencies
            if (${GUI})
                # append debian GUI dependencies (qt5)
                set(CPACK_DEBIAN_PACKAGE_DEPENDS "${CPACK_DEBIAN_PACKAGE_DEPENDS},qt5-default")
            endif ()
        else (NOT ${DEB_EXECUTABLE} STREQUAL "DEB_EXECUTABLE-NOTFOUND")
            message(STATUS "not found  deb build tools --> not building deb")
        endif (NOT ${DEB_EXECUTABLE} STREQUAL "DEB_EXECUTABLE-NOTFOUND")


        SET(CPACK_CMAKE_GENERATOR "Unix Makefiles")
        SET(CPACK_SOURCE_GENERATOR "TGZ;TBZ2")

        SET(CPACK_PACKAGE_SECTION "games")
    ELSE ()
        # Pack a NSIS intaller
        set(CPACK_GENERATOR NSIS)
        message(STATUS "Using NSIS Package build.")

        set(CPACK_NSIS_EXECUTABLES_DIRECTORY "DynuIpRefresher")
        set(CPACK_PACKAGE_INSTALL_DIRECTORY "DynuIpRefresher")
        SET(CPACK_NSIS_MODIFY_PATH ON)
        set(CPACK_NSIS_MENU_LINKS
                "${Application_Name}.exe" "DynuIpRefresher Console"
                "${Application_Name}-gui.exe" "DynuIpRefresher GUI")

        # varnames need uppercase strings
        string(TOUPPER ${Application_Name} APPLICATION_NAME_UPPER)
        # Define components and their display names
        set (CPACK_COMPONENTS_ALL ${Application_Name} ${Application_Name}gui)
        set (CPACK_COMPONENT_${APPLICATION_NAME_UPPER}_DISPLAY_NAME "Dynu IP Refresher Console")
        set (CPACK_COMPONENT_${APPLICATION_NAME_UPPER}GUI_DISPLAY_NAME "Dynu IP Refresher GUI")

        # Human readable component descriptions
        set(CPACK_COMPONENT_${APPLICATION_NAME_UPPER}_DESCRIPTION "An extremely useful application that makes use of MyLib")
        set(CPACK_COMPONENT_${APPLICATION_NAME_UPPER}GUI_DESCRIPTION "Static libraries used to build programs with MyLib")

        # Define dependencies between components
        set(CPACK_COMPONENT_${APPLICATION_NAME_UPPER}GUI_DEPENDS ${Application_Name})

        # Define groups
        set(CPACK_COMPONENT_${APPLICATION_NAME_UPPER}_GROUP "Runtime")
        set(CPACK_COMPONENT_${APPLICATION_NAME_UPPER}GUI_GROUP "Development")

        set(CPACK_COMPONENT_GROUP_DEVELOPMENT_DESCRIPTION "All of the tools you'll ever need to develop software")

        # Define NSIS installation types
        set(CPACK_ALL_INSTALL_TYPES Full Developer)
        set(CPACK_COMPONENT_${APPLICATION_NAME_UPPER}_INSTALL_TYPES Developer Full)
        set(CPACK_COMPONENT_${APPLICATION_NAME_UPPER}GUI_INSTALL_TYPES Full)
    ENDIF ()

    INCLUDE(CPack)

    add_custom_target(build-packages
            "${CMAKE_COMMAND}" --build "${CMAKE_BINARY_DIR}" --target package
            DEPENDS ${Application_Name}
            COMMENT "Packing ${Application_Name}")
    message("")
ENDIF ()

# check if Doxygen is installed
if (BUILD_DOC)
    message(STATUS "config of DOxygen build")
    find_package(Doxygen)
    if (DOXYGEN_FOUND)
        # SET input and output files
        SET(DOXYGEN_IN ${CMAKE_CURRENT_SOURCE_DIR}/docs/Doxyfile.in)
        SET(DOXYGEN_OUT ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile)

        # request to configure the file
        configure_file(${DOXYGEN_IN} ${DOXYGEN_OUT} @ONLY)

        # note the option ALL which allows to build the docs together with the application
        add_custom_target(doc_doxygen ALL
                COMMAND ${DOXYGEN_EXECUTABLE} ${DOXYGEN_OUT}
                WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                COMMENT "Generating API documentation with Doxygen"
                VERBATIM)
        message(STATUS "Successfully configured doxygen")
    else (DOXYGEN_FOUND)
        message(STATUS "Doxygen need to be installed to generate the doxygen documentation")
    endif (DOXYGEN_FOUND)
    message("")
endif (BUILD_DOC)

# Test Cases
if (TESTS)
    #    include(GoogleTest)
    message(STATUS "Configuring GTEST")
    find_package(GTest)
    if (GTEST_FOUND)

        mark_as_advanced(
                BUILD_GMOCK BUILD_GTEST BUILD_SHARED_LIBS
                gmock_build_tests gtest_build_samples gtest_build_tests
                gtest_disable_pthreads gtest_force_shared_crt gtest_hide_internal_symbols
        )

        enable_testing()

        macro(package_add_test TESTNAME)
            # create an exectuable in which the tests will be stored
            add_executable(${TESTNAME} ${ARGN})
            # link the Google test infrastructure, mocking library, and a default main fuction to
            target_link_libraries(${TESTNAME} gtest gtest_main -lpthread -lm libdynuiprefresher api ${CURL_LIBRARIES} ${LIBCONFIG++_LIBRARIES})
            # see https://cmake.org/cmake/help/v3.10/module/GoogleTest.html for more options to pass to it
            gtest_discover_tests(${TESTNAME})

            add_custom_command(TARGET build-test
                    POST_BUILD
                    COMMAND "${CMAKE_COMMAND}" --build "${CMAKE_BINARY_DIR}" --target ${TESTNAME}
                    COMMAND "bin/${TESTNAME}" --gtest_output=xml:${TESTNAME}-report.xml)
        endmacro()

        add_custom_target(build-test
                DEPENDS ${PROJECT_NAME}
                COMMENT "Testing ${PROJECT_NAME}")

        package_add_test(test1 tests/UnitTest.cpp ${SOURCE})

    else ()
        message(STATUS "GTEST environment not found!")
    endif ()
ENDIF ()