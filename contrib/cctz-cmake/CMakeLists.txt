option (USE_INTERNAL_CCTZ "Use internal cctz library" ${NOT_UNBUNDLED})

if (USE_INTERNAL_CCTZ)
    SET(LIBRARY_DIR ${ClickHouse_SOURCE_DIR}/contrib/cctz)

    SET (SRCS
        ${LIBRARY_DIR}/src/civil_time_detail.cc
        ${LIBRARY_DIR}/src/time_zone_fixed.cc
        ${LIBRARY_DIR}/src/time_zone_format.cc
        ${LIBRARY_DIR}/src/time_zone_if.cc
        ${LIBRARY_DIR}/src/time_zone_impl.cc
        ${LIBRARY_DIR}/src/time_zone_info.cc
        ${LIBRARY_DIR}/src/time_zone_libc.cc
        ${LIBRARY_DIR}/src/time_zone_lookup.cc
        ${LIBRARY_DIR}/src/time_zone_posix.cc
        ${LIBRARY_DIR}/src/zone_info_source.cc
    )

    add_library (cctz ${SRCS})
    target_include_directories (cctz PUBLIC ${LIBRARY_DIR}/include)

    if (OS_FREEBSD)
        # yes, need linux, because bsd check inside linux in time_zone_libc.cc:24
        target_compile_definitions (cctz PRIVATE __USE_BSD linux _XOPEN_SOURCE=600)
    endif ()

    # Build a libray with embedded tzdata

    if (OS_LINUX)
        # get the list of timezones from tzdata shipped with cctz
        set(TZDIR ${LIBRARY_DIR}/testdata/zoneinfo)
        file(STRINGS ${LIBRARY_DIR}/testdata/version TZDATA_VERSION)
        set_property(GLOBAL PROPERTY TZDATA_VERSION_PROP "${TZDATA_VERSION}")
        message(STATUS "Packaging with tzdata version: ${TZDATA_VERSION}")

        set(TZ_OBJS)

        # each file/symlink in that dir (except of tab and localtime) store the info about timezone
        execute_process(COMMAND bash -c "cd ${TZDIR} && find * -type f,l -and ! -name '*.tab' -and ! -name 'localtime' | sort | paste -sd ';'" OUTPUT_STRIP_TRAILING_WHITESPACE OUTPUT_VARIABLE TIMEZONES )

        foreach(TIMEZONE ${TIMEZONES})
            string(REPLACE "/" "_" TIMEZONE_ID ${TIMEZONE})
            string(REPLACE "+" "_PLUS_" TIMEZONE_ID ${TIMEZONE_ID})
            set(TZ_OBJ ${TIMEZONE_ID}.o)
            set(TZ_OBJS ${TZ_OBJS} ${TZ_OBJ})

            # https://stackoverflow.com/questions/14776463/compile-and-add-an-object-file-from-a-binary-with-cmake
            add_custom_command(OUTPUT ${TZ_OBJ}
                COMMAND cp ${TZDIR}/${TIMEZONE} ${CMAKE_CURRENT_BINARY_DIR}/${TIMEZONE_ID}
                COMMAND cd ${CMAKE_CURRENT_BINARY_DIR} && ${OBJCOPY_PATH} -I binary ${OBJCOPY_ARCH_OPTIONS}
                            --rename-section .data=.rodata,alloc,load,readonly,data,contents ${TIMEZONE_ID} ${TZ_OBJ}
                COMMAND rm ${CMAKE_CURRENT_BINARY_DIR}/${TIMEZONE_ID})

            set_source_files_properties(${TZ_OBJ} PROPERTIES EXTERNAL_OBJECT true GENERATED true)
        endforeach(TIMEZONE)

        add_library(tzdata STATIC ${TZ_OBJS})
        set_target_properties(tzdata PROPERTIES LINKER_LANGUAGE C)
        # whole-archive prevents symbols from being discarded for unknown reason
        # CMake can shuffle each of target_link_libraries arguments with other
        # libraries in linker command. To avoid this we hardcode whole-archive
        # library into single string.
        add_dependencies(cctz tzdata)
        target_link_libraries(cctz INTERFACE "-Wl,${WHOLE_ARCHIVE} $<TARGET_FILE:tzdata> -Wl,${NO_WHOLE_ARCHIVE}")
    endif ()

else ()
    find_library (LIBRARY_CCTZ cctz)
    find_path (INCLUDE_CCTZ NAMES cctz/civil_time.h)

    add_library (cctz UNKNOWN IMPORTED)
    set_property (TARGET cctz PROPERTY IMPORTED_LOCATION ${LIBRARY_CCTZ})
    set_property (TARGET cctz PROPERTY INTERFACE_INCLUDE_DIRECTORIES ${INCLUDE_CCTZ})

    set(CMAKE_REQUIRED_LIBRARIES cctz)
    check_cxx_source_compiles(
        "
        #include <cctz/civil_time.h>
        int main() {
            cctz::civil_day date;
        }
        "
        EXTERNAL_CCTZ_WORKS
    )

    if (NOT EXTERNAL_CCTZ_WORKS)
        message (FATAL_ERROR "cctz is unusable: ${LIBRARY_CCTZ} ${INCLUDE_CCTZ}")
    endif ()
endif ()

message (STATUS "Using cctz")
