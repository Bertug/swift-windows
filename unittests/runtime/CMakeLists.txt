if(("${SWIFT_HOST_VARIANT_SDK}" STREQUAL "${SWIFT_PRIMARY_VARIANT_SDK}") AND
   ("${SWIFT_HOST_VARIANT_ARCH}" STREQUAL "${SWIFT_PRIMARY_VARIANT_ARCH}"))

  set(PLATFORM_SOURCES)
  set(PLATFORM_TARGET_LINK_LIBRARIES)
  if(SWIFT_HOST_VARIANT MATCHES "${SWIFT_DARWIN_VARIANTS}")
    find_library(FOUNDATION_LIBRARY Foundation)
    list(APPEND PLATFORM_SOURCES
      weak.mm
      Refcounting.mm
      )
    # We need to link swiftCore on Darwin because the runtime still relies on
    # some stdlib hooks to implement SwiftObject.
    list(APPEND PLATFORM_TARGET_LINK_LIBRARIES
      ${FOUNDATION_LIBRARY}
      swiftCore${SWIFT_PRIMARY_VARIANT_SUFFIX}
      swiftStdlibUnittest${SWIFT_PRIMARY_VARIANT_SUFFIX}
      )
  endif()

  add_swift_unittest(SwiftRuntimeTests
    Metadata.cpp
    Enum.cpp
    Refcounting.cpp
    ${PLATFORM_SOURCES}
    )

  # The runtime tests link to internal runtime symbols, which aren't exported
  # from the swiftCore dylib, so we need to link to both the runtime archive
  # and the stdlib.
  # FIXME: cross-compile for all variants.
  target_link_libraries(SwiftRuntimeTests
    swiftRuntime${SWIFT_PRIMARY_VARIANT_SUFFIX}
    ${PLATFORM_TARGET_LINK_LIBRARIES}
    )
endif()

