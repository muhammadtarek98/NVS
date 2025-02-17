cmake_minimum_required(VERSION 3.10)
project(NVS)
set(COLMAP_INCLUDE_DIRS "/home/cplus/projects/m.tarek_master/packages/COLMAP_repos/colmap_install/include")
set(COLMAP_LIBRARIES "/home/cplus/projects/m.tarek_master/packages/COLMAP_repos/colmap_install/lib/libcolmap.a")

find_package(COLMAP REQUIRED)
add_executable(NVS test.cpp)
# Include COLMAP headers
include_directories(
    "/home/cplus/projects/m.tarek_master/packages/COLMAP_repos/colmap_install/include"
    ${COLMAP_INCLUDE_DIRS}
)

# Link COLMAP libraries
target_link_libraries(NVS PRIVATE ${COLMAP_LIBRARIES})