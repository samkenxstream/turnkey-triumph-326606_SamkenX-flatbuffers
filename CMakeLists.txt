# This is the legacy minimum version flatbuffers supported for a while.
cmake_minimum_required(VERSION 2.8.12...3.22.1)

# CMake version 3.16 is the 'de-facto' minimum version for flatbuffers. If the
# current cmake is older than this, warn the user and include the legacy file to
# provide some level of support.
if(CMAKE_VERSION VERSION_LESS 3.16)
  message(WARNING "Using cmake version ${CMAKE_VERSION} which is older than "
  "our target version of 3.16. This will use the legacy CMakeLists.txt that "
  "supports version 2.8.12 and higher, but not actively maintained. Consider "
  "upgrading cmake to a newer version, as this may become a fatal error in the "
  "future.")
  # Use the legacy version of CMakeLists.txt
  include(CMake/CMakeLists_legacy.cmake.in)
  return()
endif()

# Attempt to read the current version of flatbuffers by looking at the latest tag.
include(CMake/Version.cmake)

if (POLICY CMP0048)
  cmake_policy(SET CMP0048 NEW)
  project(FlatBuffers
        DESCRIPTION "Flatbuffers serialization library"
        VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}
        LANGUAGES CXX)
else()
  project(FlatBuffers)
endif (POLICY CMP0048)

# generate compile_commands.json
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# NOTE: Code coverage only works on Linux & OSX.
option(FLATBUFFERS_CODE_COVERAGE "Enable the code coverage build option." OFF)
option(FLATBUFFERS_BUILD_TESTS "Enable the build of tests and samples." ON)
option(FLATBUFFERS_INSTALL "Enable the installation of targets." ON)
option(FLATBUFFERS_BUILD_FLATLIB "Enable the build of the flatbuffers library"
       ON)
option(FLATBUFFERS_BUILD_FLATC "Enable the build of the flatbuffers compiler"
       ON)
option(FLATBUFFERS_STATIC_FLATC "Build flatbuffers compiler with -static flag"
       OFF)
option(FLATBUFFERS_BUILD_FLATHASH "Enable the build of flathash" ON)
option(FLATBUFFERS_BUILD_BENCHMARKS "Enable the build of flatbenchmark."
       OFF)
option(FLATBUFFERS_BUILD_GRPCTEST "Enable the build of grpctest" OFF)
option(FLATBUFFERS_BUILD_SHAREDLIB
       "Enable the build of the flatbuffers shared library"
       OFF)
option(FLATBUFFERS_LIBCXX_WITH_CLANG "Force libc++ when using Clang" ON)
# NOTE: Sanitizer check only works on Linux & OSX (gcc & llvm).
option(FLATBUFFERS_CODE_SANITIZE
      "Add '-fsanitize' flags to 'flattests' and 'flatc' targets."
      OFF)
option(FLATBUFFERS_PACKAGE_REDHAT
       "Build an rpm using the 'package' target."
       OFF)
option(FLATBUFFERS_PACKAGE_DEBIAN
       "Build an deb using the 'package' target."
       OFF)
option(FLATBUFFERS_BUILD_CPP17
       "Enable the build of c++17 test target. \"
       Requirements: Clang6, GCC7, MSVC2017 (_MSC_VER >= 1914)  or higher."
       OFF)
option(FLATBUFFERS_BUILD_LEGACY
       "Run C++ code generator with '--cpp-std c++0x' switch."
       OFF)
option(FLATBUFFERS_ENABLE_PCH
       "Enable precompile headers support for 'flatbuffers' and 'flatc'. \"
        Only work if CMake supports 'target_precompile_headers'. \"
        This can speed up compilation time."
       OFF)
option(FLATBUFFERS_SKIP_MONSTER_EXTRA 
      "Skip generating monster_extra.fbs that contains non-supported numerical\"
      types." OFF)

if(NOT FLATBUFFERS_BUILD_FLATC AND FLATBUFFERS_BUILD_TESTS)
    message(WARNING
    "Cannot build tests without building the compiler. Tests will be disabled.")
    set(FLATBUFFERS_BUILD_TESTS OFF)
endif()

if(DEFINED FLATBUFFERS_MAX_PARSING_DEPTH)
  # Override the default recursion depth limit.
  add_definitions(-DFLATBUFFERS_MAX_PARSING_DEPTH=${FLATBUFFERS_MAX_PARSING_DEPTH})
  message(STATUS "FLATBUFFERS_MAX_PARSING_DEPTH: ${FLATBUFFERS_MAX_PARSING_DEPTH}")
endif()

# Auto-detect locale-narrow 'strtod_l' and  'strtoull_l' functions.
if(NOT DEFINED FLATBUFFERS_LOCALE_INDEPENDENT)
  include(CheckCXXSymbolExists)

  set(FLATBUFFERS_LOCALE_INDEPENDENT 0)
  if(MSVC)
    check_cxx_symbol_exists(_strtof_l stdlib.h FLATBUFFERS_HAS_STRTOF_L)
    check_cxx_symbol_exists(_strtoui64_l stdlib.h FLATBUFFERS_HAS_STRTOULL_L)
  else()
    check_cxx_symbol_exists(strtof_l stdlib.h FLATBUFFERS_HAS_STRTOF_L)
    check_cxx_symbol_exists(strtoull_l stdlib.h FLATBUFFERS_HAS_STRTOULL_L)
  endif()
  if(FLATBUFFERS_HAS_STRTOF_L AND FLATBUFFERS_HAS_STRTOULL_L)
    set(FLATBUFFERS_LOCALE_INDEPENDENT 1)
  endif()
endif()
add_definitions(-DFLATBUFFERS_LOCALE_INDEPENDENT=$<BOOL:${FLATBUFFERS_LOCALE_INDEPENDENT}>)

set(FlatBuffers_Library_SRCS
  include/flatbuffers/allocator.h
  include/flatbuffers/array.h
  include/flatbuffers/base.h
  include/flatbuffers/bfbs_generator.h
  include/flatbuffers/buffer.h
  include/flatbuffers/buffer_ref.h
  include/flatbuffers/default_allocator.h
  include/flatbuffers/detached_buffer.h
  include/flatbuffers/flatbuffer_builder.h
  include/flatbuffers/flatbuffers.h
  include/flatbuffers/flexbuffers.h
  include/flatbuffers/flex_flat_util.h
  include/flatbuffers/hash.h
  include/flatbuffers/idl.h
  include/flatbuffers/minireflect.h
  include/flatbuffers/reflection.h
  include/flatbuffers/reflection_generated.h
  include/flatbuffers/registry.h
  include/flatbuffers/stl_emulation.h
  include/flatbuffers/string.h
  include/flatbuffers/struct.h
  include/flatbuffers/table.h
  include/flatbuffers/util.h
  include/flatbuffers/vector.h
  include/flatbuffers/vector_downward.h
  include/flatbuffers/verifier.h
  src/idl_parser.cpp
  src/idl_gen_text.cpp
  src/reflection.cpp
  src/util.cpp
)

set(FlatBuffers_Compiler_SRCS
  ${FlatBuffers_Library_SRCS}
  src/idl_gen_cpp.cpp
  src/idl_gen_csharp.cpp
  src/idl_gen_dart.cpp
  src/idl_gen_kotlin.cpp
  src/idl_gen_go.cpp
  src/idl_gen_java.cpp
  src/idl_gen_ts.cpp
  src/idl_gen_php.cpp
  src/idl_gen_python.cpp
  src/idl_gen_lobster.cpp
  src/idl_gen_lua.cpp
  src/idl_gen_rust.cpp
  src/idl_gen_fbs.cpp
  src/idl_gen_grpc.cpp
  src/idl_gen_json_schema.cpp
  src/idl_gen_swift.cpp
  src/idl_namer.h
  src/namer.h
  src/flatc.cpp
  src/flatc_main.cpp
  src/bfbs_gen.h
  src/bfbs_gen_lua.h
  src/bfbs_namer.h
  include/flatbuffers/code_generators.h
  src/binary_annotator.h
  src/binary_annotator.cpp
  src/annotated_binary_text_gen.h
  src/annotated_binary_text_gen.cpp
  src/bfbs_gen_lua.cpp
  src/code_generators.cpp
  grpc/src/compiler/schema_interface.h
  grpc/src/compiler/cpp_generator.h
  grpc/src/compiler/cpp_generator.cc
  grpc/src/compiler/go_generator.h
  grpc/src/compiler/go_generator.cc
  grpc/src/compiler/java_generator.h
  grpc/src/compiler/java_generator.cc
  grpc/src/compiler/python_generator.h
  grpc/src/compiler/python_generator.cc
  grpc/src/compiler/swift_generator.h
  grpc/src/compiler/swift_generator.cc
  grpc/src/compiler/ts_generator.h
  grpc/src/compiler/ts_generator.cc
)

set(FlatHash_SRCS
  include/flatbuffers/hash.h
  src/flathash.cpp
)

set(FlatBuffers_Tests_SRCS
  ${FlatBuffers_Library_SRCS}
  src/idl_gen_fbs.cpp
  tests/test.cpp
  tests/test_assert.h
  tests/test_assert.cpp
  tests/test_builder.h
  tests/test_builder.cpp
  tests/native_type_test_impl.h
  tests/native_type_test_impl.cpp
  include/flatbuffers/code_generators.h
  src/code_generators.cpp
  # file generate by running compiler on tests/monster_test.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/tests/monster_test_generated.h
  # file generate by running compiler on namespace_test/namespace_test1.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/tests/namespace_test/namespace_test1_generated.h
  ${CMAKE_CURRENT_BINARY_DIR}/tests/namespace_test/namespace_test2_generated.h
  # file generate by running compiler on union_vector/union_vector.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/tests/union_vector/union_vector_generated.h
  # file generate by running compiler on tests/arrays_test.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/tests/arrays_test_generated.h
  # file generate by running compiler on tests/native_type_test.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/tests/native_type_test_generated.h
  # file generate by running compiler on tests/monster_extra.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/tests/monster_extra_generated.h
  # file generate by running compiler on tests/monster_test.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/tests/monster_test_bfbs_generated.h
  # file generate by running compiler on tests/optional_scalars.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/tests/optional_scalars_generated.h
)

set(FlatBuffers_Tests_CPP17_SRCS
  ${FlatBuffers_Library_SRCS}
  tests/test_assert.h
  tests/test_assert.cpp
  tests/cpp17/test_cpp17.cpp
  # file generate by running compiler on tests/monster_test.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/tests/cpp17/generated_cpp17/monster_test_generated.h
  ${CMAKE_CURRENT_BINARY_DIR}/tests/monster_test_generated.h
  ${CMAKE_CURRENT_BINARY_DIR}/tests/cpp17/generated_cpp17/optional_scalars_generated.h
  ${CMAKE_CURRENT_BINARY_DIR}/tests/optional_scalars_generated.h
)

set(FlatBuffers_Sample_Binary_SRCS
  include/flatbuffers/flatbuffers.h
  samples/sample_binary.cpp
  # file generated by running compiler on samples/monster.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/samples/monster_generated.h
)

set(FlatBuffers_Sample_Text_SRCS
  ${FlatBuffers_Library_SRCS}
  samples/sample_text.cpp
  # file generated by running compiler on samples/monster.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/samples/monster_generated.h
)

set(FlatBuffers_Sample_BFBS_SRCS
  ${FlatBuffers_Library_SRCS}
  samples/sample_bfbs.cpp
  # file generated by running compiler on samples/monster.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/samples/monster_generated.h
)

set(FlatBuffers_GRPCTest_SRCS
  include/flatbuffers/flatbuffers.h
  include/flatbuffers/grpc.h
  include/flatbuffers/util.h
  src/util.cpp
  tests/monster_test.grpc.fb.h
  tests/test_assert.h
  tests/test_builder.h
  tests/monster_test.grpc.fb.cc
  tests/test_assert.cpp
  tests/test_builder.cpp
  grpc/tests/grpctest.cpp
  grpc/tests/message_builder_test.cpp
  # file generate by running compiler on tests/monster_test.fbs
  ${CMAKE_CURRENT_BINARY_DIR}/tests/monster_test_generated.h
)

# TODO(dbaileychess): Figure out how this would now work. I posted a question on
# https://stackoverflow.com/questions/71772330/override-target-compile-options-via-cmake-command-line.
# Append FLATBUFFERS_CXX_FLAGS to CMAKE_CXX_FLAGS.
if(DEFINED FLATBUFFERS_CXX_FLAGS AND NOT EXISTS "${CMAKE_TOOLCHAIN_FILE}")
  message(STATUS "extend CXX_FLAGS with ${FLATBUFFERS_CXX_FLAGS}")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${FLATBUFFERS_CXX_FLAGS}")
endif()
message(STATUS "CMAKE_CXX_FLAGS: ${CMAKE_CXX_FLAGS}")

function(add_fsanitize_to_target _target _sanitizer)
  if(WIN32)
    target_compile_definitions(${_target} PRIVATE FLATBUFFERS_MEMORY_LEAK_TRACKING)
    message(STATUS "Sanitizer MSVC::_CrtDumpMemoryLeaks added to ${_target}")
  else()
    # FLATBUFFERS_CODE_SANITIZE: boolean {ON,OFF,YES,NO} or string with list of sanitizer.
    # List of sanitizer is string starts with '=': "=address,undefined,thread,memory".
    if((${CMAKE_CXX_COMPILER_ID} MATCHES "Clang") OR
      ((${CMAKE_CXX_COMPILER_ID} MATCHES "GNU") AND NOT (CMAKE_CXX_COMPILER_VERSION VERSION_LESS "4.9"))
    )
      set(_sanitizer_flags "=address,undefined")
      if(_sanitizer MATCHES "=.*")
        # override default by user-defined sanitizer list
        set(_sanitizer_flags ${_sanitizer})
      endif()
      target_compile_options(${_target} PRIVATE
        -g -fsigned-char -fno-omit-frame-pointer
        "-fsanitize${_sanitizer_flags}")
      target_link_libraries(${_target} PRIVATE
        "-fsanitize${_sanitizer_flags}")
      set_property(TARGET ${_target} PROPERTY POSITION_INDEPENDENT_CODE ON)
      message(STATUS "Sanitizer ${_sanitizer_flags} added to ${_target}")
    endif()
  endif()
endfunction()

function(add_pch_to_target _target _pch_header)
  if(COMMAND target_precompile_headers)
    target_precompile_headers(${_target} PRIVATE ${_pch_header})
    if(NOT MSVC)
      set_source_files_properties(src/util.cpp PROPERTIES SKIP_PRECOMPILE_HEADERS ON)
    endif()
  endif()
endfunction()

include_directories(include)
include_directories(grpc)

# Creates an interface library that stores the configuration settings that each
# target links too. This is a compromise between setting configuration globally
# with add_compile_options() and the more targetted target_compile_options(). 
# This way each target in this file can share settings and override them if 
# needed.
add_library(ProjectConfig INTERFACE)
target_compile_features(ProjectConfig
  INTERFACE
    cxx_std_11
)

# Force the standard to be met.
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# We shouldn't rely on any compiler-extensions to make things work.
set(CMAKE_CXX_EXTENSIONS OFF)

if(MSVC) 
  target_compile_options(ProjectConfig
    INTERFACE 
      /W4
      /WX
      /wd4512   # C4512: assignment operator could not be generated
      /wd4316   # C4316: object allocated on the heap may not be aligned
      /wd4456   # C4456: hides previous local declaration
      $<$<CXX_COMPILER_ID:CLANG>:
        /D_CRT_SECURE_NO_WARNINGS
      >
  )
else()
  target_compile_options(ProjectConfig
    INTERFACE
      -Wall
      -Werror
      -pedantic 
      -Werror 
      -Wextra 
      -Wno-unused-parameter
      -Wold-style-cast
      -Wimplicit-fallthrough
      -fsigned-char
      -Wnon-virtual-dtor

      $<$<CXX_COMPILER_ID:CLANG>:
        $<$<VERSION_GREATER:$<CXX_COMPILER_VERSION>,3.8>:
          -Wimplicit-fallthrough
          -Wextra-semi
          -Werror=unused-private-field
        >
      >

      $<$<CXX_COMPILER_ID:GNU>:
        $<$<VERSION_GREATER:$<CXX_COMPILER_VERSION>,4.4>:
          -Wunused-result 
          -Werror=unused-result 
          -Wunused-parameter 
          -Werror=unused-parameter
        >
        $<$<VERSION_GREATER:$<CXX_COMPILER_VERSION>,7.0>:
          -faligned-new 
          -Werror=implicit-fallthrough=2
        >
        $<$<VERSION_GREATER:$<CXX_COMPILER_VERSION>,8.0>:
          -Wextra-semi
        >
      >

      $<$<BOOL:${FLATBUFFERS_CODE_COVERAGE}>:
        -g
        -fprofile-arcs
        -ftest-coverage
      >
    )

  if(FLATBUFFERS_CODE_COVERAGE)
    target_link_options(ProjectConfig
      INTERFACE
        -fprofile-arcs
        -ftest-coverage
    )
  endif()
endif()

if(FLATBUFFERS_BUILD_FLATLIB)
  add_library(flatbuffers STATIC ${FlatBuffers_Library_SRCS})

  # Attach header directory for when build via add_subdirectory().
  target_include_directories(flatbuffers 
    INTERFACE
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  )
  target_link_libraries(flatbuffers PRIVATE $<BUILD_INTERFACE:ProjectConfig>)

  if(FLATBUFFERS_ENABLE_PCH)
    add_pch_to_target(flatbuffers include/flatbuffers/pch/pch.h)
  endif()
endif()

if(FLATBUFFERS_BUILD_FLATC)
  add_executable(flatc ${FlatBuffers_Compiler_SRCS})
  if(FLATBUFFERS_ENABLE_PCH)
    add_pch_to_target(flatc include/flatbuffers/pch/flatc_pch.h)
  endif()

  target_link_libraries(flatc PRIVATE $<BUILD_INTERFACE:ProjectConfig>)
  target_compile_options(flatc 
    PUBLIC
      $<$<AND:$<CXX_COMPILER_ID:MSVC>,$<CONFIG:Release>>:
        /MT
      >
  )

  if(FLATBUFFERS_CODE_SANITIZE AND NOT WIN32)
    add_fsanitize_to_target(flatc ${FLATBUFFERS_CODE_SANITIZE})
  endif()
  if(NOT FLATBUFFERS_FLATC_EXECUTABLE)
    set(FLATBUFFERS_FLATC_EXECUTABLE $<TARGET_FILE:flatc>)
  endif()
  if(FLATBUFFERS_STATIC_FLATC AND NOT MSVC)
    target_link_libraries(flatc PRIVATE -static)
  endif()
endif()

if(FLATBUFFERS_BUILD_FLATHASH)
  add_executable(flathash ${FlatHash_SRCS})
  target_link_libraries(flathash PRIVATE $<BUILD_INTERFACE:ProjectConfig>)
endif()

if(FLATBUFFERS_BUILD_SHAREDLIB)
  add_library(flatbuffers_shared SHARED ${FlatBuffers_Library_SRCS})
  target_link_libraries(flatbuffers_shared PRIVATE $<BUILD_INTERFACE:ProjectConfig>)
  # Shared object version: "major.minor.micro"
  # - micro updated every release when there is no API/ABI changes
  # - minor updated when there are additions in API/ABI
  # - major (ABI number) updated when there are changes in ABI (or removals)
  set(FlatBuffers_Library_SONAME_MAJOR ${VERSION_MAJOR})
  set(FlatBuffers_Library_SONAME_FULL "${FlatBuffers_Library_SONAME_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}")
  set_target_properties(flatbuffers_shared PROPERTIES OUTPUT_NAME flatbuffers
                        SOVERSION "${FlatBuffers_Library_SONAME_MAJOR}"
                        VERSION "${FlatBuffers_Library_SONAME_FULL}")
  if(FLATBUFFERS_ENABLE_PCH)
    add_pch_to_target(flatbuffers_shared include/flatbuffers/pch/pch.h)
  endif()
endif()

# Global list of generated files.
# Use the global property to be independent of PARENT_SCOPE.
set_property(GLOBAL PROPERTY FBS_GENERATED_OUTPUTS)

function(get_generated_output generated_files)
  get_property(tmp GLOBAL PROPERTY FBS_GENERATED_OUTPUTS)
  set(${generated_files} ${tmp} PARENT_SCOPE)
endfunction(get_generated_output)

function(register_generated_output file_name)
  get_property(tmp GLOBAL PROPERTY FBS_GENERATED_OUTPUTS)
  list(APPEND tmp ${file_name})
  set_property(GLOBAL PROPERTY FBS_GENERATED_OUTPUTS ${tmp})
endfunction(register_generated_output)

function(compile_flatbuffers_schema_to_cpp_opt SRC_FBS OPT)
  if(FLATBUFFERS_BUILD_LEGACY)
    set(OPT ${OPT};--cpp-std c++0x)
  else()
    # --cpp-std is defined by flatc default settings.
  endif()
  message(STATUS "`${SRC_FBS}`: add generation of C++ code with '${OPT}'")
  get_filename_component(SRC_FBS_DIR ${SRC_FBS} PATH)
  string(REGEX REPLACE "\\.fbs$" "_generated.h" GEN_HEADER ${SRC_FBS})
  add_custom_command(
    OUTPUT ${GEN_HEADER}
    COMMAND "${FLATBUFFERS_FLATC_EXECUTABLE}"
            --cpp --gen-mutable --gen-object-api --reflect-names
            --cpp-ptr-type flatbuffers::unique_ptr # Used to test with C++98 STLs
            ${OPT}
            -I "${CMAKE_CURRENT_SOURCE_DIR}/tests/include_test"
            -o "${SRC_FBS_DIR}"
            "${CMAKE_CURRENT_SOURCE_DIR}/${SRC_FBS}"
    DEPENDS flatc
    COMMENT "Run generation: '${GEN_HEADER}'")
  register_generated_output(${GEN_HEADER})
endfunction()

function(compile_flatbuffers_schema_to_cpp SRC_FBS)
  compile_flatbuffers_schema_to_cpp_opt(${SRC_FBS} "--no-includes;--gen-compare")
endfunction()

function(compile_flatbuffers_schema_to_binary SRC_FBS)
  message(STATUS "`${SRC_FBS}`: add generation of binary (.bfbs) schema")
  get_filename_component(SRC_FBS_DIR ${SRC_FBS} PATH)
  string(REGEX REPLACE "\\.fbs$" ".bfbs" GEN_BINARY_SCHEMA ${SRC_FBS})
  # For details about flags see generate_code.py
  add_custom_command(
    OUTPUT ${GEN_BINARY_SCHEMA}
    COMMAND "${FLATBUFFERS_FLATC_EXECUTABLE}"
            -b --schema --bfbs-comments --bfbs-builtins
            --bfbs-filenames ${SRC_FBS_DIR}
            -I "${CMAKE_CURRENT_SOURCE_DIR}/tests/include_test"
            -o "${SRC_FBS_DIR}"
            "${CMAKE_CURRENT_SOURCE_DIR}/${SRC_FBS}"
    DEPENDS flatc
    COMMENT "Run generation: '${GEN_BINARY_SCHEMA}'")
  register_generated_output(${GEN_BINARY_SCHEMA})
endfunction()

function(compile_flatbuffers_schema_to_embedded_binary SRC_FBS OPT)
  if(FLATBUFFERS_BUILD_LEGACY)
    set(OPT ${OPT};--cpp-std c++0x)
  else()
    # --cpp-std is defined by flatc default settings.
  endif()
  message(STATUS "`${SRC_FBS}`: add generation of C++ embedded binary schema code with '${OPT}'")
  get_filename_component(SRC_FBS_DIR ${SRC_FBS} PATH)
  string(REGEX REPLACE "\\.fbs$" "_bfbs_generated.h" GEN_BFBS_HEADER ${SRC_FBS})
  # For details about flags see generate_code.py
  add_custom_command(
          OUTPUT ${GEN_BFBS_HEADER}
          COMMAND "${FLATBUFFERS_FLATC_EXECUTABLE}"
          --cpp --gen-mutable --gen-object-api --reflect-names
          --cpp-ptr-type flatbuffers::unique_ptr # Used to test with C++98 STLs
          ${OPT}
          --bfbs-comments --bfbs-builtins --bfbs-gen-embed
          --bfbs-filenames ${SRC_FBS_DIR}
          -I "${CMAKE_CURRENT_SOURCE_DIR}/tests/include_test"
          -o "${SRC_FBS_DIR}"
          "${CMAKE_CURRENT_SOURCE_DIR}/${SRC_FBS}"
          DEPENDS flatc
          COMMENT "Run generation: '${GEN_BFBS_HEADER}'")
  register_generated_output(${GEN_BFBS_HEADER})
endfunction()

if(FLATBUFFERS_BUILD_TESTS)
  file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/tests" DESTINATION "${CMAKE_CURRENT_BINARY_DIR}")
  file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/samples" DESTINATION "${CMAKE_CURRENT_BINARY_DIR}")

  # TODO Add (monster_test.fbs monsterdata_test.json)->monsterdata_test.mon
  compile_flatbuffers_schema_to_cpp(tests/monster_test.fbs)
  compile_flatbuffers_schema_to_binary(tests/monster_test.fbs)
  compile_flatbuffers_schema_to_cpp_opt(tests/namespace_test/namespace_test1.fbs "--no-includes;--gen-compare;--gen-name-strings")
  compile_flatbuffers_schema_to_cpp_opt(tests/namespace_test/namespace_test2.fbs "--no-includes;--gen-compare;--gen-name-strings")
  compile_flatbuffers_schema_to_cpp_opt(tests/union_vector/union_vector.fbs "--no-includes;--gen-compare;--gen-name-strings")
  compile_flatbuffers_schema_to_cpp(tests/optional_scalars.fbs)
  compile_flatbuffers_schema_to_cpp_opt(tests/native_type_test.fbs "")
  compile_flatbuffers_schema_to_cpp_opt(tests/arrays_test.fbs "--scoped-enums;--gen-compare")
  compile_flatbuffers_schema_to_binary(tests/arrays_test.fbs)
  compile_flatbuffers_schema_to_embedded_binary(tests/monster_test.fbs "--no-includes;--gen-compare")
  if(NOT (MSVC AND (MSVC_VERSION LESS 1900)))
    compile_flatbuffers_schema_to_cpp(tests/monster_extra.fbs) # Test floating-point NAN/INF.
  endif()
  include_directories(${CMAKE_CURRENT_BINARY_DIR}/tests)
  add_executable(flattests ${FlatBuffers_Tests_SRCS})
  target_link_libraries(flattests PRIVATE $<BUILD_INTERFACE:ProjectConfig>)

  add_dependencies(flattests generated_code)

  if(FLATBUFFERS_CODE_SANITIZE)
    add_fsanitize_to_target(flattests ${FLATBUFFERS_CODE_SANITIZE})
  endif()

  compile_flatbuffers_schema_to_cpp(samples/monster.fbs)
  compile_flatbuffers_schema_to_binary(samples/monster.fbs)
  include_directories(${CMAKE_CURRENT_BINARY_DIR}/samples)

  add_executable(flatsamplebinary ${FlatBuffers_Sample_Binary_SRCS})
  target_link_libraries(flatsamplebinary PRIVATE $<BUILD_INTERFACE:ProjectConfig>)
  add_dependencies(flatsamplebinary generated_code)

  add_executable(flatsampletext ${FlatBuffers_Sample_Text_SRCS})
  target_link_libraries(flatsampletext PRIVATE $<BUILD_INTERFACE:ProjectConfig>)
  add_dependencies(flatsampletext generated_code)

  add_executable(flatsamplebfbs ${FlatBuffers_Sample_BFBS_SRCS})
  target_link_libraries(flatsamplebfbs PRIVATE $<BUILD_INTERFACE:ProjectConfig>)
  add_dependencies(flatsamplebfbs generated_code)

  if(FLATBUFFERS_BUILD_CPP17)
    # Don't generate header for flattests_cpp17 target.
    # This target uses "generated_cpp17/monster_test_generated.h"
    add_executable(flattests_cpp17 ${FlatBuffers_Tests_CPP17_SRCS})
    add_dependencies(flattests_cpp17 generated_code)
    target_link_libraries(flattests_cpp17 PRIVATE $<BUILD_INTERFACE:ProjectConfig>)
    target_compile_features(flattests_cpp17 PRIVATE cxx_std_17)

    if(FLATBUFFERS_CODE_SANITIZE)
      add_fsanitize_to_target(flattests_cpp17 ${FLATBUFFERS_CODE_SANITIZE})
    endif()
  endif(FLATBUFFERS_BUILD_CPP17)
endif()

if(FLATBUFFERS_BUILD_GRPCTEST)
  if(NOT GRPC_INSTALL_PATH)
    message(SEND_ERROR "GRPC_INSTALL_PATH variable is not defined. See grpc/README.md")
  endif()
  if(NOT PROTOBUF_DOWNLOAD_PATH)
    message(SEND_ERROR "PROTOBUF_DOWNLOAD_PATH variable is not defined. See grpc/README.md")
  endif()
  INCLUDE_DIRECTORIES(${GRPC_INSTALL_PATH}/include)
  INCLUDE_DIRECTORIES(${PROTOBUF_DOWNLOAD_PATH}/src)
  find_package(Threads REQUIRED)
  list(APPEND CMAKE_PREFIX_PATH ${GRPC_INSTALL_PATH})
  find_package(absl CONFIG REQUIRED)
  find_package(protobuf CONFIG REQUIRED)
  find_package(gRPC CONFIG REQUIRED)
  add_executable(grpctest ${FlatBuffers_GRPCTest_SRCS})
  add_dependencies(grpctest generated_code)
  target_link_libraries(grpctext 
    PRIVATE 
      $<BUILD_INTERFACE:ProjectConfig>
      gRPC::grpc++_unsecure 
      gRPC::gpr 
      pthread
      dl 
  )
endif()

if(FLATBUFFERS_INSTALL)
  include(GNUInstallDirs)

  install(DIRECTORY include/flatbuffers DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

  set(FB_CMAKE_DIR "${CMAKE_INSTALL_LIBDIR}/cmake/flatbuffers")

  configure_file(CMake/FlatbuffersConfigVersion.cmake.in FlatbuffersConfigVersion.cmake @ONLY)
  install(
      FILES "CMake/FlatbuffersConfig.cmake" "${CMAKE_CURRENT_BINARY_DIR}/FlatbuffersConfigVersion.cmake"
      DESTINATION ${FB_CMAKE_DIR}
  )

  if(FLATBUFFERS_BUILD_FLATLIB)
    install(
      TARGETS flatbuffers EXPORT FlatbuffersTargets
      ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
      INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )

    install(EXPORT FlatbuffersTargets
      FILE FlatbuffersTargets.cmake
      NAMESPACE flatbuffers::
      DESTINATION ${FB_CMAKE_DIR}
    )
  endif()

  if(FLATBUFFERS_BUILD_FLATC)
    install(
      TARGETS flatc EXPORT FlatcTargets
      RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )

    install(
      EXPORT FlatcTargets
      FILE FlatcTargets.cmake
      NAMESPACE flatbuffers::
      DESTINATION ${FB_CMAKE_DIR}
    )
  endif()

  if(FLATBUFFERS_BUILD_SHAREDLIB)
    install(
      TARGETS flatbuffers_shared EXPORT FlatbuffersSharedTargets
      ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
      RUNTIME DESTINATION ${CMAKE_INSTALL_LIBDIR}
      LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
      INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )

    install(
      EXPORT FlatbuffersSharedTargets
      FILE FlatbuffersSharedTargets.cmake
      NAMESPACE flatbuffers::
      DESTINATION ${FB_CMAKE_DIR}
    )
  endif()

  if(FLATBUFFERS_BUILD_SHAREDLIB OR FLATBUFFERS_BUILD_FLATLIB)
      configure_file(CMake/flatbuffers.pc.in flatbuffers.pc @ONLY)
      install(
        FILES "${CMAKE_CURRENT_BINARY_DIR}/flatbuffers.pc"
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
      )
  endif()
endif()

if(FLATBUFFERS_BUILD_TESTS)
  enable_testing()

  add_test(NAME flattests COMMAND flattests)
  if(FLATBUFFERS_BUILD_CPP17)
    add_test(NAME flattests_cpp17 COMMAND flattests_cpp17)
  endif()
  if(FLATBUFFERS_BUILD_GRPCTEST)
    add_test(NAME grpctest COMMAND grpctest)
  endif()
endif()

# This target is sync-barrier.
# Other generate-dependent targets can depend on 'generated_code' only.
get_generated_output(fbs_generated)
if(fbs_generated)
  # message(STATUS "Add generated_code target with files:${fbs_generated}")
  add_custom_target(generated_code
    DEPENDS ${fbs_generated}
    COMMENT "All generated files were updated.")
endif()

include(CMake/BuildFlatBuffers.cmake)

if(UNIX)
    # Use of CPack only supported on Linux systems.
    if(FLATBUFFERS_PACKAGE_DEBIAN)
        include(CMake/PackageDebian.cmake)
        include(CPack)
    endif()
    if (FLATBUFFERS_PACKAGE_REDHAT)
        include(CMake/PackageRedhat.cmake)
        include(CPack)
    endif()
endif()

# Include for running Google Benchmarks.
if(FLATBUFFERS_BUILD_BENCHMARKS)
  add_subdirectory(benchmarks)
endif()

# Add FlatBuffers::FlatBuffers interface, needed for FetchContent_Declare
add_library(FlatBuffers INTERFACE)
add_library(FlatBuffers::FlatBuffers ALIAS FlatBuffers)
target_include_directories(
  FlatBuffers
  INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
            $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/include>)