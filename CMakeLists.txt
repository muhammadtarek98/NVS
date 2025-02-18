cmake_minimum_required(VERSION 3.10)
project(NVS LANGUAGES C CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(COLMAP_INCLUDE_DIRS)
add_subdirectory(colmap)
add_subdirectory(COLMAP_SLAM)

add_executable(NVS test.cpp)

target_link_libraries(NVS PRIVATE colmap::colmap)