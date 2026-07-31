cmake_minimum_required(VERSION 3.5)
project(nav2_mppi_controller)

list(REMOVE_ITEM CMAKE_CXX_IMPLICIT_INCLUDE_DIRECTORIES "/usr/local/include")
add_compile_options("-I/usr/local/include")

# ==============================================================================
# Configurable Build Options
# Overrides: colcon build --cmake-args -DWITH_OPENMP=ON -DWITH_XSIMD=ON
# ==============================================================================
option(WITH_XSIMD "Enable SIMD acceleration for xtensor via xsimd" ON)
option(WITH_OPENMP "Enable OpenMP multi-threading" ON)
option(WITH_TBB "Enable TBB multi-threading" OFF)

# Configure SIMD / xsimd
if(WITH_XSIMD)
  add_compile_definitions(XTENSOR_ENABLE_XSIMD XTENSOR_USE_XSIMD)
  find_package(xsimd REQUIRED)
endif()

# Configure OpenMP & handle xtensorTargets.cmake dependency
if(WITH_OPENMP)
  add_compile_definitions(XTENSOR_USE_OPENMP)
  find_package(OpenMP REQUIRED)
  if(NOT TARGET OpenMP::OpenMP_CXX_xtensor)
    add_library(OpenMP::OpenMP_CXX_xtensor INTERFACE IMPORTED)
    if(TARGET OpenMP::OpenMP_CXX)
      set_target_properties(OpenMP::OpenMP_CXX_xtensor PROPERTIES
        INTERFACE_LINK_LIBRARIES OpenMP::OpenMP_CXX
      )
    else()
      set_target_properties(OpenMP::OpenMP_CXX_xtensor PROPERTIES
        INTERFACE_COMPILE_OPTIONS "-qopenmp"
        INTERFACE_LINK_LIBRARIES "-qopenmp"
      )
    endif()
  endif()
else()
  # Stub target so system xtensorTargets.cmake won't fail if installed xtensor expects it
  if(NOT TARGET OpenMP::OpenMP_CXX_xtensor)
    add_library(OpenMP::OpenMP_CXX_xtensor INTERFACE IMPORTED)
  endif()
endif()

# Configure TBB
if(WITH_TBB)
  add_compile_definitions(XTENSOR_USE_TBB)
  find_package(TBB REQUIRED)
endif()

find_package(ament_cmake REQUIRED)


find_package(xtensor REQUIRED)

include_directories(
  include
  /usr/local/include
 
)

set(dependencies_pkgs
  rclcpp
  nav2_common
  pluginlib
  tf2
  geometry_msgs
  visualization_msgs
  nav_msgs
  nav2_core
  nav2_costmap_2d
  nav2_util
  tf2_geometry_msgs
  tf2_eigen
  tf2_ros
)

foreach(pkg IN LISTS dependencies_pkgs)
  find_package(${pkg} REQUIRED)
endforeach()

nav2_package()

include(CheckCXXCompilerFlag)

check_cxx_compiler_flag("-mno-avx512f" COMPILER_SUPPORTS_AVX512)
check_cxx_compiler_flag("-msse4.2" COMPILER_SUPPORTS_SSE4)
check_cxx_compiler_flag("-mavx2" COMPILER_SUPPORTS_AVX2)
check_cxx_compiler_flag("-mfma" COMPILER_SUPPORTS_FMA)

if(COMPILER_SUPPORTS_AVX512)
  add_compile_options(-mno-avx512f)
endif()

if(COMPILER_SUPPORTS_SSE4)
  add_compile_options(-msse4.2)
endif()

if(COMPILER_SUPPORTS_AVX2)
  add_compile_options(-mavx2)
endif()

if(COMPILER_SUPPORTS_FMA)
  add_compile_options(-mfma)
endif()

# If building one the same hardware to be deployed on, try `-march=native`!
# add_compile_options(-O3 -finline-limit=10000000 -ffp-contract=fast -ffast-math -mtune=generic)

add_library(mppi_controller SHARED
  src/controller.cpp
  src/optimizer.cpp
  src/critic_manager.cpp
  src/trajectory_visualizer.cpp
  src/path_handler.cpp
  src/parameters_handler.cpp
  src/noise_generator.cpp
)

add_library(mppi_critics SHARED
  src/critics/obstacles_critic.cpp
  src/critics/cost_critic.cpp
  src/critics/goal_critic.cpp
  src/critics/goal_angle_critic.cpp
  src/critics/path_align_critic.cpp
  src/critics/path_follow_critic.cpp
  src/critics/path_angle_critic.cpp
  src/critics/prefer_forward_critic.cpp
  src/critics/twirling_critic.cpp
  src/critics/constraint_critic.cpp
  src/critics/velocity_deadband_critic.cpp
)

set(libraries mppi_controller mppi_critics)

foreach(lib IN LISTS libraries)
  # Extract include directory from the target if set by find_package(xtensor)
  if(TARGET xtensor)
    get_target_property(XTENSOR_TARGET_INC xtensor INTERFACE_INCLUDE_DIRECTORIES)
  endif()

  # ✅ Include target property, all CMake variable variants, and system fallback
  target_include_directories(${lib} PUBLIC
    ${XTENSOR_INCLUDE_DIRS}
    ${XTENSOR_INCLUDE_DIR}
    ${xtensor_INCLUDE_DIR}
    ${XTENSOR_TARGET_INC}
    ${xsimd_INCLUDE_DIRS}
    /usr/local/include
  )
  target_link_libraries(${lib} xtensor xtensor::optimize)
  
  # Conditionally link SIMD and parallelization targets
  if(WITH_XSIMD)
    target_include_directories(${lib} PUBLIC ${xsimd_INCLUDE_DIRS})
    target_link_libraries(${lib} xtensor::use_xsimd)
    target_compile_definitions(${lib} PRIVATE XSIMD_DEFAULT_ARCH=xsimd::avx2)
  endif()

  if(WITH_OPENMP)
    target_link_libraries(${lib} OpenMP::OpenMP_CXX)
  endif()

  if(WITH_TBB)
    target_link_libraries(${lib} TBB::tbb)
  endif()
  
  ament_target_dependencies(${lib} ${dependencies_pkgs})
endforeach()

install(TARGETS mppi_controller mppi_critics
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)

install(DIRECTORY include/
  DESTINATION include/
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  find_package(ament_cmake_gtest REQUIRED)
  set(ament_cmake_copyright_FOUND TRUE)
  ament_lint_auto_find_test_dependencies()
  add_subdirectory(test)
  # add_subdirectory(benchmark)
endif()

ament_export_libraries(${libraries})
ament_export_dependencies(${dependencies_pkgs})
ament_export_include_directories(include)
pluginlib_export_plugin_description_file(nav2_core mppic.xml)
pluginlib_export_plugin_description_file(nav2_mppi_controller critics.xml)

ament_package()
