pkgname=curfew
pkgver=0.2.1
pkgrel=1
pkgdesc="Distraction blocker and bedtime screen lock that is hard to undo on impulse"
arch=('any')
url="https://github.com/YOURNAME/curfew"
license=('MIT')
depends=('python' 'pyside6' 'python-jeepney' 'qt6-declarative' 'systemd' 'hicolor-icon-theme')
optdepends=('desktop-file-utils: refresh the application menu when apps are hidden'
            'breeze-icons: application icons in the app list on KDE')
makedepends=()
backup=()
install="${pkgname}.install"
source=()
sha256sums=()

prepare() {
    # Built from the working tree next to this PKGBUILD.
    rm -rf "$srcdir/tree"
    mkdir -p "$srcdir/tree"
    cp -a "$startdir/curfew" "$startdir/bin" "$startdir/data" "$srcdir/tree/"
    for f in README.md LICENSE; do
        [ -f "$startdir/$f" ] && cp -a "$startdir/$f" "$srcdir/tree/"
    done
    find "$srcdir/tree" -name '__pycache__' -prune -exec rm -rf {} + 2>/dev/null || true
}

check() {
    cd "$startdir"
    python tests/test_core.py
    # Interface regression tests; they render offscreen and need no display.
    QT_QPA_PLATFORM=offscreen python tests/test_timefield.py
}

package() {
    cd "$srcdir/tree"
    local site
    site="$(python -c 'import sysconfig; print(sysconfig.get_path("purelib"))')"

    # python package
    install -d "$pkgdir$site"
    cp -a curfew "$pkgdir$site/"
    find "$pkgdir$site/curfew" -type d -exec chmod 755 {} +
    find "$pkgdir$site/curfew" -type f -exec chmod 644 {} +

    # executables
    install -Dm755 bin/curfewd  "$pkgdir/usr/bin/curfewd"
    install -Dm755 bin/curfew   "$pkgdir/usr/bin/curfew"
    install -Dm755 bin/curfewctl "$pkgdir/usr/bin/curfewctl"

    # the root enforcement service
    install -Dm644 data/systemd/curfewd.service \
        "$pkgdir/usr/lib/systemd/system/curfewd.service"

    # user timer that refreshes subscribed blocklists; it runs as the user
    # because curfewd is not allowed to make network requests
    install -Dm644 data/systemd/curfew-lists.service \
        "$pkgdir/usr/lib/systemd/user/curfew-lists.service"
    install -Dm644 data/systemd/curfew-lists.timer \
        "$pkgdir/usr/lib/systemd/user/curfew-lists.timer"

    # desktop integration
    install -Dm644 data/applications/org.curfew.Curfew.desktop \
        "$pkgdir/usr/share/applications/org.curfew.Curfew.desktop"
    install -Dm644 data/applications/org.curfew.Curfew.autostart.desktop \
        "$pkgdir/etc/xdg/autostart/org.curfew.Curfew.desktop"
    install -Dm644 data/icons/curfew.svg \
        "$pkgdir/usr/share/icons/hicolor/scalable/apps/curfew.svg"

    # docs
    [ -f README.md ] && install -Dm644 README.md "$pkgdir/usr/share/doc/$pkgname/README.md"
    [ -f LICENSE ]   && install -Dm644 LICENSE   "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
    return 0
}
