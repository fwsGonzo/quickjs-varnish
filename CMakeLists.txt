cmake_minimum_required (VERSION 3.0.2)
cmake_policy(SET CMP0069 NEW)
project (qjs_app C)

include(CheckIPOSupported)
check_ipo_supported()

option(NATIVE      "Enable native performance" OFF)
option(LTO         "Enable link-time optimizations" OFF)
option(GCSECTIONS  "Garbage collect empty sections" ON)
option(STRIP_ALL   "Remove debug info" ON)

set(CMAKE_C_FLAGS "-Wall -Wextra -std=gnu11 -O2 -g")

set(SOURCES
	src/main.c
	src/http.c
	src/embed_js.c
	static_site.c
)
set_source_files_properties(src/main.c OBJECT_DEPENDS
	${CMAKE_SOURCE_DIR}/src/my.js
)

add_executable(jsapp ${SOURCES})
set_target_properties(jsapp PROPERTIES C_STANDARD 11)
if (NATIVE)
	target_compile_options(jsapp PUBLIC -march=native -Ofast -fno-fast-math)
endif()

set (QJS_SOURCES
	quickjs/cutils.c
	quickjs/libregexp.c
	quickjs/quickjs.c
	quickjs/libbf.c
	quickjs/quickjs-libc.c
	quickjs/libunicode.c
	quickjs/tests/bjson.c
)
add_library(quickjs STATIC ${QJS_SOURCES})
set_target_properties(quickjs PROPERTIES C_STANDARD 11)
target_include_directories(quickjs PUBLIC ${CMAKE_SOURCE_DIR})
target_compile_definitions(quickjs PUBLIC
	_GNU_SOURCE=1
	CONFIG_BIGNUM=1
	CONFIG_VERSION="2021-03-27"
)
target_compile_options(quickjs PRIVATE "-Wno-unused-parameter" "-Wno-sign-compare")

target_link_libraries(jsapp -static quickjs pthread m)

if (LTO)
	set_property(TARGET quickjs PROPERTY INTERPROCEDURAL_OPTIMIZATION true)
	set_property(TARGET jsapp PROPERTY INTERPROCEDURAL_OPTIMIZATION true)
	set_source_files_properties("static_site.c" PROPERTIES COMPILE_FLAGS "-fno-lto")
	set_source_files_properties("src/embed_js.c" PROPERTIES COMPILE_FLAGS "-fno-lto")
endif()
if (GCSECTIONS)
	target_compile_options(quickjs PUBLIC -ffunction-sections -fdata-sections)
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-gc-sections")
endif()
if (STRIP_ALL)
	add_custom_command(TARGET jsapp POST_BUILD
		COMMAND ${CMAKE_STRIP} --strip-debug -R .note -R .comment -- ${CMAKE_CURRENT_BINARY_DIR}/jsapp
	)
endif()

function (target_keep_symbols NAME )
	foreach(SYMBOL ${ARGN})
		#message(STATUS "Symbol retained: ${SYMBOL}")
		target_link_libraries(${NAME} "-Wl,--undefined=${SYMBOL}")
	endforeach()
endfunction()

target_keep_symbols(jsapp
	"my_backend"
	"my_post_backend"
	"on_live_update"
	"on_resume_update"
)
