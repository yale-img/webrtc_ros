cmake_minimum_required(VERSION 2.8.12)
cmake_policy(SET CMP0048 NEW)
project(webrtc_ros VERSION 59.0.4)

find_package(catkin REQUIRED COMPONENTS
  async_web_server_cpp
  cv_bridge
  image_transport
  message_generation
  nodelet
  roscpp
  std_msgs
)
find_package(webrtc REQUIRED)
find_package(X11 REQUIRED)
find_package(jsoncpp REQUIRED)

add_message_files(
    FILES
    IceServer.msg
)

add_service_files(
    FILES
    GetIceServers.srv
)

generate_messages(
    DEPENDENCIES
    std_msgs
)

catkin_package(
  CATKIN_DEPENDS
    async_web_server_cpp
    cv_bridge
    image_transport
    message_runtime
    nodelet
    roscpp
    std_msgs
  DEPENDS webrtc
)

catkin_install_python(
    PROGRAMS scripts/ice_server_service.py
    DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)

###########
## Build ##
###########

include_directories(
  include
  ${catkin_INCLUDE_DIRS}
  ${webrtc_INCLUDE_DIRS}
)
add_definitions(${webrtc_DEFINITIONS})

# This library contains code that cannot be compiled with c++11 enabled
# This is due to ABI incompatibilities between the two
# While most of ROS appears to work with the webrtc compiler options,
# this is needed for async_web_server_cpp, which relies on Boost ASIO types
# Boost ASIO cannot be mixed between c++11 and non-c++11 compilation units


add_library(${PROJECT_NAME}_server_noncpp11
  src/webrtc_web_server.cpp
)
add_dependencies(${PROJECT_NAME}_server_noncpp11 ${catkin_EXPORTED_TARGETS} ${${PROJECT_NAME}_EXPORTED_TARGETS})
target_link_libraries(${PROJECT_NAME}_server_noncpp11
  ${catkin_LIBRARIES}
)



# This library is built with the same options used to build the webrtc library
add_library(${PROJECT_NAME}_server
  src/configure_message.cpp
  src/ice_candidate_message.cpp
  src/image_transport_factory.cpp
  src/ros_video_capturer.cpp
  src/ros_video_renderer.cpp
  src/sdp_message.cpp
  src/webrtc_client.cpp
  src/webrtc_ros_message.cpp
  src/webrtc_ros_server.cpp
  src/webrtc_ros_json_parser.cpp
)
add_dependencies(${PROJECT_NAME}_server ${catkin_EXPORTED_TARGETS} ${${PROJECT_NAME}_EXPORTED_TARGETS})
target_link_libraries(${PROJECT_NAME}_server
  ${catkin_LIBRARIES}
  ${webrtc_LIBRARIES}
  ${X11_LIBRARIES}
  jsoncpp_lib
)

set_target_properties(${PROJECT_NAME}_server PROPERTIES COMPILE_OPTIONS "-std=c++14")

add_executable(${PROJECT_NAME}_server_node
  src/webrtc_ros_server_node.cpp
)
target_link_libraries(${PROJECT_NAME}_server_node
  ${catkin_LIBRARIES}
  ${PROJECT_NAME}_server
  ${PROJECT_NAME}_server_noncpp11
)
set_target_properties(${PROJECT_NAME}_server_node PROPERTIES COMPILE_OPTIONS "-std=c++14")

#add_library(${PROJECT_NAME}_server_nodelet
#  src/webrtc_ros_server_nodelet.cpp
#)
#target_link_libraries(${PROJECT_NAME}_server_nodelet
#  ${catkin_LIBRARIES}
#  ${PROJECT_NAME}_server
#)

#############
## Install ##
#############

# Mark nodes and libraries for installation
install(TARGETS
  ${PROJECT_NAME}_server
  ${PROJECT_NAME}_server_node
  ${PROJECT_NAME}_server_noncpp11

  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)

# Mark nodelet plugin configuration for installation
install(FILES
  nodelet_plugins.xml
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}
)

# Mark cpp header files for installation
install(DIRECTORY include/${PROJECT_NAME}/
  DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
  FILES_MATCHING PATTERN "*.h"
)

# Install web files
install(DIRECTORY web
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}
  PATTERN "Gruntfile.js" EXCLUDE
  PATTERN "node_modules" EXCLUDE
  PATTERN "package.json" EXCLUDE
  PATTERN ".gitignore" EXCLUDE
)

# Install license files
install(FILES
  LICENSE
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}
)
install(FILES
  LICENSE_webrtc_adapter
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}
)
