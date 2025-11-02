cmake_minimum_required(VERSION 3.15)
if(CMAKE_VERSION GREATER_EQUAL 3.31)
    cmake_policy(SET CMP0177 NEW) # install destination
endif ()

project(elsdc VERSION 0.9.0 LANGUAGES C)

option(BUILD_SHARED_LIBS "Build shared library" ON)
option(ELSDC_BUILD_WITH_COVERAGE "Build with code coverage support" OFF)
option(ELSDC_ENABLE_WARNINGS_AS_ERRORS "Enable warnings as errors" OFF)
option(ELSDC_BUILD_APP "Build main applications" ON)
option(ELSDC_INSTALL "Install the library and headers" ON)
option(ELSDC_BUILD_DOCS "Build documentation" OFF)
option(ELSDC_BUILD_TESTS "Build tests" OFF)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build type" FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug;Release;RelWithDebInfo;MinSizeRel")
endif()

# coverage is only supported for Debug and RelWithDebInfo builds with GCC or Clang
if(ELSDC_BUILD_WITH_COVERAGE)
    if(NOT (CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU"))
        message(WARNING "Code coverage is only supported with GCC or Clang. Disabling ELSDC_BUILD_WITH_COVERAGE.")
        set(ELSDC_BUILD_WITH_COVERAGE OFF)
    elseif(NOT (CMAKE_BUILD_TYPE MATCHES "Debug|RelWithDebInfo"))
        message(WARNING "Code coverage is only supported for Debug or RelWithDebInfo builds. Disabling ELSDC_BUILD_WITH_COVERAGE.")
        set(ELSDC_BUILD_WITH_COVERAGE OFF)
    endif()
endif()

message(STATUS "ELSDc is main project: ${ELSDC_IS_MAIN_PROJECT}")
message(STATUS "CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}")
message(STATUS "ELSDc version: ${PROJECT_VERSION}")
message(STATUS "Build shared libraries: ${BUILD_SHARED_LIBS}")
message(STATUS "Build main application: ${ELSDC_BUILD_APP}")
message(STATUS "Treat warnings as errors: ${ELSDC_ENABLE_WARNINGS_AS_ERRORS}")
message(STATUS "Install the library: ${ELSDC_INSTALL}")
message(STATUS "Build documentation: ${ELSDC_BUILD_DOCS}")
message(STATUS "Build tests: ${ELSDC_BUILD_TESTS}")
message(STATUS "Build with code coverage: ${ELSDC_BUILD_WITH_COVERAGE}")

set(ELSDC_C_STANDARD 17)
if(NOT DEFINED CMAKE_C_STANDARD)
  set(CMAKE_C_STANDARD ${ELSDC_C_STANDARD})
  set(CMAKE_C_STANDARD_REQUIRED ON)
endif()
message(STATUS "Using C standard: ${CMAKE_C_STANDARD}")

set(ELSDC_C_FEATURE c_std_${CMAKE_C_STANDARD})

set(CMAKE_C_STANDARD ${CMAKE_C_STANDARD})
set(CMAKE_C_STANDARD_REQUIRED ON)

# compile options for all targets depending on the compiler and the system
if(CMAKE_C_COMPILER_ID MATCHES "Clang|GNU")
    set(ELSDC_COMPILE_OPTIONS "-Wall;-Wextra;-pedantic;-Wno-comment;-Wshadow;-Wnon-virtual-dtor;-Wpedantic;-Wconversion;-Wmisleading-indentation;-Wsign-conversion;-Wfloat-equal")
    if(ELSDC_BUILD_WITH_COVERAGE)
        list(APPEND ELSDC_COMPILE_OPTIONS "--coverage")
    endif()
    if(ELSDC_ENABLE_WARNINGS_AS_ERRORS)
        list(APPEND ELSDC_COMPILE_OPTIONS "-Werror")
    endif()
elseif(MSVC)
    set(ELSDC_COMPILE_OPTIONS "/W4;/permissive-")
    if(ELSDC_ENABLE_WARNINGS_AS_ERRORS)
        list(APPEND ELSDC_COMPILE_OPTIONS "/WX")
    endif()
endif()
if(APPLE)
    list(APPEND ELSDC_COMPILE_OPTIONS "-Wno-deprecated-declarations")
endif()

# compile definitions for all targets depending on the compiler and the system
set(ELSDC_COMPILE_DEFINITIONS "")
if(MSVC)
    list(APPEND ELSDC_COMPILE_DEFINITIONS "-DNOMINMAX;-D_USE_MATH_DEFINES;-D_CRT_SECURE_NO_WARNINGS;-DAR_LIBRARY")
endif()

if(BUILD_SHARED_LIBS)
    if(WIN32)
        # export all symbols on a windows build of dynamic libs
        set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS  ON)
    endif()
endif()

find_package(LAPACK REQUIRED)

# Generate version header
set(ELSDC_generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(ELSDC_version_header_file "${ELSDC_generated_dir}/elsdc/version.h")
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/version.h.in"
    "${ELSDC_version_header_file}"
    @ONLY
)

set(ELSDC_SOURCES
    src/curve_grow.c
    src/ellipse_fit.c
    src/elsdc.c
    src/gauss.c
    src/image.c
    src/iterator.c
    src/lapack_wrapper.c
    src/misc.c
    src/pgm.c
    src/polygon.c
    src/rectangle.c
    src/ring.c
    src/svg.c
)

set(ELSDC_HEADERS
    src/curve_grow.h
    src/ellipse_fit.h
    src/elsdc.h
    src/gauss.h
    src/image.h
    src/iterator.h
    src/lapack_wrapper.h
    src/misc.h
    src/pgm.h
    src/polygon.h
    src/rectangle.h
    src/ring.h
    src/svg.h
)

add_library(elsdc ${ELSDC_SOURCES})
target_include_directories(elsdc PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
    $<BUILD_INTERFACE:${ELSDC_generated_dir}>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(elsdc PRIVATE LAPACK::LAPACK)
target_compile_features(elsdc PRIVATE ${ELSDC_C_FEATURE})
target_compile_definitions(elsdc PRIVATE ${ELSDC_COMPILE_DEFINITIONS})
target_compile_options(elsdc PRIVATE ${ELSDC_COMPILE_OPTIONS})
set_target_properties(elsdc PROPERTIES
  DEBUG_POSTFIX "d"
  VERSION ${PROJECT_VERSION}
  SOVERSION ${PROJECT_VERSION})

set(ELSDC_namespace "${PROJECT_NAME}::")
add_library(${ELSDC_namespace}elsdc ALIAS elsdc)

if(ELSDC_BUILD_APP)
    add_executable(elsdc_main src/main.c)
    target_link_libraries(elsdc_main PRIVATE elsdc)
    target_compile_features(elsdc_main PRIVATE ${ELSDC_C_FEATURE})
    target_compile_definitions(elsdc_main PRIVATE ${ELSDC_COMPILE_DEFINITIONS})
    target_compile_options(elsdc_main PRIVATE ${ELSDC_COMPILE_OPTIONS})
endif ()

if (ELSDC_INSTALL)
    include(GNUInstallDirs)

    ## Automatically generated files are stored here
    set(ELSDC_generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

    ####
    # Installation
    set(ELSDC_config_install_dir "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")
    set(ELSDC_include_install_dir "${CMAKE_INSTALL_INCLUDEDIR}/")

    # Configuration
    set(ELSDC_version_config "${ELSDC_generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
    set(ELSDC_project_config "${ELSDC_generated_dir}/${PROJECT_NAME}Config.cmake")
    set(ELSDC_targets_export_name "${PROJECT_NAME}Targets")

    # Include module with function 'write_basic_package_version_file'
    include(CMakePackageConfigHelpers)

    # Configure '<PROJECT-NAME>ConfigVersion.cmake'
    write_basic_package_version_file(
        "${ELSDC_version_config}" VERSION ${PROJECT_VERSION} COMPATIBILITY SameMajorVersion
    )

    # Configure '<PROJECT-NAME>Config.cmake'
    configure_package_config_file(
        "${CMAKE_SOURCE_DIR}/cmake/Config.cmake.in"
        "${ELSDC_project_config}"
        INSTALL_DESTINATION "${ELSDC_config_install_dir}"
    )

    # Targets:
    install(
        TARGETS elsdc
        EXPORT "${ELSDC_targets_export_name}"
        LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
        INCLUDES DESTINATION "${ELSDC_include_install_dir}"
    )

    # Headers:
    install(
#        DIRECTORY "${PROJECT_SOURCE_DIR}/include/${PROJECT_NAME}"
#        TYPE INCLUDE
        FILES ${ELSDC_HEADERS}
        DESTINATION "${ELSDC_include_install_dir}/${PROJECT_NAME}"

    )
    # install generated version header
    install(
        FILES "${ELSDC_version_header_file}"
        DESTINATION "${ELSDC_include_install_dir}/${PROJECT_NAME}")


    # Config
    install(
        FILES "${ELSDC_project_config}" "${ELSDC_version_config}"
        DESTINATION "${ELSDC_config_install_dir}"
    )

    # Config
    install(
        EXPORT "${ELSDC_targets_export_name}"
        NAMESPACE "${ELSDC_namespace}"
        DESTINATION "${ELSDC_config_install_dir}"
    )

endif()