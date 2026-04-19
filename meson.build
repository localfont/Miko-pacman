name: Build Miko Package Manager

on:
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install Build Dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            build-essential meson ninja-build \
            libarchive-dev libcurl4-openssl-dev \
            libssl-dev pkg-config

      - name: Check meson.build exists
        run: |
          ls -la
          cat meson.build | head -5

      - name: Configure
        run: |
          meson setup build \
            --prefix=/usr \
            --sysconfdir=/etc \
            --localstatedir=/var \
            -Ddoc=disabled \
            -Dgpgme=disabled \
            -Dscriptlet-shell=/bin/sh

      - name: Build
        run: ninja -C build

      - name: Show Built Binaries
        run: find build -type f -executable | sort

      - name: Package
        run: |
          mkdir -p dist/usr/bin dist/etc
          MIKO_BIN=$(find build -type f -executable -name "miko" | head -1)
          echo "Binary: $MIKO_BIN"
          cp "$MIKO_BIN" dist/usr/bin/miko

      - name: Upload Artifact
        uses: actions/upload-artifact@v4
        with:
          name: miko-${{ github.sha }}
          path: dist/18n = import('i18n')
  subdir('lib/libalpm/po')
  subdir('src/miko/po')
  subdir('scripts/po')
endif

want_doc = get_option('doc')
ASCIIDOC = find_program('asciidoc', required : want_doc)
A2X = find_program('a2x', required : want_doc)
build_doc = A2X.found() and not want_doc.disabled()
if build_doc
  subdir('doc')
endif

includes = include_directories('src/common', 'lib/libalpm')

libcommon = static_library(
  'common',
  libcommon_sources,
  include_directories : includes,
  gnu_symbol_visibility : 'hidden',
  install : false)

alpm_deps = [crypto_provider, libarchive, libcurl, libintl, libseccomp, gpgme]

libalpm_a = static_library(
  'alpm_objlib',
  libalpm_sources,
  # https://github.com/mesonbuild/meson/issues/3937
  objects : libcommon.extract_all_objects(recursive: true),
  include_directories : includes,
  gnu_symbol_visibility : 'hidden',
  dependencies : alpm_deps)

libalpm = library(
  'alpm',
  version : libalpm_version,
  objects: libalpm_a.extract_all_objects(recursive: true),
  include_directories : includes,
  dependencies : alpm_deps,
  install : true)

install_headers(
  'lib/libalpm/alpm.h',
  'lib/libalpm/alpm_list.h')

pkgconfig = import('pkgconfig')
pkgconfig.generate(
  libalpm,
  name : 'libalpm',
  description : 'Arch Linux package management library',
  version : libalpm_version,
  url : 'http://archlinux.org/miko/',
  extra_cflags: [ '-D_FILE_OFFSET_BITS=64' ],
)

miko_bin = executable(
  'miko',
  miko_sources,
  include_directories : includes,
  link_with : [libalpm, libcommon],
  dependencies : [libarchive],
  install : true,
)

executable(
  'miko-conf',
  miko_conf_sources,
  include_directories : includes,
  link_with : [libalpm, libcommon],
  dependencies : [libarchive],
  install : true,
)

executable(
  'testpkg',
  testpkg_sources,
  include_directories : includes,
  link_with : [libalpm],
  dependencies : [libarchive],
  install : true,
)

executable(
  'vercmp',
  vercmp_sources,
  include_directories : includes,
  link_with : [libalpm_a],
  install : true,
)

foreach wrapper : script_wrappers
  cdata = configuration_data()
  cdata.set_quoted('BASH', BASH.full_path())
  cdata.set_quoted('BUILDDIR', wrapper[2])
  cdata.set_quoted('REAL_PROGPATH', wrapper[1].full_path())

  # Create a wrapper script that bootstraps the real script within the build
  # directory. Use configure_file instead of a custom_target to ensure that
  # permissions on the input script wrapper are preserved.
  configure_file(
    input : join_paths(meson.project_source_root(), 'build-aux', 'script-wrapper.sh.in'),
    output : wrapper[0],
    configuration : cdata)
endforeach

configure_file(
  input : 'etc/makepkg.conf.in',
  output : 'makepkg.conf',
  configuration : substs,
  install_dir : SYSCONFDIR)

configure_file(
  input : 'etc/makepkg.conf.d/fortran.conf.in',
  output : 'fortran.conf',
  configuration : substs,
  install_dir : join_paths(SYSCONFDIR, 'makepkg.conf.d/'))

configure_file(
  input : 'etc/makepkg.conf.d/rust.conf.in',
  output : 'rust.conf',
  configuration : substs,
  install_dir : join_paths(SYSCONFDIR, 'makepkg.conf.d/'))

install_emptydir(join_paths(SYSCONFDIR, 'makepkg.d/'))

configure_file(
  input : 'etc/miko.conf.in',
  output : 'miko.conf',
  configuration : substs,
  install_dir : SYSCONFDIR)

install_data(
  'proto/PKGBUILD-split.proto',
  'proto/PKGBUILD-vcs.proto',
  'proto/PKGBUILD.proto',
  'proto/proto.install',
  install_dir : join_paths(DATAROOTDIR, 'miko'))

foreach path : [
	join_paths(LOCALSTATEDIR, 'lib/miko/'),
	join_paths(LOCALSTATEDIR, 'cache/miko/pkg/'),
	join_paths(DATAROOTDIR, 'makepkg-template/'),
	join_paths(DATAROOTDIR, 'libalpm/hooks/'),
	]
	meson.add_install_script('sh', '-c', 'mkdir -p "$DESTDIR/@0@"'.format(path))
endforeach

TEST_ENV = environment()
TEST_ENV.set('PMTEST_SCRIPTLIB_DIR', join_paths(meson.project_source_root(), 'scripts/library/'))
TEST_ENV.set('PMTEST_LIBMAKEPKG_DIR', join_paths(meson.project_build_root(), 'scripts/libmakepkg/'))
TEST_ENV.set('PMTEST_UTIL_DIR', meson.project_build_root() + '/')
TEST_ENV.set('PMTEST_SCRIPT_DIR', join_paths(meson.project_build_root(), 'scripts/'))

subdir('test/miko')
subdir('test/scripts')
subdir('test/util')

summary({
  'prefix': PREFIX,
  'sysconfdir': SYSCONFDIR,
  '    conf file': join_paths(SYSCONFDIR, 'miko.conf'),
  'localstatedir': LOCALSTATEDIR,
  '    database dir': join_paths(LOCALSTATEDIR, 'lib/miko/'),
  '    cache dir': join_paths(LOCALSTATEDIR, 'cache/miko/pkg/'),
  'keyringdir': KEYRINGDIR,
  'compiler': '@0@ @1@\n'.format(cc.get_id(), cc.version()),
  'Architecture': carch,
  'Host Type': chost,
  'File inode command': inodecmd,
  'File seccomp command': filecmd,
  'libalpm version': libalpm_version,
  'miko version': PACKAGE_VERSION,
}, section: 'Build information')

summary({
  'root working directory': ROOTDIR,
  'package extension': get_option('pkg-ext'),
  'source pkg extension': get_option('src-ext'),
  'build script name': BUILDSCRIPT,
  'template directory': get_option('makepkg-template-dir'),
}, section: 'Directory and file information')

summary({
  'i18n support': get_option('i18n'),
  'Build docs': build_doc,
  'debug build': get_option('buildtype') == 'debug',
  'Use libcurl': conf.get('HAVE_LIBCURL'),
  'Use GPGME': conf.get('HAVE_LIBGPGME'),
  'Use OpenSSL': conf.has('HAVE_LIBSSL') and conf.get('HAVE_LIBSSL') == 1,
  'Use nettle': conf.has('HAVE_LIBNETTLE') and conf.get('HAVE_LIBNETTLE') == 1,
}, section: 'Compilation options')
