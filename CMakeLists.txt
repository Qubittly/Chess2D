# CMakeList.txt : Top-level CMake project file, do global configuration
# and include sub-projects here.
#
cmake_minimum_required (VERSION 3.20)

project ("DeepLearningCPP")

set(CMAKE_MSVC_DEBUG_INFORMATION_FORMAT Embedded)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

project ("Chess2D")

option(CHESS2D_ENABLE_TRACY "Enable Tracy instrumentation" ON)

if (CHESS2D_ENABLE_TRACY)
  find_package(Tracy CONFIG REQUIRED)
endif()

enable_testing()

# Include sub-projects.
add_subdirectory ("src")
add_subdirectory ("tests")

# Copy compile_commands.json to build root for clangd
add_custom_target(copy_compile_commands ALL
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        ${CMAKE_BINARY_DIR}/compile_commands.json
        ${CMAKE_SOURCE_DIR}/out/build/compile_commands.json
    COMMENT "Copying compile_commands.json to build root"
)
