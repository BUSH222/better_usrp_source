cmake_minimum_required(VERSION 3.13)
project(better_usrp_source)

set(SDRPP_CORE_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/lib/SDRPlusPlus" CACHE PATH "path to sdrpp core")
set(SDRPP_CORE_LIB_DIR "" CACHE PATH "path to the built sdrpp_core import library directory")
set(SDRPP_CORE_DLL_DIR "" CACHE PATH "path to the built sdrpp_core runtime library directory")

file(GLOB SRC "src/*.cpp")

# When building out-of-tree on Windows CI, bind the module link against the
# exact sdrpp_core import library produced by the core build.
if (MSVC AND SDRPP_CORE_LIB_DIR AND NOT TARGET sdrpp_core)
    set(_SDRPP_CORE_IMPLIB "${SDRPP_CORE_LIB_DIR}/sdrpp_core.lib")
    if (NOT EXISTS "${_SDRPP_CORE_IMPLIB}")
        message(FATAL_ERROR "sdrpp_core.lib not found at ${_SDRPP_CORE_IMPLIB}")
    endif ()

    add_library(sdrpp_core SHARED IMPORTED GLOBAL)
    set_target_properties(sdrpp_core PROPERTIES
        IMPORTED_IMPLIB "${_SDRPP_CORE_IMPLIB}"
        IMPORTED_IMPLIB_RELEASE "${_SDRPP_CORE_IMPLIB}"
    )

    if (SDRPP_CORE_DLL_DIR)
        set(_SDRPP_CORE_DLL "${SDRPP_CORE_DLL_DIR}/sdrpp_core.dll")
        if (EXISTS "${_SDRPP_CORE_DLL}")
            set_target_properties(sdrpp_core PROPERTIES
                IMPORTED_LOCATION "${_SDRPP_CORE_DLL}"
                IMPORTED_LOCATION_RELEASE "${_SDRPP_CORE_DLL}"
            )
        endif ()
    endif ()
endif ()

include("${SDRPP_CORE_ROOT}/sdrpp_module.cmake")

# Explicitly append the core and imgui paths here
target_include_directories(${PROJECT_NAME} PRIVATE 
    "src/"
    "${SDRPP_CORE_ROOT}/core/src/"
    "${SDRPP_CORE_ROOT}/core/src/imgui/"
)

if (MSVC)
    # UHD headers include Boost headers on Windows.
    find_path(_BOOST_INCLUDE_DIR
        NAMES boost/config.hpp
        HINTS
            "$ENV{BOOST_ROOT}"
            "$ENV{BOOST_ROOT}/include"
            "$ENV{BOOST_ROOT_1_83_0}"
            "$ENV{BOOST_ROOT_1_83_0}/include"
            "C:/vcpkg/installed/x64-windows/include"
            "C:/Program Files/PothosSDR/include"
    )

    if (NOT _BOOST_INCLUDE_DIR)
        message(FATAL_ERROR
            "Boost headers not found. Install boost-core:x64-windows in vcpkg or set BOOST_ROOT so boost/config.hpp can be found."
        )
    endif ()

    target_include_directories(${PROJECT_NAME} PRIVATE
        "${_BOOST_INCLUDE_DIR}"
    )

    # On Windows CI, SDR++ core headers pull GNU Radio VOLK headers from PothosSDR.
    target_include_directories(${PROJECT_NAME} PRIVATE
        "C:/Program Files/PothosSDR/include"
        "C:/vcpkg/installed/x64-windows/include"
    )

    # Note: Includes both lib/ and bin/ just in case for linking uhd properly depending on its location.
    target_link_directories(${PROJECT_NAME} PRIVATE
        "C:/Program Files/PothosSDR/lib"
        "C:/Program Files/PothosSDR/bin"
    )

    if (SDRPP_CORE_LIB_DIR)
        target_link_directories(${PROJECT_NAME} PRIVATE
            "${SDRPP_CORE_LIB_DIR}"
        )
    endif ()
    
    target_link_libraries(${PROJECT_NAME} PRIVATE volk uhd)
else ()
    find_package(PkgConfig)
    pkg_check_modules(LIBUHD REQUIRED uhd)

    target_include_directories(${PROJECT_NAME} PRIVATE ${LIBUHD_INCLUDE_DIRS})
    target_link_directories(${PROJECT_NAME} PRIVATE ${LIBUHD_LIBRARY_DIRS})
    
    target_link_libraries(${PROJECT_NAME} PRIVATE volk ${LIBUHD_LIBRARIES})
endif ()