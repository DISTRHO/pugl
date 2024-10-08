# Copyright 2021-2022 David Robillard <d@drobilla.net>
# SPDX-License-Identifier: CC0-1.0 OR ISC

project('pugl', ['c'],
        version: '0.4.0',
        license: 'ISC',
        meson_version: '>= 0.49.2',
        default_options: [
          'c_std=c99',
          'cpp_std=c++14',
          'default_library=shared'
        ])

pugl_src_root = meson.current_source_dir()
major_version = meson.project_version().split('.')[0]
version_suffix = '-@0@'.format(major_version)
versioned_name = 'pugl' + version_suffix

# Load build tools
pkg = import('pkgconfig')
cc = meson.get_compiler('c')

# Enable C++ support if we're building the examples
if get_option('examples') or get_option('tests')
  if add_languages(['cpp'], required: false)
    cpp = meson.get_compiler('cpp')
  endif
endif

# Enable Objective C support if we're building for MacOS
if host_machine.system() == 'darwin'
  add_languages(['objc'])
  objcc = meson.get_compiler('objc')
endif

# Set ultra strict warnings for developers, if requested
# These are for the implementation, tests and examples add more suppressions
if get_option('strict')
  subdir('meson')

  c_warnings = all_c_warnings
  if cc.get_id() == 'clang'
    c_warnings += [
      '-Wno-bad-function-cast',
      '-Wno-documentation', # Cairo
      '-Wno-documentation-unknown-command', # Cairo
      '-Wno-padded',
      '-Wno-reserved-identifier', # Cairo, FD_ZERO
      '-Wno-switch-default',
      '-Wno-switch-enum',
    ]

    if host_machine.system() == 'darwin'
      c_warnings += [
        '-Wno-poison-system-directories',
      ]
    elif host_machine.system() == 'windows'
      c_warnings += [
        '-Wno-deprecated-declarations',
        '-Wno-format-nonliteral',
        '-Wno-nonportable-system-include-path',
        '-Wno-unused-macros',
      ]
    endif
  elif cc.get_id() == 'gcc'
    c_warnings += [
      '-Wno-bad-function-cast',
      '-Wno-float-equal',
      '-Wno-inline',
      '-Wno-padded',
      '-Wno-suggest-attribute=const',
      '-Wno-suggest-attribute=malloc',
      '-Wno-suggest-attribute=pure',
      '-Wno-switch-default',
      '-Wno-switch-enum',
      '-Wno-unsuffixed-float-constants',
    ]

    if host_machine.system() == 'windows'
      c_warnings += [
        '-Wno-suggest-attribute=format',
      ]
    endif
  elif cc.get_id() == 'msvc'
    c_warnings += [
      '/wd4061',  # enumerator in switch is not explicitly handled
      '/wd4191',  # unsafe conversion from type to type
      '/wd4514',  # unreferenced inline function has been removed
      '/wd4706',  # assignment within conditional expression
      '/wd4710',  # function not inlined
      '/wd4711',  # function selected for automatic inline expansion
      '/wd4820',  # padding added after construct
      '/wd4996',  # POSIX name for this item is deprecated
      '/wd5045',  # will insert Spectre mitigation for memory load
    ]
  endif

  add_project_arguments(cc.get_supported_arguments(c_warnings),
                        language: ['c', 'objc'])

  # Objective C warnings
  if is_variable('objcc')
    objc_warnings = all_objc_warnings

    objc_warnings += [
      '-Wno-deprecated-declarations',
      '-Wno-direct-ivar-access',
      '-Wno-pedantic',
    ]

    add_project_arguments(objcc.get_supported_arguments(objc_warnings),
                          language: ['objc'])
  endif
endif

# Disable deprecated API which is not used by tests or examples
add_project_arguments(['-DPUGL_DISABLE_DEPRECATED'],
                      language: ['c', 'cpp', 'objc'])

# System libraries
m_dep = cc.find_library('m', required: false)
dl_dep = cc.find_library('dl', required: false)
thread_dep = dependency('threads')

# Cairo (optional backend)
cairo_dep = dependency('cairo',
                       required: get_option('cairo'))

# OpenGL (optional backend)
opengl_dep = dependency('GL',
                        required: get_option('opengl'))

# Vulkan (optional backend)
vulkan_dep = dependency('vulkan',
                        required: get_option('vulkan'))

core_args = []

# MacOS
if host_machine.system() == 'darwin'
  cocoa_dep = dependency('Cocoa', required: false, modules: 'foundation')
  corevideo_dep = dependency('CoreVideo', required: false)

  platform = 'mac'
  platform_sources = files('src/mac.m')
  core_deps = [cocoa_dep, corevideo_dep]
  extension = '.m'

  add_project_arguments(['-DGL_SILENCE_DEPRECATION'],
                        language: ['c', 'objc'])

  add_project_link_arguments(['-Wl,-framework,Cocoa'],
                             language: ['c', 'objc'])

# Windows
elif host_machine.system() == 'windows'
  if cc.get_id() == 'msvc'
    msvc_args = [
      '/TP',
      '/experimental:external',
      '/external:W0',
      '/external:anglebrackets',
    ]

    add_project_arguments(msvc_args, language: ['c', 'cpp'])
  endif

  win_args = [
    '-DWIN32_LEAN_AND_MEAN',
    '-D_CRT_SECURE_NO_WARNINGS',
  ]

  add_project_arguments(win_args, language: ['c', 'cpp'])

  user32_dep = cc.find_library('user32')
  shlwapi_dep = cc.find_library('shlwapi')

  platform = 'win'
  platform_sources = files('src/win.c')
  core_deps = [user32_dep, shlwapi_dep]
  extension = '.c'

else # X11
  x11_dep = cc.find_library('X11')
  core_args += '-D_POSIX_C_SOURCE=200809L'

  xcursor_dep = cc.find_library('Xcursor', required: false)
  if xcursor_dep.found()
    core_args += ['-DHAVE_XCURSOR']
  endif

  xrandr_dep = cc.find_library('Xrandr', required: false)
  if xrandr_dep.found()
    core_args += ['-DHAVE_XRANDR']
  endif

  xext_dep = cc.find_library('Xext', required: false)
  if xext_dep.found()
    xsync_fragment = '''#include <X11/Xlib.h>
      #include <X11/extensions/sync.h>
      int main(void) { XSyncQueryExtension(0, 0, 0); return 0; }'''
    if cc.compiles(xsync_fragment, name: 'Xsync')
      core_args += ['-DHAVE_XSYNC']
    endif
  endif

  platform = 'x11'
  platform_sources = files('src/x11.c')
  core_deps = [x11_dep, xcursor_dep, xrandr_dep, xext_dep]
  extension = '.c'
endif

# Build core library

core_deps += [m_dep]
core_name = 'pugl_@0@@1@'.format(platform, version_suffix)

# Determine library type and set up defines for building them
library_args = ['-DPUGL_INTERNAL']
if get_option('default_library') == 'both'
  if host_machine.system() == 'windows'
    error('default_library=both is not supported on Windows')
  endif

  library_type = 'both_libraries'
elif get_option('default_library') == 'shared'
  library_type = 'shared_library'
else
  library_type = 'static_library'
  add_project_arguments(['-DPUGL_STATIC'], language: ['c', 'cpp', 'objc'])
endif

# Build core library
libpugl = build_target(
  core_name,
  files('src/implementation.c') + platform_sources,
  version: meson.project_version(),
  include_directories: include_directories(['include']),
  c_args: library_args + core_args,
  dependencies: core_deps,
  gnu_symbol_visibility: 'hidden',
  install: true,
  target_type: library_type)

pugl_dep = declare_dependency(link_with: libpugl, dependencies: core_deps)

# Build pkg-config file
pkg.generate(libpugl,
             name: 'Pugl',
             filebase: versioned_name,
             subdirs: [versioned_name],
             version: meson.project_version(),
             description: 'Pugl GUI library core')

# Build stub backend

name = 'pugl_' + platform + '_stub' + version_suffix
sources = files('src/' + platform + '_stub' + extension)

stub_backend = build_target(
  name, sources,
  version: meson.project_version(),
  include_directories: include_directories(['include']),
  c_args: library_args,
  dependencies: [pugl_dep],
  gnu_symbol_visibility: 'hidden',
  install: true,
  target_type: library_type)

stub_backend_dep = declare_dependency(link_with: stub_backend)

pkg.generate(stub_backend,
             name: 'Pugl Stub',
             filebase: 'pugl-stub-@0@'.format(major_version),
             subdirs: [versioned_name],
             version: meson.project_version(),
             description: 'Native window pugl graphics backend')

# Build GL backend
if opengl_dep.found()
  name = 'pugl_' + platform + '_gl' + version_suffix
  sources = files('src/' + platform + '_gl' + extension)

  gl_backend = build_target(
    name, sources,
    version: meson.project_version(),
    include_directories: include_directories(['include']),
    c_args: library_args,
    dependencies: [pugl_dep, opengl_dep],
    gnu_symbol_visibility: 'hidden',
    install: true,
    target_type: library_type)

  gl_backend_dep = declare_dependency(link_with: gl_backend,
                                      dependencies: [pugl_dep, opengl_dep])

  pkg.generate(gl_backend,
               name: 'Pugl OpenGL',
               filebase: 'pugl-gl-@0@'.format(major_version),
               subdirs: [versioned_name],
               version: meson.project_version(),
               description: 'Pugl GUI library with OpenGL backend')
endif

# Build Cairo backend
if cairo_dep.found()
  name = 'pugl_' + platform + '_cairo' + version_suffix
  sources = files('src/' + platform + '_cairo' + extension)

  cairo_backend = build_target(
    name, sources,
    version: meson.project_version(),
    include_directories: include_directories(['include']),
    c_args: library_args,
    dependencies: [pugl_dep, cairo_dep],
    gnu_symbol_visibility: 'hidden',
    install: true,
    target_type: library_type)

  cairo_backend_dep = declare_dependency(
    link_with: cairo_backend,
    dependencies: [pugl_dep, cairo_dep])

  pkg.generate(cairo_backend,
               name: 'Pugl Cairo',
               filebase: 'pugl-cairo-@0@'.format(major_version),
               subdirs: [versioned_name],
               version: meson.project_version(),
               description: 'Pugl GUI library with Cairo backend')
endif

# Build Vulkan backend
if vulkan_dep.found()
  name = 'pugl_' + platform + '_vulkan' + version_suffix
  sources = files('src/' + platform + '_vulkan' + extension)

  vulkan_deps = [pugl_dep, vulkan_dep, dl_dep]
  vulkan_c_args = library_args
  vulkan_link_args = []
  if platform == 'mac'
    metal_dep = dependency('Metal', modules: 'foundation')
    quartzcore_dep = dependency('QuartzCore', modules: 'foundation')

    vulkan_deps += [metal_dep, quartzcore_dep]
  endif

  vulkan_backend = build_target(
    name, sources,
    version: meson.project_version(),
    include_directories: include_directories(['include']),
    c_args: library_args,
    dependencies: vulkan_deps,
    gnu_symbol_visibility: 'hidden',
    install: true,
    target_type: library_type)

  vulkan_backend_dep = declare_dependency(
    link_with: vulkan_backend,
    dependencies: [pugl_dep, vulkan_dep, thread_dep])

  pkg.generate(vulkan_backend,
               name: 'Pugl Vulkan',
               filebase: 'pugl-vulkan-@0@'.format(major_version),
               subdirs: [versioned_name],
               version: meson.project_version(),
               description: 'Pugl GUI library with Vulkan backend')
endif

# Process all headers first so they are available for building documentation
subdir('include')
subdir('bindings/cpp')

if not get_option('docs').disabled()
  subdir('doc')
else
  build_docs = false
endif

if get_option('examples')
  subdir('examples')
endif

if get_option('tests')
  subdir('test')
endif

if meson.version().version_compare('>=0.53.0')
  summary('Platform', platform)
  summary('Cairo backend', cairo_dep.found(), bool_yn: true)
  summary('OpenGL backend', opengl_dep.found(), bool_yn: true)
  summary('Vulkan backend', vulkan_dep.found(), bool_yn: true)
  summary('Tests', get_option('tests'), bool_yn: true)
  summary('Examples', get_option('examples'), bool_yn: true)
  summary('Documentation', build_docs, bool_yn: true)

  summary('Install prefix', get_option('prefix'))
  summary('Headers', get_option('prefix') / get_option('includedir'))
  summary('Libraries', get_option('prefix') / get_option('libdir'))

  if get_option('examples')
    summary('Executables', get_option('prefix') / get_option('bindir'))
  endif
endif
