cmake_minimum_required(VERSION 3.0)

### Hunter snapshot that will be used ###
include("cmake/HunterGate.cmake")

## This is synonmous with building the examples, off by default for package builds
option(DEST_BUILD_EXAMPLES "Build DEST examples." OFF)
option(DEST_WITH_OPENCV "Build DEST with OpenCV support." OFF)

set(HUNTER_DEST_URL "https://github.com/ruslo/hunter/archive/v0.18.9.tar.gz")
set(HUNTER_DEST_SHA1 "90bf10e2895531145332a8963725a351d7345c58")

if(DEST_WITH_OPENCV)
  HunterGate(
    URL "${HUNTER_DEST_URL}"
    SHA1 "${HUNTER_DEST_SHA1}"
    LOCAL # Disable some opencv functionality for sample apps as workaround for build errors
    )
else()
  # Lib only builds don't need a LOCAL config
  HunterGate(
    URL "${HUNTER_DEST_URL}"
    SHA1 "${HUNTER_DEST_SHA1}"
    )
endif()

project(deformable-shape-tracking VERSION 0.8.0)

set(DEST_LINK_TARGETS)

option(DEST_BUILD_TESTS "Build tests." OFF)

### Eigen3 (Header only)
hunter_add_package(Eigen)
find_package(Eigen3 REQUIRED)

### flatbuffers (header only)
hunter_add_package(flatbuffers)
find_package(flatbuffers CONFIG REQUIRED)

### OpenCV
if(DEST_WITH_OPENCV)

  hunter_add_package(OpenCV)
  find_package(OpenCV REQUIRED)

  include_directories(${OpenCV_INCLUDE_DIRS})
  list(APPEND DEST_LINK_TARGETS ${OpenCV_LIBRARIES})
  message(STATUS "Compiling with OpenCV support")

  hunter_add_package(tinydir)
  find_package(tinydir REQUIRED)
  list(APPEND DEST_LINK_TARGETS tinydir::tinydir)

else()
  message(STATUS "Compiling without OpenCV support")
endif()

option(DEST_WITH_OPENMP "Build DEST with OpenMP support" OFF)
if(DEST_WITH_OPENMP)
    find_package(OpenMP)
    if (OPENMP_FOUND)
        set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
        set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
        message(STATUS "Compiling with OpenMP support")
    else()
        message(STATUS "No OpenMP support detected.")
    endif()
else()
    message(STATUS "Compiling without OpenMP support")
endif()

#include_directories("${CMAKE_CURRENT_BINARY_DIR}" "inc" "ext")

include_directories("${CMAKE_CURRENT_BINARY_DIR}" "inc" )

# Library
option(DEST_VERBOSE "Build DEST in verbose mode." OFF)
configure_file(inc/dest/core/config.h.in dest/core/config.h)

set(DEST_SOURCES
  ${CMAKE_CURRENT_BINARY_DIR}/dest/core/config.h
  inc/dest/dest.h
  inc/dest/io/dest_io.fbs
  inc/dest/io/dest_io_generated.h
  inc/dest/io/rect_io.h
  inc/dest/core/image.h
  inc/dest/core/regressor.h
  inc/dest/core/shape.h
  inc/dest/core/tracker.h
  inc/dest/core/tree.h
  inc/dest/io/database_io.h
  inc/dest/io/matrix_io.h
  inc/dest/util/log.h
  src/core/image.cpp
  src/core/regressor.cpp
  src/core/shape.cpp
  src/core/tracker.cpp
  src/core/tree.cpp
  src/io/database_io.cpp
  )

if(DEST_WITH_OPENCV)
  # These files are only required for
  list(APPEND DEST_SOURCES
    inc/dest/core/tester.h
    inc/dest/core/training_data.h
    inc/dest/face/face_detector.h
    inc/dest/util/convert.h
    inc/dest/util/draw.h
    inc/dest/util/glob.h
    inc/dest/util/triangulate.h
    src/core/tester.cpp
    src/core/training_data.cpp
    src/face/face_detector.cpp
    src/io/rect_io.cpp
    src/util/draw.cpp
    src/util/glob.cpp
    src/util/triangulate.cpp
    )
endif(DEST_WITH_OPENCV)

add_library(dest ${DEST_SOURCES})
target_link_libraries(dest PUBLIC ${DEST_LINK_TARGETS} Eigen3::Eigen flatbuffers::flatbuffers)

if(HUNTER_ENABLED AND DEST_WITH_OPENCV)
  target_compile_definitions(dest PRIVATE DEST_INTERNAL_TINYDIR=0)
else()
  target_compile_definitions(dest PRIVATE DEST_INTERNAL_TINYDIR=1)
endif()

if (WIN32)
  target_compile_definitions(dest PUBLIC _SCL_SECURE_NO_WARNINGS _CRT_SECURE_NO_WARNINGS)
endif()

# Samples
if(DEST_WITH_OPENCV AND DEST_BUILD_EXAMPLES)
    add_executable(dest_gen_rects examples/dest_gen_rects.cpp)
    target_link_libraries(dest_gen_rects dest ${DEST_LINK_TARGETS})

    add_executable(dest_train examples/dest_train.cpp)
    target_link_libraries(dest_train dest ${DEST_LINK_TARGETS})

    add_executable(dest_evaluate examples/dest_evaluate.cpp)
    target_link_libraries(dest_evaluate dest ${DEST_LINK_TARGETS})

    add_executable(dest_align examples/dest_align.cpp)
    target_link_libraries(dest_align dest ${DEST_LINK_TARGETS})

    add_executable(dest_track_video examples/dest_track_video.cpp)
    target_link_libraries(dest_track_video dest ${DEST_LINK_TARGETS})

    add_executable(dest_show_landmarks examples/dest_show_landmarks.cpp)
    target_link_libraries(dest_show_landmarks dest ${DEST_LINK_TARGETS})

    add_executable(dest_expression_transfer examples/dest_expression_transfer.cpp)
    target_link_libraries(dest_expression_transfer dest ${DEST_LINK_TARGETS})

    add_executable(dest_face_swap examples/dest_face_swap.cpp)
    target_link_libraries(dest_face_swap dest ${DEST_LINK_TARGETS})

endif()

# Tests
if(DEST_BUILD_TESTS)
  add_executable(dest_tests
    tests/catch.hpp
    tests/test_transform.cpp
    tests/test_image.cpp
    tests/test_shape.cpp
    tests/test_matrix_io.cpp
    tests/test_rect_io.cpp
    )
  target_link_libraries(dest_tests dest ${DEST_LINK_TARGETS})
endif(DEST_BUILD_TESTS)

## install

set(config_install_dir "lib/cmake/dest")
set(include_install_dir "include")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

# Configuration
set(version_config "${generated_dir}/destConfigVersion.cmake")
set(project_config "${generated_dir}/destConfig.cmake")
set(targets_export_name "destTargets")
set(namespace "dest::")

include(CMakePackageConfigHelpers)

write_basic_package_version_file(
  "${version_config}" COMPATIBILITY SameMajorVersion
  )

configure_package_config_file(
  "cmake/Config.cmake.in"
  "${project_config}"
  INSTALL_DESTINATION "${config_install_dir}"
  )

install(
  TARGETS dest
  EXPORT "${targets_export_name}"
  LIBRARY DESTINATION "lib"
  ARCHIVE DESTINATION "lib"
  RUNTIME DESTINATION "bin"
  INCLUDES DESTINATION "${include_install_dir}"
  )

configure_file(inc/dest/core/config.h.in "${include_install_dir}/dest/core/config.h")

install(
  DIRECTORY inc/dest/ # Trailing slash is required
  DESTINATION "${include_install_dir}/dest"
  FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
  )

## install runtime generated file
install(
  FILES "${CMAKE_CURRENT_BINARY_DIR}/dest/core/config.h"
  DESTINATION "${include_install_dir}/dest/core/"
  )

install(
  FILES "${project_config}" "${version_config}"
  DESTINATION "${config_install_dir}"
  )

install(
  EXPORT "${targets_export_name}"
  NAMESPACE "${namespace}"
  DESTINATION "${config_install_dir}"
  )
