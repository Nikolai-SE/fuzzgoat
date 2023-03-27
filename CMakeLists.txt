project(cmake_fuzz)
add_executable(cfuzz main.c fuzzgoat.c)
target_link_libraries(cfuzz m)
