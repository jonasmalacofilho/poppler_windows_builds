project(poppler)

cmake_minimum_required(VERSION 3.1.0 FATAL_ERROR)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/modules)

include(PopplerDefaults)
include(PopplerMacros)
include(MacroOptionalFindPackage)
find_package(PkgConfig)
include(MacroEnsureVersion)
include(MacroBoolTo01)
if (WIN32)
  message("-- Using win32 threads")
else()
  set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
  set(THREADS_PREFER_PTHREAD_FLAG TRUE)
  find_package(Threads)
endif()
include(TestBigEndian)
test_big_endian(WORDS_BIGENDIAN)
include(CheckFileOffsetBits)
CHECK_FILE_OFFSET_BITS()

include(GNUInstallDirs)

set(ENABLE_FUZZER FALSE)

find_package (ECM 1.6.0 QUIET NO_MODULE)
if (ECM_FOUND)
    include("${ECM_MODULE_DIR}/ECMEnableSanitizers.cmake")

    if(ECM_ENABLE_SANITIZERS MATCHES fuzzer)
        set(ENABLE_FUZZER TRUE)
    endif()
endif()

set(POPPLER_MAJOR_VERSION "0")
set(POPPLER_MINOR_VERSION "80")
set(POPPLER_MICRO_VERSION "0")
set(POPPLER_VERSION "${POPPLER_MAJOR_VERSION}.${POPPLER_MINOR_VERSION}.${POPPLER_MICRO_VERSION}")

set (CMAKE_CXX_STANDARD 14)
set (CMAKE_CXX_EXTENSIONS OFF)

# command line switches
option(ENABLE_UNSTABLE_API_ABI_HEADERS "Install API/ABI unstable xpdf headers." OFF)
option(BUILD_GTK_TESTS "Whether to compile the GTK+ test programs." ON)
option(BUILD_QT5_TESTS "Whether to compile the Qt5 test programs." ON)
option(BUILD_CPP_TESTS "Whether to compile the CPP test programs." ON)
option(ENABLE_SPLASH "Build the Splash graphics backend." ON)
option(ENABLE_UTILS "Compile poppler command line utils." ON)
option(ENABLE_CPP "Compile poppler cpp wrapper." ON)
option(ENABLE_GLIB "Compile poppler glib wrapper." ON)
option(ENABLE_GOBJECT_INTROSPECTION "Whether to generate GObject introspection." ON)
option(ENABLE_GTK_DOC "Whether to generate glib API documentation." OFF)
option(ENABLE_QT5 "Compile poppler qt5 wrapper." ON)
set(ENABLE_LIBOPENJPEG "openjpeg2" CACHE STRING "Use libopenjpeg for JPX streams. Possible values: openjpeg2, unmaintained, none. 'unmaintained' gives you the internal unmaintained decoder. Use at your own risk. 'none' compiles no JPX decoder at all. Default: openjpeg2")
set(ENABLE_CMS "lcms2" CACHE STRING "Use color management system. Possible values: lcms2, none. 'none' disables color management system.")
set(ENABLE_DCTDECODER "libjpeg" CACHE STRING "Use libjpeg for DCT streams. Possible values: libjpeg, unmaintained, none. will use libjpeg if available or fail if not. 'unmaintained' gives you the internal unmaintained decoder. Use at your own risk. 'none' compiles no DCT decoder at all. Default: libjpeg")
option(ENABLE_LIBCURL "Build libcurl based HTTP support." ON)
option(ENABLE_ZLIB "Build with zlib." ON)
option(ENABLE_ZLIB_UNCOMPRESS "Use zlib to uncompress flate streams (not totally safe)." OFF)
option(USE_FLOAT "Use single precision arithmetic in the Splash backend" OFF)
option(BUILD_SHARED_LIBS "Build poppler as a shared library" ON)
if(WIN32)
  option(ENABLE_RELOCATABLE "Do not hardcode the poppler library location (on Windows)." ON)
else()
  set(ENABLE_RELOCATABLE OFF)
endif()
option(EXTRA_WARN "Enable extra compile warnings" OFF)

set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

set(TESTDATADIR "${CMAKE_SOURCE_DIR}/../test" CACHE STRING "Specify test data dir.")
if(NOT (EXISTS ${TESTDATADIR} AND EXISTS ${TESTDATADIR}/test-poppler.c))
  message(WARNING "
 No test data found in $testdatadir.
 You will not be able to run 'make test' successfully.

 The test data is not included in the source packages
 and is also not part of the main git repository. Instead,
 you can checkout the test data from its own git
 repository with:

   git clone git://git.freedesktop.org/git/poppler/test

 You should checkout the test data as a sibling of your
 poppler source folder or specify the location of your
 checkout with -DTESTDATADIR=/path/to/checkoutdir/test.
  ")
endif()

if(WIN32)
  set(_default_fontconfiguration "win32")
elseif(ANDROID)
  # on android we don't have fontconfig and we don't want windows-specific code
  set(_default_fontconfiguration "generic")
else()
  set(_default_fontconfiguration "fontconfig")
endif()
set(FONT_CONFIGURATION "${_default_fontconfiguration}" CACHE STRING "The font configuration backend (win32|generic|fontconfig).")
string(TOLOWER "${FONT_CONFIGURATION}" font_configuration)
set(WITH_FONTCONFIGURATION_WIN32 OFF)
set(WITH_FONTCONFIGURATION_FONTCONFIG OFF)
if(font_configuration STREQUAL "win32")
  set(WITH_FONTCONFIGURATION_WIN32 ON)
elseif(font_configuration STREQUAL "fontconfig")
  set(WITH_FONTCONFIGURATION_FONTCONFIG ON)
elseif(font_configuration STREQUAL "generic")
  message(STATUS "no fontconfig or win32 specific code")
else()
  message(FATAL_ERROR "Invalid font configuration setting: ${FONT_CONFIGURATION}")
endif()

# Enable these unconditionally.
set(OPI_SUPPORT ON)
set(TEXTOUT_WORD_LIST ON)

# setting the minimum required versions for some components
set(CAIRO_VERSION "1.10.0")
set(GLIB_REQUIRED "2.41")

macro_bool_to_01(ENABLE_SPLASH HAVE_SPLASH)
find_package(Freetype REQUIRED)
if(WITH_FONTCONFIGURATION_FONTCONFIG)
  find_package(Fontconfig REQUIRED)
endif()
macro_optional_find_package(JPEG)
macro_optional_find_package(PNG)
macro_optional_find_package(TIFF)
macro_optional_find_package(NSS3)
if(ENABLE_DCTDECODER STREQUAL "libjpeg")
  if(JPEG_FOUND)
    set(ENABLE_LIBJPEG ${JPEG_FOUND})
  else()
    message(FATAL_ERROR "Install libjpeg before trying to build poppler. You can also decide to use the internal unmaintained DCT decoder or none at all.")
  endif()
  set(HAVE_DCT_DECODER ON)
elseif(ENABLE_DCTDECODER STREQUAL "unmaintained")
  set(ENABLE_LIBJPEG OFF)
  set(HAVE_DCT_DECODER ON)
elseif(ENABLE_DCTDECODER STREQUAL "none")
  set(ENABLE_LIBJPEG OFF)
  set(HAVE_DCT_DECODER OFF)
else()
  message(FATAL_ERROR "Invalid ENABLE_DCTDECODER value.")
endif()

if (ENABLE_QT5)
  find_package(Qt5Core)
  find_package(Qt5Gui)
  find_package(Qt5Xml)
  find_package(Qt5Widgets)
  find_package(Qt5Test)
  if (NOT (Qt5Core_FOUND AND Qt5Gui_FOUND AND Qt5Xml_FOUND AND Qt5Widgets_FOUND AND Qt5Test_FOUND))
    message("-- Package Qt5Core or Qt5Gui or Qt5Xml or Qt5Widgets or Qt5Test not found")
    set(ENABLE_QT5 OFF)
  endif()
endif()

macro_optional_find_package(Cairo ${CAIRO_VERSION})
if(CAIRO_FOUND)
  set(HAVE_CAIRO ${CAIRO_FOUND})
  set(CAIRO_FEATURE "#define POPPLER_HAS_CAIRO 1")
  set(CAIRO_REQ "cairo")
  set(POPPLER_GLIB_DISABLE_DEPRECATED "")
  set(POPPLER_GLIB_DISABLE_SINGLE_INCLUDES "")
  if(ENABLE_GLIB)
    macro_optional_find_package(GLIB)
    if(NOT GLIB_FOUND)
      set(ENABLE_GLIB OFF)
    endif()
  endif()
  if(ENABLE_GLIB)
    if(ENABLE_GOBJECT_INTROSPECTION)
      # Check for introspection
      macro_optional_find_package(GObjectIntrospection 0.9.12)
      set(HAVE_INTROSPECTION ${INTROSPECTION_FOUND})
    endif()
    set(POPPLER_GLIB_DISABLE_DEPRECATED "${POPPLER_GLIB_DISABLE_DEPRECATED} -DG_DISABLE_DEPRECATED")
    set(POPPLER_GLIB_DISABLE_SINGLE_INCLUDES "${POPPLER_GLIB_DISABLE_SINGLE_INCLUDES} -DG_DISABLE_SINGLE_INCLUDES")
    macro_optional_find_package(GTK)
  endif()
else()
  set(CAIRO_FEATURE "#undef POPPLER_HAS_CAIRO")
  set(ENABLE_GLIB OFF)
endif()
if(ENABLE_CPP)
  macro_optional_find_package(Iconv)
  set(ENABLE_CPP ${ICONV_FOUND})
  set(HAVE_ICONV ${ICONV_FOUND})
endif()
if(ENABLE_ZLIB)
  find_package(ZLIB)
  set(ENABLE_ZLIB ${ZLIB_FOUND})
endif()
if(ENABLE_ZLIB_UNCOMPRESS AND NOT ENABLE_ZLIB)
  message("Warning: ENABLE_ZLIB_UNCOMPRESS requires ENABLE_ZLIB")
  set(ENABLE_ZLIB_UNCOMPRESS FALSE)
endif()
set(WITH_OPENJPEG FALSE)
if(ENABLE_LIBOPENJPEG STREQUAL "openjpeg2")
  find_package(OpenJPEG)
  set(WITH_OPENJPEG ${OpenJPEG_FOUND})
  if(NOT OpenJPEG_FOUND OR OPENJPEG_MAJOR_VERSION VERSION_LESS 2)
    message(FATAL_ERROR "Install libopenjpeg2 before trying to build poppler. You can also decide to use the internal unmaintained JPX decoder or none at all.")
  endif()
  set(HAVE_JPX_DECODER ON)
elseif(ENABLE_LIBOPENJPEG STREQUAL "unmaintained")
  set(WITH_OPENJPEG OFF)
  set(HAVE_JPX_DECODER ON)
elseif(ENABLE_LIBOPENJPEG STREQUAL "none")
  set(WITH_OPENJPEG OFF)
  set(HAVE_JPX_DECODER OFF)
else()
  message(FATAL_ERROR "Invalid ENABLE_LIBOPENJPEG value: ${ENABLE_LIBOPENJPEG}")
endif()
set(ENABLE_LIBOPENJPEG "${WITH_OPENJPEG}")
if(ENABLE_CMS STREQUAL "lcms2")
  find_package(LCMS2)
  set(USE_CMS ${LCMS2_FOUND})
elseif(NOT ENABLE_CMS STREQUAL "none")
  message(FATAL_ERROR "Invalid ENABLE_CMS value: ${ENABLE_CMS}")
endif()
if(ENABLE_LIBCURL)
  find_package(CURL)
  if(CURL_FOUND)
    include_directories(SYSTEM ${CURL_INCLUDE_DIR})
    set(POPPLER_HAS_CURL_SUPPORT ON)
  else()
    set(ENABLE_LIBCURL OFF)
  endif()
endif()

if(MINGW)
  # Use mingw's ansi stdio extensions
  add_definitions(-D__USE_MINGW_ANSI_STDIO=1)
endif()
if(FONTCONFIG_FOUND)
  add_definitions(${FONTCONFIG_DEFINITIONS})
  include_directories(SYSTEM ${FONTCONFIG_INCLUDE_DIR})
endif()
if(WITH_FONTCONFIGURATION_WIN32)
  if(MINGW)
    # Set the minimum required Internet Explorer version to 5.0
    add_definitions(-D_WIN32_IE=0x0500)
  endif()
endif()
include_directories(
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${CMAKE_CURRENT_SOURCE_DIR}/fofi
  ${CMAKE_CURRENT_SOURCE_DIR}/goo
  ${CMAKE_CURRENT_SOURCE_DIR}/poppler
  ${CMAKE_CURRENT_BINARY_DIR}
  ${CMAKE_CURRENT_BINARY_DIR}/poppler
  SYSTEM
  ${FREETYPE_INCLUDE_DIRS}
)

if(ENABLE_ZLIB)
  include_directories(SYSTEM ${ZLIB_INCLUDE_DIR})
endif()

if (NSS3_FOUND)
  add_definitions(${NSS3_CFLAGS})
  set(ENABLE_NSS3 ON)
endif()
if(JPEG_FOUND)
  include_directories(SYSTEM ${JPEG_INCLUDE_DIR})
endif()
if(PNG_FOUND)
  include_directories(SYSTEM ${PNG_INCLUDE_DIR})
  set(ENABLE_LIBPNG ON)
endif()
if(TIFF_FOUND)
  include_directories(SYSTEM ${TIFF_INCLUDE_DIR})
  set(ENABLE_LIBTIFF ON)
endif()
if(OpenJPEG_FOUND)
  include_directories(SYSTEM ${OPENJPEG_INCLUDE_DIRS})
endif()
if(LCMS2_FOUND)
  include_directories(SYSTEM ${LCMS2_INCLUDE_DIR})
endif()

if(ENABLE_SPLASH)
  find_package(Boost 1.58.0)
  if(Boost_FOUND)
    include_directories(SYSTEM ${Boost_INCLUDE_DIRS})
    set(USE_BOOST_HEADERS ON)
  endif()
endif()

# Recent versions of poppler-data install a .pc file.
# Use it to determine the encoding data path, if available.
# Default to the same prefix otherwise.
pkg_check_modules(POPPLER_DATA poppler-data)
if(POPPLER_DATA_FOUND)
  execute_process(COMMAND "${PKG_CONFIG_EXECUTABLE}" --variable=poppler_datadir poppler-data
                  RESULT_VARIABLE _result_var
                  OUTPUT_VARIABLE _output_var OUTPUT_STRIP_TRAILING_WHITESPACE)
  if(_result_var STREQUAL "0" AND NOT _output_var STREQUAL "")
    set(POPPLER_DATADIR "${_output_var}")
  endif()
endif()
if(NOT DEFINED POPPLER_DATADIR)
  set(POPPLER_DATADIR "${CMAKE_INSTALL_PREFIX}/share/poppler")
endif()

if(EXTRA_WARN)
  set(CMAKE_C_FLAGS "-Wall ${CMAKE_C_FLAGS}")
  set(CMAKE_CXX_FLAGS "${DEFAULT_COMPILE_WARNINGS_EXTRA} ${CMAKE_CXX_FLAGS}")
else()
  set(CMAKE_C_FLAGS "-Wall ${CMAKE_C_FLAGS}")
  set(CMAKE_CXX_FLAGS "${DEFAULT_COMPILE_WARNINGS} ${CMAKE_CXX_FLAGS}")
endif()

include(ConfigureChecks.cmake)
configure_file(config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/config.h)
configure_file(poppler/poppler-config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/poppler/poppler-config.h)

set(poppler_SRCS
  goo/gbase64.cc
  goo/gbasename.cc
  goo/gfile.cc
  goo/GooTimer.cc
  goo/GooString.cc
  goo/NetPBMWriter.cc
  goo/PNGWriter.cc
  goo/TiffWriter.cc
  goo/JpegWriter.cc
  goo/ImgWriter.cc
  goo/gstrtod.cc
  goo/grandom.cc
  goo/glibc.cc
  goo/glibc_strtok_r.cc
  fofi/FoFiBase.cc
  fofi/FoFiEncodings.cc
  fofi/FoFiTrueType.cc
  fofi/FoFiType1.cc
  fofi/FoFiType1C.cc
  fofi/FoFiIdentifier.cc
  poppler/Annot.cc
  poppler/Array.cc
  poppler/BuiltinFont.cc
  poppler/BuiltinFontTables.cc
  poppler/CachedFile.cc
  poppler/Catalog.cc
  poppler/CharCodeToUnicode.cc
  poppler/CMap.cc
  poppler/DateInfo.cc
  poppler/Decrypt.cc
  poppler/Dict.cc
  poppler/Error.cc
  poppler/FileSpec.cc
  poppler/FontEncodingTables.cc
  poppler/Form.cc
  poppler/FontInfo.cc
  poppler/Function.cc
  poppler/Gfx.cc
  poppler/GfxFont.cc
  poppler/GfxState.cc
  poppler/GlobalParams.cc
  poppler/Hints.cc
  poppler/JArithmeticDecoder.cc
  poppler/JBIG2Stream.cc
  poppler/Lexer.cc
  poppler/Link.cc
  poppler/Linearization.cc
  poppler/LocalPDFDocBuilder.cc
  poppler/MarkedContentOutputDev.cc
  poppler/NameToCharCode.cc
  poppler/Object.cc
  poppler/OptionalContent.cc
  poppler/Outline.cc
  poppler/OutputDev.cc
  poppler/Page.cc
  poppler/PageTransition.cc
  poppler/Parser.cc
  poppler/PDFDoc.cc
  poppler/PDFDocEncoding.cc
  poppler/PDFDocFactory.cc
  poppler/ProfileData.cc
  poppler/PreScanOutputDev.cc
  poppler/PSTokenizer.cc
  poppler/SignatureInfo.cc
  poppler/Stream.cc
  poppler/StructTreeRoot.cc
  poppler/StructElement.cc
  poppler/UnicodeMap.cc
  poppler/UnicodeMapFuncs.cc
  poppler/UnicodeTypeTable.cc
  poppler/UTF.cc
  poppler/XRef.cc
  poppler/PSOutputDev.cc
  poppler/TextOutputDev.cc
  poppler/PageLabelInfo.cc
  poppler/SecurityHandler.cc
  poppler/StdinCachedFile.cc
  poppler/StdinPDFDocBuilder.cc
  poppler/Sound.cc
  poppler/ViewerPreferences.cc
  poppler/Movie.cc
  poppler/Rendition.cc
  poppler/CertificateInfo.cc
)
set(poppler_LIBS ${FREETYPE_LIBRARIES})
if(ENABLE_SPLASH)
  set(poppler_SRCS ${poppler_SRCS}
    poppler/SplashOutputDev.cc
    splash/Splash.cc
    splash/SplashBitmap.cc
    splash/SplashClip.cc
    splash/SplashFTFont.cc
    splash/SplashFTFontEngine.cc
    splash/SplashFTFontFile.cc
    splash/SplashFont.cc
    splash/SplashFontEngine.cc
    splash/SplashFontFile.cc
    splash/SplashFontFileID.cc
    splash/SplashPath.cc
    splash/SplashPattern.cc
    splash/SplashScreen.cc
    splash/SplashState.cc
    splash/SplashXPath.cc
    splash/SplashXPathScanner.cc
  )
endif()
if(FONTCONFIG_FOUND)
  set(poppler_LIBS ${poppler_LIBS} ${FONTCONFIG_LIBRARIES})
endif()

if(JPEG_FOUND)
  set(poppler_SRCS ${poppler_SRCS}
    poppler/DCTStream.cc
  )
  set(poppler_LIBS ${poppler_LIBS} ${JPEG_LIBRARIES})
endif()
if(ENABLE_ZLIB)
  set(poppler_SRCS ${poppler_SRCS}
    poppler/FlateEncoder.cc
  )
  set(poppler_LIBS ${poppler_LIBS} ${ZLIB_LIBRARIES})
endif()
if(ENABLE_ZLIB_UNCOMPRESS)
  set(poppler_SRCS ${poppler_SRCS}
    poppler/FlateStream.cc
  )
endif()
if(ENABLE_LIBCURL)
  set(poppler_SRCS ${poppler_SRCS}
    poppler/CurlCachedFile.cc
    poppler/CurlPDFDocBuilder.cc
  )
  set(poppler_LIBS ${poppler_LIBS} ${CURL_LIBRARIES})
endif()
if (ENABLE_NSS3)
  set(poppler_SRCS ${poppler_SRCS}
    poppler/SignatureHandler.cc
  )
  if(${CMAKE_VERSION} VERSION_LESS "3.6.0")
    set(poppler_LIBS ${poppler_LIBS} ${NSS3_LIBRARIES})
  else()
    set(poppler_LIBS ${poppler_LIBS} PkgConfig::NSS3)
  endif()
endif()
if (OpenJPEG_FOUND)
  set(poppler_SRCS ${poppler_SRCS}
    poppler/JPEG2000Stream.cc
  )
  set(poppler_LIBS ${poppler_LIBS} openjp2)
else ()
  set(poppler_SRCS ${poppler_SRCS}
    poppler/JPXStream.cc
  )
endif()
if(USE_CMS)
  set(poppler_LIBS ${poppler_LIBS} ${LCMS2_LIBRARIES})
endif()
if(WIN32)
  # use clean APIs
  add_definitions(-DWIN32_LEAN_AND_MEAN)
  # gdi32 is needed under win32
  set(poppler_LIBS ${poppler_LIBS} gdi32)
endif()
if(PNG_FOUND)
  set(poppler_LIBS ${poppler_LIBS} ${PNG_LIBRARIES})
endif()
if(TIFF_FOUND)
  set(poppler_LIBS ${poppler_LIBS} ${TIFF_LIBRARIES})
endif()

if(MSVC)
add_definitions(-D_CRT_SECURE_NO_WARNINGS)
add_library(poppler STATIC ${poppler_SRCS})
else()
add_library(poppler ${poppler_SRCS})
endif()
set_target_properties(poppler PROPERTIES VERSION 90.0.0 SOVERSION 90)
if(MINGW AND BUILD_SHARED_LIBS)
    get_target_property(POPPLER_SOVERSION poppler SOVERSION)
    set_target_properties(poppler PROPERTIES SUFFIX "-${POPPLER_SOVERSION}${CMAKE_SHARED_LIBRARY_SUFFIX}")
endif()
target_link_libraries(poppler LINK_PRIVATE ${poppler_LIBS})
if(CMAKE_USE_PTHREADS_INIT)
   target_link_libraries(poppler LINK_PRIVATE Threads::Threads)
endif()
install(TARGETS poppler RUNTIME DESTINATION bin LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

if(ENABLE_UNSTABLE_API_ABI_HEADERS)
  install(FILES
    poppler/Annot.h
    poppler/Array.h
    poppler/BuiltinFont.h
    poppler/BuiltinFontTables.h
    poppler/CachedFile.h
    poppler/Catalog.h
    poppler/CharCodeToUnicode.h
    poppler/CMap.h
    poppler/DateInfo.h
    poppler/Decrypt.h
    poppler/Dict.h
    poppler/Error.h
    poppler/FileSpec.h
    poppler/FontEncodingTables.h
    poppler/FontInfo.h
    poppler/Form.h
    poppler/Function.h
    poppler/Gfx.h
    poppler/GfxFont.h
    poppler/GfxState.h
    poppler/GfxState_helpers.h
    poppler/GlobalParams.h
    poppler/Hints.h
    poppler/JArithmeticDecoder.h
    poppler/JBIG2Stream.h
    poppler/Lexer.h
    poppler/Link.h
    poppler/Linearization.h
    poppler/LocalPDFDocBuilder.h
    poppler/MarkedContentOutputDev.h
    poppler/Movie.h
    poppler/NameToCharCode.h
    poppler/Object.h
    poppler/OptionalContent.h
    poppler/Outline.h
    poppler/OutputDev.h
    poppler/Page.h
    poppler/PageTransition.h
    poppler/Parser.h
    poppler/PDFDoc.h
    poppler/PDFDocBuilder.h
    poppler/PDFDocEncoding.h
    poppler/PDFDocFactory.h
    poppler/PopplerCache.h
    poppler/ProfileData.h
    poppler/PreScanOutputDev.h
    poppler/PSTokenizer.h
    poppler/Rendition.h
    poppler/CertificateInfo.h
    poppler/Stream-CCITT.h
    poppler/Stream.h
    poppler/StructElement.h
    poppler/StructTreeRoot.h
    poppler/UnicodeMap.h
    poppler/UnicodeMapFuncs.h
    poppler/UnicodeMapTables.h
    poppler/UnicodeTypeTable.h
    poppler/UnicodeCClassTables.h
    poppler/UnicodeCompTables.h
    poppler/UnicodeDecompTables.h
    poppler/ViewerPreferences.h
    poppler/XRef.h
    poppler/CharTypes.h
    poppler/CompactFontTables.h
    poppler/ErrorCodes.h
    poppler/NameToUnicodeTable.h
    poppler/PSOutputDev.h
    poppler/TextOutputDev.h
    poppler/SecurityHandler.h
    poppler/StdinCachedFile.h
    poppler/StdinPDFDocBuilder.h
    poppler/UTF.h
    poppler/Sound.h
    ${CMAKE_CURRENT_BINARY_DIR}/poppler/poppler-config.h
    DESTINATION include/poppler)
  install(FILES
    goo/GooTimer.h
    goo/GooString.h
    goo/gmem.h
    goo/gdir.h
    goo/gfile.h
    goo/ImgWriter.h
    goo/GooCheckedOps.h
    goo/GooLikely.h
    goo/gstrtod.h
    goo/grandom.h
    DESTINATION include/poppler/goo)
  if(PNG_FOUND)
    install(FILES
      goo/PNGWriter.h
      DESTINATION include/poppler/goo)
  endif()
  if(TIFF_FOUND)
    install(FILES
      goo/TiffWriter.h
      DESTINATION include/poppler/goo)
  endif()
  if(JPEG_FOUND)
    install(FILES
      goo/JpegWriter.h
      DESTINATION include/poppler/goo)
  endif()
  install(FILES
    fofi/FoFiBase.h
    fofi/FoFiEncodings.h
    fofi/FoFiTrueType.h
    fofi/FoFiType1.h
    fofi/FoFiType1C.h
    fofi/FoFiIdentifier.h
    DESTINATION include/poppler/fofi)
  if(ENABLE_LIBCURL)
    install(FILES
      poppler/CurlCachedFile.h
      poppler/CurlPDFDocBuilder.h
      DESTINATION include/poppler)
  endif()
  if(OpenJPEG_FOUND)
    install(FILES
      poppler/JPEG2000Stream.h
      DESTINATION include/poppler)
  else()
    install(FILES
      poppler/JPXStream.h
      DESTINATION include/poppler)
  endif()
  if(ENABLE_SPLASH)
    install(FILES
      poppler/SplashOutputDev.h
      DESTINATION include/poppler)
    install(FILES
      splash/Splash.h
      splash/SplashBitmap.h
      splash/SplashClip.h
      splash/SplashErrorCodes.h
      splash/SplashFTFont.h
      splash/SplashFTFontEngine.h
      splash/SplashFTFontFile.h
      splash/SplashFont.h
      splash/SplashFontEngine.h
      splash/SplashFontFile.h
      splash/SplashFontFileID.h
      splash/SplashGlyphBitmap.h
      splash/SplashMath.h
      splash/SplashPath.h
      splash/SplashPattern.h
      splash/SplashScreen.h
      splash/SplashState.h
      splash/SplashTypes.h
      splash/SplashXPath.h
      splash/SplashXPathScanner.h
      DESTINATION include/poppler/splash)
  endif()
endif()


if(ENABLE_UTILS)
  add_subdirectory(utils)
endif()
if(ENABLE_GLIB)
  add_subdirectory(glib)
endif()
add_subdirectory(test)
if(ENABLE_QT5)
  add_subdirectory(qt5)
endif()
if(ENABLE_CPP)
  add_subdirectory(cpp)
endif()

set(PKG_CONFIG_VERSION_0_18 TRUE)
if(PKG_CONFIG_EXECUTABLE)
  exec_program(${PKG_CONFIG_EXECUTABLE} ARGS --version RETURN_VALUE _return_VALUE OUTPUT_VARIABLE _output_VAR)
  macro_ensure_version("0.18" "${_output_VAR}" PKG_CONFIG_VERSION_0_18)
endif()
if(PKG_CONFIG_VERSION_0_18)
  set(PC_REQUIRES "")
  set(PC_REQUIRES_PRIVATE "Requires.private: poppler = ${POPPLER_VERSION}")
else()
  set(PC_REQUIRES "poppler = ${POPPLER_VERSION}")
  set(PC_REQUIRES_PRIVATE "")
endif()

if(PKG_CONFIG_EXECUTABLE)
  poppler_create_install_pkgconfig(poppler.pc ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
  if(ENABLE_SPLASH)
    poppler_create_install_pkgconfig(poppler-splash.pc ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
  endif()
  if(ENABLE_QT5)
    poppler_create_install_pkgconfig(poppler-qt5.pc ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
  endif()
  if(ENABLE_GLIB)
    poppler_create_install_pkgconfig(poppler-glib.pc ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
  endif()
  if(CAIRO_FOUND)
    poppler_create_install_pkgconfig(poppler-cairo.pc ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
  endif()
  if(ENABLE_CPP)
    poppler_create_install_pkgconfig(poppler-cpp.pc ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
  endif()
else()
  MESSAGE(STATUS ".pc files will not be installed becasue of missing 'pkg-config'!")
endif()

message("Building Poppler with support for:")
show_end_message("font configuration" ${font_configuration})
show_end_message_yesno("splash output" ENABLE_SPLASH)
show_end_message_yesno("cairo output" CAIRO_FOUND)
show_end_message_yesno("qt5 wrapper" ENABLE_QT5)
show_end_message_yesno("glib wrapper" ENABLE_GLIB)
show_end_message_yesno("  introspection" INTROSPECTION_FOUND)
show_end_message_yesno("  gtk-doc" ENABLE_GTK_DOC)
show_end_message_yesno("cpp wrapper" ENABLE_CPP)
show_end_message_yesno("use libjpeg" ENABLE_LIBJPEG)
show_end_message_yesno("use libpng" ENABLE_LIBPNG)
show_end_message_yesno("use libtiff" ENABLE_LIBTIFF)
show_end_message_yesno("use zlib compress" ENABLE_ZLIB)
show_end_message_yesno("use zlib uncompress" ENABLE_ZLIB_UNCOMPRESS)
show_end_message_yesno("use nss3" ENABLE_NSS3)
show_end_message_yesno("use curl" ENABLE_LIBCURL)
show_end_message_yesno("use libopenjpeg2" WITH_OPENJPEG)
show_end_message_yesno("use lcms2" USE_CMS)
show_end_message_yesno("use boost" Boost_FOUND)
show_end_message_yesno("command line utils" ENABLE_UTILS)
show_end_message_yesno("fuzz target" ENABLE_FUZZER)
show_end_message("test data dir" ${TESTDATADIR})

if(NOT ENABLE_SPLASH AND NOT CAIRO_FOUND)
  message("Warning: There is no rendering backend enabled")
endif()

if(NOT ENABLE_LIBJPEG AND HAVE_DCT_DECODER)
  message("Warning: Using libjpeg is recommended. The internal DCT decoder is unmaintained.")
endif()

if(NOT HAVE_DCT_DECODER)
  message("Warning: You're not compiling any DCT decoder. Some files will fail to display properly.")
endif()

if(ENABLE_ZLIB_UNCOMPRESS)
  message("Warning: Using zlib is not totally safe")
endif()

if(NOT WITH_OPENJPEG AND HAVE_JPX_DECODER)
  message("Warning: Using libopenjpeg2 is recommended. The internal JPX decoder is unmaintained.")
endif()

if(NOT HAVE_JPX_DECODER)
  message("Warning: You're not compiling any JPX decoder. Some files will fail to display properly.")
endif()

if(ENABLE_SPLASH AND NOT Boost_FOUND)
  message("Warning: Use of boost is recommended for better performance.")
endif()

set(ARCHIVE_NAME ${CMAKE_PROJECT_NAME}-${POPPLER_VERSION})
add_custom_target(dist
    COMMAND
        COMMAND git log --stat | fmt --split-only > ${CMAKE_BINARY_DIR}/ChangeLog
        COMMAND git archive --prefix=${ARCHIVE_NAME}/ HEAD > ${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar
        COMMAND tar -C ${CMAKE_BINARY_DIR} -rf ${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar ChangeLog --transform='s,,${ARCHIVE_NAME}/,' --owner root:0 --group root:0
        COMMAND tar -C ${CMAKE_BINARY_DIR} -rf ${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar glib/reference/html --transform='s,,${ARCHIVE_NAME}/,' --owner root:0 --group root:0
        COMMAND xz -9 ${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
