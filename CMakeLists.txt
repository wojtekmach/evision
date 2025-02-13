cmake_minimum_required(VERSION 3.3 FATAL_ERROR)
project(evision)

if(NOT DEFINED CMAKE_BUILD_TYPE OR "${CMAKE_BUILD_TYPE}" STREQUAL "")
    set(CMAKE_BUILD_TYPE "Release")
endif()

if(DEFINED MIX_APP_PATH AND NOT "${MIX_APP_PATH}" STREQUAL "")
    set(PRIV_DIR "${MIX_APP_PATH}/priv")
else()
    set(PRIV_DIR "${CMAKE_CURRENT_SOURCE_DIR}/priv")
endif()
message(STATUS "Using PRIV_DIR: ${PRIV_DIR}")

if(DEFINED ERTS_INCLUDE_DIR AND NOT "${ERTS_INCLUDE_DIR}" STREQUAL "")
    set(ERTS_INCLUDE_DIR "${ERTS_INCLUDE_DIR}")
else()
    if(WIN32)
        execute_process(COMMAND powershell -command "erl -noshell -s init stop -eval \"io:format('~ts/erts-~ts/include/', [code:root_dir(), erlang:system_info(version)]).\"" OUTPUT_VARIABLE ERTS_INCLUDE_DIR)
    else()
        execute_process(COMMAND bash -c "erl -noshell -s init stop -eval \"io:format('~ts/erts-~ts/include/', [code:root_dir(), erlang:system_info(version)]).\"" OUTPUT_VARIABLE ERTS_INCLUDE_DIR)
    endif()
    set(ERTS_INCLUDE_DIR "${ERTS_INCLUDE_DIR}")
endif()
message(STATUS "Using ERTS_INCLUDE_DIR: ${ERTS_INCLUDE_DIR}")

find_package(OpenCV REQUIRED PATHS "${PRIV_DIR}/" NO_DEFAULT_PATH)
include_directories("${OpenCV_INCLUDE_DIRS}")

if(POLICY CMP0068)
    cmake_policy(SET CMP0068 NEW)
endif()

if(WIN32)
    execute_process(COMMAND "rmdir ${GENERATED_ELIXIR_SRC_DIR} /s /q && rmdir ${GENERATED_ERLANG_SRC_DIR} /s /q  && mkdir ${GENERATED_ELIXIR_SRC_DIR} && mkdir ${GENERATED_ERLANG_SRC_DIR}")
    message(STATUS "Enabled modules: ${ENABLED_CV_MODULES}")
    execute_process(COMMAND python3.exe "${PY_SRC}\\gen2.py" "${C_SRC}" "${GENERATED_ELIXIR_SRC_DIR}" "${GENERATED_ERLANG_SRC_DIR}" "${C_SRC}\\headers.txt" "${EVISION_GENERATE_LANG}" "${ENABLED_CV_MODULES}" RESULT_VARIABLE STATUS)
else()
    execute_process(COMMAND bash -c "rm -rf ${GENERATED_ELIXIR_SRC_DIR} && rm -rf ${GENERATED_ERLANG_SRC_DIR} && mkdir -p ${GENERATED_ELIXIR_SRC_DIR} && mkdir -p ${GENERATED_ERLANG_SRC_DIR}")
    message(STATUS "Enabled modules: ${ENABLED_CV_MODULES}")
    execute_process(COMMAND bash -c "python3 ${PY_SRC}/gen2.py ${C_SRC} ${GENERATED_ELIXIR_SRC_DIR} ${GENERATED_ERLANG_SRC_DIR} ${C_SRC}/headers.txt ${EVISION_GENERATE_LANG} ${ENABLED_CV_MODULES}" RESULT_VARIABLE STATUS)
endif()
if(STATUS STREQUAL "0")
    message(STATUS "Successfully generated binding code for: ${EVISION_GENERATE_LANG}")
else()
    message(FATAL_ERROR "Failed to generate binding code for: ${EVISION_GENERATE_LANG}")
endif()

if(WIN32)
    string(REPLACE "\\" "/" C_SRC "${C_SRC}")
endif()
file(GLOB evision_bindings_main "${C_SRC}/*.hpp" "${C_SRC}/*.h" "${C_SRC}/*.cpp")
file(GLOB evision_custom_headers "${C_SRC}/evision_custom_headers/*.hpp")
file(GLOB evision_modules "${C_SRC}/modules/*.hpp" "${C_SRC}/modules/*.h")
file(GLOB evision_backend "${C_SRC}/modules/evision_backend/*.h")

add_library(evision SHARED
    ${evision_bindings_main}
    ${evision_custom_headers}
    ${evision_modules}
    ${evision_backend}
)
target_link_libraries(evision "${OpenCV_LIBS}")
set_property(TARGET evision PROPERTY CXX_STANDARD 14)
set_target_properties(evision PROPERTIES PREFIX "")
set_target_properties(evision PROPERTIES SUFFIX ".so")

set_target_properties(evision PROPERTIES
    INSTALL_RPATH_USE_LINK_PATH TRUE
    BUILD_WITH_INSTALL_RPATH TRUE
)

if(UNIX AND NOT APPLE)
    set_target_properties(evision PROPERTIES INSTALL_RPATH "\$ORIGIN/lib")
elseif(UNIX AND APPLE)
    # Although the compiler complains about not using these,
    # things only work with them set
    set(CMAKE_SHARED_LINKER_FLAGS "-flat_namespace -undefined suppress -undefined dynamic_lookup")
    set_target_properties(evision PROPERTIES INSTALL_RPATH "@loader_path/lib")
endif()

include_directories(${ERTS_INCLUDE_DIR})
if(WIN32)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /bigobj /wd4996 /wd4267 /wd4068")
else()
    if (CMAKE_BUILD_TYPE STREQUAL "Debug")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g3")
    else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3")
    endif()

    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -Wall -Wextra -Wno-unused-parameter -Wno-missing-field-initializers -Wno-deprecated-declarations -Wno-unused-but-set-variable")
endif()
