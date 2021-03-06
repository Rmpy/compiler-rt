# Build for the undefined behavior sanitizer runtime support library.

set(UBSAN_SOURCES
  ubsan_diag.cc
  ubsan_init.cc
  ubsan_flags.cc
  ubsan_handlers.cc
  ubsan_value.cc
  )

set(UBSAN_STANDALONE_SOURCES
  ubsan_init_standalone.cc
  )

set(UBSAN_CXX_SOURCES
  ubsan_handlers_cxx.cc
  ubsan_type_hash.cc
  )

include_directories(..)

set(UBSAN_CFLAGS ${SANITIZER_COMMON_CFLAGS})
append_no_rtti_flag(UBSAN_CFLAGS)
set(UBSAN_STANDALONE_CFLAGS ${SANITIZER_COMMON_CFLAGS})
append_no_rtti_flag(UBSAN_STANDALONE_CFLAGS)
set(UBSAN_CXXFLAGS ${SANITIZER_COMMON_CFLAGS})

add_custom_target(ubsan)

if(APPLE)
  # Common parts of UBSan runtime.
  add_compiler_rt_object_libraries(RTUbsan
    OS ${SANITIZER_COMMON_SUPPORTED_OS}
    ARCHS ${UBSAN_COMMON_SUPPORTED_ARCH}
    SOURCES ${UBSAN_SOURCES} ${UBSAN_CXX_SOURCES}
    CFLAGS ${UBSAN_CXXFLAGS})

  if(COMPILER_RT_HAS_UBSAN)
    # Initializer of standalone UBSan runtime.
    add_compiler_rt_object_libraries(RTUbsan_standalone
      OS ${SANITIZER_COMMON_SUPPORTED_OS}
      ARCHS ${UBSAN_SUPPORTED_ARCH}
      SOURCES ${UBSAN_STANDALONE_SOURCES}
      CFLAGS ${UBSAN_STANDALONE_CFLAGS})
    foreach(os ${SANITIZER_COMMON_SUPPORTED_OS})
      add_compiler_rt_darwin_dynamic_runtime(clang_rt.ubsan_${os}_dynamic ${os}
        ARCHS ${UBSAN_SUPPORTED_ARCH}
        SOURCES $<TARGET_OBJECTS:RTUbsan.${os}>
                $<TARGET_OBJECTS:RTUbsan_standalone.${os}>
                $<TARGET_OBJECTS:RTSanitizerCommon.${os}>)

      add_dependencies(ubsan clang_rt.ubsan_${os}_dynamic)
    endforeach()
  endif()

else()
  # Common parts of UBSan runtime.
  add_compiler_rt_object_libraries(RTUbsan
    ARCHS ${UBSAN_COMMON_SUPPORTED_ARCH}
    SOURCES ${UBSAN_SOURCES} CFLAGS ${UBSAN_CFLAGS})
  # C++-specific parts of UBSan runtime. Requires a C++ ABI library.
  add_compiler_rt_object_libraries(RTUbsan_cxx
    ARCHS ${UBSAN_COMMON_SUPPORTED_ARCH}
    SOURCES ${UBSAN_CXX_SOURCES} CFLAGS ${UBSAN_CXXFLAGS})

  if(COMPILER_RT_HAS_UBSAN)
    # Initializer of standalone UBSan runtime.
    add_compiler_rt_object_libraries(RTUbsan_standalone
      ARCHS ${UBSAN_SUPPORTED_ARCH}
      SOURCES ${UBSAN_STANDALONE_SOURCES} CFLAGS ${UBSAN_STANDALONE_CFLAGS})
    
    foreach(arch ${UBSAN_SUPPORTED_ARCH})
      # Standalone UBSan runtimes.
      add_compiler_rt_runtime(clang_rt.ubsan_standalone-${arch} ${arch} STATIC
        SOURCES $<TARGET_OBJECTS:RTSanitizerCommon.${arch}>
                $<TARGET_OBJECTS:RTSanitizerCommonLibc.${arch}>
                $<TARGET_OBJECTS:RTUbsan.${arch}>
                $<TARGET_OBJECTS:RTUbsan_standalone.${arch}>
        CFLAGS ${UBSAN_CFLAGS})
      add_compiler_rt_runtime(clang_rt.ubsan_standalone_cxx-${arch} ${arch} STATIC
        SOURCES $<TARGET_OBJECTS:RTUbsan_cxx.${arch}>
        CFLAGS ${UBSAN_CXXFLAGS})

      add_dependencies(ubsan
        clang_rt.ubsan_standalone-${arch}
        clang_rt.ubsan_standalone_cxx-${arch})
      if (UNIX AND NOT ${arch} MATCHES "i386|i686")
        add_sanitizer_rt_symbols(clang_rt.ubsan_standalone-${arch} ubsan.syms.extra)
        add_sanitizer_rt_symbols(clang_rt.ubsan_standalone_cxx-${arch} ubsan.syms.extra)
        add_dependencies(ubsan
          clang_rt.ubsan_standalone-${arch}-symbols
          clang_rt.ubsan_standalone_cxx-${arch}-symbols)
      endif()
    endforeach()
  endif()
endif()

add_dependencies(compiler-rt ubsan)
