cmake_minimum_required(VERSION 3.1.0)

project(Birdtray CXX)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

find_package(SQLite3 REQUIRED)
find_package(Qt5Core CONFIG REQUIRED)
find_package(Qt5Gui CONFIG REQUIRED)
find_package(Qt5Widgets CONFIG REQUIRED)
find_package(Qt5Network CONFIG REQUIRED)
find_package(Qt5LinguistTools CONFIG QUIET)
set(REQUIRED_MODULES ${SQLITE3_LIBRARIES} Qt5::Core Qt5::Gui Qt5::Widgets Qt5::Network)

set(PLATFORM_SOURCES)
set(PLATFORM_HEADERS)
set(EXECUTABLE_OPTIONS)

option( OPT_THUNDERBIRD_CMDLINE "Default Thunderbird startup command-line" [])
option( OPT_THUNDERBIRD_PROFILE "Default Thunderbird profile path" [])

if (OPT_THUNDERBIRD_CMDLINE)
    message(STATUS "Setting Thunderbird command-line to ${OPT_THUNDERBIRD_CMDLINE}")
    add_definitions(-DOPT_THUNDERBIRD_CMDLINE="${OPT_THUNDERBIRD_CMDLINE}")
endif(OPT_THUNDERBIRD_CMDLINE)

if (OPT_THUNDERBIRD_PROFILE)
    message(STATUS "Setting Thunderbird command line to ${OPT_THUNDERBIRD_PROFILE}")
    add_definitions(-DOPT_THUNDERBIRD_PROFILE="${OPT_THUNDERBIRD_PROFILE}")
endif(OPT_THUNDERBIRD_PROFILE)

if(NOT Qt5LinguistTools_FOUND)
    message(STATUS "Qt5LinguistTools package not found: translations will not be available")
endif(NOT Qt5LinguistTools_FOUND)

if(WIN32)
    set(EXECUTABLE_OPTIONS WIN32)
    ENABLE_LANGUAGE(RC)
    add_definitions(-DUNICODE -DNOMINMAX)
    set(PLATFORM_SOURCES src/windowtools_win.cpp src/birdtrayeventfilter.cpp
            src/processhandle.cpp src/res/birdtray.rc)
    set(PLATFORM_HEADERS src/windowtools_win.h src/birdtrayeventfilter.h src/processhandle.h)
    find_package(Qt5WinExtras CONFIG REQUIRED)
    list(APPEND REQUIRED_MODULES Qt5::WinExtras)
else()
    find_package(Qt5X11Extras REQUIRED)
    find_library(X11 X11)
    list(APPEND REQUIRED_MODULES Qt5::X11Extras ${X11})
    set(PLATFORM_SOURCES src/windowtools_x11.cpp)
    set(PLATFORM_HEADERS src/windowtools_x11.h)
endif(WIN32)

set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)
include_directories(${SQLITE3_INCLUDE_DIRS} src)
add_definitions(-DQT_DEPRECATED_WARNINGS)

set(SOURCES
        src/colorbutton.cpp
        src/databaseaccounts.cpp
        src/databaseunreadfixer.cpp
        src/dialogaddeditaccount.cpp
        src/dialogaddeditnewemail.cpp
        src/dialogsettings.cpp
        src/main.cpp
        src/modelaccounttree.cpp
        src/modelnewemails.cpp
        src/morkparser.cpp
        src/setting_newemail.cpp
        src/settings.cpp
        src/sqlite_statement.cpp
        src/trayicon.cpp
        src/unreadmonitor.cpp
        src/utils.cpp
        src/windowtools.cpp
        src/autoupdater.cpp
        src/updatedialog.cpp
        src/updatedownloaddialog.cpp
        src/mailaccountdialog.cpp
        src/birdtrayapp.cpp
        src/dialogaddeditaccount.ui
        src/dialogaddeditnewemail.ui
        src/dialogsettings.ui
        src/updatedialog.ui
        src/updatedownloaddialog.ui
        src/mailaccountdialog.ui)

set(HEADERS
        src/colorbutton.h
        src/databaseaccounts.h
        src/databaseunreadfixer.h
        src/dialogaddeditaccount.h
        src/dialogaddeditnewemail.h
        src/dialogsettings.h
        src/modelaccounttree.h
        src/modelnewemails.h
        src/morkparser.h
        src/setting_newemail.h
        src/settings.h
        src/sqlite_statement.h
        src/trayicon.h
        src/unreadmonitor.h
        src/utils.h
        src/version.h
        src/windowtools.h
        src/autoupdater.h
        src/updatedialog.h
        src/updatedownloaddialog.h
        src/mailaccountdialog.h
        src/birdtrayapp.h)

set(RESOURCES src/resources.qrc)
source_group("Resources" FILES ${RESOURCES})
qt5_add_resources(RESOURCES_SOURCES ${RESOURCES})
set_source_files_properties(${RESOURCES_SOURCES} PROPERTIES GENERATED ON)

if(Qt5LinguistTools_FOUND)
    set(MAIN_TRANSLATION_FILES
            src/translations/main_en.ts
            src/translations/main_de.ts
            src/translations/main_nl.ts)
    set(DYN_TRANSLATION_FILES
            src/translations/dynamic_en.ts
            src/translations/dynamic_de.ts
            src/translations/dynamic_nl.ts)

    set_source_files_properties(${MAIN_TRANSLATION_FILES} PROPERTIES OUTPUT_LOCATION
            ${CMAKE_CURRENT_BINARY_DIR}/translations)
    set_source_files_properties(${DYN_TRANSLATION_FILES} PROPERTIES OUTPUT_LOCATION
            ${CMAKE_CURRENT_BINARY_DIR}/translations)

    qt5_create_translation(GEN_TRANSLATION_FILES
            ${MAIN_TRANSLATION_FILES}
            ${SOURCES} ${PLATFORM_SOURCES} ${HEADERS} ${PLATFORM_HEADERS}
            OPTIONS -locations none)
    qt5_add_translation(GEN_DYN_TRANSLATION_FILES ${DYN_TRANSLATION_FILES})
    list(APPEND GEN_TRANSLATION_FILES ${GEN_DYN_TRANSLATION_FILES})
endif(Qt5LinguistTools_FOUND)

add_executable(birdtray ${EXECUTABLE_OPTIONS}
        ${SOURCES} ${PLATFORM_SOURCES} ${RESOURCES_SOURCES} ${GEN_TRANSLATION_FILES}
        ${HEADERS} ${PLATFORM_HEADERS})
target_link_libraries(birdtray ${REQUIRED_MODULES})
if(MSVC)
    target_compile_options(birdtray PRIVATE /utf-8)
endif()

option(COMPILER_WARNINGS_AS_ERRORS "Fail the build on compiler warnings" OFF)
if(COMPILER_WARNINGS_AS_ERRORS)
    if(MSVC)
        target_compile_options(birdtray PRIVATE /W4 /WX)
    else()
        target_compile_options(birdtray PRIVATE -Wall -Wextra -Wpedantic -Werror)
    endif()
endif()

# Installation
if(WIN32)
    # Find ssl libraries
    find_library(SSL_LIBRARY NAMES ssl)
    get_filename_component(SSL_PATH ${SSL_LIBRARY} DIRECTORY)
    if(SSL_LIBRARY)
        STRING(REGEX REPLACE "/" "\\\\\\\\" SSL_PATH ${SSL_PATH})
    endif()

    # Create installer
    STRING(REGEX REPLACE "/" "\\\\\\\\" BIRDTRAY_EXE ${CMAKE_BINARY_DIR}/birdtray.exe)
    STRING(REGEX REPLACE "/" "\\\\\\\\" SQLITE3_PATH ${SQLITE3_LIBRARY})
    install(CODE "execute_process(
        COMMAND cmd /C buildInstaller.bat
            \"${BIRDTRAY_EXE}\" \"${SQLITE3_PATH}\" \"${SSL_PATH}\" --install
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/installer)")
else()
    include(GNUInstallDirs)
    install(TARGETS birdtray RUNTIME DESTINATION bin)
    install(FILES src/res/com.ulduzsoft.Birdtray.desktop
            DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/applications)
    install(FILES src/res/com.ulduzsoft.Birdtray.appdata.xml
            DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/metainfo)
    foreach(size 32 48 64 128)
        install(FILES src/res/icons/${size}/com.ulduzsoft.Birdtray.png
                DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/${size}x${size}/apps)
    endforeach()
    install (FILES src/res/birdtray.svg
             DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/scalable/apps 
             RENAME com.ulduzsoft.Birdtray.svg)
    install(DIRECTORY ${CMAKE_BINARY_DIR}/translations
            DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/ulduzsoft/birdtray)
endif()

if(Qt5LinguistTools_FOUND)
    add_custom_target(clean_translations COMMENT "Remove obsolete translation entries")
    foreach(TRANSLATION_FILE ${MAIN_TRANSLATION_FILES})
        get_filename_component(TRANSLATION_NAME ${TRANSLATION_FILE} NAME)
        set(TRANSLATION_LST_FILE
                "${CMAKE_CURRENT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/${TRANSLATION_NAME}_lst_file")
        if(NOT EXISTS ${TRANSLATION_LST_FILE})
            get_filename_component(TRANSLATION_NAME ${TRANSLATION_FILE} NAME_WE)
            set(TRANSLATION_LST_FILE
                    "${CMAKE_CURRENT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/${TRANSLATION_NAME}_lst_file")
        endif()
        add_custom_command(TARGET clean_translations
                COMMAND ${Qt5_LUPDATE_EXECUTABLE}
                ARGS -locations none -no-obsolete "@${TRANSLATION_LST_FILE}"
                -ts ${CMAKE_CURRENT_SOURCE_DIR}/${TRANSLATION_FILE}
                DEPENDS ${TRANSLATION_LST_FILE})
    endforeach()
endif()
