     1|include(CheckCXXCompilerFlag)
     2|include("../cmake/common.cmake")
     3|
     4|add_compile_definitions(GGML_SCHED_MAX_COPIES=${GGML_SCHED_MAX_COPIES})
     5|
     6|# enable libstdc++ assertions for debug builds
     7|if (CMAKE_SYSTEM_NAME MATCHES "Linux")
     8|    add_compile_definitions($<$<CONFIG:Debug>:_GLIBCXX_ASSERTIONS>)
     9|endif()
    10|
    11|if (NOT MSVC)
    12|    if (GGML_SANITIZE_THREAD)
    13|        add_compile_options(-fsanitize=thread)
    14|        link_libraries     (-fsanitize=thread)
    15|    endif()
    16|
    17|    if (GGML_SANITIZE_ADDRESS)
    18|        add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
    19|        link_libraries     (-fsanitize=address)
    20|    endif()
    21|
    22|    if (GGML_SANITIZE_UNDEFINED)
    23|        add_compile_options(-fsanitize=undefined)
    24|        link_libraries     (-fsanitize=undefined)
    25|    endif()
    26|endif()
    27|
    28|if (GGML_FATAL_WARNINGS)
    29|    if (CMAKE_CXX_COMPILER_ID MATCHES "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    30|        list(APPEND C_FLAGS   -Werror)
    31|        list(APPEND CXX_FLAGS -Werror)
    32|    elseif (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    33|        add_compile_options(/WX)
    34|    endif()
    35|endif()
    36|
    37|if (GGML_ALL_WARNINGS)
    38|    if (NOT MSVC)
    39|        list(APPEND WARNING_FLAGS -Wall -Wextra -Wpedantic -Wcast-qual -Wno-unused-function)
    40|        list(APPEND C_FLAGS       -Wshadow -Wstrict-prototypes -Wpointer-arith -Wmissing-prototypes
    41|                                  -Werror=implicit-int -Werror=implicit-function-declaration)
    42|        list(APPEND CXX_FLAGS     -Wmissing-declarations -Wmissing-noreturn)
    43|
    44|        list(APPEND C_FLAGS   ${WARNING_FLAGS})
    45|        list(APPEND CXX_FLAGS ${WARNING_FLAGS})
    46|
    47|        ggml_get_flags(${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION})
    48|
    49|        add_compile_options("$<$<COMPILE_LANGUAGE:C>:${C_FLAGS};${GF_C_FLAGS}>"
    50|                            "$<$<COMPILE_LANGUAGE:CXX>:${CXX_FLAGS};${GF_CXX_FLAGS}>")
    51|    else()
    52|        # todo : msvc
    53|        set(C_FLAGS   "")
    54|        set(CXX_FLAGS "")
    55|    endif()
    56|endif()
    57|
    58|if (GGML_LTO)
    59|    include(CheckIPOSupported)
    60|    check_ipo_supported(RESULT result OUTPUT output)
    61|    if (result)
    62|        set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
    63|    else()
    64|        message(WARNING "IPO is not supported: ${output}")
    65|    endif()
    66|endif()
    67|
    68|if (GGML_CCACHE AND NOT CMAKE_C_COMPILER_LAUNCHER AND NOT CMAKE_CXX_COMPILER_LAUNCHER)
    69|    find_program(GGML_CCACHE_FOUND ccache)
    70|    find_program(GGML_SCCACHE_FOUND sccache)
    71|
    72|    if (GGML_CCACHE_FOUND OR GGML_SCCACHE_FOUND)
    73|        if(GGML_CCACHE_FOUND)
    74|            set(GGML_CCACHE_VARIANT ccache)
    75|        else()
    76|            set(GGML_CCACHE_VARIANT sccache)
    77|        endif()
    78|        # TODO: should not be set globally
    79|        if (GGML_SYCL AND GGML_CCACHE_FOUND AND WIN32)
    80|            set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "ccache compiler_type=icl")
    81|        else ()
    82|            set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${GGML_CCACHE_VARIANT}")
    83|        endif ()
    84|        set(ENV{CCACHE_SLOPPINESS} time_macros)
    85|        message(STATUS "${GGML_CCACHE_VARIANT} found, compilation results will be cached. Disable with GGML_CCACHE=OFF.")
    86|    else()
    87|        message(STATUS "Warning: ccache not found - consider installing it for faster compilation or disable this warning with GGML_CCACHE=OFF")
    88|    endif ()
    89|endif()
    90|
    91|# this version of Apple ld64 is buggy
    92|execute_process(
    93|    COMMAND ${CMAKE_C_COMPILER} ${CMAKE_EXE_LINKER_FLAGS} -Wl,-v
    94|    ERROR_VARIABLE output
    95|    OUTPUT_QUIET
    96|)
    97|
    98|if (output MATCHES "dyld-1015\.7")
    99|    add_compile_definitions(HAVE_BUGGY_APPLE_LINKER)
   100|endif()
   101|
   102|# architecture specific
   103|# TODO: probably these flags need to be tweaked on some architectures
   104|#       feel free to update the Makefile for your architecture and send a pull request or issue
   105|message(STATUS "CMAKE_SYSTEM_PROCESSOR: ${CMAKE_SYSTEM_PROCESSOR}")
   106|if (MSVC)
   107|    string(TOLOWER "${CMAKE_GENERATOR_PLATFORM}" CMAKE_GENERATOR_PLATFORM_LWR)
   108|    message(STATUS "CMAKE_GENERATOR_PLATFORM: ${CMAKE_GENERATOR_PLATFORM}")
   109|else ()
   110|    set(CMAKE_GENERATOR_PLATFORM_LWR "")
   111|endif ()
   112|ggml_get_system_arch()
   113|message(STATUS "GGML_SYSTEM_ARCH: ${GGML_SYSTEM_ARCH}")
   114|
   115|if (NOT MSVC)
   116|    if (GGML_STATIC)
   117|        if (UNIX AND NOT APPLE)
   118|            set(CMAKE_FIND_LIBRARY_SUFFIXES ".a;.so")
   119|        endif()
   120|        add_link_options(-static)
   121|        if (MINGW)
   122|            add_link_options(-static-libgcc -static-libstdc++)
   123|        endif()
   124|    endif()
   125|    if (GGML_GPROF)
   126|        add_compile_options(-pg)
   127|    endif()
   128|endif()
   129|
   130|#
   131|# POSIX conformance
   132|#
   133|
   134|# clock_gettime came in POSIX.1b (1993)
   135|# CLOCK_MONOTONIC came in POSIX.1-2001 / SUSv3 as optional
   136|# posix_memalign came in POSIX.1-2001 / SUSv3
   137|# M_PI is an XSI extension since POSIX.1-2001 / SUSv3, came in XPG1 (1985)
   138|
   139|# Somehow in OpenBSD whenever POSIX conformance is specified
   140|# some string functions rely on locale_t availability,
   141|# which was introduced in POSIX.1-2008, forcing us to go higher
   142|if (CMAKE_SYSTEM_NAME MATCHES "OpenBSD")
   143|    add_compile_definitions(_XOPEN_SOURCE=700)
   144|elseif (CMAKE_SYSTEM_NAME MATCHES "AIX")
   145|    # Don't define _XOPEN_SOURCE.  We need _ALL_SOURCE, which is the default,
   146|    # in order to define _SC_PHYS_PAGES.
   147|else()
   148|    add_compile_definitions(_XOPEN_SOURCE=600)
   149|endif()
   150|
   151|# Data types, macros and functions related to controlling CPU affinity and
   152|# some memory allocation are available on Linux through GNU extensions in libc
   153|if (CMAKE_SYSTEM_NAME MATCHES "Linux" OR CMAKE_SYSTEM_NAME MATCHES "Android")
   154|    add_compile_definitions(_GNU_SOURCE)
   155|endif()
   156|
   157|# RLIMIT_MEMLOCK came in BSD, is not specified in POSIX.1,
   158|# and on macOS its availability depends on enabling Darwin extensions
   159|# similarly on DragonFly, enabling BSD extensions is necessary
   160|if (
   161|    CMAKE_SYSTEM_NAME MATCHES "Darwin" OR
   162|    CMAKE_SYSTEM_NAME MATCHES "iOS"    OR
   163|    CMAKE_SYSTEM_NAME MATCHES "tvOS"   OR
   164|    CMAKE_SYSTEM_NAME MATCHES "DragonFly"
   165|)
   166|    add_compile_definitions(_DARWIN_C_SOURCE)
   167|endif()
   168|
   169|# alloca is a non-standard interface that is not visible on BSDs when
   170|# POSIX conformance is specified, but not all of them provide a clean way
   171|# to enable it in such cases
   172|if (CMAKE_SYSTEM_NAME MATCHES "FreeBSD")
   173|    add_compile_definitions(__BSD_VISIBLE)
   174|endif()
   175|if (CMAKE_SYSTEM_NAME MATCHES "NetBSD")
   176|    add_compile_definitions(_NETBSD_SOURCE)
   177|endif()
   178|if (CMAKE_SYSTEM_NAME MATCHES "OpenBSD")
   179|    add_compile_definitions(_BSD_SOURCE)
   180|endif()
   181|
   182|if (WIN32)
   183|    add_compile_definitions(_CRT_SECURE_NO_WARNINGS)
   184|endif()
   185|
   186|# ggml
   187|
   188|if (GGML_BACKEND_DL AND NOT BUILD_SHARED_LIBS)
   189|    message(FATAL_ERROR "GGML_BACKEND_DL requires BUILD_SHARED_LIBS")
   190|endif()
   191|
   192|add_library(ggml-base
   193|            ../include/ggml.h
   194|            ../include/ggml-alloc.h
   195|            ../include/ggml-backend.h
   196|            ../include/ggml-cpp.h
   197|            ../include/ggml-opt.h
   198|            ../include/gguf.h
   199|            ggml.c
   200|            ggml.cpp
   201|            ggml-alloc.c
   202|            ggml-backend.cpp
   203|            ggml-backend-meta.cpp
   204|            ggml-opt.cpp
   205|            ggml-threading.cpp
   206|            ggml-threading.h
   207|            ggml-quants.c
    ggml-planar-quant.c
    ggml-iso-quant.c
   208|            ggml-quants.h
   209|            gguf.cpp)
   210|
   211|set_target_properties(ggml-base PROPERTIES
   212|    VERSION ${GGML_VERSION}
   213|    SOVERSION ${GGML_VERSION_MAJOR}
   214|)
   215|
   216|target_include_directories(ggml-base PRIVATE .)
   217|if (GGML_BACKEND_DL)
   218|    target_compile_definitions(ggml-base PUBLIC GGML_BACKEND_DL)
   219|endif()
   220|
   221|if (GGML_SCHED_NO_REALLOC)
   222|    target_compile_definitions(ggml-base PUBLIC GGML_SCHED_NO_REALLOC)
   223|endif()
   224|
   225|if (GGML_OPENMP)
   226|    find_package(OpenMP)
   227|    if (OpenMP_FOUND)
   228|        set(GGML_OPENMP_ENABLED "ON" CACHE INTERNAL "")
   229|    else()
   230|        set(GGML_OPENMP_ENABLED "OFF" CACHE INTERNAL "")
   231|        message(WARNING "OpenMP not found")
   232|    endif()
   233|else()
   234|    set(GGML_OPENMP_ENABLED "OFF" CACHE INTERNAL "")
   235|endif()
   236|
   237|if (GGML_OPENMP_ENABLED)
   238|    target_compile_definitions(ggml-base PRIVATE GGML_USE_OPENMP)
   239|    target_link_libraries(ggml-base PRIVATE OpenMP::OpenMP_C OpenMP::OpenMP_CXX)
   240|endif()
   241|
   242|add_library(ggml
   243|            ggml-backend-dl.cpp
   244|            ggml-backend-reg.cpp)
   245|add_library(ggml::ggml ALIAS ggml)
   246|
   247|set_target_properties(ggml PROPERTIES
   248|    VERSION ${GGML_VERSION}
   249|    SOVERSION ${GGML_VERSION_MAJOR}
   250|)
   251|
   252|if (GGML_BACKEND_DIR)
   253|    if (NOT GGML_BACKEND_DL)
   254|        message(FATAL_ERROR "GGML_BACKEND_DIR requires GGML_BACKEND_DL")
   255|    endif()
   256|    target_compile_definitions(ggml PUBLIC GGML_BACKEND_DIR="${GGML_BACKEND_DIR}")
   257|endif()
   258|
   259|target_link_libraries(ggml PUBLIC ggml-base)
   260|
   261|if (CMAKE_SYSTEM_NAME MATCHES "Linux")
   262|    target_link_libraries(ggml PRIVATE dl)
   263|endif()
   264|
   265|function(ggml_add_backend_library backend)
   266|    if (GGML_BACKEND_DL)
   267|        add_library(${backend} MODULE ${ARGN})
   268|        # write the shared library to the output directory
   269|        set_target_properties(${backend} PROPERTIES LIBRARY_OUTPUT_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
   270|        target_compile_definitions(${backend} PRIVATE GGML_BACKEND_DL)
   271|        add_dependencies(ggml ${backend})
   272|        if (GGML_BACKEND_DIR)
   273|            install(TARGETS ${backend} LIBRARY DESTINATION ${GGML_BACKEND_DIR})
   274|        else()
   275|            install(TARGETS ${backend} LIBRARY DESTINATION ${CMAKE_INSTALL_BINDIR})
   276|        endif()
   277|    else()
   278|        add_library(${backend} ${ARGN})
   279|        target_link_libraries(ggml PUBLIC ${backend})
   280|        install(TARGETS ${backend} LIBRARY)
   281|    endif()
   282|
   283|    target_link_libraries(${backend} PRIVATE ggml-base)
   284|    target_include_directories(${backend} PRIVATE ..)
   285|
   286|    if (${BUILD_SHARED_LIBS})
   287|        target_compile_definitions(${backend} PRIVATE GGML_BACKEND_BUILD)
   288|        target_compile_definitions(${backend} PUBLIC  GGML_BACKEND_SHARED)
   289|    endif()
   290|
   291|    # Set versioning properties for all backend libraries
   292|    # Building a MODULE library with a version is not supported on macOS (https://gitlab.kitware.com/cmake/cmake/-/issues/20782)
   293|    if (NOT (APPLE AND GGML_BACKEND_DL))
   294|        set_target_properties(${backend} PROPERTIES
   295|            VERSION ${GGML_VERSION}
   296|            SOVERSION ${GGML_VERSION_MAJOR}
   297|        )
   298|    endif()
   299|
   300|    if(NOT GGML_AVAILABLE_BACKENDS)
   301|        set(GGML_AVAILABLE_BACKENDS "${backend}"
   302|            CACHE INTERNAL "List of backends for cmake package")
   303|    else()
   304|        list(FIND GGML_AVAILABLE_BACKENDS "${backend}" has_backend)
   305|        if(has_backend EQUAL -1)
   306|            set(GGML_AVAILABLE_BACKENDS "${GGML_AVAILABLE_BACKENDS};${backend}"
   307|                CACHE INTERNAL "List of backends for cmake package")
   308|        endif()
   309|    endif()
   310|endfunction()
   311|
   312|function(ggml_add_backend backend)
   313|    string(TOUPPER "GGML_${backend}" backend_id)
   314|    if (${backend_id})
   315|        string(TOLOWER "ggml-${backend}" backend_target)
   316|        add_subdirectory(${backend_target})
   317|        message(STATUS "Including ${backend} backend")
   318|        if (NOT GGML_BACKEND_DL)
   319|            string(TOUPPER "GGML_USE_${backend}" backend_use)
   320|            target_compile_definitions(ggml PUBLIC ${backend_use})
   321|        endif()
   322|    endif()
   323|endfunction()
   324|
   325|function(ggml_add_cpu_backend_variant tag_name)
   326|    set(GGML_CPU_TAG_NAME ${tag_name})
   327|    # other: OPENMP LLAMAFILE CPU_HBM
   328|    if (GGML_SYSTEM_ARCH STREQUAL "x86")
   329|        foreach (feat NATIVE
   330|                      SSE42
   331|                      AVX AVX2 BMI2 AVX_VNNI FMA F16C
   332|                      AVX512 AVX512_VBMI AVX512_VNNI AVX512_BF16
   333|                      AMX_TILE AMX_INT8 AMX_BF16)
   334|            set(GGML_${feat} OFF)
   335|        endforeach()
   336|
   337|        foreach (feat ${ARGN})
   338|            set(GGML_${feat} ON)
   339|        endforeach()
   340|    elseif (GGML_SYSTEM_ARCH STREQUAL "ARM")
   341|        foreach (feat ${ARGN})
   342|            set(GGML_INTERNAL_${feat} ON)
   343|        endforeach()
   344|    elseif (GGML_SYSTEM_ARCH STREQUAL "PowerPC")
   345|        foreach (feat ${ARGN})
   346|            set(GGML_INTERNAL_${feat} ON)
   347|        endforeach()
   348|    elseif (GGML_SYSTEM_ARCH STREQUAL "s390x")
   349|        foreach (feat VXE2 NNPA)
   350|            set(GGML_INTERNAL_${feat} OFF)
   351|        endforeach()
   352|
   353|        foreach (feat ${ARGN})
   354|            set(GGML_INTERNAL_${feat} ON)
   355|        endforeach()
   356|    elseif (GGML_SYSTEM_ARCH STREQUAL "riscv64")
   357|        foreach (feat RVV)
   358|            set(GGML_INTERNAL_${feat} OFF)
   359|        endforeach()
   360|
   361|        foreach (feat ${ARGN})
   362|            set(GGML_INTERNAL_${feat} ON)
   363|        endforeach()
   364|    endif()
   365|
   366|    ggml_add_cpu_backend_variant_impl(${tag_name})
   367|endfunction()
   368|
   369|ggml_add_backend(CPU)
   370|
   371|if (GGML_CPU_ALL_VARIANTS)
   372|    if (NOT GGML_BACKEND_DL)
   373|        message(FATAL_ERROR "GGML_CPU_ALL_VARIANTS requires GGML_BACKEND_DL")
   374|    elseif (GGML_CPU_ARM_ARCH)
   375|        message(FATAL_ERROR "Cannot use both GGML_CPU_ARM_ARCH and GGML_CPU_ALL_VARIANTS")
   376|    endif()
   377|    if (GGML_SYSTEM_ARCH STREQUAL "x86")
   378|        ggml_add_cpu_backend_variant(x64)
   379|        ggml_add_cpu_backend_variant(sse42              SSE42)
   380|        ggml_add_cpu_backend_variant(sandybridge        SSE42 AVX)
   381|        if (NOT MSVC)
   382|            # __FMA__ and __F16C__ are not defined in MSVC, however they are implied with AVX2/AVX512
   383|            ggml_add_cpu_backend_variant(ivybridge      SSE42 AVX F16C)
   384|            ggml_add_cpu_backend_variant(piledriver     SSE42 AVX F16C FMA)
   385|        endif()
   386|        ggml_add_cpu_backend_variant(haswell            SSE42 AVX F16C FMA AVX2 BMI2)
   387|        ggml_add_cpu_backend_variant(skylakex           SSE42 AVX F16C FMA AVX2 BMI2 AVX512)
   388|        ggml_add_cpu_backend_variant(cannonlake         SSE42 AVX F16C FMA AVX2 BMI2 AVX512 AVX512_VBMI)
   389|        ggml_add_cpu_backend_variant(cascadelake        SSE42 AVX F16C FMA AVX2 BMI2 AVX512 AVX512_VNNI)
   390|        ggml_add_cpu_backend_variant(icelake            SSE42 AVX F16C FMA AVX2 BMI2 AVX512 AVX512_VBMI AVX512_VNNI)
   391|        if (NOT MSVC)
   392|            # MSVC 2022 doesn't support BF16 intrinsics without `/arch:AVX10.1` ?!
   393|            # https://learn.microsoft.com/en-us/cpp/intrinsics/x64-amd64-intrinsics-list?view=msvc-170
   394|            # https://learn.microsoft.com/en-us/cpp/build/reference/arch-x64?view=msvc-170
   395|            ggml_add_cpu_backend_variant(cooperlake     SSE42 AVX F16C FMA AVX2 BMI2 AVX512 AVX512_VNNI AVX512_BF16)
   396|            ggml_add_cpu_backend_variant(zen4           SSE42 AVX F16C FMA AVX2 BMI2 AVX512 AVX512_VBMI AVX512_VNNI AVX512_BF16)
   397|        endif()
   398|        ggml_add_cpu_backend_variant(alderlake          SSE42 AVX F16C FMA AVX2 BMI2 AVX_VNNI)
   399|        if (NOT MSVC)
   400|            # MSVC doesn't support AMX
   401|            ggml_add_cpu_backend_variant(sapphirerapids SSE42 AVX F16C FMA AVX2 BMI2 AVX512 AVX512_VBMI AVX512_VNNI AVX512_BF16 AMX_TILE AMX_INT8)
   402|        endif()
   403|    elseif(GGML_SYSTEM_ARCH STREQUAL "ARM")
   404|        if (CMAKE_SYSTEM_NAME MATCHES "Linux")
   405|            # Many of these features are optional so we build versions with popular
   406|            # combinations and name the backends based on the version they were
   407|            # first released with
   408|            ggml_add_cpu_backend_variant(armv8.0_1)
   409|            ggml_add_cpu_backend_variant(armv8.2_1    DOTPROD)
   410|            ggml_add_cpu_backend_variant(armv8.2_2    DOTPROD FP16_VECTOR_ARITHMETIC)
   411|            ggml_add_cpu_backend_variant(armv8.2_3    DOTPROD FP16_VECTOR_ARITHMETIC SVE)
   412|            ggml_add_cpu_backend_variant(armv8.6_1    DOTPROD FP16_VECTOR_ARITHMETIC SVE MATMUL_INT8)
   413|            ggml_add_cpu_backend_variant(armv8.6_2    DOTPROD FP16_VECTOR_ARITHMETIC SVE MATMUL_INT8 SVE2)
   414|            ggml_add_cpu_backend_variant(armv9.2_1    DOTPROD FP16_VECTOR_ARITHMETIC SVE MATMUL_INT8 SME)
   415|            ggml_add_cpu_backend_variant(armv9.2_2    DOTPROD FP16_VECTOR_ARITHMETIC SVE MATMUL_INT8 SVE2 SME)
   416|        elseif (CMAKE_SYSTEM_NAME MATCHES "Android")
   417|            # Android-specific backends with SoC-compatible feature sets
   418|            ggml_add_cpu_backend_variant(android_armv8.0_1)
   419|            ggml_add_cpu_backend_variant(android_armv8.2_1    DOTPROD)
   420|            ggml_add_cpu_backend_variant(android_armv8.2_2    DOTPROD FP16_VECTOR_ARITHMETIC)
   421|            ggml_add_cpu_backend_variant(android_armv8.6_1    DOTPROD FP16_VECTOR_ARITHMETIC MATMUL_INT8)
   422|            ggml_add_cpu_backend_variant(android_armv9.0_1    DOTPROD MATMUL_INT8 FP16_VECTOR_ARITHMETIC SVE2)
   423|            ggml_add_cpu_backend_variant(android_armv9.2_1    DOTPROD MATMUL_INT8 FP16_VECTOR_ARITHMETIC SVE SME)
   424|            ggml_add_cpu_backend_variant(android_armv9.2_2    DOTPROD MATMUL_INT8 FP16_VECTOR_ARITHMETIC SVE SVE2 SME)
   425|        elseif (APPLE)
   426|            ggml_add_cpu_backend_variant(apple_m1             DOTPROD)
   427|            ggml_add_cpu_backend_variant(apple_m2_m3          DOTPROD MATMUL_INT8)
   428|            ggml_add_cpu_backend_variant(apple_m4             DOTPROD MATMUL_INT8 NOSVE SME)
   429|        else()
   430|            message(FATAL_ERROR "Unsupported ARM target OS: ${CMAKE_SYSTEM_NAME}")
   431|        endif()
   432|    elseif (GGML_SYSTEM_ARCH STREQUAL "PowerPC")
   433|        if (CMAKE_SYSTEM_NAME MATCHES "Linux")
   434|            ggml_add_cpu_backend_variant(power0)
   435|            ggml_add_cpu_backend_variant(power7_1       POWER7)
   436|            ggml_add_cpu_backend_variant(power7_2       POWER7  VSX)
   437|            ggml_add_cpu_backend_variant(power8_1       POWER8)
   438|            ggml_add_cpu_backend_variant(power8_2       POWER8  VSX)
   439|            ggml_add_cpu_backend_variant(power9         POWER9  VSX)
   440|            ggml_add_cpu_backend_variant(power10        POWER10 VSX)
   441|            # POWER11 backend: only if compiler supports -mcpu=power11
   442|            check_cxx_compiler_flag("-mcpu=power11" GGML_CXX_SUPPORTS_POWER11)
   443|            if (GGML_CXX_SUPPORTS_POWER11)
   444|                message(STATUS "Compiler supports -mcpu=power11, enabling POWER11 backend")
   445|                ggml_add_cpu_backend_variant(power11 POWER11 VSX)
   446|            else()
   447|                message(STATUS "Skipping POWER11 backend: compiler does not support -mcpu=power11")
   448|            endif()
   449|        else()
   450|            message(FATAL_ERROR "Unsupported PowerPC target OS: ${CMAKE_SYSTEM_NAME}")
   451|        endif()
   452|    elseif (GGML_SYSTEM_ARCH STREQUAL "s390x")
   453|        if (CMAKE_SYSTEM_NAME MATCHES "Linux")
   454|            ggml_add_cpu_backend_variant(z15    Z15 VXE2)
   455|            ggml_add_cpu_backend_variant(z16    Z16 VXE2 NNPA)
   456|        else()
   457|            message(FATAL_ERROR "Unsupported s390x target OS: ${CMAKE_SYSTEM_NAME}")
   458|        endif()
   459|    elseif (GGML_SYSTEM_ARCH STREQUAL "riscv64")
   460|        if (CMAKE_SYSTEM_NAME MATCHES "Linux")
   461|            ggml_add_cpu_backend_variant(riscv64_0)
   462|            ggml_add_cpu_backend_variant(riscv64_v   RVV)
   463|        else()
   464|            message(FATAL_ERROR "Unsupported RISC-V target OS: ${CMAKE_SYSTEM_NAME}")
   465|        endif()
   466|    else()
   467|        message(FATAL_ERROR "GGML_CPU_ALL_VARIANTS not yet supported with ${GGML_SYSTEM_ARCH} on ${CMAKE_SYSTEM_NAME}")
   468|    endif()
   469|elseif (GGML_CPU)
   470|    ggml_add_cpu_backend_variant_impl("")
   471|endif()
   472|
   473|ggml_add_backend(BLAS)
   474|ggml_add_backend(CANN)
   475|ggml_add_backend(CUDA)
   476|ggml_add_backend(HIP)
   477|ggml_add_backend(METAL)
   478|ggml_add_backend(MUSA)
   479|ggml_add_backend(RPC)
   480|ggml_add_backend(VirtGPU)
   481|ggml_add_backend(SYCL)
   482|ggml_add_backend(Vulkan)
   483|ggml_add_backend(WebGPU)
   484|ggml_add_backend(zDNN)
   485|ggml_add_backend(OpenCL)
   486|ggml_add_backend(Hexagon)
   487|ggml_add_backend(ZenDNN)
   488|ggml_add_backend(OPENVINO)
   489|
   490|foreach (target ggml-base ggml)
   491|    target_include_directories(${target} PUBLIC    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include> $<INSTALL_INTERFACE:include>)
   492|    target_compile_features   (${target} PRIVATE c_std_11 cxx_std_17) # don't bump
   493|endforeach()
   494|
   495|target_link_libraries(ggml-base PRIVATE Threads::Threads)
   496|
   497|if (DEFINED MATH_LIBRARY)
   498|    target_link_libraries(ggml-base PRIVATE ${MATH_LIBRARY})
   499|elseif (NOT WIN32 AND NOT DEFINED ENV{ONEAPI_ROOT})
   500|    target_link_libraries(ggml-base PRIVATE m)
   501|endif()
   502|
   503|if (CMAKE_SYSTEM_NAME MATCHES "Android")
   504|    target_link_libraries(ggml-base PRIVATE dl)
   505|endif()
   506|
   507|if(CMAKE_SYSTEM_NAME MATCHES "visionOS")
   508|    target_compile_definitions(ggml-base PUBLIC _DARWIN_C_SOURCE)
   509|endif()
   510|
   511|if (BUILD_SHARED_LIBS)
   512|    foreach (target ggml-base ggml)
   513|        set_target_properties(${target} PROPERTIES POSITION_INDEPENDENT_CODE ON)
   514|        target_compile_definitions(${target} PRIVATE GGML_BUILD)
   515|        target_compile_definitions(${target} PUBLIC  GGML_SHARED)
   516|    endforeach()
   517|endif()
   518|