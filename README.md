# linux-from-scratch-7.7-systemd-with-ubuntu14.04
7.7-systemd版本的linux from scratch的一次实践，宿主系统是ubuntu14.04，硬件是amd的物理主机上的虚拟机：virtual-box。   

参考于：https://linux.cn/lfs/LFS-BOOK-7.7-systemd/

##宿主系统预备
【ubuntu 14.04环境设置（输入）】
```shell
sudo rm /bin/sh
sudo ln -s /bin/bash /bin/sh
sudo apt-get update
sudo apt-get install bison gawk build-essential texinfo
```
【检查宿主系统需求（输入）】
```shell
cat > version-check.sh << "EOF"
#!/bin/bash
# Simple script to list version numbers of critical development tools
export LC_ALL=C
bash --version | head -n1 | cut -d" " -f2-4
echo "/bin/sh -> `readlink -f /bin/sh`"
echo -n "Binutils: "; ld --version | head -n1 | cut -d" " -f3-
bison --version | head -n1
if [ -h /usr/bin/yacc ]; then
echo "/usr/bin/yacc -> `readlink -f /usr/bin/yacc`";
elif [ -x /usr/bin/yacc ]; then
echo yacc is `/usr/bin/yacc --version | head -n1`
else
echo "yacc not found"
fi
bzip2 --version 2>&1 < /dev/null | head -n1 | cut -d" " -f1,6-
echo -n "Coreutils: "; chown --version | head -n1 | cut -d")" -f2
diff --version | head -n1
find --version | head -n1
gawk --version | head -n1
if [ -h /usr/bin/awk ]; then
echo "/usr/bin/awk -> `readlink -f /usr/bin/awk`";
elif [ -x /usr/bin/awk ]; then
echo yacc is `/usr/bin/awk --version | head -n1`
else
echo "awk not found"
fi
gcc --version | head -n1
g++ --version | head -n1
ldd --version | head -n1 | cut -d" " -f2- # glibc version
grep --version | head -n1
gzip --version | head -n1
cat /proc/version
m4 --version | head -n1
make --version | head -n1
patch --version | head -n1
echo Perl `perl -V:version`
sed --version | head -n1
tar --version | head -n1
makeinfo --version | head -n1
xz --version | head -n1
echo 'main(){}' > dummy.c && g++ -o dummy dummy.c
if [ -x dummy ]
then echo "g++ compilation OK";
else echo "g++ compilation failed"; fi
rm -f dummy.c dummy
EOF
```
【检查宿主系统需求（输出参考）】
```shell
ubuntu@ubuntu:~$ bash version-check.sh
bash, version 4.3.11(1)-release
/bin/sh -> /bin/bash
Binutils: (GNU Binutils for Ubuntu) 2.24
bison (GNU Bison) 3.0.2
/usr/bin/yacc -> /usr/bin/bison.yacc
bzip2,  Version 1.0.6, 6-Sept-2010.
Coreutils:  8.21
diff (GNU diffutils) 3.3
find (GNU findutils) 4.4.2
GNU Awk 4.0.1
/usr/bin/awk -> /usr/bin/gawk
gcc (Ubuntu 4.8.4-2ubuntu1~14.04) 4.8.4
g++ (Ubuntu 4.8.4-2ubuntu1~14.04) 4.8.4
(Ubuntu EGLIBC 2.19-0ubuntu6.6) 2.19
grep (GNU grep) 2.16
gzip 1.6
Linux version 3.19.0-25-generic (buildd@lgw01-20) (gcc version 4.8.2 (Ubuntu 4.8.2-19ubuntu1) ) #26~14.04.1-Ubuntu SMP Fri Jul 24 21:16:20 UTC 2015
m4 (GNU M4) 1.4.17
GNU Make 3.81
GNU patch 2.7.1
Perl version='5.18.2';
sed (GNU sed) 4.2.2
tar (GNU tar) 1.27.1
makeinfo (GNU texinfo) 5.2
xz (XZ Utils) 5.1.0alpha
g++ compilation OK
ubuntu@ubuntu:~$
```
【库文件的一致性检查（输入）】
```shell
cat > library-check.sh << "EOF"
#!/bin/bash
for lib in lib{gmp,mpfr,mpc}.la; do
echo $lib: $(if find /usr/lib* -name $lib|
grep -q $lib;then :;else echo not;fi) found
done
unset lib
EOF
```
【库文件的一致性检查（输出参考）】
```shell
ubuntu@ubuntu:~$ bash library-check.sh
libgmp.la: not found
libmpfr.la: not found
libmpc.la: not found
ubuntu@ubuntu:~$
```
这些文件应该要么都在或者是都缺失，而不应该只有一两个。

##构建前准备

【分区】
```shell
sudo fdisk -l
sudo mkfs -v -t ext4 /dev/sda
```
【在分区上创建文件系统】
```shell
export LFS=/mnt/lfs

sudo mkdir -pv $LFS   #建立挂载点
sudo mount -v -t ext4 /dev/sda $LFS   #将/dev/sda挂载到$LFS
```
【设置 $LFS 变量】
```shell
sudo passwd root
su -
export LFS=/mnt/lfs
cat > /root/.bash_profile << "EOF"
#!/bin/bash
export LFS=/mnt/lfs
EOF
```
【创建$LFS/tools文件夹】
```shell
sudo ln -sv $LFS/tools /
sudo mkdir -v $LFS/sources
sudo chmod -v a+wt $LFS/sources
```
在宿主系统中创建/tools的符号链接,将其指向LFS分区中新建的文件夹，输出的提示是："/tools" -> "/mnt/lfs/tools"

【添加LFS用户】
```shell
sudo groupadd lfs
sudo useradd -s /bin/bash -g lfs -m -k /dev/null lfs
sudo passwd lfs
sudo chown -v lfs $LFS/tools
sudo chown -v lfs $LFS/sources
```
【设置环境】
```shell
su - lfs
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
source ~/.bash_profile
EOF
```
exec env -i.../bin/bash 命令用一个除了HOME 、 TERM 和 PS1 变量,完全空环境的 shell 代替运行中的 shell。
```shell
cat > ~/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/tools/bin:/bin:/usr/bin
export LFS LC_ALL LFS_TGT PATH
EOF

source ~/.bash_profile
```
【软件包与补丁】
```shell
cd $LFS/sources
wget 'https://linux.cn/lfs/LFS-BOOK-7.7-systemd/wget-list'
wget --input-file=wget-list --continue --directory-prefix=$LFS/sources

pushd $LFS/sources
wget 'https://linux.cn/lfs/LFS-BOOK-7.7-systemd/md5sums'
md5sum -c md5sums
popd
```

##构建临时系统
【Binutils-2.25 - 第一遍（安装交叉编译的 Binutils）】
```shell
cd $LFS/sources
tar -jxvf binutils-2.25.tar.bz2
cd binutils-2.25
mkdir -v ../binutils-build
cd ../binutils-build

../binutils-2.25/configure     \
    --prefix=/tools            \
    --with-sysroot=$LFS        \
    --with-lib-path=/tools/lib \
    --target=$LFS_TGT          \
    --disable-nls              \
    --disable-werror

make

case $(uname -m) in
  x86_64) mkdir -v /tools/lib && ln -sv lib /tools/lib64 ;;
esac

make install

cd $LFS/sources
rm -rf binutils-build/ binutils-2.25/
```
【GCC-4.9.2 - 第一遍（安装交叉编译的 GCC）】
```shell
cd $LFS/sources
tar -jxvf gcc-4.9.2.tar.bz2
cd gcc-4.9.2

tar -xf ../mpfr-3.1.2.tar.xz
mv -v mpfr-3.1.2 mpfr
tar -xf ../gmp-6.0.0a.tar.xz
mv -v gmp-6.0.0 gmp
tar -xf ../mpc-1.0.2.tar.gz
mv -v mpc-1.0.2 mpc

for file in \
 $(find gcc/config -name linux64.h -o -name linux.h -o -name sysv4.h)
do
  cp -uv $file{,.orig}
  sed -e 's@/lib\(64\)\?\(32\)\?/ld@/tools&@g' \
      -e 's@/usr@/tools@g' $file.orig > $file
  echo '
#undef STANDARD_STARTFILE_PREFIX_1
#undef STANDARD_STARTFILE_PREFIX_2
#define STANDARD_STARTFILE_PREFIX_1 "/tools/lib/"
#define STANDARD_STARTFILE_PREFIX_2 ""' >> $file
  touch $file.orig
done

sed -i '/k prot/agcc_cv_libc_provides_ssp=yes' gcc/configure

mkdir -v ../gcc-build
cd ../gcc-build

../gcc-4.9.2/configure                             \
    --target=$LFS_TGT                              \
    --prefix=/tools                                \
    --with-sysroot=$LFS                            \
    --with-newlib                                  \
    --without-headers                              \
    --with-local-prefix=/tools                     \
    --with-native-system-header-dir=/tools/include \
    --disable-nls                                  \
    --disable-shared                               \
    --disable-multilib                             \
    --disable-decimal-float                        \
    --disable-threads                              \
    --disable-libatomic                            \
    --disable-libgomp                              \
    --disable-libitm                               \
    --disable-libquadmath                          \
    --disable-libsanitizer                         \
    --disable-libssp                               \
    --disable-libvtv                               \
    --disable-libcilkrts                           \
    --disable-libstdc++-v3                         \
    --enable-languages=c,c++

make
make install

cd $LFS/sources
rm -rf gcc-build/ gcc-4.9.2/
```
【Linux-3.19 API 头文件】
```shell
cd $LFS/sources
tar -xvf linux-3.19.tar.xz
cd linux-3.19
make mrproper
make INSTALL_HDR_PATH=dest headers_install
cp -rv dest/include/* /tools/include

cd $LFS/sources
rm -rf linux-3.19/
```
【Glibc-2.21】
```shell
cd $LFS/sources
tar -xvf glibc-2.21.tar.xz
cd glibc-2.21

if [ ! -r /usr/include/rpc/types.h ]; then
  su -c 'mkdir -pv /usr/include/rpc'
  su -c 'cp -v sunrpc/rpc/*.h /usr/include/rpc'
fi

sed -e '/ia32/s/^/1:/' \
    -e '/SSE2/s/^1://' \
    -i  sysdeps/i386/i686/multiarch/mempcpy_chk.S

mkdir -v ../glibc-build
cd ../glibc-build

../glibc-2.21/configure                             \
      --prefix=/tools                               \
      --host=$LFS_TGT                               \
      --build=$(../glibc-2.21/scripts/config.guess) \
      --disable-profile                             \
      --enable-kernel=2.6.32                        \
      --with-headers=/tools/include                 \
      libc_cv_forced_unwind=yes                     \
      libc_cv_ctors_header=yes                      \
      libc_cv_c_cleanup=yes

make

make install

cd $LFS/sources
rm -rf gcc-build/ glibc-2.21/
```
【确认新工具链的基本功能(编译和链接)都是像预期的那样正常工作】
```shell
echo 'main(){}' > dummy.c
$LFS_TGT-gcc dummy.c
readelf -l a.out | grep ': /tools'
```
如果一切工作正常的话，这里应该没有错误，最后一个命令的输出形式会是：
[Requesting program interpreter: /tools/lib/ld-linux.so.2]
```shell
rm -v dummy.c a.out
```
【Libstdc++-4.9.2】
```shell
cd $LFS/sources
tar -jxvf gcc-4.9.2.tar.bz2
cd gcc-4.9.2

mkdir -pv ../gcc-build
cd ../gcc-build

../gcc-4.9.2/libstdc++-v3/configure \
    --host=$LFS_TGT                 \
    --prefix=/tools                 \
    --disable-multilib              \
    --disable-shared                \
    --disable-nls                   \
    --disable-libstdcxx-threads     \
    --disable-libstdcxx-pch         \
    --with-gxx-include-dir=/tools/$LFS_TGT/include/c++/4.9.2

make

make install 

cd $LFS/sources
rm -rf gcc-build/ gcc-4.9.2/
```
【Binutils-2.25 - 第2遍】
```shell
cd $LFS/sources
rm -rf binutils-build binutils-2.25

cd $LFS/sources
tar xf binutils-2.25.tar.bz2
cd binutils-2.25
mkdir -v ../binutils-build
cd ../binutils-build

CC=$LFS_TGT-gcc                \
AR=$LFS_TGT-ar                 \
RANLIB=$LFS_TGT-ranlib         \
../binutils-2.25/configure     \
    --prefix=/tools            \
    --disable-nls              \
    --disable-werror           \
    --with-lib-path=/tools/lib \
    --with-sysroot

make

make install

make -C ld clean
make -C ld LIB_PATH=/usr/lib:/lib
cp -v ld/ld-new /tools/bin

cd $LFS/sources
rm -rf binutils-build binutils-2.25
```
【GCC-4.9.2 - 第2遍】
```shell
cd $LFS/sources
rm -rf gcc-4.9.2 gcc-build
tar -xvf gcc-4.9.2.tar.bz2
cd gcc-4.9.2

cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
  `dirname $($LFS_TGT-gcc -print-libgcc-file-name)`/include-fixed/limits.h

for file in \
 $(find gcc/config -name linux64.h -o -name linux.h -o -name sysv4.h)
do
  cp -uv $file{,.orig}
  sed -e 's@/lib\(64\)\?\(32\)\?/ld@/tools&@g' \
      -e 's@/usr@/tools@g' $file.orig > $file
  echo '
#undef STANDARD_STARTFILE_PREFIX_1
#undef STANDARD_STARTFILE_PREFIX_2
#define STANDARD_STARTFILE_PREFIX_1 "/tools/lib/"
#define STANDARD_STARTFILE_PREFIX_2 ""' >> $file
  touch $file.orig
done

tar -xf ../mpfr-3.1.2.tar.xz
mv -v mpfr-3.1.2 mpfr
tar -xf ../gmp-6.0.0a.tar.xz
mv -v gmp-6.0.0 gmp
tar -xf ../mpc-1.0.2.tar.gz
mv -v mpc-1.0.2 mpc

mkdir -v ../gcc-build
cd ../gcc-build

CC=$LFS_TGT-gcc                                    \
CXX=$LFS_TGT-g++                                   \
AR=$LFS_TGT-ar                                     \
RANLIB=$LFS_TGT-ranlib                             \
../gcc-4.9.2/configure                             \
    --prefix=/tools                                \
    --with-local-prefix=/tools                     \
    --with-native-system-header-dir=/tools/include \
    --enable-languages=c,c++                       \
    --disable-libstdcxx-pch                        \
    --disable-multilib                             \
    --disable-bootstrap                            \
    --disable-libgomp

make

make install

ln -sv gcc /tools/bin/cc

cd $LFS/sources
rm -rf gcc-build/ gcc-4.9.2/
```
【确认新工具链的基本功能(编译和链接)都是像预期的那样正常工作】
```shell
echo 'main(){}' > dummy.c
cc dummy.c
readelf -l a.out | grep ': /tools'
```
如果一切工作正常的话，这里应该没有错误，最后一个命令的输出形式会是：
[Requesting program interpreter: /tools/lib/ld-linux.so.2]
```shell
rm -v dummy.c a.out
```
【Tcl-8.6.3】
```shell
cd $LFS/sources
tar -xzvf tcl8.6.3-src.tar.gz
cd tcl8.6.3
cd unix
./configure --prefix=/tools
make 
make install
chmod -v u+w /tools/lib/libtcl8.6.so
make install-private-headers
ln -sv tclsh8.6 /tools/bin/tclsh

cd $LFS/sources
rm -rf tcl8.6.3/
```

【Expect-5.45】
```shell
cd $LFS/sources
tar -xzvf expect5.45.tar.gz
cd expect5.45
cp -v configure{,.orig}
sed 's:/usr/local/bin:/bin:' configure.orig > configure
./configure --prefix=/tools       \
            --with-tcl=/tools/lib \
            --with-tclinclude=/tools/include
make
make SCRIPTS="" install

cd $LFS/sources
rm -rf expect5.45/
```
【DejaGNU-1.5.2】
```shell
cd $LFS/sources
tar -xzvf dejagnu-1.5.2.tar.gz
cd dejagnu-1.5.2
./configure --prefix=/tools
make install

cd $LFS/sources
rm -rf dejagnu-1.5.2/
```
【Check-0.9.14】
```shell
cd $LFS/sources
tar -xzvf check-0.9.14.tar.gz
cd check-0.9.14
PKG_CONFIG= ./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf check-0.9.14/
```
【Ncurses-5.9】
```shell
cd $LFS/sources
tar -xzvf ncurses-5.9.tar.gz
cd ncurses-5.9
./configure --prefix=/tools \
            --with-shared   \
            --without-debug \
            --without-ada   \
            --enable-widec  \
            --enable-overwrite
make
make install

cd $LFS/sources
rm -rf ncurses-5.9/
```
【Bash-4.3.30】
```shell
cd $LFS/sources
tar -xzvf bash-4.3.30.tar.gz
cd bash-4.3.30
./configure --prefix=/tools --without-bash-malloc
make
make install
ln -sv bash /tools/bin/sh

cd $LFS/sources
rm -rf bash-4.3.30/
```
【Bzip2-1.0.6】
```shell
cd $LFS/sources
tar -xzvf bzip2-1.0.6.tar.gz
cd bzip2-1.0.6
make
make PREFIX=/tools install

cd $LFS/sources
rm -rf bzip2-1.0.6/
```
【Coreutils-8.23】
```shell
cd $LFS/sources
tar -xvf coreutils-8.23.tar.xz
cd coreutils-8.23
./configure --prefix=/tools --enable-install-program=hostname
make
make install

cd $LFS/sources
rm -rf coreutils-8.23/
```
【Diffutils-3.3】
```shell
cd $LFS/sources
tar -xvf diffutils-3.3.tar.xz
cd diffutils-3.3
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf diffutils-3.3/
```
【File-5.22】
```shell
cd $LFS/sources
tar -xzvf file-5.22.tar.gz
cd file-5.22
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf file-5.22/
```
【Findutils-4.4.2】
```shell
cd $LFS/sources
tar -xzvf findutils-4.4.2.tar.gz
cd findutils-4.4.2
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf findutils-4.4.2/
```
【Gawk-4.1.1】
```shell
cd $LFS/sources
tar -xvf gawk-4.1.1.tar.xz
cd gawk-4.1.1
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf gawk-4.1.1/
```
【Gettext-0.19.4】
```shell
cd $LFS/sources
tar -xvf gettext-0.19.4.tar.xz
cd gettext-0.19.4
cd gettext-tools
EMACS="no" ./configure --prefix=/tools --disable-shared
make -C gnulib-lib
make -C intl pluralx.c
make -C src msgfmt
make -C src msgmerge
make -C src xgettext
cp -v src/{msgfmt,msgmerge,xgettext} /tools/bin

cd $LFS/sources
rm -rf gettext-0.19.4/
```
【Grep-2.21】
```shell
cd $LFS/sources
tar -xvf grep-2.21.tar.xz
cd grep-2.21
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf gettext-0.19.4/
```
【Gzip-1.6】
```shell
cd $LFS/sources
tar -xvf gzip-1.6.tar.xz
cd gzip-1.6
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf gzip-1.6/
```
【M4-1.4.17】
```shell
cd $LFS/sources
tar -xvf m4-1.4.17.tar.xz
cd m4-1.4.17
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf m4-1.4.17/
```
【Make-4.1】
```shell
cd $LFS/sources
tar -jxvf make-4.1.tar.bz2
cd make-4.1
./configure --prefix=/tools --without-guile
make
make install

cd $LFS/sources
rm -rf m4-1.4.17/
```
【Patch-2.7.4】
```shell
cd $LFS/sources
tar -xvf patch-2.7.4.tar.xz
cd patch-2.7.4
./configure --prefix=/tools
make 
make install

cd $LFS/sources
rm -rf patch-2.7.4/
```
【Perl-5.20.2】
```shell
cd $LFS/sources
tar -jxvf perl-5.20.2.tar.bz2
cd perl-5.20.2
sh Configure -des -Dprefix=/tools -Dlibs=-lm
make
cp -v perl cpan/podlators/pod2man /tools/bin
mkdir -pv /tools/lib/perl5/5.20.2
cp -Rv lib/* /tools/lib/perl5/5.20.2

cd $LFS/sources
rm -rf perl-5.20.2/
```
【Sed-4.2.2】
```shell
cd $LFS/sources
tar -jxvf sed-4.2.2.tar.bz2
cd sed-4.2.2
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf sed-4.2.2/
```
【Tar-1.28】
```shell
cd $LFS/sources
tar -xvf tar-1.28.tar.xz
cd tar-1.28
./configure --prefix=/tools
make 
make install

cd $LFS/sources
rm -rf tar-1.28/
```
【Texinfo-5.2】
```shell
cd $LFS/sources
tar -xvf texinfo-5.2.tar.xz
cd texinfo-5.2
./configure --prefix=/tools
make 
make install

cd $LFS/sources
rm -rf texinfo-5.2/
```
【Util-linux-2.26】
```shell
cd $LFS/sources
tar -xvf util-linux-2.26.tar.xz
cd util-linux-2.26
./configure --prefix=/tools                \
            --without-python               \
            --disable-makeinstall-chown    \
            --without-systemdsystemunitdir \
            PKG_CONFIG=""
make 
make install

cd $LFS/sources
rm -rf util-linux-2.26/
```
【Xz-5.2.0】
```shell
cd $LFS/sources
tar -xvf xz-5.2.0.tar.xz
cd xz-5.2.0
./configure --prefix=/tools
make 
make install

cd $LFS/sources
rm -rf xz-5.2.0/
```
【清理无用内容】
```shell
strip --strip-debug /tools/lib/*
/usr/bin/strip --strip-unneeded /tools/{,s}bin/*

rm -rf /tools/{,share}/{info,man,doc}
```
这两个命令会跳过一些文件，并提示不可识别的文件格式。  
【改变属主】 ＃以后部分的命令都必须以 root 用户身份执行而不再是 lfs 用户。
```shell
su -
echo $LFS #再次确认下 $LFS 变量在 root 用户环境下也有定义。
chown -R root:root $LFS/tools
```

##构建 LFS 系统
【准备虚拟内核文件系统】
```shell
mkdir -pv $LFS/{dev,proc,sys,run}

mknod -m 600 $LFS/dev/console c 5 1
mknod -m 666 $LFS/dev/null c 1 3

mount -v --bind /dev $LFS/dev

mount -vt devpts devpts $LFS/dev/pts -o gid=5,mode=620
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run

if [ -h $LFS/dev/shm ]; then
  mkdir -pv $LFS/$(readlink $LFS/dev/shm)
fi
```
【进入 Chroot 环境】
```shell
chroot "$LFS" /tools/bin/env -i \
    HOME=/root                  \
    TERM="$TERM"                \
    PS1='\u:\w\$ '              \
    PATH=/bin:/usr/bin:/sbin:/usr/sbin:/tools/bin \
    /tools/bin/bash --login +h
```
注意一下 bash 的提示符是 I have no name!。这是正常的，因为这个时候 /etc/passwd 文件还没有被创建。  
  
＃从这以后的命令，以及后续章节里的命令都要在 chroot 环境下运行。  
＃如果因为某种原因（比如说重启）离开了这个环境，  
＃请保证运行【准备虚拟内核文件系统】的所有mount命令，再运行本节的命令chroot  
  
【创建目录】
```shell
mkdir -pv /{bin,boot,etc/{opt,sysconfig},home,lib/firmware,mnt,opt}
mkdir -pv /{media/{floppy,cdrom},sbin,srv,var}
install -dv -m 0750 /root
install -dv -m 1777 /tmp /var/tmp
mkdir -pv /usr/{,local/}{bin,include,lib,sbin,src}
mkdir -pv /usr/{,local/}share/{color,dict,doc,info,locale,man}
mkdir -v  /usr/{,local/}share/{misc,terminfo,zoneinfo}
mkdir -v  /usr/libexec
mkdir -pv /usr/{,local/}share/man/man{1..8}

case $(uname -m) in
 x86_64) ln -sv lib /lib64
         ln -sv lib /usr/lib64
         ln -sv lib /usr/local/lib64 ;;
esac

mkdir -v /var/{log,mail,spool}
ln -sv /run /var/run
ln -sv /run/lock /var/lock
mkdir -pv /var/{opt,cache,lib/{color,misc,locate},local}
```
【创建必需的文件和符号链接】
```shell
ln -sv /tools/bin/{bash,cat,echo,pwd,stty} /bin
ln -sv /tools/bin/perl /usr/bin
ln -sv /tools/lib/libgcc_s.so{,.1} /usr/lib
ln -sv /tools/lib/libstdc++.so{,.6} /usr/lib
sed 's/tools/usr/' /tools/lib/libstdc++.la > /usr/lib/libstdc++.la
ln -sv bash /bin/sh

ln -sv /proc/self/mounts /etc/mtab

cat > /etc/passwd << "EOF"
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/dev/null:/bin/false
daemon:x:6:6:Daemon User:/dev/null:/bin/false
messagebus:x:18:18:D-Bus Message Daemon User:/var/run/dbus:/bin/false
systemd-bus-proxy:x:72:72:systemd Bus Proxy:/:/bin/false
systemd-journal-gateway:x:73:73:systemd Journal Gateway:/:/bin/false
systemd-journal-remote:x:74:74:systemd Journal Remote:/:/bin/false
systemd-journal-upload:x:75:75:systemd Journal Upload:/:/bin/false
systemd-network:x:76:76:systemd Network Management:/:/bin/false
systemd-resolve:x:77:77:systemd Resolver:/:/bin/false
systemd-timesync:x:78:78:systemd Time Synchronization:/:/bin/false
nobody:x:99:99:Unprivileged User:/dev/null:/bin/false
EOF

cat > /etc/group << "EOF"
root:x:0:
bin:x:1:daemon
sys:x:2:
kmem:x:3:
tape:x:4:
tty:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
usb:x:14:
cdrom:x:15:
adm:x:16:
messagebus:x:18:
systemd-journal:x:23:
input:x:24:
mail:x:34:
systemd-bus-proxy:x:72:
systemd-journal-gateway:x:73:
systemd-journal-remote:x:74:
systemd-journal-upload:x:75:
systemd-network:x:76:
systemd-resolve:x:77:
systemd-timesync:x:78:
nogroup:x:99:
users:x:999:
EOF

exec /tools/bin/bash --login +h

touch /var/log/{btmp,lastlog,wtmp}
chgrp -v utmp /var/log/lastlog
chmod -v 664  /var/log/lastlog
chmod -v 600  /var/log/btmp
```
【Linux-3.19 API 头文件】
```shell
cd /sources
tar -xvf linux-3.19.tar.xz
cd linux-3.19
make mrproper
make INSTALL_HDR_PATH=dest headers_install
find dest/include \( -name .install -o -name ..install.cmd \) -delete
cp -rv dest/include/* /usr/include

cd /sources
rm -rf linux-3.19/
```
【Man-pages-3.79】
```shell
cd /sources
tar -xvf man-pages-3.79.tar.xz
cd man-pages-3.79
make install

cd /sources
rm -rf man-pages-3.79/
```
【Glibc-2.21】
```shell
cd /sources
tar -xvf glibc-2.21.tar.xz
cd glibc-2.21
patch -Np1 -i ../glibc-2.21-fhs-1.patch
sed -e '/ia32/s/^/1:/' \
    -e '/SSE2/s/^1://' \
    -i  sysdeps/i386/i686/multiarch/mempcpy_chk.S
mkdir -v ../glibc-build
cd ../glibc-build
../glibc-2.21/configure    \
    --prefix=/usr          \
    --disable-profile      \
    --enable-kernel=2.6.32 \
    --enable-obsolete-rpc
make
make check

touch /etc/ld.so.conf

make install

cp -v ../glibc-2.21/nscd/nscd.conf /etc/nscd.conf
mkdir -pv /var/cache/nscd

install -v -Dm644 ../glibc-2.21/nscd/nscd.tmpfiles /usr/lib/tmpfiles.d/nscd.conf
install -v -Dm644 ../glibc-2.21/nscd/nscd.service /lib/systemd/system/nscd.service

mkdir -pv /usr/lib/locale
localedef -i cs_CZ -f UTF-8 cs_CZ.UTF-8
localedef -i de_DE -f ISO-8859-1 de_DE
localedef -i de_DE@euro -f ISO-8859-15 de_DE@euro
localedef -i de_DE -f UTF-8 de_DE.UTF-8
localedef -i en_GB -f UTF-8 en_GB.UTF-8
localedef -i en_HK -f ISO-8859-1 en_HK
localedef -i en_PH -f ISO-8859-1 en_PH
localedef -i en_US -f ISO-8859-1 en_US
localedef -i en_US -f UTF-8 en_US.UTF-8
localedef -i es_MX -f ISO-8859-1 es_MX
localedef -i fa_IR -f UTF-8 fa_IR
localedef -i fr_FR -f ISO-8859-1 fr_FR
localedef -i fr_FR@euro -f ISO-8859-15 fr_FR@euro
localedef -i fr_FR -f UTF-8 fr_FR.UTF-8
localedef -i it_IT -f ISO-8859-1 it_IT
localedef -i it_IT -f UTF-8 it_IT.UTF-8
localedef -i ja_JP -f EUC-JP ja_JP
localedef -i ru_RU -f KOI8-R ru_RU.KOI8-R
localedef -i ru_RU -f UTF-8 ru_RU.UTF-8
localedef -i tr_TR -f UTF-8 tr_TR.UTF-8
localedef -i zh_CN -f GB18030 zh_CN.GB18030

make localedata/install-locales

cd $LFS/sources
rm -rf glibc-2.21 glibc-build
```
```shell
cat > /etc/nsswitch.conf << "EOF"
# Begin /etc/nsswitch.conf

passwd: files
group: files
shadow: files

hosts: files dns myhostname
networks: files

protocols: files
services: files
ethers: files
rpc: files

# End /etc/nsswitch.conf
EOF

tar -xf ../tzdata2015a.tar.gz

ZONEINFO=/usr/share/zoneinfo
mkdir -pv $ZONEINFO/{posix,right}

for tz in etcetera southamerica northamerica europe africa antarctica  \
          asia australasia backward pacificnew systemv; do
    zic -L /dev/null   -d $ZONEINFO       -y "sh yearistype.sh" ${tz}
    zic -L /dev/null   -d $ZONEINFO/posix -y "sh yearistype.sh" ${tz}
    zic -L leapseconds -d $ZONEINFO/right -y "sh yearistype.sh" ${tz}
done

cp -v zone.tab zone1970.tab iso3166.tab $ZONEINFO
zic -d $ZONEINFO -p America/New_York
unset ZONEINFO
```
```shell
tzselect
```
```shell
ln -sfv /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```
```shell
cat > /etc/ld.so.conf << "EOF"
# Begin /etc/ld.so.conf
/usr/local/lib
/opt/lib

EOF

cat >> /etc/ld.so.conf << "EOF"
# Add an include directory
include /etc/ld.so.conf.d/*.conf

EOF
mkdir -pv /etc/ld.so.conf.d
```
【调整工具链】
```shell
mv -v /tools/bin/{ld,ld-old}
mv -v /tools/$(gcc -dumpmachine)/bin/{ld,ld-old}
mv -v /tools/bin/{ld-new,ld}
ln -sv /tools/bin/ld /tools/$(gcc -dumpmachine)/bin/ld

gcc -dumpspecs | sed -e 's@/tools@@g'                   \
    -e '/\*startfile_prefix_spec:/{n;s@.*@/usr/lib/ @}' \
    -e '/\*cpp:/{n;s@$@ -isystem /usr/include@}' >      \
    `dirname $(gcc --print-libgcc-file-name)`/specs
```
【检查已调整的工具链的基本功能（编译和链接）】
```shell
echo 'main(){}' > dummy.c
cc dummy.c -v -Wl,--verbose &> dummy.log
readelf -l a.out | grep ': /lib'
```
如果没有任何错误，上条命令的输出应该是（不同的平台上的动态链接器可能名字不同）：[Requesting program interpreter: /lib/ld-linux.so.2]
```shell
grep -o '/usr/lib.*/crt[1in].*succeeded' dummy.log
```
上一条命令的输出应该是：  
/usr/lib/crt1.o succeeded  
/usr/lib/crti.o succeeded  
/usr/lib/crtn.o succeeded  
```shell
grep -B1 '^ /usr/include' dummy.log
```
这条命令应该返回如下输出：
```shell
#include <...> search starts here:
/usr/include
```
```shell
grep 'SEARCH.*/usr/lib' dummy.log |sed 's|; |\n|g'
```
应该忽略指向带有 '-linux-gnu' 的路径，上条命令的输出应该是：  
SEARCH_DIR("/usr/lib")  
SEARCH_DIR("/lib");  
```shell
grep "/lib.*/libc.so.6 " dummy.log
```
上条命令的输出应该是（在 64 位主机上会有 lib64 目录）：  
attempt to open /lib/libc.so.6 succeeded  
```shell
grep found dummy.log
```
上条命令的结果应该是（不同的平台上链接器名字可以不同，64 位主机上是 lib64 目录）：  
found ld-linux.so.2 at /lib/ld-linux.so.2  
一旦所有的事情都正常了，清除测试文件：
```shell
rm -v dummy.c a.out dummy.log
```
【Zlib-1.2.8】
```shell
cd /sources
tar -xvf zlib-1.2.8.tar.xz
cd zlib-1.2.8
./configure --prefix=/usr
make
make install

mv -v /usr/lib/libz.so.* /lib
ln -sfv ../../lib/$(readlink /usr/lib/libz.so) /usr/lib/libz.so

cd /sources
rm -rf zlib-1.2.8
```
【File-5.22】
```shell
cd /sources
tar -xzvf file-5.22.tar.gz
cd file-5.22

./configure --prefix=/usr
make
make install

cd /sources
rm -rf file-5.22
```
【Binutils-2.25】  
通过一个简单测试验证在 chroot 环境下 PTY 工作正常：
```shell
expect -c "spawn ls"
```
这个命令应该输出以下内容：  
spawn ls  
假如输出包括下面的信息，那么表示没有为 PTY 操作设置好环境。在运行 Binutils 和 GCC 的测试套件之前需要解决这个问题：  
The system has no more ptys.  
Ask your system administrator to create more.
```shell
cd /sources
tar -jxvf binutils-2.25.tar.bz2
cd binutils-2.25
mkdir -v ../binutils-build
cd ../binutils-build

../binutils-2.25/configure --prefix=/usr   \
                           --enable-shared \
                           --disable-werror

make tooldir=/usr
make -k check
make tooldir=/usr install

cd /sources
rm -rf binutils-2.25
```
【GMP-6.0.0a】
```shell
cd /sources
tar -xvf gmp-6.0.0a.tar.xz
cd gmp-6.0.0

./configure --prefix=/usr \
            --enable-cxx  \
            --docdir=/usr/share/doc/gmp-6.0.0a

make
make html

make check 2>&1 | tee gmp-check-log
```
确认测试套件中所有的 188 个测试都通过了。通过输入下面的命令检查结果：  
awk '/tests passed/{total+=$2} ; END{print total}' gmp-check-log
```shell
make install
make install-html

cd /sources
rm -rf gmp-6.0.0
```
【MPFR-3.1.2】
```shell
cd /sources
tar -xvf mpfr-3.1.2.tar.xz
cd mpfr-3.1.2
patch -Np1 -i ../mpfr-3.1.2-upstream_fixes-3.patch
./configure --prefix=/usr        \../../usr/bin/cpp
            --enable-thread-safe \
            --docdir=/usr/share/doc/mpfr-3.1.2
make
make check
make install
make install-html

cd /sources
rm -rf mpfr-3.1.2
```
【MPC-1.0.2】
```shell
cd /sources
tar -xzvf mpc-1.0.2.tar.gz
cd mpc-1.0.2
./configure --prefix=/usr --docdir=/usr/share/doc/mpc-1.0.2
make
make html
make install
make install-html

cd /sources
rm -rf mpc-1.0.2
```
【GCC-4.9.2】
```shell
cd /sources
tar -jxvf gcc-4.9.2.tar.bz2
cd gcc-4.9.2

mkdir -v ../gcc-build
cd ../gcc-build

SED=sed                       \
../gcc-4.9.2/configure        \
     --prefix=/usr            \
     --enable-languages=c,c++ \
     --disable-multilib       \
     --disable-bootstrap      \
     --with-system-zlib

make

#ulimit -s 32768
#make -k check

#../gcc-4.9.2/contrib/test_summary

make install

ln -sv ../../usr/bin/cpp /lib

ln -sv gcc /usr/bin/cc

install -v -dm755 /usr/lib/bfd-plugins
ln -sfv ../../libexec/gcc/$(gcc -dumpmachine)/4.9.2/liblto_plugin.so /usr/lib/bfd-plugins/

cd /sources
rm -rf gcc-4.9.2 gcc-build
```
【检查最终的工具链】
```shell
echo 'main(){}' > dummy.c
cc dummy.c -v -Wl,--verbose &> dummy.log
readelf -l a.out | grep ': /lib'
```
这应该没有错误，最后一个命令的输出应该是（允许平台相关的动态链接器名字有差异）：[Requesting program interpreter: /lib/ld-linux.so.2]
```shell
grep -o '/usr/lib.*/crt[1in].*succeeded' dummy.log
```
最后一个命令的输出应该是：  
/usr/lib/gcc/i686-pc-linux-gnu/4.9.2/../../../crt1.o succeeded  
/usr/lib/gcc/i686-pc-linux-gnu/4.9.2/../../../crti.o succeeded  
/usr/lib/gcc/i686-pc-linux-gnu/4.9.2/../../../crtn.o succeeded
```shell
grep -B4 '^ /usr/include' dummy.log
```
这个命令应该返回下面的的输出：  
 #include <...> search starts here:  
/usr/lib/gcc/i686-pc-linux-gnu/4.9.2/include  
/usr/local/include  
/usr/lib/gcc/i686-pc-linux-gnu/4.9.2/include-fixed  
/usr/include
```shell
grep 'SEARCH.*/usr/lib' dummy.log |sed 's|; |\n|g'
```
可以忽略指向含有 '-linux-gnu' 的路径的引用，但最后一个命令的输出应该是：  
SEARCH_DIR("/usr/x86_64-unknown-linux-gnu/lib64")  
SEARCH_DIR("/usr/local/lib64")  
SEARCH_DIR("/lib64")  
SEARCH_DIR("/usr/lib64")  
SEARCH_DIR("/usr/x86_64-unknown-linux-gnu/lib")  
SEARCH_DIR("/usr/local/lib")  
SEARCH_DIR("/lib")  
SEARCH_DIR("/usr/lib");  
```shell
grep "/lib.*/libc.so.6 " dummy.log
```
最后一个命令的输出（64 位主机中应该是 lib64 目录）应该是：  
attempt to open /lib/libc.so.6 succeeded
```shell
grep found dummy.log
```
最后一个命令的输出应该是（平台相关的动态链接器是可以有所差异， 64 位系统中应该是 lib64 目录）：  
found ld-linux.so.2 at /lib/ld-linux.so.2
```shell
rm -v dummy.c a.out dummy.log

mkdir -pv /usr/share/gdb/auto-load/usr/lib
mv -v /usr/lib/*gdb.py /usr/share/gdb/auto-load/usr/lib
```
【Bzip2-1.0.6】
```shell
cd /sources
tar -xzvf bzip2-1.0.6.tar.gz
cd bzip2-1.0.6

patch -Np1 -i ../bzip2-1.0.6-install_docs-1.patch
sed -i 's@\(ln -s -f \)$(PREFIX)/bin/@\1@' Makefile
sed -i "s@(PREFIX)/man@(PREFIX)/share/man@g" Makefile
make -f Makefile-libbz2_so
make clean
make
make PREFIX=/usr install

cp -v bzip2-shared /bin/bzip2
cp -av libbz2.so* /lib
ln -sv ../../lib/libbz2.so.1.0 /usr/lib/libbz2.so
rm -v /usr/bin/{bunzip2,bzcat,bzip2}
ln -sv bzip2 /bin/bunzip2
ln -sv bzip2 /bin/bzcat

cd /sources
rm -rf bzip2-1.0.6
```
【pkg-config-0.28】
```shell
cd /sources
tar -xzvf pkg-config-0.28.tar.gz
cd pkg-config-0.28

./configure --prefix=/usr        \
            --with-internal-glib \
            --disable-host-tool  \
            --docdir=/usr/share/doc/pkg-config-0.28

make

make install

cd /sources
rm -rf pkg-config-0.28
```
【Ncurses-5.9】
```shell
cd /sources
tar -xzvf ncurses-5.9.tar.gz
cd ncurses-5.9

./configure --prefix=/usr           \
            --mandir=/usr/share/man \
            --with-shared           \
            --without-debug         \
            --enable-pc-files       \
            --enable-widec
make
make install

mv -v /usr/lib/libncursesw.so.5* /lib
ln -sfv ../../lib/$(readlink /usr/lib/libncursesw.so) /usr/lib/libncursesw.so
for lib in ncurses form panel menu ; do
    rm -vf                    /usr/lib/lib${lib}.so
    echo "INPUT(-l${lib}w)" > /usr/lib/lib${lib}.so
    ln -sfv lib${lib}w.a      /usr/lib/lib${lib}.a
    ln -sfv ${lib}w.pc        /usr/lib/pkgconfig/${lib}.pc
done

ln -sfv libncurses++w.a /usr/lib/libncurses++.a

rm -vf                     /usr/lib/libcursesw.so
echo "INPUT(-lncursesw)" > /usr/lib/libcursesw.so
ln -sfv libncurses.so      /usr/lib/libcurses.so
ln -sfv libncursesw.a      /usr/lib/libcursesw.a
ln -sfv libncurses.a       /usr/lib/libcurses.a

mkdir -v       /usr/share/doc/ncurses-5.9
cp -v -R doc/* /usr/share/doc/ncurses-5.9

make distclean
./configure --prefix=/usr    \
            --with-shared    \
            --without-normal \
            --without-debug  \
            --without-cxx-binding
make sources libs
cp -av lib/lib*.so.5* /usr/lib

cd /sources
rm -rf ncurses-5.9
```
【Attr-2.4.47】
```shell
cd /sources
tar -xzvf attr-2.4.47.src.tar.gz
cd attr-2.4.47
sed -i -e 's|/@pkg_name@|&-@pkg_version@|' include/builddefs.in
sed -i -e "/SUBDIRS/s|man2||" man/Makefile
./configure --prefix=/usr

make

make -j1 tests root-tests

make install install-dev install-lib
chmod -v 755 /usr/lib/libattr.so

mv -v /usr/lib/libattr.so.* /lib
ln -sfv ../../lib/$(readlink /usr/lib/libattr.so) /usr/lib/libattr.so

cd /sources
rm -rf attr-2.4.47
```
【Acl-2.2.52】
```shell
cd /sources
tar -xzvf acl-2.2.52.src.tar.gz
cd acl-2.2.52

sed -i -e 's|/@pkg_name@|&-@pkg_version@|' include/builddefs.in
sed -i "s:| sed.*::g" test/{sbits-restore,cp,misc}.test
sed -i -e "/TABS-1;/a if (x > (TABS-1)) x = (TABS-1);" \
    libacl/__acl_to_any_text.c

./configure --prefix=/usr --libexecdir=/usr/lib
make
make install install-dev install-lib
chmod -v 755 /usr/lib/libacl.so
mv -v /usr/lib/libacl.so.* /lib
ln -sfv ../../lib/$(readlink /usr/lib/libacl.so) /usr/lib/libacl.so

cd /sources
rm -rf acl-2.2.52
```
【Libcap-2.24】
```shell
cd /sources
tar -xvf libcap-2.24.tar.xz
cd libcap-2.24

make
make RAISE_SETFCAP=no prefix=/usr install

chmod -v 755 /usr/lib/libcap.so

mv -v /usr/lib/libcap.so.* /lib
ln -sfv ../../lib/$(readlink /usr/lib/libcap.so) /usr/lib/libcap.so

cd /sources
rm -rf libcap-2.24
```
【Sed-4.2.2】
```shell
cd /sources
tar -jxvf sed-4.2.2.tar.bz2
cd sed-4.2.2

./configure --prefix=/usr --bindir=/bin --htmldir=/usr/share/doc/sed-4.2.2
make
make html
make install
make -C doc install-html

cd /sources
rm -rf sed-4.2.2
```
【Shadow-4.2.1】
```shell
cd /sources
tar -xvf shadow-4.2.1.tar.xz
cd shadow-4.2.1

sed -i 's/groups$(EXEEXT) //' src/Makefile.in
find man -name Makefile.in -exec sed -i 's/groups\.1 / /' {} \;

sed -i -e 's@#ENCRYPT_METHOD DES@ENCRYPT_METHOD SHA512@' \
       -e 's@/var/spool/mail@/var/mail@' etc/login.defs

sed -i 's/1000/999/' etc/useradd
./configure --sysconfdir=/etc --with-group-name-max-length=32

make
make install
mv -v /usr/bin/passwd /bin

cd /sources
rm -rf shadow-4.2.1
```
【配置 Shadow】  
运行下面的命令启用 shadow 密码；
```shell
pwconv
```
运行下面的命令启用 shadow 组密码：
```shell
grpconv
```
```shell
sed -i 's/yes/no/' /etc/default/useradd
```
【设置 root 密码】  
运行下面的命令为用户 root 设置密码：
```shell
passwd root
```
【Psmisc-22.21】
```shell
cd /sources
tar -xzvf psmisc-22.21.tar.gz
cd psmisc-22.21

./configure --prefix=/usr
make
make install

mv -v /usr/bin/fuser   /bin
mv -v /usr/bin/killall /bin

cd /sources
rm -rf psmisc-22.21
```
【Procps-ng-3.3.10】
```shell
cd /sources
tar -xvf procps-ng-3.3.10.tar.xz
cd procps-ng-3.3.10

./configure --prefix=/usr                            \
            --exec-prefix=                           \
            --libdir=/usr/lib                        \
            --docdir=/usr/share/doc/procps-ng-3.3.10 \
            --disable-static                         \
            --disable-kill

make

sed -i -r 's|(pmap_initname)\\\$|\1|' testsuite/pmap.test/pmap.exp
make check

make install

mv -v /usr/bin/pidof /bin
mv -v /usr/lib/libprocps.so.* /lib
ln -sfv ../../lib/$(readlink /usr/lib/libprocps.so) /usr/lib/libprocps.so

cd /sources
rm -rf procps-ng-3.3.10
```
【E2fsprogs-1.42.12】
```shell
cd /sources
tar -xzvf e2fsprogs-1.42.12.tar.gz
cd e2fsprogs-1.42.12

sed -e '/int.*old_desc_blocks/s/int/blk64_t/' \
    -e '/if (old_desc_blocks/s/super->s_first_meta_bg/desc_blocks/' \
    -i lib/ext2fs/closefs.c

mkdir -v build
cd build

LIBS=-L/tools/lib                    \
CFLAGS=-I/tools/include              \
PKG_CONFIG_PATH=/tools/lib/pkgconfig \
../configure --prefix=/usr           \
             --bindir=/bin           \
             --with-root-prefix=""   \
             --enable-elf-shlibs     \
             --disable-libblkid      \
             --disable-libuuid       \
             --disable-uuidd         \
             --disable-fsck

make

ln -sfv /tools/lib/lib{blk,uu}id.so.1 lib
make LD_LIBRARY_PATH=/tools/lib check

make install
make install-libs

chmod -v u+w /usr/lib/{libcom_err,libe2p,libext2fs,libss}.a

gunzip -v /usr/share/info/libext2fs.info.gz
install-info --dir-file=/usr/share/info/dir /usr/share/info/libext2fs.info

makeinfo -o      doc/com_err.info ../lib/et/com_err.texinfo
install -v -m644 doc/com_err.info /usr/share/info
install-info --dir-file=/usr/share/info/dir /usr/share/info/com_err.info

cd /sources
rm -rf build
```
【Coreutils-8.23】
```shell
cd /sources
tar -xvf coreutils-8.23.tar.xz
cd coreutils-8.23

patch -Np1 -i ../coreutils-8.23-i18n-1.patch 
touch Makefile.in

FORCE_UNSAFE_CONFIGURE=1 ./configure \
            --prefix=/usr            \
            --enable-no-install-program=kill,uptime

make
make NON_ROOT_USERNAME=nobody check-root
echo "dummy:x:1000:nobody" >> /etc/group
chown -Rv nobody . 
su nobody -s /bin/bash \
          -c "PATH=$PATH make RUN_EXPENSIVE_TESTS=yes check"
sed -i '/dummy/d' /etc/group
make install

mv -v /usr/bin/{cat,chgrp,chmod,chown,cp,date,dd,df,echo} /bin
mv -v /usr/bin/{false,ln,ls,mkdir,mknod,mv,pwd,rm} /bin
mv -v /usr/bin/{rmdir,stty,sync,true,uname} /bin
mv -v /usr/bin/chroot /usr/sbin
mv -v /usr/share/man/man1/chroot.1 /usr/share/man/man8/chroot.8
sed -i s/\"1\"/\"8\"/1 /usr/share/man/man8/chroot.8

mv -v /usr/bin/{head,sleep,nice,test,[} /bin

cd /sources
rm -rf coreutils-8.23
```
【Iana-Etc-2.30】
```shell
cd /sources
tar -jxvf iana-etc-2.30.tar.bz2
cd iana-etc-2.30

make
make install

cd /sources
rm -rf iana-etc-2.30
```
【M4-1.4.17】
```shell
cd /sources
tar -xvf m4-1.4.17.tar.xz
cd m4-1.4.17

./configure --prefix=/usr
make
make install

cd /sources
rm -rf m4-1.4.17
```
【Flex-2.5.39】
```shell
cd /sources
tar -jxvf flex-2.5.39.tar.bz2
cd flex-2.5.39

sed -i -e '/test-bison/d' tests/Makefile.in
./configure --prefix=/usr --docdir=/usr/share/doc/flex-2.5.39

make
make install

ln -sv flex /usr/bin/lex

cd /sources
rm -rf flex-2.5.39
```
【Bison-3.0.4】
```shell
cd /sources
tar -xvf bison-3.0.4.tar.xz
cd bison-3.0.4

./configure --prefix=/usr --docdir=/usr/share/doc/bison-3.0.4
make
make install

cd /sources
rm -rf bison-3.0.4
```
【Grep-2.21】
```shell
cd /sources
tar -xvf grep-2.21.tar.xz
cd grep-2.21

sed -i -e '/tp++/a  if (ep <= tp) break;' src/kwset.c
./configure --prefix=/usr --bindir=/bin
make
make install

cd /sources
rm -rf grep-2.21
```
【Readline-6.3】
```shell
cd /sources
tar -xzvf readline-6.3.tar.gz
cd readline-6.3

patch -Np1 -i ../readline-6.3-upstream_fixes-3.patch

sed -i '/MV.*old/d' Makefile.in
sed -i '/{OLDSUFF}/c:' support/shlib-install

./configure --prefix=/usr --docdir=/usr/share/doc/readline-6.3
make SHLIB_LIBS=-lncurses
make SHLIB_LIBS=-lncurses install

mv -v /usr/lib/lib{readline,history}.so.* /lib
ln -sfv ../../lib/$(readlink /usr/lib/libreadline.so) /usr/lib/libreadline.so
ln -sfv ../../lib/$(readlink /usr/lib/libhistory.so ) /usr/lib/libhistory.so

install -v -m644 doc/*.{ps,pdf,html,dvi} /usr/share/doc/readline-6.3

cd /sources
rm -rf readline-6.3
```
【Bash-4.3.30】
```shell
cd /sources
tar -xzvf bash-4.3.30.tar.gz
cd bash-4.3.30

patch -Np1 -i ../bash-4.3.30-upstream_fixes-1.patch

./configure --prefix=/usr                       \
            --bindir=/bin                       \
            --docdir=/usr/share/doc/bash-4.3.30 \
            --without-bash-malloc               \
            --with-installed-readline
make
make install
exec /bin/bash --login +h

cd /sources
rm -rf bash-4.3.30
```
【Bc-1.06.95】
```shell
cd /sources
tar -jxvf bc-1.06.95.tar.bz2
cd bc-1.06.95

patch -Np1 -i ../bc-1.06.95-memory_leak-1.patch

./configure --prefix=/usr           \
            --with-readline         \
            --mandir=/usr/share/man \
            --infodir=/usr/share/info

make
echo "quit" | ./bc/bc -l Test/checklib.b
make install

cd /sources
rm -rf bc-1.06.95
```
【Libtool-2.4.6】
```shell
cd /sources
tar -xvf libtool-2.4.6.tar.xz
cd libtool-2.4.6

./configure --prefix=/usr
make
make install

cd /sources
rm -rf libtool-2.4.6
```
【GDBM-1.11】
```shell
cd /sources
tar -xzvf gdbm-1.11.tar.gz
cd gdbm-1.11

./configure --prefix=/usr --enable-libgdbm-compat
make
make install

cd /sources
rm -rf gdbm-1.11
```
【Expat-2.1.0】
```shell
cd /sources
tar -xzvf expat-2.1.0.tar.gz
cd expat-2.1.0

./configure --prefix=/usr
make
make install

install -v -dm755 /usr/share/doc/expat-2.1.0
install -v -m644 doc/*.{html,png,css} /usr/share/doc/expat-2.1.0

cd /sources
rm -rf expat-2.1.0
```
【Inetutils-1.9.2】
```shell
cd /sources
tar -xzvf inetutils-1.9.2.tar.gz
cd inetutils-1.9.2

echo '#define PATH_PROCNET_DEV "/proc/net/dev"' >> ifconfig/system/linux.h 

./configure --prefix=/usr        \
            --localstatedir=/var \
            --disable-logger     \
            --disable-whois      \
            --disable-servers

make
make install

mv -v /usr/bin/{hostname,ping,ping6,traceroute} /bin
mv -v /usr/bin/ifconfig /sbin

cd /sources
rm -rf Inetutils-1.9.2
```
【Perl-5.20.2】
```shell
cd /sources
tar -jxvf perl-5.20.2.tar.bz2
cd perl-5.20.2

echo "127.0.0.1 localhost $(hostname)" > /etc/hosts

export BUILD_ZLIB=False
export BUILD_BZIP2=0

sh Configure -des -Dprefix=/usr                 \
                  -Dvendorprefix=/usr           \
                  -Dman1dir=/usr/share/man/man1 \
                  -Dman3dir=/usr/share/man/man3 \
                  -Dpager="/usr/bin/less -isR"  \
                  -Duseshrplib

make

make install
unset BUILD_ZLIB BUILD_BZIP2

cd /sources
rm -rf Perl-5.20.2
```
【XML::Parser-2.44】
```shell
cd /sources
tar -xzvf XML-Parser-2.44.tar.gz
cd XML-Parser-2.44

perl Makefile.PL
make
make install

cd /sources
rm -rf XML-Parser-2.44
```
【Autoconf-2.69】
```shell
cd /sources
tar -xvf autoconf-2.69.tar.xz
cd autoconf-2.69

./configure --prefix=/usr
make
make install

cd /sources
rm -rf autoconf-2.69
```
【Automake-1.15】
```shell
cd /sources
tar -xvf automake-1.15.tar.xz
cd automake-1.15

./configure --prefix=/usr --docdir=/usr/share/doc/automake-1.15
make
sed -i "s:./configure:LEXLIB=/usr/lib/libfl.a &:" t/lex-{clean,depend}-cxx.sh
make install

cd /sources
rm -rf automake-1.15
```
【Diffutils-3.3】
```shell
cd /sources
tar -xvf diffutils-3.3.tar.xz
cd diffutils-3.3

sed -i 's:= @mkdir_p@:= /bin/mkdir -p:' po/Makefile.in.in
./configure --prefix=/usr

make
make install

cd /sources
rm -rf diffutils-3.3
```
【Gawk-4.1.1】
```shell
cd /sources
tar -xvf gawk-4.1.1.tar.xz
cd gawk-4.1.1

./configure --prefix=/usr
make
make install

mkdir -v /usr/share/doc/gawk-4.1.1
cp    -v doc/{awkforai.txt,*.{eps,pdf,jpg}} /usr/share/doc/gawk-4.1.1

cd /sources
rm -rf gawk-4.1.1
```
【Findutils-4.4.2】
```shell
cd /sources
tar -xzvf findutils-4.4.2.tar.gz
cd findutils-4.4.2

./configure --prefix=/usr --localstatedir=/var/lib/locate
make
make install

mv -v /usr/bin/find /bin
sed -i 's|find:=${BINDIR}|find:=/bin|' /usr/bin/updatedb

cd /sources
rm -rf findutils-4.4.2
```
【Gettext-0.19.4】
```shell
cd /sources
tar -xvf gettext-0.19.4.tar.xz
cd gettext-0.19.4

./configure --prefix=/usr --docdir=/usr/share/doc/gettext-0.19.4
make
make install

cd /sources
rm -rf gettext-0.19.4
```
【Intltool-0.50.2】
```shell
cd /sources
tar -xzvf intltool-0.50.2.tar.gz
cd intltool-0.50.2

./configure --prefix=/usr
make

make install
install -v -Dm644 doc/I18N-HOWTO /usr/share/doc/intltool-0.50.2/I18N-HOWTO

cd /sources
rm -rf intltool-0.50.2
```
【Gperf-3.0.4】
```shell
cd /sources
tar -xzvf gperf-3.0.4.tar.gz
cd gperf-3.0.4

./configure --prefix=/usr --docdir=/usr/share/doc/gperf-3.0.4
make
make install

cd /sources
rm -rf gperf-3.0.4
```
【Groff-1.22.3】
```shell
cd /sources
tar -xzvf groff-1.22.3.tar.gz
cd groff-1.22.3

PAGE=<paper_size> ./configure --prefix=/usr
make
make install

cd /sources
rm -rf groff-1.22.3
```
【Xz-5.2.0】
```shell
cd /sources
tar -xvf xz-5.2.0.tar.xz
cd xz-5.2.0

./configure --prefix=/usr --docdir=/usr/share/doc/xz-5.2.0
make
make install
mv -v   /usr/bin/{lzma,unlzma,lzcat,xz,unxz,xzcat} /bin
mv -v /usr/lib/liblzma.so.* /lib
ln -svf ../../lib/$(readlink /usr/lib/liblzma.so) /usr/lib/liblzma.so

cd /sources
rm -rf xz-5.2.0
```
【GRUB-2.02~beta2】
```shell
cd /sources
tar -xvf grub-2.02~beta2.tar.xz
cd grub-2.02~beta2

./configure --prefix=/usr          \
            --sbindir=/sbin        \
            --sysconfdir=/etc      \
            --disable-grub-emu-usb \
            --disable-efiemu       \
            --disable-werror
make
make install

cd /sources
rm -rf grub-2.02~beta2
```
【Less-458】
```shell
cd /sources
tar -xzvf less-458.tar.gz
cd less-458

./configure --prefix=/usr --sysconfdir=/etc
make
make install

cd /sources
rm -rf less-458
```
【Gzip-1.6】
```shell
cd /sources
tar -xvf gzip-1.6.tar.xz
cd gzip-1.6

./configure --prefix=/usr --bindir=/bin
make
make install

mv -v /bin/{gzexe,uncompress,zcmp,zdiff,zegrep} /usr/bin
mv -v /bin/{zfgrep,zforce,zgrep,zless,zmore,znew} /usr/bin

cd /sources
rm -rf gzip-1.6
```
【IPRoute2-3.19.0】
```shell
cd /sources
tar -xvf iproute2-3.19.0.tar.xz
cd iproute2-3.19.0

sed -i '/^TARGETS/s@arpd@@g' misc/Makefile
sed -i /ARPD/d Makefile
sed -i 's/arpd.8//' man/man8/Makefile

make
make DOCDIR=/usr/share/doc/iproute2-3.19.0 install

cd /sources
rm -rf iproute2-3.19.0
```
【Kbd-2.0.2】
```shell
cd /sources
tar -xzvf kbd-2.0.2.tar.gz
cd kbd-2.0.2

patch -Np1 -i ../kbd-2.0.2-backspace-1.patch
sed -i 's/\(RESIZECONS_PROGS=\)yes/\1no/g' configure
sed -i 's/resizecons.8 //' docs/man/man8/Makefile.in
PKG_CONFIG_PATH=/tools/lib/pkgconfig ./configure --prefix=/usr --disable-vlock

make
make install

cd /sources
rm -rf kbd-2.0.2
```
【Kmod-19】
```shell
cd /sources
tar -xvf kmod-19.tar.xz
cd kmod-19

./configure --prefix=/usr          \
            --bindir=/bin          \
            --sysconfdir=/etc      \
            --with-rootlibdir=/lib \
            --with-xz              \
            --with-zlib
make
make install

for target in depmod insmod lsmod modinfo modprobe rmmod; do
  ln -sv ../bin/kmod /sbin/$target
done

ln -sv kmod /bin/lsmod

cd /sources
rm -rf kmod-19
```
【Libpipeline-1.4.0】
```shell
cd /sources
tar -xzvf libpipeline-1.4.0.tar.gz
cd libpipeline-1.4.0

PKG_CONFIG_PATH=/tools/lib/pkgconfig ./configure --prefix=/usr
make
make install

cd /sources
rm -rf libpipeline-1.4.0
```
【Make-4.1】
```shell
cd /sources
tar -jxvf make-4.1.tar.bz2
cd make-4.1

./configure --prefix=/usr
make
make install

cd /sources
rm -rf make-4.1
```
【Patch-2.7.4】
```shell
cd /sources
tar -xvf patch-2.7.4.tar.xz
cd patch-2.7.4

./configure --prefix=/usr
make
make install

cd /sources
rm -rf patch-2.7.4
```
【Systemd-219】
```shell
cd /sources
tar -xvf systemd-219.tar.xz
cd systemd-219

cat > config.cache << "EOF"
KILL=/bin/kill
HAVE_BLKID=1
BLKID_LIBS="-lblkid"
BLKID_CFLAGS="-I/tools/include/blkid"
HAVE_LIBMOUNT=1
MOUNT_LIBS="-lmount"
MOUNT_CFLAGS="-I/tools/include/libmount"
cc_cv_CFLAGS__flto=no
EOF

sed -i "s:blkid/::" $(grep -rl "blkid/blkid.h")
patch -Np1 -i ../systemd-219-compat-1.patch
sed -i "s:test/udev-test.pl ::g" Makefile.in

./configure --prefix=/usr                                           \
            --sysconfdir=/etc                                       \
            --localstatedir=/var                                    \
            --config-cache                                          \
            --with-rootprefix=                                      \
            --with-rootlibdir=/lib                                  \
            --enable-split-usr                                      \
            --disable-gudev                                         \
            --disable-firstboot                                     \
            --disable-ldconfig                                      \
            --disable-sysusers                                      \
            --without-python                                        \
            --docdir=/usr/share/doc/systemd-219                     \
            --with-dbuspolicydir=/etc/dbus-1/system.d               \
            --with-dbussessionservicedir=/usr/share/dbus-1/services \
            --with-dbussystemservicedir=/usr/share/dbus-1/system-services
make LIBRARY_PATH=/tools/lib
make LD_LIBRARY_PATH=/tools/lib install
mv -v /usr/lib/libnss_{myhostname,mymachines,resolve}.so.2 /lib
rm -rfv /usr/lib/rpm
for tool in runlevel reboot shutdown poweroff halt telinit; do
     ln -sfv ../bin/systemctl /sbin/${tool}
done
ln -sfv ../lib/systemd/systemd /sbin/init

sed -i "s:0775 root lock:0755 root root:g" /usr/lib/tmpfiles.d/legacy.conf
sed -i "/pam.d/d" /usr/lib/tmpfiles.d/etc.conf

systemd-machine-id-setup

cd /sources
rm -rf systemd-219
```
【D-Bus-1.8.16】
```shell
cd /sources
tar -xzvf dbus-1.8.16.tar.gz
cd dbus-1.8.16

./configure --prefix=/usr                       \
            --sysconfdir=/etc                   \
            --localstatedir=/var                \
            --docdir=/usr/share/doc/dbus-1.8.16 \
            --with-console-auth-dir=/run/console
make
make install

mv -v /usr/lib/libdbus-1.so.* /lib
ln -sfv ../../lib/$(readlink /usr/lib/libdbus-1.so)/usr/lib/libdbus-1.so

ln -sfv /etc/machine-id /var/lib/dbus

cd /sources
rm -rf dbus-1.8.16
```
【Util-linux-2.26】
```shell
cd /sources
tar -xvf util-linux-2.26.tar.xz
cd util-linux-2.26

mkdir -pv /var/lib/hwclock

./configure ADJTIME_PATH=/var/lib/hwclock/adjtime   \
            --docdir=/usr/share/doc/util-linux-2.26 \
            --disable-chfn-chsh  \
            --disable-login      \
            --disable-nologin    \
            --disable-su         \
            --disable-setpriv    \
            --disable-runuser    \
            --disable-pylibmount \
            --without-python
make

make install

cd /sources
rm -rf util-linux-2.26
```
【Man-DB-2.7.1】
```shell
cd /sources
tar -xvf man-db-2.7.1.tar.xz
cd man-db-2.7.1

./configure --prefix=/usr                        \
            --docdir=/usr/share/doc/man-db-2.7.1 \
            --sysconfdir=/etc                    \
            --disable-setuid                     \
            --with-browser=/usr/bin/lynx         \
            --with-vgrind=/usr/bin/vgrind        \
            --with-grap=/usr/bin/grap
make
make install

sed -i "s:man root:root root:g" /usr/lib/tmpfiles.d/man-db.conf

cd /sources
rm -rf man-db-2.7.1
```
【Tar-1.28】
```shelll
cd /sources
tar -xvf tar-1.28.tar.xz
cd tar-1.28

FORCE_UNSAFE_CONFIGURE=1  \
./configure --prefix=/usr \
            --bindir=/bin

make
make install
make -C doc install-html docdir=/usr/share/doc/tar-1.28

cd /sources
rm -rf tar-1.28
```
【Texinfo-5.2】
```shelll
cd /sources
tar -xvf texinfo-5.2.tar.xz
cd texinfo-5.2

./configure --prefix=/usr
make
make install
make TEXMF=/usr/share/texmf install-tex

pushd /usr/share/info
rm -v dir
for f in *
  do install-info $f dir 2>/dev/null
done
popd

cd /sources
rm -rf texinfo-5.2
```
【Vim-7.4】
```shell
cd /sources
tar -jxvf vim-7.4.tar.bz2
cd vim74

echo '#define SYS_VIMRC_FILE "/etc/vimrc"' >> src/feature.h
./configure --prefix=/usr
make
make install

ln -sv vim /usr/bin/vi
for L in  /usr/share/man/{,*/}man1/vim.1; do
    ln -sv vim.1 $(dirname $L)/vi.1
done

ln -sv ../vim/vim74/doc /usr/share/doc/vim-7.4

cd /sources
rm -rf vim74
```
【设置 Vim】
```shell
cat > /etc/vimrc << "EOF"
" Begin /etc/vimrc

set nocompatible
set backspace=2
syntax on
if (&term == "iterm") || (&term == "putty")
  set background=dark
endif

" End /etc/vimrc
EOF
```
【再次清理无用内容】
```shell
logout

chroot $LFS /tools/bin/env -i            \
    HOME=/root TERM=$TERM PS1='\u:\w\$ ' \
    PATH=/bin:/usr/bin:/sbin:/usr/sbin   \
    /tools/bin/bash --login

/tools/bin/find /{,usr/}{bin,lib,sbin} -type f \
    -exec /tools/bin/strip --strip-debug '{}' ';'

rm -rf /tmp/*
```
【手动或者重启卸载了虚拟内核文件系统，重新进入 chroot 的时候确保挂载了虚拟内核文件系统】
```shell
root:~# exit
logout
root@ubuntu:
```
```shell
mount -v --bind /dev $LFS/dev

mount -vt devpts devpts $LFS/dev/pts -o gid=5,mode=620
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run

chroot "$LFS" /usr/bin/env -i              \
    HOME=/root TERM="$TERM" PS1='\u:\w\$ ' \
    PATH=/bin:/usr/bin:/sbin:/usr/sbin     \
    /bin/bash --login

```
【通用网络配置】
```shell
cat > /etc/systemd/network/10-dhcp-eth0.network << "EOF"
[Match]
Name=eth0

[Network]
DHCP=yes
EOF

cat > /etc/resolv.conf << "EOF"
# Begin /etc/resolv.conf

nameserver 223.5.5.5
nameserver 180.76.76.76

# End /etc/resolv.conf
EOF

ln -sfv /run/systemd/resolve/resolv.conf /etc/resolv.conf

echo "lfs" > /etc/hostname

cat > /etc/hosts << "EOF"
# Begin /etc/hosts (network card version)

127.0.0.1 localhost
::1       localhost

# End /etc/hosts (network card version)
EOF
```
【系统区域设置】
```shell
cat > /etc/locale.conf << "EOF"
LANG=zh_CN.UTF-8
EOF
```
【创建 /etc/inputrc 文件】
```shell
cat > /etc/inputrc << "EOF"
# Begin /etc/inputrc
# Modified by Chris Lynn <roryo@roryo.dynup.net>

# Allow the command prompt to wrap to the next line
set horizontal-scroll-mode Off

# Enable 8bit input
set meta-flag On
set input-meta On

# Turns off 8th bit stripping
set convert-meta Off

# Keep the 8th bit for display
set output-meta On

# none, visible or audible
set bell-style none

# All of the following map the escape sequence of the value
# contained in the 1st argument to the readline specific functions
"\eOd": backward-word
"\eOc": forward-word

# for linux console
"\e[1~": beginning-of-line
"\e[4~": end-of-line
"\e[5~": beginning-of-history
"\e[6~": end-of-history
"\e[3~": delete-char
"\e[2~": quoted-insert

# for xterm
"\eOH": beginning-of-line
"\eOF": end-of-line

# for Konsole
"\e[H": beginning-of-line
"\e[F": end-of-line

# End /etc/inputrc
EOF
```
【创建 /etc/shells 文件】
```shell
cat > /etc/shells << "EOF"
# Begin /etc/shells

/bin/sh
/bin/bash

# End /etc/shells
EOF
```
【Systemd 的用法与配置】
```shell
mkdir -pv /etc/systemd/system/getty@tty1.service.d

cat > /etc/systemd/system/getty@tty1.service.d/noclear.conf << EOF
[Service]
TTYVTDisallocate=no
EOF

ln -sfv /dev/null /etc/systemd/system/tmp.mount

mkdir -pv /etc/systemd/system/foobar.service.d

cat > /etc/systemd/system/foobar.service.d/foobar.conf << EOF
[Service]
Restart=always
RestartSec=30
EOF
```
【创建 /etc/fstab 文件】
```shell
cat > /etc/fstab << "EOF"
/dev/sda     /            ext4    defaults            1     1
EOF
```
【Linux-3.19】
```shell
exit
```
root@ubuntu:~#
```shell
mount -v --bind /dev $LFS/dev
mount -vt devpts devpts $LFS/dev/pts -o gid=5,mode=620
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run
chroot "$LFS" /usr/bin/env -i              \
    HOME=/root TERM="$TERM" PS1='\u:\w\$ ' \
    PATH=/bin:/usr/bin:/sbin:/usr/sbin     \
    /bin/bash --login

cd /sources
tar -xvf linux-3.19.tar.xz
cd linux-3.19

make mrproper
make defconfig

make 
make modules_install

cp -v arch/x86_64/boot/bzImage /boot/vmlinuz-3.19-lfs-7.7-systemd

cp -v System.map /boot/System.map-3.19

cp -v .config /boot/config-3.19

install -v -m755 -d /etc/modprobe.d
cat > /etc/modprobe.d/usb.conf << "EOF"
# Begin /etc/modprobe.d/usb.conf

install ohci_hcd /sbin/modprobe ehci_hcd ; /sbin/modprobe -i ohci_hcd ; true
install uhci_hcd /sbin/modprobe ehci_hcd ; /sbin/modprobe -i uhci_hcd ; true

# End /etc/modprobe.d/usb.conf
EOF

cd /sources
rm -rf linux-3.19
```

【使用 GRUB 设置启动过程】
```
grub-install --force /dev/sda

cat > /boot/grub/grub.cfg << "EOF"
# Begin /boot/grub/grub.cfg
set default=0
set timeout=5

insmod ext2
set root='hd0'

menuentry "GNU/Linux, Linux 3.19-lfs-7.7-systemd" {
        linux   /boot/vmlinuz-3.19-lfs-7.7-systemd root=/dev/sda ro
}
EOF
```
