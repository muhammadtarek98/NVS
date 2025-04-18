# Ceres Solver - A fast non-linear least squares minimizer
# Copyright 2024 Google Inc. All rights reserved.
# http://ceres-solver.org/
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met:
#
# * Redistributions of source code must retain the above copyright notice,
#   this list of conditions and the following disclaimer.
# * Redistributions in binary form must reproduce the above copyright notice,
#   this list of conditions and the following disclaimer in the documentation
#   and/or other materials provided with the distribution.
# * Neither the name of Google Inc. nor the names of its contributors may be
#   used to endorse or promote products derived from this software without
#   specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
# ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
# LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
# CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
# SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
# INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
# CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
# ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
# POSSIBILITY OF SUCH DAMAGE.
#
# Authors: keir@google.com (Keir Mierle)
#          alexs.mac@gmail.com (Alex Stewart)

cmake_minimum_required(VERSION 3.16...3.29)
project(Ceres C CXX)

# Always build position-independent code (PIC), even when building Ceres as a
# static library so that shared libraries can link against it, not just
# executables (PIC does not apply on Windows). Global variable can be overridden
# by the user whereas target properties can be not.
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# NOTE: The 'generic' CMake variables CMAKE_[SOURCE/BINARY]_DIR should not be
#       used.  Always use the project-specific variants (generated by CMake):
#       <PROJECT_NAME_MATCHING_CASE>_[SOURCE/BINARY]_DIR, e.g.
#       Ceres_SOURCE_DIR (note, *not* CERES_SOURCE_DIR) instead, as these will
#       always point to the correct directories for the Ceres project, even if
#       it is nested inside another source tree, whereas the 'generic'
#       CMake variables refer to the *first* project() declaration, i.e. the
#       top-level project, not Ceres, if Ceres is nested.

# Make CMake aware of the cmake folder for local FindXXX scripts,
# append rather than set in case the user has passed their own
# additional paths via -D.
list(APPEND CMAKE_MODULE_PATH "${Ceres_SOURCE_DIR}/cmake")
include(AddCompileFlagsIfSupported)
include(CheckCXXCompilerFlag)
include(CheckLibraryExists)
include(GNUInstallDirs)
include(UpdateCacheVariable)

check_cxx_compiler_flag(/bigobj HAVE_BIGOBJ)
check_library_exists(m pow "" HAVE_LIBM)

# Xcode 11.0-1 with macOS 10.15 (Catalina) broke alignment.
include(DetectBrokenStackCheckMacOSXcodePairing)
detect_broken_stack_check_macos_xcode_pairing()

# Set up the git hook to make Gerrit Change-Id: lines in commit messages.
include(AddGerritCommitHook)
add_gerrit_commit_hook(${Ceres_SOURCE_DIR} ${Ceres_BINARY_DIR})

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${Ceres_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${Ceres_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${Ceres_BINARY_DIR}/lib)
# Set postfixes for generated libraries based on buildtype.
set(CMAKE_RELEASE_POSTFIX "")
set(CMAKE_DEBUG_POSTFIX "-debug")

# Read the Ceres version from the source, such that we only ever have a single
# definition of the Ceres version.
include(ReadCeresVersionFromSource)
read_ceres_version_from_source(${Ceres_SOURCE_DIR})

enable_testing()

include(CMakeDependentOption)
include(PrettyPrintCMakeList)

option(SUITESPARSE "Enable SuiteSparse." ON)
if (APPLE)
  option(ACCELERATESPARSE
    "Enable use of sparse solvers in Apple's Accelerate framework." ON)
  option(ENABLE_BITCODE
    "Enable bitcode for iOS builds (disables inline optimizations for Eigen)." OFF)
endif()
# We can't have an option called 'CUDA' since that is a reserved word -- a
# language definition.
set(USE_CUDA "default" CACHE STRING "Enable use of CUDA linear algebra solvers.")
option(LAPACK "Enable use of LAPACK directly within Ceres." ON)
# Template specializations for the Schur complement based solvers. If
# compile time, binary size or compiler performance is an issue, you
# may consider disabling this.
option(SCHUR_SPECIALIZATIONS "Enable fixed-size schur specializations." ON)
option(CUSTOM_BLAS
       "Use handcoded BLAS routines (usually faster) instead of Eigen."
       ON)
# Enable the use of Eigen as a sparse linear algebra library for
# solving the nonlinear least squares problems.
option(EIGENSPARSE "Enable Eigen as a sparse linear algebra library." ON)
cmake_dependent_option(EIGENMETIS "Enable Eigen METIS support." ON EIGENSPARSE OFF)
option(EXPORT_BUILD_DIR
  "Export build directory using CMake (enables external use without install)." OFF)
option(BUILD_TESTING "Enable tests" ON)
option(BUILD_DOCUMENTATION "Build User's Guide (html)" OFF)
option(BUILD_EXAMPLES "Build examples" ON)
option(BUILD_BENCHMARKS "Build Ceres benchmarking suite" ON)
option(BUILD_SHARED_LIBS "Build Ceres as a shared library." OFF)
option(PROVIDE_UNINSTALL_TARGET "Add a custom target to ease removal of installed targets" ON)
set(SANITIZERS "" CACHE STRING "Semicolon-separated list of sanitizers to use (e.g address, memory, thread)")
include(EnableSanitizer)
enable_sanitizer(${SANITIZERS})
if (ANDROID)
  option(ANDROID_STRIP_DEBUG_SYMBOLS "Strip debug symbols from Android builds (reduces file sizes)" ON)
endif()

# Abseil
if (EXISTS "${Ceres_SOURCE_DIR}/third_party/abseil-cpp/CMakeLists.txt")
  # extract submodule absl_VERSION
  file(STRINGS "${Ceres_SOURCE_DIR}/third_party/abseil-cpp/CMakeLists.txt" project_calls REGEX "[ \t]*project\\([ \t]*absl")
  foreach(project_call IN LISTS project_calls)
    string(REGEX MATCH "VERSION[ ]+([0-9]+(\.(rc)?[0-9]+)?)" version_param "${project_call}")
      if (version_param)
        set(absl_VERSION "${CMAKE_MATCH_1}")
        message("-- Using the version of abseil in ceres-solver/third_party/abseil-cpp with version ${absl_VERSION}")
        break ()
      endif()
  endforeach()
  unset(project_calls)
  unset(project_call)
  unset(version_param)
  if (NOT absl_VERSION)
    message("-- Using the version of abseil in ceres-solver/third_party/abseil-cpp")
  endif()

  # In the normal course of things, we would not need the following constraint.
  # However, if we do not set the standard, then std::basic_string_view is not
  # available, which absl depends on and linking the test fails.
  set(CMAKE_CXX_STANDARD 17)
  # Abseil does not handle symbol visibility correctly. As a workaround, we let it
  # expose all the symbols to avoid linker errors.
  set(CMAKE_CXX_VISIBILITY_PRESET default)
  # ABSL plans to enable this flag to be on by default in the future. If it is not
  # set, then it issues a warning. We force enable it, both to silence the warning
  # but to also be safe going forward.

  option(ABSL_PROPAGATE_CXX_STD "Use CMake C++ standard meta features
  (e.g. cxx_std_14) that propagate to targets that link to Abseil" ON)
  option(ABSL_ENABLE_INSTALL "Enable install rules" ON)

  add_subdirectory(third_party/abseil-cpp)

  unset(CMAKE_CXX_STANDARD)
  unset(CMAKE_CXX_VISIBILITY_PRESET)

  # Set the default symbol visibility to hidden to unify the behavior among
  # the various compilers and to get smaller binaries
  set(CMAKE_C_VISIBILITY_PRESET hidden)
  set(CMAKE_CXX_VISIBILITY_PRESET hidden)
  set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)

  set(CERES_USE_SYSTEM_ABSL NO)
else()
  message("-- ceres-solver/third_party/abseil-cpp is empty, so falling back to system installed abseil")
  find_package(absl REQUIRED 20240116)
  message("-- Found abseil version ${absl_VERSION}: ${absl_DIR}")
  if (NOT absl_VERSION)
    # In some installs, abseil will not export the version number, in
    # which case we check for vlog_is_on, which is a feature we need
    # before being able to use abseil.
    if (NOT TARGET absl::absl_vlog_is_on)
      message(FATAL_ERROR "The version of abseil installed on the system provides no version info and is missing TARGET 'absl::absl_vlog_is_on', need at least 20240116")
    endif()
  endif()
  set(CERES_USE_SYSTEM_ABSL YES)
endif()
if (NOT absl_VERSION)
  if (EXPORT_BUILD_DIR)
    message(FATAL_ERROR "Cannot generate CeresConfig.cmake due to missing Abseil version. Using resulting package configuration will likely result in integration errors. Set absl_VERSION manually to disable this error message")
  endif()
  install (CODE [=[message (FATAL_ERROR "Cannot generate CeresConfig.cmake due to missing Abseil version. Using resulting package configuration will likely result in integration errors. Set absl_VERSION manually to disable this error message.")]=])
endif()

# Google Test
if (BUILD_TESTING)
  if (EXISTS "${Ceres_SOURCE_DIR}/third_party/googletest/CMakeLists.txt")
    message("-- Using the version of googletest in ceres-solver/third_party/googletest")
    # Now that we have absl as a dependency, it would make sense to tell
    # googletest to use absl also, but doing so seems to be problematic
    # for now, since if there is a version of absl and re2 installed in
    # the system then we start getting errors because we maybe using the
    # absl from our submodule, but the re2 installed on the system only
    # sees the absl installed on the system. It is possible that we will
    # need to include re2 as a submodule also, for the following to
    # work.

    # set(GTEST_HAS_ABSL ON CACHE BOOL "Use ABSL when building gtest")
    add_subdirectory(${Ceres_SOURCE_DIR}/third_party/googletest ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/googletest)
  else()
    message("-- ceres-solver/third_party/googletest is empty, so falling back to system installed googletest")
    find_package(GTest 1.14.0 REQUIRED)
  endif()
endif(BUILD_TESTING)

# IOS is defined iff using the iOS.cmake CMake toolchain to build a static
# library for iOS.
if (IOS)
  message(STATUS "Building Ceres for iOS platform: ${IOS_PLATFORM}")

  # Ceres requires at least iOS 7.0+.
  if (IOS_DEPLOYMENT_TARGET VERSION_LESS 7.0)
    message(FATAL_ERROR "Unsupported iOS version: ${IOS_DEPLOYMENT_TARGET}, Ceres "
      "requires at least iOS version 7.0")
  endif()

  # Apple claims that the BLAS call dsyrk_ is a private API, and will not allow
  # you to submit to the Apple Store if the symbol is present.
  update_cache_variable(LAPACK OFF)
  message(STATUS "Building for iOS: SuiteSparse, LAPACK are not available.")

  update_cache_variable(BUILD_EXAMPLES OFF)
  message(STATUS "Building for iOS: Will not build examples.")
endif (IOS)

unset(CERES_COMPILE_OPTIONS)

# Eigen.
# Eigen delivers Eigen3Config.cmake since v3.3.3
find_package(Eigen3 3.3.4 REQUIRED NO_MODULE)
if (Eigen3_FOUND)
  message("-- Found Eigen version ${Eigen3_VERSION}: ${Eigen3_DIR}")
  if (EIGENSPARSE)
    message("-- Enabling use of Eigen as a sparse linear algebra library.")
    list(APPEND CERES_COMPILE_OPTIONS CERES_USE_EIGEN_SPARSE)
  else (EIGENSPARSE)
    message("-- Disabling use of Eigen as a sparse linear algebra library.")
    message("   This does not affect the covariance estimation algorithm ")
    message("   which can still use the EIGEN_SPARSE_QR algorithm.")
    add_definitions(-DEIGEN_MPL2_ONLY)
  endif (EIGENSPARSE)
endif (Eigen3_FOUND)

if (CMAKE_VERSION VERSION_LESS 3.17)
  set_property(CACHE USE_CUDA PROPERTY STRINGS OFF default)
else (CMAKE_VERSION VERSION_LESS 3.17)
  set_property(CACHE USE_CUDA PROPERTY STRINGS OFF default static)
endif (CMAKE_VERSION VERSION_LESS 3.17)

if (USE_CUDA)
  if (CMAKE_VERSION VERSION_LESS 3.17)
    # On older versions of CMake (20.04 default is 3.16) FindCUDAToolkit was
    # not available, but FindCUDA was deprecated. To avoid special-case handling
    # elsewhere, emulate the effects of FindCUDAToolkit locally in terms of the
    # expected CMake imported targets and defined variables. This can be removed
    # from as soon as the min CMake version is >= 3.17.
    find_package(CUDA QUIET)
    if (CUDA_FOUND)
      message("-- Found CUDA version ${CUDA_VERSION} installed in: "
        "${CUDA_TOOLKIT_ROOT_DIR} via legacy (< 3.17) CMake module. "
        "Using the legacy CMake module means that any installation of "
        "Ceres will require that the CUDA libraries be installed in a "
        "location included in the LD_LIBRARY_PATH.")
      enable_language(CUDA)

      macro(DECLARE_IMPORTED_CUDA_TARGET COMPONENT)
        add_library(CUDA::${COMPONENT} INTERFACE IMPORTED)
        target_include_directories(
          CUDA::${COMPONENT} INTERFACE ${CUDA_INCLUDE_DIRS})
        target_link_libraries(
          CUDA::${COMPONENT} INTERFACE ${CUDA_${COMPONENT}_LIBRARY} ${ARGN})
      endmacro()

      declare_imported_cuda_target(cublas)
      declare_imported_cuda_target(cusolver)
      declare_imported_cuda_target(cusparse)
      declare_imported_cuda_target(cudart ${CUDA_LIBRARIES})

      set(CERES_CUDA_TARGET_SUFFIX "")
      set(CUDAToolkit_BIN_DIR ${CUDA_TOOLKIT_ROOT_DIR}/bin)

    else (CUDA_FOUND)
      message("-- Did not find CUDA, disabling CUDA support.")
      update_cache_variable(USE_CUDA OFF)
    endif (CUDA_FOUND)
  else (CMAKE_VERSION VERSION_LESS 3.17)
    find_package(CUDAToolkit QUIET)
    if (CUDAToolkit_FOUND)
      message("-- Found CUDA version ${CUDAToolkit_VERSION} installed in: "
        "${CUDAToolkit_TARGET_DIR}")
      set(CUDAToolkit_DEPENDENCY
        "find_dependency(CUDAToolkit ${CUDAToolkit_VERSION})")
      enable_language(CUDA)
      if (CMAKE_VERSION VERSION_GREATER_EQUAL "3.18")
        # Support Maxwell GPUs (Default).
        set(CMAKE_CUDA_ARCHITECTURES "50")
        # Support other architectures depending on CUDA toolkit version.
        if (CUDAToolkit_VERSION VERSION_GREATER_EQUAL "8.0")
          # Support Pascal GPUs.
          list(APPEND CMAKE_CUDA_ARCHITECTURES "60")
        endif(CUDAToolkit_VERSION VERSION_GREATER_EQUAL "8.0")
        if (CUDAToolkit_VERSION VERSION_GREATER_EQUAL "9.0")
          # Support Volta GPUs.
          list(APPEND CMAKE_CUDA_ARCHITECTURES "70")
        endif(CUDAToolkit_VERSION VERSION_GREATER_EQUAL "9.0")
        if (CUDAToolkit_VERSION VERSION_GREATER_EQUAL "10.0")
          # Support Turing  GPUs.
          list(APPEND CMAKE_CUDA_ARCHITECTURES "75")
        endif(CUDAToolkit_VERSION VERSION_GREATER_EQUAL "10.0")
        if (CUDAToolkit_VERSION VERSION_GREATER_EQUAL "11.0")
          # Support Ampere GPUs.
          list(APPEND CMAKE_CUDA_ARCHITECTURES "80")
        endif(CUDAToolkit_VERSION VERSION_GREATER_EQUAL "11.0")
        if (CUDAToolkit_VERSION VERSION_GREATER_EQUAL "11.8")
          # Support Hopper GPUs.
          list(APPEND CMAKE_CUDA_ARCHITECTURES "90")
        endif(CUDAToolkit_VERSION VERSION_GREATER_EQUAL "11.8")
        message("-- Setting CUDA Architecture to ${CMAKE_CUDA_ARCHITECTURES}")
      endif()

      if (USE_CUDA STREQUAL "static")
        set(CERES_CUDA_TARGET_SUFFIX "_static")
      else (USE_CUDA STREQUAL "static")
        set(CERES_CUDA_TARGET_SUFFIX "")
      endif (USE_CUDA STREQUAL "static")
    else (CUDAToolkit_FOUND)
      message("-- Did not find CUDA, disabling CUDA support.")
      update_cache_variable(USE_CUDA OFF)
    endif (CUDAToolkit_FOUND)
  endif (CMAKE_VERSION VERSION_LESS 3.17)
endif (USE_CUDA)

if (USE_CUDA)
  list(APPEND CERES_CUDA_LIBRARIES
    CUDA::cublas${CERES_CUDA_TARGET_SUFFIX}
    CUDA::cudart${CERES_CUDA_TARGET_SUFFIX}
    CUDA::cusolver${CERES_CUDA_TARGET_SUFFIX}
    CUDA::cusparse${CERES_CUDA_TARGET_SUFFIX})

  find_package(cudss CONFIG)
  if (cudss_FOUND)
    set(cudss_DEPENDENCY "find_dependency(cudss ${cudss_VERSION})")
    list(APPEND CERES_CUDA_LIBRARIES cudss${CERES_CUDA_TARGET_SUFFIX})
  else (cudss_FOUND)
    list(APPEND CERES_COMPILE_OPTIONS CERES_NO_CUDSS)
    message("-- Did not find cuDSS, SPARSE_SCHUR and SPARSE_NORMAL_CHOLESKY with CUDA_SPARSE will not be available.")
  endif (cudss_FOUND)

  unset (CERES_CUDA_TARGET_SUFFIX)
  set(CMAKE_CUDA_RUNTIME_LIBRARY NONE)
else (USE_CUDA)
  message("-- Building without CUDA.")
  list(APPEND CERES_COMPILE_OPTIONS CERES_NO_CUDSS)
  list(APPEND CERES_COMPILE_OPTIONS CERES_NO_CUDA)
endif (USE_CUDA)

if (LAPACK)
  find_package(LAPACK QUIET)
  if (LAPACK_FOUND)
    message("-- Found LAPACK library: ${LAPACK_LIBRARIES}")
  else (LAPACK_FOUND)
    message("-- Did not find LAPACK library, disabling LAPACK support.")
    update_cache_variable(LAPACK OFF)
    list(APPEND CERES_COMPILE_OPTIONS CERES_NO_LAPACK)
  endif (LAPACK_FOUND)
else (LAPACK)
  message("-- Building without LAPACK.")
  list(APPEND CERES_COMPILE_OPTIONS CERES_NO_LAPACK)
endif (LAPACK)

# Set the install path for the installed CeresConfig.cmake configuration file
# relative to CMAKE_INSTALL_PREFIX.
set(RELATIVE_CMAKECONFIG_INSTALL_DIR ${CMAKE_INSTALL_LIBDIR}/cmake/Ceres)

if (SUITESPARSE)
  # By default, if SuiteSparse and all dependencies are found, Ceres is
  # built with SuiteSparse support.

  # Check for SuiteSparse and dependencies.
  find_package(SuiteSparse 4.5.6 COMPONENTS CHOLMOD SPQR
    OPTIONAL_COMPONENTS Partition)
  if (SuiteSparse_FOUND)
    set(SuiteSparse_DEPENDENCY "find_dependency(SuiteSparse ${SuiteSparse_VERSION})")
    # By default, if all of SuiteSparse's dependencies are found, Ceres is
    # built with SuiteSparse support.
    message("-- Found SuiteSparse ${SuiteSparse_VERSION}, "
            "building with SuiteSparse.")

    if (SuiteSparse_VERSION VERSION_LESS 7.4.0)
      list(APPEND CERES_COMPILE_OPTIONS CERES_NO_CHOLMOD_FLOAT)
    else()
      message("-- CHOLMOD supports single precision factorization")
    endif()
    if (SuiteSparse_NO_CMAKE OR NOT SuiteSparse_DIR)
      install(FILES ${Ceres_SOURCE_DIR}/cmake/FindSuiteSparse.cmake
                    ${Ceres_SOURCE_DIR}/cmake/FindMETIS.cmake
              DESTINATION ${RELATIVE_CMAKECONFIG_INSTALL_DIR})
    endif (SuiteSparse_NO_CMAKE OR NOT SuiteSparse_DIR)
  else (SuiteSparse_FOUND)
    # Disable use of SuiteSparse if it cannot be found and continue.
    message("-- Did not find all SuiteSparse dependencies, disabling "
      "SuiteSparse support.")
    update_cache_variable(SUITESPARSE OFF)
    list(APPEND CERES_COMPILE_OPTIONS CERES_NO_SUITESPARSE)
  endif (SuiteSparse_FOUND)
else (SUITESPARSE)
  message("-- Building without SuiteSparse.")
  list(APPEND CERES_COMPILE_OPTIONS CERES_NO_SUITESPARSE)
endif (SUITESPARSE)

if (NOT SuiteSparse_Partition_FOUND)
  list (APPEND CERES_COMPILE_OPTIONS CERES_NO_CHOLMOD_PARTITION)
endif (NOT SuiteSparse_Partition_FOUND)

if (EIGENMETIS)
  find_package (METIS)
  if (METIS_FOUND)
    # Since METIS is a private dependency of Ceres, it requires access to the
    # link-only METIS::METIS target to avoid undefined linker errors in projects
    # relying on Ceres. We do not actually need to propagate anything besides
    # the link libraries (such as include directories.)
    set(METIS_DEPENDENCY "find_dependency(METIS ${METIS_VERSION})")
    # METIS find module must be installed unless a package config is being used.
    if (NOT METIS_DIR)
      install(FILES ${Ceres_SOURCE_DIR}/cmake/FindMETIS.cmake
              DESTINATION ${RELATIVE_CMAKECONFIG_INSTALL_DIR})
    endif (NOT METIS_DIR)
  else (METIS_FOUND)
    message("-- Did not find METIS, disabling Eigen METIS support.")
    update_cache_variable(EIGENMETIS OFF)
    list (APPEND CERES_COMPILE_OPTIONS CERES_NO_EIGEN_METIS)
  endif (METIS_FOUND)
else (EIGENMETIS)
  message("-- Building without Eigen METIS support.")
  list (APPEND CERES_COMPILE_OPTIONS CERES_NO_EIGEN_METIS)
endif (EIGENMETIS)

if (ACCELERATESPARSE)
  find_package(AccelerateSparse)
  if (AccelerateSparse_FOUND)
    message("-- Found Apple's Accelerate framework with sparse solvers, "
      "building with Accelerate sparse support.")
  else()
    message("-- Failed to find Apple's Accelerate framework with sparse solvers, "
      "building without Accelerate sparse support.")
    update_cache_variable(ACCELERATESPARSE OFF)
    list(APPEND CERES_COMPILE_OPTIONS CERES_NO_ACCELERATE_SPARSE)
  endif()
else()
  message("-- Building without Apple's Accelerate sparse support.")
  list(APPEND CERES_COMPILE_OPTIONS CERES_NO_ACCELERATE_SPARSE)
  mark_as_advanced(FORCE AccelerateSparse_INCLUDE_DIR
                         AccelerateSparse_LIBRARY)
endif()

# Ensure that the user understands they have disabled all sparse libraries.
if (NOT SUITESPARSE AND NOT EIGENSPARSE AND NOT ACCELERATESPARSE AND NOT cudss_FOUND)
  message("   ===============================================================")
  message("   Compiling without any sparse library: SuiteSparse, EigenSparse")
  message("   Apple's Accelerate & cuDSS are all disabled or unavailable.  ")
  message("   No sparse linear solvers (SPARSE_NORMAL_CHOLESKY & SPARSE_SCHUR)")
  message("   will be available when Ceres is used.")
  message("   ===============================================================")
endif()

# ANDROID define is set by the Android CMake toolchain file.
if (ANDROID)
  message("  ================================================================")
  if (ANDROID_STRIP_DEBUG_SYMBOLS)
    # Strip debug information unconditionally to avoid +200MB library file sizes.
    set( CMAKE_EXE_LINKER_FLAGS  "${CMAKE_EXE_LINKER_FLAGS} -s" )
    set( CMAKE_SHARED_LINKER_FLAGS  "${CMAKE_SHARED_LINKER_FLAGS} -s" )
    message("  Stripping debug information from Android build of Ceres library ")
    message("  to avoid +200MB library files.")
  else()
    message("  Warning: not stripping debug information from Android build of ")
    message("  Ceres library.  This will result in a large (+200MB) library.")
  endif()
  message("")
  message("  You can control whether debug information is stripped via the ")
  message("  ANDROID_STRIP_DEBUG_SYMBOLS CMake option when configuring Ceres.")
  message("  ================================================================")
endif()

if (NOT SCHUR_SPECIALIZATIONS)
  list(APPEND CERES_COMPILE_OPTIONS CERES_RESTRICT_SCHUR_SPECIALIZATION)
  message("-- Disabling Schur specializations (faster compiles)")
endif (NOT SCHUR_SPECIALIZATIONS)

if (NOT CUSTOM_BLAS)
  list(APPEND CERES_COMPILE_OPTIONS CERES_NO_CUSTOM_BLAS)
  message("-- Disabling custom blas")
endif (NOT CUSTOM_BLAS)

if (BUILD_BENCHMARKS)
  # Version 1.3 was first to provide import targets
  find_package(benchmark 1.3 QUIET)
  if (benchmark_FOUND)
     message("-- Found Google benchmark library. Building Ceres benchmarks.")
  else()
     message("-- Failed to find Google benchmark library, disabling build of benchmarks.")
     update_cache_variable(BUILD_BENCHMARKS OFF)
  endif()
  mark_as_advanced(benchmark_DIR)
endif()

# TODO Report features using the FeatureSummary CMake module
if (BUILD_SHARED_LIBS)
  message("-- Building Ceres as a shared library.")
else (BUILD_SHARED_LIBS)
  message("-- Building Ceres as a static library.")
endif (BUILD_SHARED_LIBS)

# Change the default build type from Debug to Release, while still
# supporting overriding the build type.
#
# The CACHE STRING logic here and elsewhere is needed to force CMake
# to pay attention to the value of these variables.
if (NOT CMAKE_BUILD_TYPE)
  message("-- No build type specified; defaulting to CMAKE_BUILD_TYPE=Release.")
  set(CMAKE_BUILD_TYPE Release CACHE STRING
    "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel."
    FORCE)
else (NOT CMAKE_BUILD_TYPE)
  if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    message("\n=================================================================================")
    message("\n-- Build type: Debug. Performance will be terrible!")
    message("-- Add -DCMAKE_BUILD_TYPE=Release to the CMake command line to get an optimized build.")
    message("\n=================================================================================")
  endif (CMAKE_BUILD_TYPE STREQUAL "Debug")
endif (NOT CMAKE_BUILD_TYPE)

# After the tweaks for the compile settings, disable some warnings on MSVC.
if (MSVC)
  # Insecure standard library functions
  add_compile_definitions(_CRT_SECURE_NO_WARNINGS)
  # std::numeric_limits<T>::has_denorm is deprecated in C++23
  add_compile_definitions($<$<COMPILE_LANGUAGE:CXX>:_SILENCE_CXX23_DENORM_DEPRECATION_WARNING>)
  # std::aligned_storage is deprecated in C++23
  add_compile_definitions($<$<COMPILE_LANGUAGE:CXX>:_SILENCE_CXX23_ALIGNED_STORAGE_DEPRECATION_WARNING>)
  # Disable signed/unsigned int conversion warnings.
  add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/wd4018>)
  add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/wd4267>)
  # Disable warning about using struct/class for the same symbol.
  add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/wd4099>)
  # Disable performance warning about int-to-bool conversion.
  add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/wd4800>)
  # Disable warning about int64 to int32 conversion. Disabling
  # this warning may not be correct; needs investigation.
  # TODO(keir): Investigate these warnings in more detail.
  add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/wd4244>)
  # It's not possible to use STL types in DLL interfaces in a portable and
  # reliable way. However, that's what happens with Google Log and Google Flags
  # on Windows. MSVC gets upset about this and throws warnings that we can't do
  # much about. The real solution is to link static versions of Google Log and
  # Google Test, but that seems tricky on Windows. So, disable the warning.
  add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/wd4251>)

  # Add bigobj flag otherwise the build would fail due to large object files
  # probably resulting from generated headers (like the fixed-size schur
  # specializations).
  add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/bigobj>)

  # Google Flags doesn't have their DLL import/export stuff set up correctly,
  # which results in linker warnings. This is irrelevant for Ceres, so ignore
  # the warnings.
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /ignore:4049")

  # Tuple sizes of 10 are used by Gtest.
  add_definitions("-D_VARIADIC_MAX=10")
endif (MSVC)

if (UNIX)
  # Flags which we add to GCC to make it more picky about stuff
  # we do care about,
  add_cxx_compiler_flag_if_supported(CERES_STRICT_CXX_FLAGS
                                     -Wmissing-declarations)
  # Flags which we add to GCC to silence lots of annoying false-positives.
  add_cxx_compiler_flag_if_supported(CERES_STRICT_CXX_FLAGS
                                     -Wno-unknown-pragmas)
  add_cxx_compiler_flag_if_supported(CERES_STRICT_CXX_FLAGS
                                     -Wno-sign-compare)
  add_cxx_compiler_flag_if_supported(CERES_STRICT_CXX_FLAGS
                                     -Wno-unused-parameter)
  add_cxx_compiler_flag_if_supported(CERES_STRICT_CXX_FLAGS
                                     -Wno-missing-field-initializers)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CERES_STRICT_CXX_FLAGS}")
endif (UNIX)

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang") # Matches Clang & AppleClang.
  # Optimize for Eigen OR enable bitcode; you cannot do both since bitcode is an
  # intermediate representation.
  if (ENABLE_BITCODE)
    set(CMAKE_CXX_FLAGS
      "${CMAKE_CXX_FLAGS} -fembed-bitcode")
  else ()
    # Use a larger inlining threshold for Clang, since it hobbles Eigen,
    # resulting in an unreasonably slow version of the blas routines. The
    # -Qunused-arguments is needed because CMake passes the inline
    # threshold to the linker and clang complains about it and dies.
    set(CMAKE_CXX_FLAGS
      "${CMAKE_CXX_FLAGS} -Qunused-arguments -mllvm -inline-threshold=600")
  endif ()
  # Older versions of Clang (<= 2.9) do not support the 'return-type-c-linkage'
  # option, so check for its presence before adding it to the default flags set.
  check_cxx_compiler_flag("-Wno-return-type-c-linkage"
                          HAVE_RETURN_TYPE_C_LINKAGE)
  if (HAVE_RETURN_TYPE_C_LINKAGE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-return-type-c-linkage")
  endif(HAVE_RETURN_TYPE_C_LINKAGE)
endif ()

add_compile_definitions($<$<BOOL:${WIN32}>:NOMINMAX>)

# Configure the Ceres config.h compile options header using the current
# compile options and put the configured header into the Ceres build
# directory.  Note that the ceres/internal subdir in <build>/config where
# the configured config.h is placed is important, because Ceres will be
# built against this configured header, it needs to have the same relative
# include path as it would if it were in the source tree (or installed).
list(REMOVE_DUPLICATES CERES_COMPILE_OPTIONS)
include(CreateCeresConfig)
create_ceres_config("${CERES_COMPILE_OPTIONS}"
  ${Ceres_BINARY_DIR}/${CMAKE_INSTALL_INCLUDEDIR}/ceres/internal)

add_subdirectory(internal/ceres)

if (BUILD_DOCUMENTATION)
  find_package (Sphinx REQUIRED COMPONENTS sphinx_rtd_theme)
  if (NOT Sphinx_FOUND)
    message("-- Failed to find Sphinx and/or its dependencies, disabling build of documentation.")
    update_cache_variable(BUILD_DOCUMENTATION OFF)
  else()
    # Generate the User's Guide (html).
    # The corresponding target is ceres_docs, but is included in ALL.
    message("-- Build the HTML documentation.")
    add_subdirectory(docs)
  endif()
endif (BUILD_DOCUMENTATION)

if (BUILD_EXAMPLES)
  message("-- Build the examples.")
  add_subdirectory(examples)
else (BUILD_EXAMPLES)
  message("-- Do not build any example.")
endif (BUILD_EXAMPLES)

# Setup installation of Ceres public headers.
file(GLOB CERES_HDRS ${Ceres_SOURCE_DIR}/include/ceres/*.h)
install(FILES ${CERES_HDRS} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/ceres)

file(GLOB CERES_PUBLIC_INTERNAL_HDRS ${Ceres_SOURCE_DIR}/include/ceres/internal/*.h)
install(FILES ${CERES_PUBLIC_INTERNAL_HDRS} DESTINATION
  ${CMAKE_INSTALL_INCLUDEDIR}/ceres/internal)

# Also setup installation of Ceres config.h configured with the current
# build options and export.h into the installed headers directory.
install(DIRECTORY ${Ceres_BINARY_DIR}/${CMAKE_INSTALL_INCLUDEDIR}/
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

# Ceres supports two mechanisms by which it can be detected & imported into
# client code which uses CMake via find_package(Ceres):
#
#   1) Installation (e.g. to /usr/local), using CMake's install() function.
#
#   2) (Optional) Export of the current build directory into the local CMake
#      package registry, using CMake's export() function.  This allows use of
#      Ceres from other projects without requiring installation.
#
# In both cases, we need to generate a configured CeresConfig.cmake which
# includes additional autogenerated files which in concert create an imported
# target for Ceres in a client project when find_package(Ceres) is invoked.
# The key distinctions are where this file is located, and whether client code
# references installed copies of the compiled Ceres headers/libraries,
# (option #1: installation), or the originals in the source/build directories
# (option #2: export of build directory).
#
# NOTE: If Ceres is both exported and installed, provided that the installation
#       path is present in CMAKE_MODULE_PATH when find_package(Ceres) is called,
#       the installed version is preferred.

# Build the list of Ceres components for CeresConfig.cmake from the current set
# of compile options.
include(CeresCompileOptionsToComponents)
ceres_compile_options_to_components("${CERES_COMPILE_OPTIONS}"
  CERES_COMPILED_COMPONENTS)

include(CMakePackageConfigHelpers)

# Create a CeresConfigVersion.cmake file containing the version information,
# used by both export() & install().
write_basic_package_version_file("${Ceres_BINARY_DIR}/CeresConfigVersion.cmake"
  VERSION ${CERES_VERSION}
  COMPATIBILITY SameMajorVersion)

# Install method #1: Put Ceres in CMAKE_INSTALL_PREFIX: /usr/local or equivalent.

# This "exports" for installation all targets which have been put into the
# export set "CeresExport". This generates a CeresTargets.cmake file which,
# when read in by a client project as part of find_package(Ceres) creates
# imported library targets for Ceres (with dependency relations) which can be
# used in target_link_libraries() calls in the client project to use Ceres.
#
install(EXPORT CeresExport
        NAMESPACE Ceres::
        DESTINATION ${RELATIVE_CMAKECONFIG_INSTALL_DIR} FILE CeresTargets.cmake)

# Save the relative path from the installed CeresConfig.cmake file to the
# install prefix.  We do not save an absolute path in case the installed package
# is subsequently relocated after installation (on Windows).
file(RELATIVE_PATH INSTALL_ROOT_REL_CONFIG_INSTALL_DIR
     ${CMAKE_INSTALL_PREFIX}/${RELATIVE_CMAKECONFIG_INSTALL_DIR}
     ${CMAKE_INSTALL_PREFIX})

# Configure a CeresConfig.cmake file for an installed version of Ceres from the
# template, reflecting the current build options.
#
# NOTE: The -install suffix is necessary to distinguish the install version from
#       the exported version, which must be named CeresConfig.cmake in
#       Ceres_BINARY_DIR to be detected.  The suffix is removed when
#       it is installed.
set(SETUP_CERES_CONFIG_FOR_INSTALLATION TRUE)
configure_file("${Ceres_SOURCE_DIR}/cmake/CeresConfig.cmake.in"
               "${Ceres_BINARY_DIR}/CeresConfig-install.cmake" @ONLY)

# Install the configuration files into the same directory as the autogenerated
# CeresTargets.cmake file.  We include the find_package() scripts for libraries
# whose headers are included in the public API of Ceres and should thus be
# present in CERES_INCLUDE_DIRS.
install(FILES "${Ceres_BINARY_DIR}/CeresConfig-install.cmake"
        RENAME CeresConfig.cmake
        DESTINATION ${RELATIVE_CMAKECONFIG_INSTALL_DIR})
install(FILES "${Ceres_BINARY_DIR}/CeresConfigVersion.cmake"
        DESTINATION ${RELATIVE_CMAKECONFIG_INSTALL_DIR})

if (PROVIDE_UNINSTALL_TARGET)
  # Create an uninstall target to remove all installed files.
  configure_file("${Ceres_SOURCE_DIR}/cmake/uninstall.cmake.in"
                 "${Ceres_BINARY_DIR}/cmake/uninstall.cmake"
                 @ONLY)
  add_custom_target(uninstall
                    COMMAND ${CMAKE_COMMAND} -P ${Ceres_BINARY_DIR}/cmake/uninstall.cmake)
endif()

# Install method #2: Put Ceres build into local CMake registry.
#
# Optionally export the Ceres build directory into the local CMake package
# registry (~/.cmake/packages on *nix & OS X).  This allows the detection &
# use of Ceres without requiring that it be installed.
if (EXPORT_BUILD_DIR)
  message("-- Export Ceres build directory to local CMake package registry.")

  # Save the relative path from the build directory to the source directory.
  file(RELATIVE_PATH INSTALL_ROOT_REL_CONFIG_INSTALL_DIR
    ${Ceres_BINARY_DIR}
    ${Ceres_SOURCE_DIR})

  set (Ceres_EXPORT_TARGETS ceres)

  if (TARGET ceres_cuda_kernels)
    # The target ceres depends on ceres_cuda_kernels requiring the latter to be
    # exported as part of the same export set.
    list (APPEND Ceres_EXPORT_TARGETS ceres_cuda_kernels)
  endif (TARGET ceres_cuda_kernels)

  # Analogously to install(EXPORT ...), export the Ceres target from the build
  # directory as a package called Ceres into the local CMake package registry.
  export(TARGETS ${Ceres_EXPORT_TARGETS}
         NAMESPACE Ceres::
         FILE ${Ceres_BINARY_DIR}/CeresTargets.cmake)
  export(PACKAGE ${CMAKE_PROJECT_NAME})

  unset (Ceres_EXPORT_TARGETS)

  # Configure a CeresConfig.cmake file for the export of the Ceres build
  # directory from the template, reflecting the current build options.
  set(SETUP_CERES_CONFIG_FOR_INSTALLATION FALSE)
  configure_file("${Ceres_SOURCE_DIR}/cmake/CeresConfig.cmake.in"
    "${Ceres_BINARY_DIR}/CeresConfig.cmake" @ONLY)

endif (EXPORT_BUILD_DIR)
