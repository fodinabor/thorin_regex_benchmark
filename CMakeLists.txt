cmake_minimum_required(VERSION 3.0)

set(CMAKE_CXX_STANDARD 20)

project("benchmark_mail" CXX)

find_package(Thorin REQUIRED CONFIG)

if(NOT THORIN_TARGET_NAMESPACE)
    set(THORIN_TARGET_NAMESPACE "")
endif()

option(REGEX_COMPILE_TIME_BENCHMARK "Enable to run the relevant compilation commands with 'time' - see ReadMe.md" OFF)

if(REGEX_COMPILE_TIME_BENCHMARK)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CMAKE_COMMAND} -E time")
    set_property(GLOBAL PROPERTY RULE_LAUNCH_CUSTOM "${CMAKE_COMMAND} -E time")
endif()

option(REGEX_DETERMINISTIC "Enable to select the deterministic benchmark regex" ON)

if(REGEX_DETERMINISTIC)
    add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/thorin_match.o
                    COMMAND $<TARGET_FILE:${THORIN_TARGET_NAMESPACE}thorin> "${CMAKE_SOURCE_DIR}/match_mail.thorin" --output-ll "${CMAKE_CURRENT_BINARY_DIR}/thorin_match.ll" --aggr-lam-spec
                    COMMAND ${CMAKE_CXX_COMPILER} -O3 "${CMAKE_CURRENT_BINARY_DIR}/thorin_match.ll" -c -o "${CMAKE_CURRENT_BINARY_DIR}/thorin_match.o"
                    MAIN_DEPENDENCY "${CMAKE_SOURCE_DIR}/match_mail.thorin"
                    BYPRODUCTS ${CMAKE_CURRENT_BINARY_DIR}/thorin_match.ll)
    set(THORIN_MATCH_FILE ${CMAKE_CURRENT_BINARY_DIR}/thorin_match.o)
else()
    add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/thorin_match_nondet.o
                    COMMAND $<TARGET_FILE:${THORIN_TARGET_NAMESPACE}thorin> "${CMAKE_SOURCE_DIR}/match_mail_nondet.thorin" --output-ll "${CMAKE_CURRENT_BINARY_DIR}/thorin_match_nondet.ll" --aggr-lam-spec
                    COMMAND ${CMAKE_CXX_COMPILER} -O3 "${CMAKE_CURRENT_BINARY_DIR}/thorin_match_nondet.ll" -c -o "${CMAKE_CURRENT_BINARY_DIR}/thorin_match_nondet.o"
                    MAIN_DEPENDENCY "${CMAKE_SOURCE_DIR}/match_mail_nondet.thorin"
                    BYPRODUCTS ${CMAKE_CURRENT_BINARY_DIR}/thorin_match_nondet.ll)
    set(THORIN_MATCH_FILE ${CMAKE_CURRENT_BINARY_DIR}/thorin_match_nondet.o)
endif()

add_executable(${PROJECT_NAME}
    benchmark_mail.cpp
    ctre_match_mail.cpp
    pcre2_match_mail.cpp
    std_match_mail.cpp
    manual_match_mail.cpp
    ${THORIN_MATCH_FILE}
)

target_include_directories(${PROJECT_NAME} PRIVATE compile-time-regular-expressions/include)

target_link_libraries(${PROJECT_NAME} PRIVATE
    pcre2-8
)

if(REGEX_DETERMINISTIC)
    target_compile_definitions(${PROJECT_NAME} PRIVATE
        -DREGEX_DETERMINISTIC
    )
endif()

add_executable(eval
    eval.cpp
    ctre_match_mail.cpp
    pcre2_match_mail.cpp
    std_match_mail.cpp
    manual_match_mail.cpp
    ${THORIN_MATCH_FILE}
)

target_include_directories(eval PRIVATE compile-time-regular-expressions/include)

target_link_libraries(eval PRIVATE
    pcre2-8
)

if(REGEX_DETERMINISTIC)
    target_compile_definitions(eval PRIVATE
        -DREGEX_DETERMINISTIC
    )
endif()

set_target_properties(eval PROPERTIES COMPILE_FLAGS "-fno-omit-frame-pointer -fsanitize=address")
set_target_properties(eval PROPERTIES LINK_FLAGS " -fno-omit-frame-pointer -fsanitize=address")
