# ==============================================================================
# SWAT+ CMakeLists.txt — with permafrost extension (swatplus_pf)
#
# Executables produced:
#   swatplus_default   vanilla SWAT+, unchanged flags, no permafrost files
#   swatplus_pf        permafrost-enabled, performance-optimised Release flags
#
# Build:
#   cmake -B build -DCMAKE_BUILD_TYPE=Release
#   cmake --build build --target swatplus_pf -j$(nproc)
#
# Permafrost-only rebuild after source changes:
#   cmake --build build --target swatplus_pf -j$(nproc)
#
# Debug build:
#   cmake -B build_dbg -DCMAKE_BUILD_TYPE=Debug
#   cmake --build build_dbg --target swatplus_pf -j$(nproc)
# ==============================================================================

if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}" OR
   "${CMAKE_SOURCE_DIR}/src" STREQUAL "${CMAKE_BINARY_DIR}")
   message(FATAL_ERROR "'in-source' builds not permitted, e.g. use 'cmake -B build' instead.")
endif()

cmake_minimum_required(VERSION 3.22)

cmake_host_system_information(RESULT os_id      QUERY DISTRIB_ID)
cmake_host_system_information(RESULT os_version QUERY DISTRIB_RELEASE)
cmake_host_system_information(RESULT os_like    QUERY DISTRIB_ID_LIKE)
cmake_host_system_information(RESULT pretty_name QUERY DISTRIB_PRETTY_NAME)

project(swatplus VERSION 0.2 LANGUAGES Fortran)

# All .mod files written to and read from build/
set(CMAKE_Fortran_MODULE_DIRECTORY ${CMAKE_BINARY_DIR})

include(CTest)

# -- Git tag ------------------------------------------------------------------
if(NOT TAG)
    find_package(Git REQUIRED)
    execute_process(
        COMMAND ${GIT_EXECUTABLE} describe --tags
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE TAG
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    if(TAG STREQUAL "")
        set(TAG "unknown")
    endif()
endif()

# ==============================================================================
# Compiler flag sets
#
# fdialect   language/safety flags applied to EVERY target via CMAKE_Fortran_FLAGS
# fdebug     extra warnings, Debug builds only
# frelease   optimisation flags for the DEFAULT executable (conservative, -O2)
#
# pf_cancel  GNU-only: flags that cancel the safety options in fdialect that
#            conflict with -ffast-math.  GFortran respects last flag on command
#            line, so appending these after fdialect disables them:
#              -fno-check          cancels -fcheck=all
#              -ffpe-trap=none     cancels -ffpe-trap=invalid,zero,overflow,underflow
#              -fno-signaling-nans cancels -fsignaling-nans
#              -fno-rounding-math  cancels -frounding-math
#            -ffast-math then also cancels -fno-unsafe-math-optimizations.
#            Safe for swatplus_pf because all sqrt/exp inputs are physically
#            bounded and no NaN-sentinel values are used.
#
# pf_flags   performance flags added to swatplus_pf Release builds only.
#            NOT applied to swatplus_default.
# ==============================================================================

if(UNIX)
    if(CMAKE_Fortran_COMPILER_ID STREQUAL Intel)
        set(fdialect  "-fpp -free -fpe0 -traceback -diag-disable=10448")
        set(fdebug    "-warn all")
        set(frelease  "-O2")
        set(FFC       "ifo")
        set(pf_cancel "")
        set(pf_flags  "-O3 -xHost -unroll -ftz -DPERMAFROST")
        link_libraries("-static")
    elseif(CMAKE_Fortran_COMPILER_ID STREQUAL IntelLLVM)
        set(fdialect  "-fpp -free -fpe0 -traceback")
        set(fdebug    "-warn all -O0")
        set(frelease  "-O2")
        set(FFC       "ifx")
        set(pf_cancel "")
        set(pf_flags  "-O3 -xHost -unroll -ftz -DPERMAFROST")
        link_libraries("-static")
    elseif(CMAKE_Fortran_COMPILER_ID MATCHES GNU)
        set(fdialect  "-cpp -fcheck=all -ffpe-trap=invalid,zero,overflow,underflow -fimplicit-none -ffree-line-length-none -fbacktrace -finit-local-zero -fno-unsafe-math-optimizations -frounding-math -fsignaling-nans")
        set(fdebug    "-Wall")
        set(frelease  "-O2")
        set(FFC       "gnu")
        set(pf_cancel "-fno-check -ffpe-trap=none -fno-signaling-nans -fno-rounding-math")
        set(pf_flags  "-O3 -march=native -funroll-loops -ffast-math -flto -DPERMAFROST")
        if(NOT APPLE)
            if(NOT ("${os_id}" STREQUAL "Arch" OR
                    "${os_like}" MATCHES ".*arch.*" OR
                    "${os_id}" STREQUAL "fedora"))
                link_libraries("-static")
            endif()
        endif()
    endif()
elseif(WIN32)
    if(CMAKE_Fortran_COMPILER_ID STREQUAL Intel)
        set(fdialect  "/fpp /free /fpe0 /traceback /Qdiag-disable:all /Qdiag-disable:remarks")
        set(fdebug    "/warn:all")
        set(frelease  "/O2")
        set(FFC       "ifo")
        set(pf_cancel "")
        set(pf_flags  "/O3 /QxHost /Qunroll /ftz /DPERMAFROST")
        link_libraries("-static")
    elseif(CMAKE_Fortran_COMPILER_ID STREQUAL IntelLLVM)
        set(fdialect  "/fpp /free /fpe0 /traceback")
        set(fdebug    "/warn:all")
        set(frelease  "/O2")
        set(FFC       "ifx")
        set(pf_cancel "")
        set(pf_flags  "/O3 /QxHost /Qunroll /ftz /DPERMAFROST")
    elseif(CMAKE_Fortran_COMPILER_ID MATCHES GNU)
        # Windows GNU dialect has no -fcheck=all or -ffpe-trap so no conflicts
        set(fdialect  "-cpp -ffree-line-length-none -fbacktrace -finit-local-zero")
        set(fdebug    "-Wall")
        set(frelease  "-O2")
        set(FFC       "gnu")
        set(pf_cancel "")
        set(pf_flags  "-O3 -march=native -funroll-loops -ffast-math -DPERMAFROST")
        
    endif()
endif()

# Convert space-separated flag strings into proper CMake lists.
# This is critical: target_compile_options needs a LIST (semicolon-separated
# items), not a single space-separated string.  If a string is passed inside
# a generator expression CMake treats the whole string as one token and the
# compiler sees it as a filename rather than individual flags.
# separate_arguments(... UNIX_COMMAND) splits on spaces and handles quoting.
separate_arguments(pf_cancel_list    UNIX_COMMAND "${pf_cancel}")
separate_arguments(pf_flags_list     UNIX_COMMAND "${pf_flags}")
set(pf_release_compile_flags ${pf_cancel_list} ${pf_flags_list})

# ==============================================================================
# CRITICAL: -DPERMAFROST must be defined for ALL permafrost targets in EVERY
# configuration (Release, Debug, RelWithDebInfo, or none).  Previously the
# define was attached only on the non-Release branch and was expected to ride
# inside ${pf_release_compile_flags} on Release.  That coupling is fragile: if
# the compiler-detection if/elseif above does not match (e.g. an unexpected
# CMAKE_Fortran_COMPILER_ID on Windows), pf_flags is empty, the define vanishes
# on Release, and the ENTIRE permafrost module (#ifdef PERMAFROST) is silently
# compiled out — the build still succeeds but swatplus_pf contains no permafrost
# code.  Defining it unconditionally here removes that failure mode for good.
set(PF_DEFINE -DPERMAFROST)

# Hard guard: if no compiler branch populated pf_flags, the optimisation flags
# (and historically the define) would be empty.  Fail loudly rather than
# silently producing a permafrost-less binary.
if(pf_flags STREQUAL "")
    message(WARNING
        "pf_flags is EMPTY — compiler '${CMAKE_Fortran_COMPILER_ID}' did not "
        "match any optimisation branch.  swatplus_pf will still get "
        "-DPERMAFROST (via PF_DEFINE) but NOT the -O3/native optimisation "
        "flags.  Check the compiler-detection block if you expected an "
        "optimised build.")
endif()

get_property(CURRENT_LINK_LIBS DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY LINK_LIBRARIES)

# -- Platform/arch string -----------------------------------------------------
string(TOLOWER ${CMAKE_HOST_SYSTEM_PROCESSOR} ARCH)
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(AR "lin_${ARCH}")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    set(AR "win_${ARCH}")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set(AR "mac_${ARCH}")
else()
    set(AR "unknown")
endif()

if(CMAKE_BUILD_TYPE STREQUAL "Release")
    set(TY "-Rel")
elseif(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(TY "-Dbg")
else()
    set(TY "")
endif()

set(SWAT_VERSION ${TAG})
set(SWATPLUS_EXE "swatplus_default")
set(CMAKE_VERBOSE_MAKEFILE FALSE)

set(CMAKE_Fortran_FLAGS         "${CMAKE_Fortran_FLAGS} ${fdialect}")
set(CMAKE_Fortran_FLAGS_DEBUG   "${CMAKE_Fortran_FLAGS_DEBUG} ${fdebug}")
set(CMAKE_Fortran_FLAGS_RELEASE "${frelease}")

# ==============================================================================
# Generate main.f90 from .in templates
# ==============================================================================
if(EXISTS "${PROJECT_SOURCE_DIR}/src/main.f90.in")
    string(TIMESTAMP ISO   "%Y-%m-%d %H:%M:%S")
    string(TIMESTAMP TODAY "%Y-%m-%d")
    string(TIMESTAMP YEAR  "%Y")
    configure_file(
        "${PROJECT_SOURCE_DIR}/src/main.f90.in"
        "${PROJECT_SOURCE_DIR}/src/main.f90"
    )
endif()

if(EXISTS "${PROJECT_SOURCE_DIR}/patched/main.f90.in")
    configure_file(
        "${PROJECT_SOURCE_DIR}/patched/main.f90.in"
        "${PROJECT_SOURCE_DIR}/patched/main.f90"
    )
endif()

if(EXISTS "${PROJECT_SOURCE_DIR}/ford.md.in")
    configure_file(
        "${PROJECT_SOURCE_DIR}/ford.md.in"
        "${PROJECT_SOURCE_DIR}/ford.md"
    )
endif()

# ==============================================================================
# DEFAULT executable — pure vanilla SWAT+
# All permafrost source files excluded; flags and behaviour unchanged.
# ==============================================================================
file(GLOB sources_default src/*.f90)
list(FILTER sources_default EXCLUDE REGEX ".*/permafrost_.*\\.f90$")
list(FILTER sources_default EXCLUDE REGEX ".*/pf_.*\\.f90$")

add_executable(swatplus_default ${sources_default})

# ==============================================================================
# PERMAFROST executable — swatplus_pf
#
# Two-stage static lib compilation + OBJECT_DEPENDS for intra-target ordering:
#
#   Stage 1   pf_parms_lib   permafrost_parms.f90
#                            -> permafrost_parms_module.mod
#                            No SWAT+ .mod deps -> safe as standalone static lib
#
#   Stage 2   pf_hru_lib     patched/hru_module.f90
#                            -> hru_module.mod  (with pf_flag field)
#                            Only needs permafrost_parms_module.mod -> safe
#
#   Stage 3   swatplus_pf    all sources in one target:
#                              src/*.f90 (vanilla, filtered)
#                              src/pf_parms_read.f90
#                              src/permafrost_heat.f90
#                              src/permafrost_output.f90
#                              patched/*.f90
#
#             pf_parms_read / permafrost_heat / permafrost_output CANNOT be in
#             a separate static lib because they USE vanilla SWAT+ modules
#             (maximum_data_module, hydrograph_module, soil_module, ...) whose
#             .mod files only exist within this target's build context.
#
#             Intra-target order is enforced via OBJECT_DEPENDS: each patched
#             source that USEs a pf .mod declares that .mod file as a file-level
#             dependency, so the build system waits for the producer to finish
#             before starting the consumer, even with parallel -j builds.
# ==============================================================================

# -- Stage 1: permafrost_parms module -----------------------------------------
add_library(pf_parms_lib STATIC
    ${PROJECT_SOURCE_DIR}/src/permafrost_parms.f90
)
set_target_properties(pf_parms_lib PROPERTIES
    Fortran_MODULE_DIRECTORY ${CMAKE_BINARY_DIR}
)
target_compile_options(pf_parms_lib PRIVATE
    $<$<CONFIG:Release>:${pf_release_compile_flags}>
    $<$<NOT:$<CONFIG:Release>>:-DPERMAFROST>
)

# -- Stage 2: patched hru_module ----------------------------------------------
add_library(pf_hru_lib STATIC
    ${PROJECT_SOURCE_DIR}/patched/hru_module.f90
)
set_target_properties(pf_hru_lib PROPERTIES
    Fortran_MODULE_DIRECTORY ${CMAKE_BINARY_DIR}
)
target_compile_options(pf_hru_lib PRIVATE
    $<$<CONFIG:Release>:${pf_release_compile_flags}>
    $<$<NOT:$<CONFIG:Release>>:-DPERMAFROST>
)
add_dependencies(pf_hru_lib pf_parms_lib swatplus_default)
# -- Stage 2b: pf physics OBJECT library --------------------------------------
#
# permafrost_heat.f90, permafrost_output.f90 and pf_parms_read.f90 all USE
# vanilla SWAT+ modules (maximum_data_module, hydrograph_module, soil_module,
# time_module, ...).  These .mod files are produced by swatplus_default, which
# writes them to CMAKE_BINARY_DIR — the same directory searched by all targets.
#
# Making pf_physics_obj depend on swatplus_default guarantees every vanilla
# .mod exists before any pf physics file is touched, without creating a
# circular dependency (swatplus_default has no pf deps).
#
# OBJECT library (not STATIC): compiled .obj files are contributed directly
# to swatplus_pf via $<TARGET_OBJECTS:pf_physics_obj>, and the .mod files
# land in CMAKE_BINARY_DIR where Stage 3 patched sources find them.

add_library(pf_physics_obj OBJECT
    ${PROJECT_SOURCE_DIR}/src/pf_parms_read.f90
    ${PROJECT_SOURCE_DIR}/src/permafrost_heat.f90
    ${PROJECT_SOURCE_DIR}/src/permafrost_output.f90
)
set_target_properties(pf_physics_obj PROPERTIES
    Fortran_MODULE_DIRECTORY ${CMAKE_BINARY_DIR}
)
target_compile_options(pf_physics_obj PRIVATE
    $<$<CONFIG:Release>:${pf_release_compile_flags}>
    $<$<NOT:$<CONFIG:Release>>:-DPERMAFROST>
)
target_include_directories(pf_physics_obj PRIVATE ${CMAKE_BINARY_DIR})

# Wait for: all vanilla .mods (swatplus_default) + pf_parms + hru mods
add_dependencies(pf_physics_obj swatplus_default pf_parms_lib pf_hru_lib)

# -- Stage 3: swatplus_pf executable ------------------------------------------
#
# Sources: vanilla src/*.f90 (filtered) + patched replacements.
# pf physics .obj files come from $<TARGET_OBJECTS:pf_physics_obj>.
# By the time Stage 3 compiles:
#   permafrost_heat.mod / permafrost_output.mod / pf_parms_read_mod.mod
# all exist in CMAKE_BINARY_DIR — no race condition.

file(GLOB sources_pf src/*.f90)

# Remove files replaced by patched versions or compiled in earlier stages
list(FILTER sources_pf EXCLUDE REGEX ".*/hru_module\\.f90$")
list(FILTER sources_pf EXCLUDE REGEX ".*/proc_hru\\.f90$")
list(FILTER sources_pf EXCLUDE REGEX ".*/hru_control\\.f90$")
list(FILTER sources_pf EXCLUDE REGEX ".*/stmp_solt\\.f90$")
list(FILTER sources_pf EXCLUDE REGEX ".*/swr_percmain\\.f90$")
list(FILTER sources_pf EXCLUDE REGEX ".*/main\\.f90$")
list(FILTER sources_pf EXCLUDE REGEX ".*/cal_parmchg_read\\.f90$")
list(FILTER sources_pf EXCLUDE REGEX ".*/cal_parm_select\\.f90$")
list(FILTER sources_pf EXCLUDE REGEX ".*/permafrost_coupling_patches\\.f90$")
# === NEW: exclude vanilla aqu_1d_control.f90 — replaced by patched version ===
list(FILTER sources_pf EXCLUDE REGEX ".*/aqu_1d_control\\.f90$")
# pf physics compiled as pf_physics_obj (Stage 2b)
list(FILTER sources_pf EXCLUDE REGEX ".*/pf_parms_read\\.f90$")
list(FILTER sources_pf EXCLUDE REGEX ".*/permafrost_heat\\.f90$")
list(FILTER sources_pf EXCLUDE REGEX ".*/permafrost_output\\.f90$")
# permafrost_parms and hru_module compiled as Stages 1 and 2
list(FILTER sources_pf EXCLUDE REGEX ".*/permafrost_parms\\.f90$")

# Add patched SWAT+ replacements
list(APPEND sources_pf
    ${PROJECT_SOURCE_DIR}/patched/proc_hru.f90
    ${PROJECT_SOURCE_DIR}/patched/hru_control.f90
    ${PROJECT_SOURCE_DIR}/patched/stmp_solt.f90
    ${PROJECT_SOURCE_DIR}/patched/swr_percmain.f90
    ${PROJECT_SOURCE_DIR}/patched/main.f90
    ${PROJECT_SOURCE_DIR}/patched/cal_parmchg_read.f90
    ${PROJECT_SOURCE_DIR}/patched/cal_parm_select.f90
    # === NEW: patched aquifer control with permafrost gating ===
    ${PROJECT_SOURCE_DIR}/patched/aqu_1d_control.f90
)

add_executable(swatplus_pf
    ${sources_pf}
    $<TARGET_OBJECTS:pf_physics_obj>
)

target_compile_options(swatplus_pf PRIVATE
    $<$<CONFIG:Release>:${pf_release_compile_flags}>
    $<$<NOT:$<CONFIG:Release>>:-DPERMAFROST>
)
target_include_directories(swatplus_pf PRIVATE ${CMAKE_BINARY_DIR})

# Stage 3 waits for Stage 2b (which already waited for default + 1 + 2)
add_dependencies(swatplus_pf pf_physics_obj)
target_link_libraries(swatplus_pf PRIVATE pf_parms_lib pf_hru_lib)

# LTO requires the linker to also receive -flto
if(CMAKE_BUILD_TYPE STREQUAL "Release" AND CMAKE_Fortran_COMPILER_ID MATCHES GNU
   AND NOT WIN32)
    target_link_options(swatplus_pf PRIVATE -flto)
endif()

# ==============================================================================
# Install both executables
# ==============================================================================
install(TARGETS swatplus_default swatplus_pf
    RUNTIME DESTINATION ${PROJECT_SOURCE_DIR})

# ==============================================================================
# Testing
# ==============================================================================
find_package(Python3 QUIET)

set(spcheck  "${PROJECT_SOURCE_DIR}/test/spcheck.py")
set(test_dir "${PROJECT_BINARY_DIR}/data")
set(ref_dir  "${PROJECT_SOURCE_DIR}/data")
set(rerr "0.01")
set(aerr "1e-8")

add_test(Ames_sub1 ${Python3_EXECUTABLE} ${spcheck} ctest
    ${PROJECT_BINARY_DIR}/swatplus_default
    ${ref_dir}/Ames_sub1 ${test_dir} --abserr ${aerr} --relerr ${rerr})

# ==============================================================================
# Build summary
# ==============================================================================
message("-- Verbose Makefile      ${CMAKE_VERBOSE_MAKEFILE}")
message("-- Detected OS ID:       ${os_id}")
message("-- Build Type            ${CMAKE_BUILD_TYPE}")
message("-- Compiler              ${CMAKE_Fortran_COMPILER} (${CMAKE_Fortran_COMPILER_ID} ${CMAKE_Fortran_COMPILER_VERSION})")
message("-- FFLAGS (global)       ${CMAKE_Fortran_FLAGS}")
message("--  RELEASE (default)    ${CMAKE_Fortran_FLAGS_RELEASE}")
message("--  DEBUG                ${CMAKE_Fortran_FLAGS_DEBUG}")
message("--  pf Release extras    ${pf_release_compile_flags}")
message("-- Link_libraries        ${CURRENT_LINK_LIBS}")
message("-- PROJECT_DIR           ${PROJECT_SOURCE_DIR}")
message("-- BINARY_DIR            ${PROJECT_BINARY_DIR}")
message("-- Default exe           swatplus_default")
message("-- Permafrost exe        swatplus_pf  (Stage1: pf_parms_lib, Stage2: pf_hru_lib, Stage2b: pf_physics_obj OBJECT, Stage3: swatplus_pf)")
message("-- SWAT version/tag      ${SWAT_VERSION}")