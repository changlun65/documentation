Android Notes
=============

Porting
-------
[Android Porting On Real Target](https://wiki.kldp.org/wiki.php/AndroidPortingOnRealTarget)

[Android on OMAP](http://elinux.org/Android_on_OMAP)

Tools
-----
## Busybox symlink installation on Android

1. Compile busybox
    * Get an ARM cross-compile toolchain, e.g. Sourcery G++ Lite 2008q1-126 for ARM GNU/Linux
    * Get the busybox sources from the Busybox download page
    * make menuconfig and configure busybox as desired
    * Compile as given:
    ```
    LDFLAGS="--static" CFLAGS="--static" make CROSS_COMPILE=arm-none-linux-gnueabi-
    At the end, you should have something like:
    (hgschmidt@dai190)-(15:10:52)-| ~/trunk/bcollect/busybox-1.13.2
    |:. file busybox
    busybox: ELF 32-bit LSB executable, ARM, version 1 (SYSV), statically linked, stripped
    ```
    * Push busybox to Android

2. Install busybox
    * Modify target/product/generic/root/init.rc to add /data/bin and /data/sbin to the PATH. This allows you to add binaries as I go without having to rebuild the ramdisk.
    * Modify the install_links() function in libbb/appletlib.c from the busybox distro and rebuilt the binary.
    ```
    static const char usr_bin [] ALIGN1 = \"./bin\";
    static const char usr_sbin[] ALIGN1 = \"./sbin\";
    static const char *const install_dir[] = {
    &usr_bin [0], /* \"\", equivalent to \"/\" for concat_path_file() */
    &usr_bin [0], /* \"/bin\" */
    &usr_sbin[0], /* \"/sbin\" */
    usr_bin,
    usr_sbin
    };
    ```
    * Create /data/bin and /data/sbin on the target.
    * Push busybox into /data/bin.
    * Run 'busybox --install -s' from the /data directory on the target.
    * Remove the 'ls' symlink from /data/bin, because it seems to be outputting a bunch of garbage. I don't know why.

## Bash on Android for ARM

1. 自 http://www.gnu.org/software/bash/ 下載 bash-4.1.tar.gz。解開下載的 bash-4.1.tar.gz，然後選擇一個適當的 cross compiler，並執行 configure.
    ```
    CC=arm-none-linux-gnueabi-gcc ./configure --host=arm-linux \
     --target=arm-none-linux-gnueabi --enable-static-link \
     --enable-history --without-bash-malloc
    ```
2. 原則上執行完 configure 就可以執行 make 編譯了。若想得到靜態編譯的 bash 可執行檔，則再對 Makefile 稍作修改即可.
3. diff -Naur old/Makefile new/Makefile
```
--- old/Makefile       2010-08-15 15:34:58.489728811 +0000
+++ new/Makefile    2010-08-15 15:35:34.877136059 +0000
@@ -120,7 +120,7 @@
 # with gprof, or nothing (the default).
 PROFILE_FLAGS=
-CFLAGS = -g -O2
+CFLAGS = -g -O2 -static
CFLAGS_FOR_BUILD = -g -DCROSS_COMPILING
CPPFLAGS =
CPPFLAGS_FOR_BUILD =
```