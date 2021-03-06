cmake_minimum_required(VERSION 2.8)

find_package(Git QUIET)

set(test_suites
    c-sharp.test
    c.test
    cpp.test
    d.test
    ecma.test
    imported.test
    java.test
    objective-c.test
    pawn.test
    vala.test
)

option("UNCRUSTIFY_SEPARATE_TESTS"
  "Create a separate CTest test for each test case;\
\ this is slower, especially with Python 3" OFF
)
if (UNCRUSTIFY_SEPARATE_TESTS)
  set(tests_ctest_file "${CMAKE_CURRENT_BINARY_DIR}/tests.cmake")
  set_property(DIRECTORY PROPERTY TEST_INCLUDE_FILE ${tests_ctest_file})

  add_custom_command(
    OUTPUT ${tests_ctest_file}
    COMMAND ${PYTHON_EXECUTABLE} run_tests.py
      --executable $<TARGET_FILE:uncrustify>
      --git ${GIT_EXECUTABLE}
      --python ${PYTHON_EXECUTABLE}
      --write-ctest ${tests_ctest_file}
      --cmake-config "$<CONFIGURATION>"
    VERBATIM
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS ${test_suites}
  )

  add_custom_target(register_tests ALL DEPENDS ${tests_ctest_file})
else()
  foreach(suite IN LISTS test_suites)
    string(REPLACE ".test" "" lang "${suite}")
    add_test(NAME ${lang}
      COMMAND ${PYTHON_EXECUTABLE} run_tests.py ${lang}
        --executable $<TARGET_FILE:uncrustify>
        -d --git ${GIT_EXECUTABLE}
        --result-dir ${CMAKE_CURRENT_BINARY_DIR}
      WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    )
  endforeach()
endif()

if (CMAKE_CONFIGURATION_TYPES)
  # Not really, but just to make the following if() true
  set(_build_type release)
  set(_configs CONFIGURATIONS Release RelWithDebInfo MinSizeRel)
else()
  string(TOLOWER "${CMAKE_BUILD_TYPE}" _build_type)
  set(_configs)
endif()

# Only enable test_cli_options stuff if building a release configuration,
# or using a multi-configuration generator (note: in the latter case, these
# will fail if a release configuration has not been built)
if (_build_type MATCHES "release|relwithdebinfo|minsizerel")
  add_test(
    NAME cli_options
    COMMAND ${PYTHON_EXECUTABLE}
      test_cli_options.py
      --build ${uncrustify_BINARY_DIR}
      --cache ${CMAKE_BINARY_DIR}/CMakeCache.txt
      --diff
    ${_configs}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/cli
  )

  add_custom_target(update_cli_options
    COMMAND ${PYTHON_EXECUTABLE}
      test_cli_options.py
      --build ${uncrustify_BINARY_DIR}
      --cache ${CMAKE_BINARY_DIR}/CMakeCache.txt
      --apply
    DEPENDS uncrustify
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/cli
  )
endif()

add_test(NAME sanity COMMAND uncrustify --help)
