cmake_minimum_required(VERSION 2.8.7)
project(lsd_slam_core)

# Set the build type. Options are:
#  Coverage : w/ debug symbols, w/o optimization, w/ code-coverage
#  Debug : w/ debug symbols, w/o optimization
#  Release : w/o debug symbols, w/ optimization
#  RelWithDebInfo : w/ debug symbols, w/ optimization
#  MinSizeRel : w/o debug symbols, w/ optimization, stripped binaries
set(CMAKE_BUILD_TYPE Release)
set(OpenCV_DIR "/home/jack/Documents/opencv3.4.2/OpenCV/build")
find_package(catkin REQUIRED COMPONENTS
  cv_bridge
  OpenCV
  dynamic_reconfigure
  sensor_msgs
  image_transport
  roscpp
  rosbag
  cmake_modules
  lsd_slam_viewer
)

find_package(Eigen3 REQUIRED)
find_package(X11 REQUIRED)
include(cmake/FindG2O.cmake)
include(cmake/FindSuiteParse.cmake)


message("-- CHOLMOD_INCLUDE_DIR : " ${CHOLMOD_INCLUDE_DIR})
message("-- CSPARSE_INCLUDE_DIR : " ${CSPARSE_INCLUDE_DIR})
message("-- G2O_INCLUDE_DIR : " ${G2O_INCLUDE_DIR})

# FabMap
# uncomment this part to enable fabmap
#add_subdirectory(${PROJECT_SOURCE_DIR}/thirdparty/openFabMap)
#include_directories(${PROJECT_SOURCE_DIR}/thirdparty/openFabMap/include)
#add_definitions("-DHAVE_FABMAP")
#set(FABMAP_LIB openFABMAP )

generate_dynamic_reconfigure_options(
  cfg/LSDDebugParams.cfg
  cfg/LSDParams.cfg
)

catkin_package(
  LIBRARIES lsdslam
  DEPENDS EIGEN3 SuiteSparse
  CATKIN_DEPENDS libg2o
)

# SSE flags
add_definitions("-DUSE_ROS")
add_definitions("-DENABLE_SSE")

# Also add some useful compiler flag
set(CMAKE_CXX_FLAGS
   "${CMAKE_CXX_FLAGS} -march=native -Wall -std=c++0x"
)

# Set source files
set(lsd_SOURCE_FILES
  ${PROJECT_SOURCE_DIR}/src/DataStructures/Frame.cpp
  ${PROJECT_SOURCE_DIR}/src/DataStructures/FramePoseStruct.cpp
  ${PROJECT_SOURCE_DIR}/src/DataStructures/FrameMemory.cpp
  ${PROJECT_SOURCE_DIR}/src/SlamSystem.cpp
  ${PROJECT_SOURCE_DIR}/src/LiveSLAMWrapper.cpp
  ${PROJECT_SOURCE_DIR}/src/DepthEstimation/DepthMap.cpp
  ${PROJECT_SOURCE_DIR}/src/DepthEstimation/DepthMapPixelHypothesis.cpp
  ${PROJECT_SOURCE_DIR}/src/util/globalFuncs.cpp
  ${PROJECT_SOURCE_DIR}/src/util/SophusUtil.cpp
  ${PROJECT_SOURCE_DIR}/src/util/settings.cpp
  ${PROJECT_SOURCE_DIR}/src/util/Undistorter.cpp
  ${PROJECT_SOURCE_DIR}/src/Tracking/Sim3Tracker.cpp
  ${PROJECT_SOURCE_DIR}/src/Tracking/Relocalizer.cpp
  ${PROJECT_SOURCE_DIR}/src/Tracking/SE3Tracker.cpp
  ${PROJECT_SOURCE_DIR}/src/Tracking/TrackingReference.cpp
  ${PROJECT_SOURCE_DIR}/src/IOWrapper/Timestamp.cpp
  ${PROJECT_SOURCE_DIR}/src/GlobalMapping/FabMap.cpp
  ${PROJECT_SOURCE_DIR}/src/GlobalMapping/KeyFrameGraph.cpp
  ${PROJECT_SOURCE_DIR}/src/GlobalMapping/g2oTypeSim3Sophus.cpp
  ${PROJECT_SOURCE_DIR}/src/GlobalMapping/TrackableKeyFrameSearch.cpp
)
set(SOURCE_FILES
  ${lsd_SOURCE_FILES}
  ${PROJECT_SOURCE_DIR}/src/IOWrapper/ROS/ROSImageStreamThread.cpp
  ${PROJECT_SOURCE_DIR}/src/IOWrapper/ROS/ROSOutput3DWrapper.cpp
  ${PROJECT_SOURCE_DIR}/src/IOWrapper/OpenCV/ImageDisplay_OpenCV.cpp
)

include_directories(
  include
  ${EIGEN3_INCLUDE_DIR}
  ${PROJECT_SOURCE_DIR}/src
  ${PROJECT_SOURCE_DIR}/thirdparty/Sophus
  ${CSPARSE_INCLUDE_DIR} #Has been set by SuiteParse
  ${CHOLMOD_INCLUDE_DIR} #Has been set by SuiteParse
  ${OpenCV_INCLUDE_DIRS}
  ${catkin_INCLUDE_DIRS}
)

# build shared library.
add_library(lsdslam SHARED ${SOURCE_FILES})
target_link_libraries(lsdslam ${FABMAP_LIB} ${G2O_LIBRARIES} ${catkin_LIBRARIES} ${CSPARSE_INCLUDE_DIR} cxsparse X11)
#rosbuild_link_boost(lsdslam thread)



# build live ros node
add_executable(live_slam src/main_live_odometry.cpp)
target_link_libraries(live_slam lsdslam ${catkin_LIBRARIES} ${G2O_LIBRARIES})


# build image node
add_executable(dataset_slam src/main_on_images.cpp)
add_dependencies(lsdslam lsd_slam_viewer_generate_messages_cpp)
add_dependencies(live_slam lsd_slam_viewer_generate_messages_cpp)
add_dependencies(dataset_slam lsd_slam_viewer_generate_messages_cpp)
target_link_libraries(dataset_slam lsdslam ${catkin_LIBRARIES} ${G2O_LIBRARIES})
target_link_libraries(live_slam lsdslam ${OpenCV_LIBRARIES})

# TODO add INSTALL
