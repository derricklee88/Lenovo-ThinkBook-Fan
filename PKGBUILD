# Maintainer: Derrick <derrick@localhost>
pkgname=thinkbook-fan-modes
pkgver=0.5.0
pkgrel=1
pkgdesc="Thermal mode GUI and fan RPM readout for the Lenovo ThinkBook 15p IMH (20V3) via ACPI WMAA"
arch=('any')
license=('MIT')
depends=('python' 'python-pyqt6' 'qt6-svg' 'acpi_call-dkms' 'polkit')
backup=('etc/thinkbook-fan-modes.conf')
options=(!strip !debug)

# Local-only package: the sources are the files next to this PKGBUILD.
_srcdir="$startdir"

check() {
	cd "$_srcdir"
	python3 -m unittest discover -s tests
}

package() {
	install -d "$pkgdir/usr/lib/thinkbook-fan/thinkbook_fan"
	for source in "$_srcdir"/thinkbook_fan/*.py; do
		install -Dm644 "$source" \
			"$pkgdir/usr/lib/thinkbook-fan/thinkbook_fan/$(basename "$source")"
	done

	install -Dm755 "$_srcdir/packaging/thinkbook-fan" \
		"$pkgdir/usr/bin/thinkbook-fan"
	install -Dm755 "$_srcdir/packaging/thinkbook-fan-gui" \
		"$pkgdir/usr/bin/thinkbook-fan-gui"
	# Root-only: pkexec runs this for the GUI, and pkexec authenticates. A
	# user-executable copy would bypass the polkit prompt entirely.
	install -Dm700 "$_srcdir/packaging/thinkbook-fan-helper" \
		"$pkgdir/usr/lib/thinkbook-fan/thinkbook-fan-helper"
	install -Dm644 "$_srcdir/packaging/thinkbook-fan.desktop" \
		"$pkgdir/usr/share/applications/thinkbook-fan.desktop"
	install -Dm644 "$_srcdir/packaging/thinkbook-fan.svg" \
		"$pkgdir/usr/share/icons/hicolor/scalable/apps/thinkbook-fan.svg"
	install -Dm644 "$_srcdir/packaging/org.thinkbookfan.helper.policy" \
		"$pkgdir/usr/share/polkit-1/actions/org.thinkbookfan.helper.policy"
	install -Dm644 "$_srcdir/packaging/thinkbook-fan-mode.service" \
		"$pkgdir/usr/lib/systemd/system/thinkbook-fan-mode.service"
	install -Dm644 "$_srcdir/packaging/thinkbook-fan-modes.conf" \
		"$pkgdir/etc/thinkbook-fan-modes.conf"
	# acpi_call-dkms already ships /usr/lib/modules-load.d/acpi_call.conf.
	install -Dm644 "$_srcdir/README.md" \
		"$pkgdir/usr/share/doc/$pkgname/README.md"
	install -Dm644 "$_srcdir/docs/superpowers/specs/2026-11-22-thinkbook-fan-modes-design.md" \
		"$pkgdir/usr/share/doc/$pkgname/design.md"
}
