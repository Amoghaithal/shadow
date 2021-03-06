## tgen: an all-purpose traffic generation plug-in for Shadow
project(shadow-plugin-tgen)

## skip checks if built in shadow source dir
if(NOT TARGET LLVMHoistGlobals)
    message(FATAL_ERROR "shadow-plugin-tgen must currently be built in shadow source tree")

    ## fill in LLVM variables since we did not inherit from shadow cmake file
    include(LLVMTools)
    add_custom_target(LLVMHoistGlobals)
    set(LLVMHoistGlobalsPATH "${SHADOW_ROOT}/lib/LLVMHoistGlobals.so")

    ## make sure our LLVM plugin exists
    if(EXISTS "${LLVMHoistGlobalsPATH}")
        message(STATUS "LLVMHoistGlobalsPATH = ${LLVMHoistGlobalsPATH}")
    else()
        message(FATAL_ERROR "LLVMHoistGlobalsPATH path does not exist: '${LLVMHoistGlobalsPATH}'")
    endif()
endif()

## plug-ins need to disable fortification to ensure syscalls are intercepted
add_definitions(-D_GNU_SOURCE)
add_cflags("-fno-inline -fno-strict-aliasing -U_FORTIFY_SOURCE")

## CFLAGS status update
message(STATUS "CMAKE_C_FLAGS = ${CMAKE_C_FLAGS}")

set(tgen_sources
    shd-tgen-action.c
    shd-tgen-driver.c
    shd-tgen-graph.c
    shd-tgen-io.c
    shd-tgen-peer.c
    shd-tgen-pool.c
    shd-tgen-server.c
    shd-tgen-timer.c
    shd-tgen-transfer.c
    shd-tgen-transport.c
)

find_package(RT REQUIRED)
find_package(M REQUIRED)
find_package(IGRAPH REQUIRED)
find_package(GLIB REQUIRED)

include_directories(AFTER ${RT_INCLUDES} ${M_INCLUDES} ${GLIB_INCLUDES})

## library to manage global locking
add_library(shadow-tgen-global-lock SHARED shd-tgen-global-lock.c)
target_link_libraries(shadow-tgen-global-lock ${GLIB_LIBRARIES})
install(TARGETS shadow-tgen-global-lock DESTINATION lib)

## executable that can run outside of shadow
add_executable(tgen shd-tgen-main.c ${tgen_sources})
add_dependencies(tgen shadow-tgen-global-lock)
target_link_libraries(tgen shadow-tgen-global-lock ${RT_LIBRARIES} ${M_LIBRARIES} ${IGRAPH_LIBRARIES} ${GLIB_LIBRARIES})
install(TARGETS tgen DESTINATION bin)
set_target_properties(tgen PROPERTIES INSTALL_RPATH ${CMAKE_INSTALL_PREFIX}/lib INSTALL_RPATH_USE_LINK_PATH TRUE LINK_FLAGS "-Wl,--no-as-needed")

## build bitcode
add_bitcode(shadow-plugin-tgen-bitcode shd-tgen-plugin.c ${tgen_sources})

## create and install the shared library that plugs into shadow
add_plugin(shadow-plugin-tgen shadow-plugin-tgen-bitcode)
add_dependencies(shadow-plugin-tgen shadow-tgen-global-lock)
target_link_libraries(shadow-plugin-tgen shadow-tgen-global-lock ${RT_LIBRARIES} ${IGRAPH_LIBRARIES} ${GLIB_LIBRARIES})
install(TARGETS shadow-plugin-tgen DESTINATION plugins)
