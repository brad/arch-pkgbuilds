# Maintainer: Alfredo Palhares <alfredo at palhares dot me>
# Contributor: Mark Wagie <mark dot wagie at tutanota dot com>
# Contributor:  Matteo Parolari
# Contributor: gardar <aur@gardar.net>

# Please contribute to:
# https://github.com/alfredopalhares/arch-pkgbuilds

pkgbase="joplin"
pkgname=('joplin' 'joplin-desktop')
pkgver=2.6.10
groups=('joplin')
pkgrel=2
install="joplin.install"
depends=('electron' 'gtk3' 'libexif' 'libgsf' 'libjpeg-turbo' 'libwebp' 'libxss' 'nodejs'
         'nss' 'orc' 'rsync' )
optdepends=('libappindicator-gtk3: for tray icon')
arch=('x86_64' 'i686')
makedepends=('git' 'yarn' 'python2' 'rsync' 'jq' 'electron' 'libgsf' )
url="https://joplinapp.org/"
license=('MIT')
source=("joplin.desktop" "joplin-desktop.sh" "joplin.sh"
  "joplin-${pkgver}.tar.gz::https://github.com/laurent22/joplin/archive/v${pkgver}.tar.gz"
  "joplin-patches.tar.xz")
sha256sums=('c7c5d8b0ff9edb810ed901ea21352c9830bfa286f3c18b1292deca5b2f8febd2'
            'a450284fe66d89aa463d129ce8fff3a0a1a783a64209e4227ee47449d5737be8'
            'dc1236767ee055ea1d61f10e5266a23e70f3e611b405fe713ed24ca18ee9eeb5'
            '1994cf5a32cf72f60f0455ad8204ff1d5ebb70933ad50ade78431fa359b561c6'
            '43f86e589f20141d7a2478fd12b4aa2dc73c6a52d7037137515358f4cd65616f')

# local npm cache directory
_npm_cache="npm-cache"

_get_cache() {
  if [[ "${_npm_cache}" =~ ^/ ]]; then
    printf "%s" "${_npm_cache}"
  else
    printf "%s" "${srcdir}/${_npm_cache}"
  fi
}

prepare() {
  local cache=$(_get_cache)
  msg2 "npm cache directory: $cache"

  msg2 "Disabling husky (git hooks)"
  sed -i '/"husky": ".*"/d' "${srcdir}/joplin-${pkgver}/package.json"

  msg2 "Appliying patches..."
  cd "${srcdir}/joplin-${pkgver}"
  tar xvJf "${srcdir}/joplin-patches.tar.xz"
  patch -p1 < "${srcdir}/0005-All-Fixed-issue-where-synchroniser-would-try-to-upda.patch"
  patch -p1 < "${srcdir}/0007-Tools-Use-Yarn-3-to-manage-monorepo-5833.patch"

  msg2 "Tweaking lerna.json"
  local tmp_json="$(mktemp --tmpdir="$srcdir")"
  local lerna_json="${srcdir}/joplin-${pkgver}/lerna.json"
  # TODO: Add check for lerna.json file
  jq ".packages = [
        \"packages/app-cli\", \"packages/app-desktop\",
        \"packages/fork-htmlparser2\", \"packages/fork-sax\",
        \"packages/lib\", \"packages/renderer\", \"packages/tools\",
        \"packages/turndown\", \"packages/turndown-plugin-gfm\"
        ] |
      . += {\"npmClient\": \"yarn\", \"npmClientArgs\": [\"--cache $cache\", \"--no-optional\"]}" \
    "$lerna_json" > "$tmp_json"
  cat "$tmp_json" > "$lerna_json"
  rm "$tmp_json"
}


build() {
  local cache=$(_get_cache)
  msg2 "npm cache directory: $cache"
  cd "${srcdir}/joplin-${pkgver}"

  # Force Lang
  # INFO: https://github.com/alfredopalhares/joplin-pkgbuild/issues/25
  export LANG=en_US.utf8

  msg2 "Installing dependencies through Lerna"
  # FSevents is on the optinal dependencies and its Mac Only
  yarn install --cache "$cache" --no-optional

}

#FIXME: These checks fail on some machines, even with the exit 0
# Something related with the number of allowed processes I guess
check() {
  cd "${srcdir}/joplin-${pkgver}"
  msg2 "Not Running any tests for now"
  #npm run test || exit 0
}

package_joplin() {
  pkgdesc="A note taking and to-do application with synchronization capabilities - CLI App"
  depends=('coreutils' 'libsecret' 'nodejs' 'python')

  local cache=$(_get_cache)
  msg2 "npm cache directory: $cache"

  msg2 "Building CLI"
  cd "${srcdir}/joplin-${pkgver}/packages/app-cli"
  yarn run build

  msg2 "Packaging CLI"
  cd "${srcdir}/joplin-${pkgver}/packages/app-cli/build"
  local pack="$(npm pack | tail -n 1)"

  msg2 "Installing CLI ($pack)"
  npm install --global --production --user root --cache "$cache" \
    --prefix "${pkgdir}/tmp" "$pack"

  msg2 "Rearranging directory tree"
  mkdir -p "${pkgdir}/usr/share/"
  mv "${pkgdir}/tmp/lib/node_modules/joplin/" "${pkgdir}/usr/share/"
  rm -r "${pkgdir}/tmp"

  msg2 "Fixing Directories Permissions"
  # Non-deterministic race in npm gives 777 permissions to random directories.
  # See https://github.com/npm/cli/issues/1103 for details.
  find "${pkgdir}/usr" -type d -exec chmod 755 {} +

  msg2 "Removing References to \$pkgdir"
  find "$pkgdir" -name package.json -print0 | xargs -0 sed -i "/_where/d"

  msg2 "Removing References to \$srcdir"
  local tmppackage="$(mktemp --tmpdir="$srcdir")"
  local pkgjson="$pkgdir/usr/share/joplin/package.json" # TODO joplin name
  jq '.|=with_entries(select(.key|test("_.+")|not))' "$pkgjson" > "$tmppackage"
  mv "$tmppackage" "$pkgjson"
  chmod 644 "$pkgjson"

  msg2 "Fixing Permissions set by npm"
  # npm gives ownership of ALL FILES to build user
  # https://bugs.archlinux.org/task/63396
  chown -R root:root "${pkgdir}"

  msg2 "Installing LICENSE"
  install -Dm644 "${srcdir}/joplin-${pkgver}/LICENSE" -t "${pkgdir}/usr/share/licenses/${pkgname}/"

  msg2 "Installing Startup Script"
  cd "${srcdir}"
  install -Dm755 joplin.sh "${pkgdir}/usr/bin/joplin"
}


package_joplin-desktop() {
  pkgdesc="A note taking and to-do application with synchronization capabilities - Desktop"
  depends=('electron' 'gtk3' 'libexif' 'libgsf' 'libjpeg-turbo' 'libwebp' 'libxss' 'nodejs'
         'nss' 'orc')
  optdepends=('libappindicator-gtk3: for tray icon')
  conflicts=('joplin-desktop-electron')

  # ./generateSha512.js fails if AppImage is not built
  mkdir -p "${srcdir}/joplin-${pkgver}/packages/app-desktop/dist/"
  touch "${srcdir}/joplin-${pkgver}/packages/app-desktop/dist/AppImage"

  msg2 "Building Desktop with packaged Electron..."
  mkdir -p "${pkgdir}/usr/share/joplin-desktop"
  cd "${srcdir}/joplin-${pkgver}/packages/app-desktop"
  electron_dir="/usr/lib/electron"
  electron_version=$(cat /usr/lib/electron/version)
  msg2 "Using Electron Version ${electron_version}"
  # Current version of electron does not work
  USE_HARD_LINKS=false yarn run dist -- --publish=never  --linux  --x64 \
    --dir="dist/" -c.electronDist=$electron_dir -c.electronVersion=$electron_version
    #--dir="dist/"

  # TODO: Cleanup app.asar file
  cd dist/linux-unpacked/
  cp -R "." "${pkgdir}/usr/share/joplin-desktop"

  msg2 "Installing LICENSE..."
  cd "${srcdir}/joplin-${pkgver}/"
  install -Dm644 LICENSE -t "${pkgdir}/usr/share/licenses/${pkgname}"

  msg2 "Installing startup script and desktop file..."
  cd "${srcdir}"
  install -Dm755 ${srcdir}/joplin-desktop.sh "${pkgdir}/usr/bin/joplin-desktop"
  install -Dm644 ${srcdir}/joplin.desktop -t "${pkgdir}/usr/share/applications"

  msg2 "Installing icons"
  local -r src_icon_dir="${srcdir}/joplin-${pkgver}/packages/app-desktop/build/icons"
  local -i size
  for size in 16 22 24 32 36 48 64 72 96 128 192 256 512; do
    [[ -f "${src_icon_dir}/${size}x${size}.png" ]] &&
      install -Dm644 \
        "${src_icon_dir}/${size}x${size}.png" \
        "${pkgdir}/usr/share/icons/hicolor/${size}x${size}/apps/joplin.png"
  done
}

