#!/bin/sh

# Slackware build script for srb2

# Written by B. Watson (yalhcru@gmail.com). Updated to 2.2.6 by Lenon Jose (lenonbaby@gmail.com)

# Licensed under the WTFPL. See http://www.wtfpl.net/txt/copying/ for details.

PRGNAM=srb2
VERSION=${VERSION:-2.2.6}
BUILD=${BUILD:-1}
TAG=${TAG:-_SBo}

if [ -z "$ARCH" ]; then
  case "$( uname -m )" in
    i?86) ARCH=i586 ;;
    arm*) ARCH=arm ;;
       *) ARCH=$( uname -m ) ;;
  esac 
fi

CWD=$(pwd)
TMP=${TMP:-/tmp/SBo}
PKG=$TMP/package-$PRGNAM
OUTPUT=${OUTPUT:-/tmp}

if [ "$ARCH" = "i586" ]; then
  SLKCFLAGS="-O2 -march=i586 -mtune=i686"
  LIBDIRSUFFIX=""
elif [ "$ARCH" = "i686" ]; then
  SLKCFLAGS="-O2 -march=i686 -mtune=i686"
  LIBDIRSUFFIX=""
elif [ "$ARCH" = "x86_64" ]; then
  SLKCFLAGS="-O2 -fPIC"
  LIBDIRSUFFIX="64"
else
  SLKCFLAGS="-O2"
  LIBDIRSUFFIX=""
fi

set -e

rm -rf $PKG
mkdir -p $TMP/$PRGNAM $PKG $OUTPUT
cd $TMP/$PRGNAM
rm -rf SRB2-SRB2_release_$VERSION

# There's a lot of stuff in the source that we don't need. All the
# --exclude stuff saves 182MB of space in $TMP.
tar xvf $CWD/SRB2-SRB2_release_$VERSION.tar.gz \
  --exclude '*/libs' \
  --exclude '*/android' \
  --exclude '*/windows-installer' \
  --exclude '*/bin' \
  --exclude '*/objs' \
  --exclude '*/tools'

cd SRB2-SRB2_release_$VERSION

chown -R root:root .
find -L .  -perm /111 -a \! -perm 755 -a -exec chmod 755 {} \+ -o \
        \! -perm /111 -a \! -perm 644 -a -exec chmod 644 {} \+
        
mkdir assets/installer        

# Assets (actually WAD files) aren't found in the source, have to download
# them separately. The build actually checks for them & refuses to compile
# if they're missing, which is kinda unfair since it doesn't ship with
# the damn things... To save 208MB of space in $TMP, we symlink the files.
# Can't just touch them, since the md5sums of the files get hardcoded
# into the binary (and it'll refuse to run if they don't match).
DATAFILES="srb2.pk3 zones.pk3 player.dta music.dta patch.pk3 patch_music.pk3"
for i in $DATAFILES; do
  ln -s $CWD/$i assets/installer/$i
done

mkdir -p build
cd build
  cmake \
    -DCMAKE_C_FLAGS_RELEASE:STRING="$SLKCFLAGS -DNDEBUG" \
    -DCMAKE_CXX_FLAGS_RELEASE:STRING="$SLKCFLAGS -DNDEBUG" \
    -DCMAKE_INSTALL_PREFIX=/usr \
    -DLIB_SUFFIX=${LIBDIRSUFFIX} \
    -DMAN_INSTALL_DIR=/usr/man \
    -DCMAKE_BUILD_TYPE=Release ..
  make VERBOSE=1
  #make install/strip DESTDIR=$PKG # don't bother, it's broken
cd ..

# 'make install' puts all the files in the same dir, so manual install:
mkdir -p $PKG/usr/games $PKG/usr/share/games/SRB2 \
         $PKG/usr/share/pixmaps $PKG/usr/share/applications \
         $PKG/usr/doc/$PRGNAM-$VERSION
install -s -m0755 build/bin/lsdlsrb2-$VERSION $PKG/usr/games
ln -s $PRGNAM-$VERSION $PKG/usr/games/$PRGNAM
install -m0644 assets/LICENSE* assets/README* $PKG/usr/doc/$PRGNAM-$VERSION
install -m0644 $PRGNAM.png $PKG/usr/share/pixmaps

# Install data files from $CWD, not the symlinks in assets/
echo -n "Copying data files: "
for i in $DATAFILES; do
  echo -n "$i "
  cat $CWD/$i > $PKG/usr/share/games/SRB2/$i
done
echo

# desktop file is a modified version of debian/srb2.desktop. I took out
# the absolute paths.
cat $CWD/$PRGNAM.desktop > $PKG/usr/share/applications/$PRGNAM.desktop

# dev and modding docs in doc/, config files for cwiid and various doom
# level editors in extras/. We don't need yet another copy of the GPL
# in doc/, so:
rm -f doc/.gitignore doc/copying
cp -a doc extras $PKG/usr/doc/$PRGNAM-$VERSION
cat $CWD/$PRGNAM.SlackBuild > $PKG/usr/doc/$PRGNAM-$VERSION/$PRGNAM.SlackBuild

mkdir -p $PKG/install
cat $CWD/slack-desc > $PKG/install/slack-desc
cat $CWD/doinst.sh > $PKG/install/doinst.sh

cd $PKG
/sbin/makepkg -l y -c n $OUTPUT/$PRGNAM-$VERSION-$ARCH-$BUILD$TAG.${PKGTYPE:-tgz}
