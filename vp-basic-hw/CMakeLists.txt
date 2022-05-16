file(GLOB_RECURSE HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/*.h)

add_library(platform-basic-hw
ethernet.cpp
display.cpp
${HEADERS})

target_include_directories(platform-basic-hw PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

add_executable(riscv-vp-hw
        main.cpp)

target_link_libraries(riscv-vp-hw rv32 platform-basic-hw platform-common gdb-mc ${Boost_LIBRARIES} ${SystemC_LIBRARIES} pthread)

INSTALL(TARGETS riscv-vp RUNTIME DESTINATION bin)
