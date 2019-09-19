project(
    'minimap2',
    ['c'],
    default_options: [
        'b_ndebug=if-release',
        'buildtype=release',
        'c_std=gnu23',
        'warning_level=3',
    ],
    license: 'MIT',
    meson_version: '>= 1.1',
    version: '2.31',
)

##########
# CFLAGS #
##########

c = meson.get_compiler('c')

add_project_arguments(
    '-DHAVE_KALLOC',
    language: 'c',
)

add_project_arguments(
    c.get_supported_arguments(
        '-Wc++-compat',
        '-Wno-sign-compare',
        '-Wno-unused-but-set-variable',
        '-Wno-unused-parameter',
        '-Wno-unused-variable',
        '-fsigned-char',
    ),
    language: 'c',
)

############
# includes #
############

inc_dir_labels = ['.']
if host_machine.cpu_family() == 'aarch64'
    inc_dir_labels += 'sse2neon'
endif

inc_dir = [include_directories(inc_dir_labels)]

################
# dependencies #
################

external_deps = [
    c.find_library('m'),
    dependency('threads'),
    dependency('zlib', include_type: 'system'),
]

deps = external_deps

###########
# headers #
###########

if not meson.is_subproject()
    install_headers(
        files(
            'bseq.h',
            'kalloc.h',
            'kdq.h',
            'ketopt.h',
            'khash.h',
            'kseq.h',
            'ksort.h',
            'ksw2.h',
            'kthread.h',
            'kvec.h',
            'minimap.h',
            'mmpriv.h',
            'sdust.h',
        ),
        subdir: meson.project_name(),
    )
endif

#############
# libraries #
#############

simd_mod = import('unstable-simd')
simd_flag = []
simd_libs = []

simd_libs += simd_mod.check(
    'll_sse',
    compiler: c,
    include_directories: inc_dir,
    sse2: files('ksw2_ll_sse.c'),
)[0]

if get_option('sse41')
    # SSE 4.1
    simd_flag += '-DKSW_CPU_DISPATCH'

    simd_libs += simd_mod.check(
        'ksw2_sse41',
        c_args: simd_flag,
        compiler: c,
        include_directories: inc_dir,
        sse41: files(
            'ksw2_extd2_sse.c',
            'ksw2_exts2_sse.c',
            'ksw2_extz2_sse.c',
        ),
    )[0]

    simd_libs += simd_mod.check(
        'ksw2_dispatch',
        c_args: simd_flag,
        compiler: c,
        include_directories: inc_dir,
        sse41: files('ksw2_dispatch.c'),
    )[0]

    simd_flag += '-DKSW_SSE2_ONLY'
endif

# SSE2
simd_libs += simd_mod.check(
    'ksw2_sse2',
    c_args: simd_flag,
    compiler: c,
    include_directories: inc_dir,
    sse2: files(
        'ksw2_extd2_sse.c',
        'ksw2_exts2_sse.c',
        'ksw2_extz2_sse.c',
    ),
)[0]

# Unify all compiled objects into one library to avoid
# multiple libraries in the pkg-config file.
# https://github.com/mesonbuild/meson/issues/2478#issuecomment-351132872
simd_all_objects = []
foreach i : simd_libs
    simd_all_objects += i.extract_all_objects(recursive: true)
endforeach

lib_install = (not meson.is_subproject()) or (
    get_option('default_library') != 'static'
)

lib_sources = files(
    'align.c',
    'bseq.c',
    'esterr.c',
    'format.c',
    'hit.c',
    'index.c',
    'jump.c',
    'kalloc.c',
    'kthread.c',
    'lchain.c',
    'map.c',
    'misc.c',
    'options.c',
    'pe.c',
    'sdust.c',
    'seed.c',
    'sketch.c',
    'splitidx.c',
)

# aarch64: compile SIMD sources directly and add NEON-compatibility defines
aarch64_flags = []
if host_machine.cpu_family() == 'aarch64'
    lib_sources += files(
        'ksw2_extd2_sse.c',
        'ksw2_exts2_sse.c',
        'ksw2_extz2_sse.c',
        'ksw2_ll_sse.c',
    )

    # These flags are intended for only the ksw2_ext* items above.
    # Meson won't detect the m1 macbook as neon-capable in the simd
    # module, so the flags are global here instead.
    aarch64_flags = ['-DKSW_SSE2_ONLY', '-D__SSE2__']
endif

lib = library(
    meson.project_name(),
    lib_sources,
    c_args: aarch64_flags,
    dependencies: deps,
    include_directories: inc_dir,
    install: lib_install,
    objects: simd_all_objects,
    soversion: meson.project_version(),
    version: meson.project_version(),
)

###############
# executables #
###############

exe = executable(
    meson.project_name(),
    files('main.c'),
    dependencies: deps,
    include_directories: inc_dir,
    install: not meson.is_subproject(),
    link_with: lib,
)

#############
# pkgconfig #
#############

if not meson.is_subproject()
    import('pkgconfig').generate(
        lib,
        subdirs: meson.project_name(),
    )
endif

###################
# dependency info #
###################

meson.override_dependency(
    meson.project_name(),
    declare_dependency(
        dependencies: deps,
        include_directories: inc_dir,
        link_with: lib,
        version: meson.project_version(),
    ),
)
