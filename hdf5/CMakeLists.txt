# sources
set(FILTER_SOURCES blosc_filter.c)
set(PLUGIN_SOURCES blosc_plugin.c)

include_directories("${PROJECT_SOURCE_DIR}/blosc")

# dependencies
if(MSVC)
    # FindHDF5.cmake does not find Windows installations. Try to
    # use an environment variable instead until the official "find"
    # file can be updated for Windows.
    #
    # Note that you have to set this environment variable by hand.
    file(TO_CMAKE_PATH "$ENV{HDF5_DIR}" HDF5_HINT)
    set(HDF5_DIR ${HDF5_HINT} CACHE STRING "Path to HDF5 CMake config directory.")
    find_package(HDF5 REQUIRED HINTS ${HDF5_DIR})
else(MSVC)
    find_package(HDF5 REQUIRED)
endif(MSVC)
include_directories(${HDF5_INCLUDE_DIRS})

# targets

# filter library
set(FILTER_NAME blosc_filter)

# shared
set(SHARED_FILTER_TARGET blosc_filter_shared)
add_library(${SHARED_FILTER_TARGET} SHARED ${FILTER_SOURCES})
set_target_properties(${SHARED_FILTER_TARGET} PROPERTIES OUTPUT_NAME ${FILTER_NAME})
if (MSVC)
    set_target_properties(${SHARED_FILTER_TARGET} PROPERTIES COMPILE_FLAGS -DDLL_EXPORT=TRUE)
endif(MSVC)
target_link_libraries(${SHARED_FILTER_TARGET} blosc_shared ${HDF5_LIBRARIES})

# static
if(BUILD_STATIC)
    set(STATIC_FILTER_TARGET blosc_filter_static)
    add_library(${STATIC_FILTER_TARGET} ${FILTER_SOURCES})
    set_target_properties(${STATIC_FILTER_TARGET} PROPERTIES OUTPUT_NAME ${FILTER_NAME})
    if (MSVC)
        set_target_properties(${STATIC_FILTER_TARGET} PROPERTIES PREFIX lib)
    endif()
    target_link_libraries(${STATIC_FILTER_TARGET} blosc_static ${HDF5_LIBRARIES})
endif(BUILD_STATIC)

# dynamic filter plugin
set(PLUGIN_TARGET blosc_plugin)
set(PLUGIN_NAME blosc_plugin)
add_library(${PLUGIN_TARGET} SHARED ${PLUGIN_SOURCES})
set_target_properties(${PLUGIN_TARGET} PROPERTIES OUTPUT_NAME ${PLUGIN_NAME})
target_link_libraries(${PLUGIN_TARGET} blosc_shared ${HDF5_LIBRARIES})

# Have to copy blosc dlls for Visual Studio
#
# TODO: Ensure that HDF5 dlls are either copied to the test directory or
#       available in via the PATH on Windows.
if(MSVC)
    add_custom_command(
        TARGET      ${SHARED_FILTER_TARGET}
        POST_BUILD
        COMMAND     ${CMAKE_COMMAND}
        ARGS        -E copy_if_different
                    "${PROJECT_BINARY_DIR}/blosc/\$\(Configuration\)/blosc.dll"
                    "${CMAKE_CURRENT_BINARY_DIR}/\$\(Configuration\)/blosc.dll")
                    
    add_custom_command(
        TARGET      ${PLUGIN_TARGET}
        POST_BUILD
        COMMAND     ${CMAKE_COMMAND}
        ARGS        -E copy_if_different
                    "${PROJECT_BINARY_DIR}/blosc/\$\(Configuration\)/blosc.dll"
                    "${CMAKE_CURRENT_BINARY_DIR}/\$\(Configuration\)/blosc.dll")
endif(MSVC)

# install
install(FILES blosc_filter.h DESTINATION include COMPONENT HDF5_FILTER_DEV)
install(TARGETS ${SHARED_FILTER_TARGET} DESTINATION lib COMPONENT HDF5_FILTER)
if(BUILD_STATIC)
    # TODO: Why is the component dev?
    install(TARGETS ${STATIC_FILTER_TARGET} DESTINATION lib COMPONENT HDF5_FILTER_DEV)
endif(BUILD_STATIC)
install(TARGETS ${PLUGIN_TARGET} DESTINATION lib COMPONENT HDF5_FILTER)

# test
if(BUILD_TESTS)
    
    set(PLUGIN_SOURCES filter_example.c)

    # shared filter
    SET(SHARED_EXAMPLE_TARGET filter_example_shared)
    add_executable(${SHARED_EXAMPLE_TARGET} ${PLUGIN_SOURCES})
    target_link_libraries(${SHARED_EXAMPLE_TARGET} ${SHARED_FILTER_TARGET} ${HDF5_LIBRARIES})
    add_test(test_hdf5_filter_shared ${SHARED_EXAMPLE_TARGET})

    # static filter
    if(BUILD_STATIC)
        SET(STATIC_EXAMPLE_TARGET filter_example_static)
        add_executable(${STATIC_EXAMPLE_TARGET} ${PLUGIN_SOURCES})
        target_link_libraries(${STATIC_EXAMPLE_TARGET} ${STATIC_FILTER_TARGET} ${HDF5_LIBRARIES})
        add_test(${STATIC_EXAMPLE_TARGET} filter_example_static)
    endif(BUILD_STATIC)

endif(BUILD_TESTS)
