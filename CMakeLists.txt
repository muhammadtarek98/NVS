cmake_minimum_required(VERSION 3.10)

# Set the CUDA compiler and architecture
set(CMAKE_CUDA_COMPILER /usr/local/cuda-11.8/bin/nvcc)
set(CMAKE_CUDA_ARCHITECTURES 75)

project(NVS LANGUAGES C CXX CUDA)

# Set C++ standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Define paths to dependencies
set(Ceres_DIR "/home/muahmmad/ceres_install/lib/cmake/Ceres/")
set(GTest_DIR "/home/muahmmad/ceres_install/lib/cmake/GTest/")
set(Abseil_DIR "/home/muahmmad/ceres_install/lib/cmake/absl/")

# Update CMAKE_PREFIX_PATH
set(CMAKE_PREFIX_PATH "${Abseil_DIR};${GTest_DIR};${Ceres_DIR};${CMAKE_PREFIX_PATH}")

# Find packages
find_package(Abseil REQUIRED)
find_package(GTest REQUIRED)
find_package(Ceres REQUIRED)

# Add subdirectory
add_subdirectory(colmap)

# Add executable
add_executable(NVS test.cpp)

# Link libraries
target_link_libraries(NVS PRIVATE colmap::colmap Ceres::ceres)
target_link_directories(NVS PRIVATE )
