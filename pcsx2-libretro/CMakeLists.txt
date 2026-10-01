# add PORTANDROID options
if(PORTANDROID)
	#set(USE_VULKAN OFF CACHE BOOL "Force vulkan off" FORCE)
	set(distribution_DIR ${CFG_LIBS_DIR})
	set(import_libs_DIR ${CFG_LIBS_DIR})
	set(main_entry_source ${CFG_ENTRY_DIR}/emu_${CFG_MODULE_NAME}_main.cpp)

	# add include directories
	include_directories(${import_libs_DIR}/classicboy/include
			${import_libs_DIR}/libBlocksRuntime/include
			${import_libs_DIR}/libcpufeatures/include
			${import_libs_DIR}/libemuCore/include
			${import_libs_DIR}/libemuRetro/include
			${import_libs_DIR}/libfex/include
			${import_libs_DIR}/libpng/include
			${import_libs_DIR}/libzip/include)

	# added imported libraries
	add_library(lib_emuCore STATIC IMPORTED)
	set_target_properties(lib_emuCore PROPERTIES IMPORTED_LOCATION
		${import_libs_DIR}/libemuCore/${CFG_BUILD_TYPE}/${ANDROID_ABI}/libemuCore.a)

	# added imported libraries
	add_library(lib_emuRetro STATIC IMPORTED)
	set_target_properties(lib_emuRetro PROPERTIES IMPORTED_LOCATION
		${import_libs_DIR}/libemuRetro/${CFG_BUILD_TYPE}/${ANDROID_ABI}/libemuRetro.a)

	add_library(pcsx2-libretro SHARED
		Main.cpp
		${main_entry_source}
	)

else()

	add_library(pcsx2-libretro SHARED)

	# Libretro cores are named <core>_libretro.so with no "lib" prefix.
	set_target_properties(pcsx2-libretro PROPERTIES
		OUTPUT_NAME "armsx2_libretro"
		PREFIX ""
	)

	if (PACKAGE_MODE)
		install(TARGETS pcsx2-libretro DESTINATION ${CMAKE_INSTALL_LIBDIR})
	else()
		install(TARGETS pcsx2-libretro DESTINATION ${CMAKE_SOURCE_DIR}/bin)
	endif()

	target_sources(pcsx2-libretro PRIVATE
		Main.cpp
	)

endif()

target_include_directories(pcsx2-libretro PRIVATE
	"${CMAKE_BINARY_DIR}/common/include"
	"${CMAKE_SOURCE_DIR}/pcsx2"
	"${CMAKE_CURRENT_SOURCE_DIR}"
)

if(PORTANDROID)
	target_link_libraries(pcsx2-libretro PRIVATE
			android
			log
			lib_emuCore
			lib_emuRetro
			PCSX2_FLAGS
			PCSX2
			libretro-headers
	)

	set_target_properties(pcsx2-libretro PROPERTIES OUTPUT_NAME "core_${CFG_MODULE_NAME}")
	set_target_properties(pcsx2-libretro
			PROPERTIES
			LIBRARY_OUTPUT_DIRECTORY
			"${distribution_DIR}/core_${CFG_MODULE_NAME}/${CFG_BUILD_TYPE}/${ANDROID_ABI}")
else()
	target_link_libraries(pcsx2-libretro PRIVATE
		PCSX2_FLAGS
		PCSX2
		libretro-headers
	)
endif()
# A shared core must be fully position-independent — the pcsx2-sdl ET_EXEC
# fixed-address trick (persisted-JIT VU cache) does not apply here.
set_target_properties(pcsx2-libretro PROPERTIES POSITION_INDEPENDENT_CODE ON)

# Only export the retro_* entry points; PCSX2's internal symbols must not
# leak into (or collide with) the frontend process.
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
	target_link_options(pcsx2-libretro PRIVATE
		"-Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/link.T")
	set_target_properties(pcsx2-libretro PROPERTIES
		LINK_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/link.T")
endif()

if(NOT PORTANDROID)
	# The libretro buildbot's ci-templates build the target named
	# <CORENAME>_libretro (see .gitlab-ci.yml).
	add_custom_target(armsx2_libretro)
	add_dependencies(armsx2_libretro pcsx2-libretro)
endif()