cmake_minimum_required(VERSION 2.8.3)
project(dynamic_map_server)

find_package(catkin REQUIRED
        COMPONENTS
            roscpp
            std_msgs
            nav_msgs
            tf2
            message_generation
        )

find_package(SDL REQUIRED)
find_package(SDL_image REQUIRED)

find_package(PkgConfig REQUIRED)
pkg_check_modules(YAMLCPP yaml-cpp REQUIRED)
if(YAMLCPP_VERSION VERSION_GREATER "0.5.0")
    add_definitions(-DHAVE_YAMLCPP_GT_0_5_0)
endif()

add_service_files(
    FILES
    SaveMap.srv
)

generate_messages(
    DEPENDENCIES
    std_msgs
)

catkin_package(
    INCLUDE_DIRS
        include
    LIBRARIES
        map_server_image_loader
    CATKIN_DEPENDS
        roscpp
        std_msgs
        nav_msgs
        tf2
        message_runtime
)

include_directories(
    include
    ${catkin_INCLUDE_DIRS}
    ${SDL_INCLUDE_DIR}
    ${SDL_IMAGE_INCLUDE_DIRS}
    ${YAMLCPP_INCLUDE_DIRS}
)

add_library(map_server_image_loader src/image_loader.cpp)
add_dependencies(map_server_image_loader ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
target_link_libraries(map_server_image_loader
    ${catkin_LIBRARIES}
    ${SDL_LIBRARY}
    ${SDL_IMAGE_LIBRARIES}
)

add_executable(map_server src/main.cpp)
add_dependencies(map_server ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
target_link_libraries(map_server
    map_server_image_loader
    ${YAMLCPP_LIBRARIES}
    ${catkin_LIBRARIES}
)

# copy test data to same place as tests are run
function(copy_test_data)
    cmake_parse_arguments(PROJECT "" "" "FILES" ${ARGN})
    foreach(datafile ${PROJECT_FILES})
        file(COPY ${datafile} DESTINATION ${PROJECT_BINARY_DIR}/test)
    endforeach()
endfunction()

## Tests
if(CATKIN_ENABLE_TESTING)
    copy_test_data( FILES
        test/testmap.bmp
        test/testmap.png )
    catkin_add_gtest(${PROJECT_NAME}_utest test/utest.cpp test/test_constants.cpp)
    target_link_libraries(${PROJECT_NAME}_utest
        map_server_image_loader
        ${SDL_LIBRARY}
        ${SDL_IMAGE_LIBRARIES}
    )

    add_executable(rtest test/rtest.cpp test/test_constants.cpp)
    add_dependencies(rtest ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})
    target_link_libraries( rtest
        gtest
        ${catkin_LIBRARIES}
    )

    # This has to be done after we've already built targets, or catkin variables get broken
    find_package(rostest REQUIRED)
    add_rostest(test/rtest.xml)
endif()

# Install libraries
install(TARGETS map_server_image_loader
    ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
    LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
    RUNTIME DESTINATION ${CATKIN_GLOBAL_BIN_DESTINATION}
)

# Install executable
install(TARGETS map_server
    RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)

## Install project namespaced headers
install(DIRECTORY include/map_server/
    DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
    FILES_MATCHING PATTERN "*.h"
    PATTERN ".svn" EXCLUDE
)

## Install excutable python script
install(
    PROGRAMS
        scripts/crop_map
    DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)
