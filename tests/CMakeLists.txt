find_package(SDL3 REQUIRED CONFIG)
find_package(SDL3_image REQUIRED CONFIG)
find_package(SDL3_ttf REQUIRED CONFIG)
find_package(spdlog REQUIRED CONFIG)

# Helper functions to reduce boilerplate
function(chess2d_set_cxx_standard target)
    set_property(TARGET ${target} PROPERTY CXX_STANDARD 20)
    set_property(TARGET ${target} PROPERTY CXX_STANDARD_REQUIRED ON)
    set_property(TARGET ${target} PROPERTY CXX_EXTENSIONS OFF)
endfunction()

function(chess2d_target_tracy target)
    if (CHESS2D_ENABLE_TRACY)
        target_link_libraries(${target} PRIVATE Tracy::TracyClient)
        target_compile_definitions(${target} PRIVATE TRACY_ENABLE)
    endif()
endfunction()

# Common source lists
set(ENGINE_SOURCES
    ${CMAKE_SOURCE_DIR}/src/chess_engine/board_state.cpp
    ${CMAKE_SOURCE_DIR}/src/chess_engine/piece_list.cpp
    ${CMAKE_SOURCE_DIR}/src/chess_engine/precomp_move_data.cpp
    ${CMAKE_SOURCE_DIR}/src/chess_engine/hash_keys.cpp
    ${CMAKE_SOURCE_DIR}/src/chess_engine/magics.cpp
)

set(ENGINE_MOVE_SOURCES
    ${ENGINE_SOURCES}
    ${CMAKE_SOURCE_DIR}/src/chess_engine/move_generator.cpp
)

set(COMMON_INCLUDES
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_SOURCE_DIR}/include/chess_engine
    ${CMAKE_SOURCE_DIR}/include/chess2D
    ${CMAKE_SOURCE_DIR}/include/search
    ${CMAKE_SOURCE_DIR}/utils/include
    ${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/include
)

# clay_test (integration) - needs clay repo for SDL3 renderer and shared layouts
include(FetchContent)
FetchContent_Declare(clay_repo
    GIT_REPOSITORY https://github.com/nicbarker/clay.git
    GIT_TAG main
)
FetchContent_Populate(clay_repo)

# Patch clay's SDL3 renderer to replace VLAs with fixed-size arrays for MSVC compatibility
set(_RENDERER_FILE "${clay_repo_SOURCE_DIR}/renderers/SDL3/clay_renderer_SDL3.c")
file(READ "${_RENDERER_FILE}" _RENDERER_CONTENT)
string(REPLACE "SDL_Vertex vertices[totalVertices];" "SDL_Vertex vertices[4096];" _RENDERER_CONTENT "${_RENDERER_CONTENT}")
string(REPLACE "int indices[totalIndices];" "int indices[4096];" _RENDERER_CONTENT "${_RENDERER_CONTENT}")
string(REPLACE "SDL_FPoint points[numCircleSegments + 1];" "SDL_FPoint points[4096];" _RENDERER_CONTENT "${_RENDERER_CONTENT}")
file(WRITE "${_RENDERER_FILE}" "${_RENDERER_CONTENT}")

add_executable(clay_test clay_test.c)
set_property(TARGET clay_test PROPERTY C_STANDARD 11)
set_property(TARGET clay_test PROPERTY C_STANDARD_REQUIRED ON)
set_property(TARGET clay_test PROPERTY C_EXTENSIONS OFF)

if (CMAKE_C_COMPILER_ID MATCHES "Clang")
    target_compile_options(clay_test PRIVATE -Wno-narrowing)
elseif (MSVC)
    target_compile_options(clay_test PRIVATE /wd4244 /wd4267)
endif()

target_include_directories(clay_test PRIVATE
    ${clay_repo_SOURCE_DIR}
    ${clay_repo_SOURCE_DIR}/examples
    ${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/include
)

target_link_libraries(clay_test PRIVATE
    SDL3::SDL3
    SDL3_image::SDL3_image
    SDL3_ttf::SDL3_ttf
)

add_custom_command(TARGET clay_test POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
        "${CMAKE_SOURCE_DIR}/tests/resources"
        "$<TARGET_FILE_DIR:clay_test>/resources"
)

add_test(NAME clay_test COMMAND clay_test)
set_tests_properties(clay_test PROPERTIES LABELS "integration")

# evaluation_test (unit)
add_executable(evaluation_test
    evaluation_test.cpp
    ${ENGINE_SOURCES}
    ${CMAKE_SOURCE_DIR}/utils/logger.cpp
)
chess2d_set_cxx_standard(evaluation_test)

target_include_directories(evaluation_test PRIVATE ${COMMON_INCLUDES})

target_link_libraries(evaluation_test PRIVATE spdlog::spdlog)
chess2d_target_tracy(evaluation_test)

add_test(NAME evaluation_test COMMAND evaluation_test)
set_tests_properties(evaluation_test PROPERTIES LABELS "unit")

# versus_test (integration)
add_executable(versus_test
    versus_test.cpp
    ${ENGINE_MOVE_SOURCES}
    ${CMAKE_SOURCE_DIR}/src/chess2D/board.cpp
    ${CMAKE_SOURCE_DIR}/src/chess2D/board_renderer.cpp
    ${CMAKE_SOURCE_DIR}/utils/logger.cpp
)
chess2d_set_cxx_standard(versus_test)

target_include_directories(versus_test PRIVATE ${COMMON_INCLUDES})

target_link_libraries(versus_test PRIVATE
    SDL3::SDL3
    SDL3_image::SDL3_image
    spdlog::spdlog
)

target_compile_definitions(versus_test PRIVATE
    CHESS2D_SOURCE_DIR="${CMAKE_SOURCE_DIR}"
)

chess2d_target_tracy(versus_test)

add_custom_command(TARGET versus_test POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
        "${CMAKE_SOURCE_DIR}/resources"
        "$<TARGET_FILE_DIR:versus_test>/resources"
)

add_test(NAME versus_test COMMAND versus_test)
set_tests_properties(versus_test PROPERTIES LABELS "integration")

# search_benchmark_test (benchmark)
add_executable(search_benchmark_test
    search_benchmark_test.cpp
    ${ENGINE_MOVE_SOURCES}
    ${CMAKE_SOURCE_DIR}/utils/logger.cpp
)
chess2d_set_cxx_standard(search_benchmark_test)

target_include_directories(search_benchmark_test PRIVATE ${COMMON_INCLUDES})

target_compile_definitions(search_benchmark_test PRIVATE
    CHESS2D_SOURCE_DIR="${CMAKE_SOURCE_DIR}"
)

target_link_libraries(search_benchmark_test PRIVATE spdlog::spdlog)
chess2d_target_tracy(search_benchmark_test)

add_test(NAME search_benchmark_test COMMAND search_benchmark_test)
set_tests_properties(search_benchmark_test PROPERTIES LABELS "benchmark" TIMEOUT 120)
