set(swift_platform_name)
set(swift_platform_flags)
set(swift_platform_sources
    Platform.swift
    Misc.c
    tgmath.swift.gyb)

if(SWIFT_HOST_VARIANT MATCHES "${SWIFT_DARWIN_VARIANTS}")
  set(swift_platform_name swiftDarwin)
  set(swift_platform_sources
      Darwin.swift
      ${swift_platform_sources}
      POSIXError.swift
      MachError.swift)
  set(swift_platform_flags
      SWIFT_COMPILE_FLAGS -Xfrontend -disable-objc-attr-requires-foundation-module
      API_NOTES_NON_OVERLAY)
else()
  set(swift_platform_name swiftGlibc)
  set(swift_platform_sources
      Glibc.swift
      ${swift_platform_sources})
  set(swift_platform_flags
      DEPENDS glibc_modulemap)
endif()

add_swift_library(${swift_platform_name} IS_SDK_OVERLAY
    ${swift_platform_sources}
    ${swift_platform_flags})

set(glibc_modulemap_target_list)
foreach(sdk ${SWIFT_SDKS})
  if("${sdk}" STREQUAL "LINUX" OR "${sdk}" STREQUAL "FREEBSD" OR "${sdk}" STREQUAL "CYGWIN")
    foreach(arch ${SWIFT_SDK_${sdk}_ARCHITECTURES})
      set(arch_subdir "${SWIFT_SDK_${sdk}_LIB_SUBDIR}/${arch}")
      set(module_dir "${SWIFTLIB_DIR}/${arch_subdir}")

      set(glibc_modulemap_source "glibc.${SWIFT_SDK_${sdk}_TRIPLE_NAME}.modulemap.in")
      if(NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${glibc_modulemap_source}")
        set(glibc_modulemap_source "glibc.modulemap.in")
      endif()

      # Determine the location of glibc headers based on the target.
      set(GLIBC_INCLUDE_PATH "/usr/include")
      set(GLIBC_ARCH_INCLUDE_PATH ${GLIBC_INCLUDE_PATH})
      if(CMAKE_LIBRARY_ARCHITECTURE)
          # FIXME: Some distributions install headers in
          #        "/usr/include/x86_64-linux-gnu/sys/...". Here we use the host
          #        machine's path, regardless of the SDK target we're building for.
          #        This will break if cross-compiling from a distro that uses the
          #        architecture as part of the path to a distro that does not.
          set(GLIBC_ARCH_INCLUDE_PATH
              "${GLIBC_INCLUDE_PATH}/${CMAKE_LIBRARY_ARCHITECTURE}")
      endif()

      # Configure the module map based on the target. Each platform needs to
      # reference different headers, based on what's available in their glibc.
      set(glibc_modulemap_configured
          "${CMAKE_CURRENT_BINARY_DIR}/${arch_subdir}/glibc.modulemap")
      configure_file(${glibc_modulemap_source} ${glibc_modulemap_configured} @ONLY)

      add_custom_command_target(glibc_modulemap_target
          COMMAND
              ${CMAKE_COMMAND} -E make_directory ${module_dir}
          COMMAND
              ${CMAKE_COMMAND} -E copy_if_different
              ${glibc_modulemap_configured}
              "${module_dir}/glibc.modulemap"
          OUTPUT "${module_dir}/glibc.modulemap"
          DEPENDS ${glibc_modulemap_configured}
          COMMENT "Copying Glibc module to ${module_dir}")

      swift_install_in_component(stdlib
          FILES "${module_dir}/glibc.modulemap"
          DESTINATION "lib/swift/${arch_subdir}")

      list(APPEND glibc_modulemap_target_list ${glibc_modulemap_target})
    endforeach()
  endif()
endforeach()
add_custom_target(glibc_modulemap ALL DEPENDS ${glibc_modulemap_target_list})
