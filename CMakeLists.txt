cmake_minimum_required(VERSION 3.16.3)
project(tensorboard_logger)

option(BUILD_TEST "Build test" OFF)

if(APPLE)
    find_package(Protobuf CONFIG REQUIRED)
else()
    find_package(Protobuf REQUIRED)
endif()

# -----------------------------------------------------------------------------
# Building the tensorboard_logger library
# -----------------------------------------------------------------------------
if(Protobuf_FOUND)
    message(STATUS "Found Protobuf ${Protobuf_VERSION}")
    message(STATUS "Found Protobuf Headers ${PROTOBUF_INCLUDE_DIRS}")
    get_target_property(PROTOBUF_INCLUDE_DIRS protobuf::libprotobuf INTERFACE_INCLUDE_DIRECTORIES)
    message(STATUS "Found Protobuf Include Dirs ${PROTOBUF_INCLUDE_DIRS}")
else()
    message(FATAL_ERROR "Protobuf not found")
endif()

file(GLOB protos "${CMAKE_CURRENT_SOURCE_DIR}/proto/*.proto")

foreach(FILE ${protos})
message(STATUS "Generating ${FILE}")
endforeach()

add_library(tensorboard_logger
    "src/crc.cc"
    "src/tensorboard_logger.cc"
)


if(Protobuf_VERSION VERSION_LESS "3.21.12")
    message(STATUS "Using protobuf_generate_cpp")
    protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS ${protos})
    target_link_libraries(tensorboard_logger PUBLIC ${Protobuf_LIBRARIES})
    target_sources(tensorboard_logger PRIVATE ${PROTO_SRCS})
    target_include_directories(tensorboard_logger
            PUBLIC
            $<INSTALL_INTERFACE:include>
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
            $<BUILD_INTERFACE:${PROTOBUF_INCLUDE_DIR}>
            $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
    )
else()
    message(STATUS "Using protobuf_generate")
    protobuf_generate(
            LANGUAGE cpp
            TARGET tensorboard_logger
            PROTOS ${protos}
            IMPORT_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/proto
    )
    target_include_directories(tensorboard_logger
            PUBLIC
            $<INSTALL_INTERFACE:include>
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
            $<BUILD_INTERFACE:${PROTOBUF_INCLUDE_DIRS}>
            $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
    )
    target_link_libraries(tensorboard_logger PUBLIC protobuf::libprotobuf)
endif()



target_compile_features(tensorboard_logger PRIVATE cxx_std_11)
# target_compile_options(tensorboard_logger PRIVATE -Wall -O2)


if (BUILD_TEST)
    add_executable(tensorboard_logger_test tests/test_tensorboard_logger.cc)
    target_compile_features(tensorboard_logger_test PRIVATE cxx_std_11)
    target_compile_options(tensorboard_logger_test PRIVATE -Wall -O2)
    target_include_directories(tensorboard_logger_test
    PRIVATE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
    )
    target_link_libraries(tensorboard_logger_test tensorboard_logger)
endif()

# -----------------------------------------------------------------------------
# Installing the tensorboard_logger library
# -----------------------------------------------------------------------------

install(
    TARGETS tensorboard_logger
    EXPORT tensorboard_logger_Targets
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    INCLUDES DESTINATION include
)

include(CMakePackageConfigHelpers)
configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/tensorboard_loggerConfig.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/tensorboard_loggerConfig.cmake"
    INSTALL_DESTINATION lib/cmake/tensorboard_logger
)

install(
    EXPORT tensorboard_logger_Targets
    FILE tensorboard_loggerTargets.cmake
    NAMESPACE tensorboard_logger::
    DESTINATION lib/cmake/tensorboard_logger
)

install(
    FILES "${CMAKE_CURRENT_BINARY_DIR}/tensorboard_loggerConfig.cmake"
    DESTINATION lib/cmake/tensorboard_logger
)

install(
    DIRECTORY include/
    DESTINATION include
)

install(
    DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/"
    DESTINATION include/tensorboard_logger
    FILES_MATCHING
    PATTERN "*.pb.h"
    PATTERN "CMakeFiles" EXCLUDE
)
