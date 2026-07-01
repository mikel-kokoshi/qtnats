cmake_minimum_required(VERSION 3.16)

project(QtNatsProject VERSION 0.1.0)

include(CMakePrintHelpers)
include(FetchContent)
include(GenerateExportHeader)
include(CMakePackageConfigHelpers)
include(GNUInstallDirs)

set(default_build_type "Release")

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED True)
set(CMAKE_AUTOMOC ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)  # to find qtnats_export.h

# according to https://gitlab.kitware.com/cmake/cmake/-/issues/20843 I need 2 find_package's to detect Qt5/Qt6
find_package(QT NAMES Qt6 Qt5 COMPONENTS Core REQUIRED)
find_package(Qt${QT_VERSION_MAJOR} COMPONENTS Core Test REQUIRED)
message("Using Qt version ${QT_VERSION}")

# Check if the QT quick and QML components are found
find_package(Qt_Qml COMPONENTS Qml OPTIONAL)
find_package(Qt_Quick COMPONENTS Quick OPTIONAL)

if(Qt_Qml_FOUND AND Qt_Quick_FOUND)
    option(BUILD_QMLNATS "Build the QML NATS plugin (Qt6-only)" OFF)
endif()

option(QTNATS_BUILD_LIB_STATIC "Build static QtNats linked with static Nats library" OFF)
option(QTNATS_BUILD_LIB_SHARED "Build shared QtNats linked with shared Nats library" ON)
option(QTNATS_BUILD_TESTS "Build test programs" ON)

find_package(cnats PATHS 3.13.0 OPTIONAL "$ENV{cnats_ROOT}")
if(cnats_FOUND)
    message(STATUS "Using cnats module from ${cnats_DIR}")
else()
    # show that we're git-cloning cnats
    set(FETCHCONTENT_QUIET FALSE)
    FetchContent_Declare(
      cnats
      GIT_REPOSITORY https://github.com/nats-io/nats.c.git
      GIT_TAG        v3.13.0
      GIT_SHALLOW    ON
    )

    set(NATS_UPDATE_VERSION ON)
    set(NATS_BUILD_WITH_TLS OFF)
    set(NATS_BUILD_TLS_FORCE_HOST_VERIFY OFF)
    set(NATS_BUILD_EXAMPLES OFF)
    set(NATS_BUILD_STREAMING OFF)
    if (QTNATS_BUILD_LIB_SHARED)
        set(NATS_BUILD_LIB_SHARED ON)
    endif()
    if (QTNATS_BUILD_LIB_STATIC)
        set(NATS_BUILD_LIB_STATIC ON)
    endif()

    set(BUILD_TESTING OFF)  # do not build cnats tests

    FetchContent_MakeAvailable(cnats)
endif()

include_directories(${cnats_SOURCE_DIR}/src src)

# ------------- qtnats library ----------------------
file(GLOB SOURCES "src/*.cpp")
file(GLOB HEADERS "src/*.h")

if (QTNATS_BUILD_LIB_STATIC)
    add_library(qtnats_static STATIC ${SOURCES} ${HEADERS})
    target_sources(qtnats_static PUBLIC FILE_SET headers TYPE HEADERS BASE_DIRS src FILES ${HEADERS})
    target_include_directories(qtnats_static PRIVATE $<TARGET_PROPERTY:cnats::nats_static,INTERFACE_INCLUDE_DIRECTORIES>)
    target_link_libraries(qtnats_static cnats::nats_static $<TARGET_PROPERTY:cnats::nats_static,INTERFACE_LINK_LIBRARIES> Qt::Core)
    # libnats_static.a needs to link the "ws2_32" library when building on Windows.
    if(WIN32)
        target_link_libraries(qtnats_static ws2_32)
    endif()
    GENERATE_EXPORT_HEADER(qtnats_static BASE_NAME qtnats)
    target_include_directories(qtnats_static PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/qtnats>
    )
    set(TARGETS qtnats_static)
endif()

if (QTNATS_BUILD_LIB_SHARED)
    add_library(qtnats SHARED ${SOURCES} ${HEADERS})
    target_sources(qtnats PUBLIC FILE_SET headers TYPE HEADERS BASE_DIRS src FILES ${HEADERS})
    target_include_directories(qtnats PRIVATE $<TARGET_PROPERTY:cnats::nats,INTERFACE_INCLUDE_DIRECTORIES>)
    target_link_libraries(qtnats cnats::nats $<TARGET_PROPERTY:cnats::nats,INTERFACE_LINK_LIBRARIES> Qt::Core)
    GENERATE_EXPORT_HEADER(qtnats)
    target_include_directories(qtnats PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/qtnats>
    )
    set(TARGETS qtnats ${TARGETS})
endif()

# this has no effect on Windows due to https://gitlab.kitware.com/cmake/cmake/-/issues/19618
set_target_properties(${TARGETS} PROPERTIES VERSION ${PROJECT_VERSION} SOVERSION 0.1)

install(TARGETS ${TARGETS}
    EXPORT QtNatsTargets
    FILE_SET headers DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/qtnats
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/qtnats_export.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/qtnats
)

install(EXPORT QtNatsTargets
    FILE QtNatsTargets.cmake
    NAMESPACE QtNats::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/QtNats
)

configure_package_config_file(cmake/QtNatsConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/QtNatsConfig.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/QtNats
)
write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/QtNatsConfigVersion.cmake
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)
install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/QtNatsConfig.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/QtNatsConfigVersion.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/QtNats
)

if(QTNATS_BUILD_LIB_SHARED)
    configure_file(cmake/qtnats.pc.in qtnats.pc @ONLY)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/qtnats.pc
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
    )
endif()
if(QTNATS_BUILD_LIB_STATIC)
    configure_file(cmake/qtnats_static.pc.in qtnats_static.pc @ONLY)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/qtnats_static.pc
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
    )
endif()

# ------------- unit tests ----------------------
if(QTNATS_BUILD_TESTS)
    add_executable(test_core test/test_core.cpp)
    add_test(NAME test_core COMMAND test_core)
    target_link_libraries(test_core PRIVATE qtnats Qt::Test)

    add_executable(test_jetstream test/test_jetstream.cpp)
    add_test(NAME test_jetstream COMMAND test_jetstream)
    target_link_libraries(test_jetstream PRIVATE qtnats Qt::Test)
endif()

if(BUILD_QMLNATS)
    if(${QT_VERSION_MAJOR} EQUAL 6)
        add_subdirectory(qml)
    else()
        message(FATAL_ERROR "NATS QML is supported only with Qt6")
    endif()
endif()
