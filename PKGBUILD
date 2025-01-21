pkgname=strongbox-tpm2-git
pkgver=r32.8e795f9
pkgrel=1

source=("git+https://github.com/kkernick/strongbox.git")
sha256sums=("SKIP")
depends=(zfs-utils)
optdepends=(handle-tpm)
arch=(x86_64)
provides=(strongbox-tpm2)
conflicts=(strongbox-tpm2)

pkgver() {
    cd $srcdir/strongbox
    printf "r%s.%s" "$(git rev-list --count HEAD)" "$(git rev-parse --short=7 HEAD)"
}

package() {
    cd $srcdir/strongbox

    cd bin
    for binary in *; do
        install -Dm755 "$binary" "$pkgdir/usr/bin/$binary"
    done

    cd ../hooks
    for hook in *; do
        install -Dm644 "$hook" "$pkgdir/usr/lib/initcpio/install/$hook"
    done

    cd ../services
    for service in *; do
        install -Dm644 "$service" "$pkgdir/usr/lib/systemd/system/$service"
    done

    echo "
Add:
    auth       required		       pam_exec.so 	    expose_authtok  /usr/bin/home-decrypt
to /etc/pam.d/system-auth for login with home-decrypt
"
}

