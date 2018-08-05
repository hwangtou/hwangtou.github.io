---
layout: post
title:  "nginx源碼學習——configure脚本"
date:   2017-02-02 22:00:00 +0800
categories: 代码分析
---

# 安裝配置腳本（草稿）
在研究nginx之前，我總是覺得configure和Makefile深不可測，自己的項目用得最多的就是CMake。而當我研究nginx之後，我發現configure也並不難，configure其實就是一個bash腳本。接下來讓以configure script作為入口，分析nginx的configure思路。

第1行代碼證明這是一個bash腳本，當我們在執行configure腳本時，shell會用/bin/sh程序來對configure文件進行解析。

第3、4行是版權信息，還有大神Igor Sysoev的“簽名”。

第7、8行的目的是去除shell中的本地化設置（所有LC\_*設置值），讓命令能夠正確地運行。LC for Locale，LC\_ALL是軟件在運行時的語言環境，包括語言（Language）、地域（Territory）和字符集（Code set）。

```bash
// configure 1~12

#!/bin/sh

# Copyright (C) Igor Sysoev
# Copyright (C) Nginx, Inc.


LC_ALL=C
export LC_ALL

. auto/options
. auto/init
. auto/sources

```

第10到12行執行auto目錄下的三個腳本：auto/options腳本，顧名思義，是用於處理configure命令的附加選項。在該腳本的6～166行，先是聲明了一系列的配置相關變量並賦予默認初值。

```bash
// auto/options 6~166
help=no

NGX_PREFIX=
NGX_SBIN_PATH=
NGX_MODULES_PATH=
...

NGX_CPU_CACHE_LINE=
```

168～394行，這是一個用於處理configure所帶參數的for循環，語句

```bash
// auto/options 168~394

NGX_POST_CONF_MSG=

opt=

for option
do
    opt="$opt `echo $option | sed -e \"s/\(--[^=]*=\)\(.* .*\)/\1'\2'/\"`"

    case "$option" in
        -*=*) value=`echo "$option" | sed -e 's/[-_a-zA-Z0-9]*=//'` ;;
           *) value="" ;;
    esac

    case "$option" in
        --help)                          help=yes                   ;;

        --prefix=)                       NGX_PREFIX="!"             ;;
        --prefix=*)                      NGX_PREFIX="$value"        ;;
        --sbin-path=*)
...

        --test-build-devpoll)            NGX_TEST_BUILD_DEVPOLL=YES ;;
        --test-build-eventport)          NGX_TEST_BUILD_EVENTPORT=YES ;;
        --test-build-epoll)              NGX_TEST_BUILD_EPOLL=YES   ;;
        --test-build-solaris-sendfilev)  NGX_TEST_BUILD_SOLARIS_SENDFILEV=YES ;;

        *)
            echo "$0: error: invalid option \"$option\""
            exit 1
        ;;
    esac
done
```

```bash 397~574
// auto/options 

NGX_CONFIGURE="$opt"


if [ $help = yes ]; then

cat << END

  --help                             print this message

  --prefix=PATH                      set installation prefix
  --sbin-path=PATH                   set nginx binary pathname
  --modules-path=PATH                set modules path
  --conf-path=PATH                   set nginx.conf pathname
  --error-log-path=PATH              set error log pathname
...

  --with-openssl=DIR                 set path to OpenSSL library sources
  --with-openssl-opt=OPTIONS         set additional build options for OpenSSL

  --with-debug                       enable debug logging

END

    exit 1
fi
```

餘下的代碼大概是HTTP功能開關的設置，是否為Win32平台編譯的設置，和nginx目錄文件配置。

```bash
if [ $HTTP = NO ]; then
    HTTP_CHARSET=NO
    HTTP_GZIP=NO
    HTTP_SSI=NO
    HTTP_USERID=NO
    HTTP_ACCESS=NO
    HTTP_STATUS=NO
    HTTP_REWRITE=NO
    HTTP_PROXY=NO
    HTTP_FASTCGI=NO
fi


if [ ".$NGX_PLATFORM" = ".win32" ]; then
    NGX_WINE=$WINE
fi


NGX_SBIN_PATH=${NGX_SBIN_PATH:-sbin/nginx}
NGX_MODULES_PATH=${NGX_MODULES_PATH:-modules}
NGX_CONF_PATH=${NGX_CONF_PATH:-conf/nginx.conf}
NGX_CONF_PREFIX=`dirname $NGX_CONF_PATH`
NGX_PID_PATH=${NGX_PID_PATH:-logs/nginx.pid}
NGX_LOCK_PATH=${NGX_LOCK_PATH:-logs/nginx.lock}

if [ ".$NGX_ERROR_LOG_PATH" = ".stderr" ]; then
    NGX_ERROR_LOG_PATH=
else
    NGX_ERROR_LOG_PATH=${NGX_ERROR_LOG_PATH:-logs/error.log}
fi

NGX_HTTP_LOG_PATH=${NGX_HTTP_LOG_PATH:-logs/access.log}
NGX_HTTP_CLIENT_TEMP_PATH=${NGX_HTTP_CLIENT_TEMP_PATH:-client_body_temp}
NGX_HTTP_PROXY_TEMP_PATH=${NGX_HTTP_PROXY_TEMP_PATH:-proxy_temp}
NGX_HTTP_FASTCGI_TEMP_PATH=${NGX_HTTP_FASTCGI_TEMP_PATH:-fastcgi_temp}
NGX_HTTP_UWSGI_TEMP_PATH=${NGX_HTTP_UWSGI_TEMP_PATH:-uwsgi_temp}
NGX_HTTP_SCGI_TEMP_PATH=${NGX_HTTP_SCGI_TEMP_PATH:-scgi_temp}

case ".$NGX_PERL_MODULES" in
    ./*)
    ;;

    .)
    ;;

    *)
        NGX_PERL_MODULES=$NGX_PREFIX/$NGX_PERL_MODULES
    ;;
esac
```

auto/init腳本，定義了Makefile文件路徑變量（NGX\_MAKEFILE）、nginx模塊選項源文件路徑變量（NX\_MODULE\_S\_C）、configure腳本生成的項目頭文件路徑變量（NGX\_AUTO\_HEADERS\_H、NGX\_AUTO\_CONFIG\_H）、自動測試文件路徑變量（NGX\_AUTOTEST）、自動配置錯誤文件路徑變量（NGX\_AUTOCONF\_ERR），以及Makefile文件路徑變量（MAKEFILE）。

```bash
// auto/init 6~21

NGX_MAKEFILE=$NGX_OBJS/Makefile
NGX_MODULES_C=$NGX_OBJS/ngx_modules.c

NGX_AUTO_HEADERS_H=$NGX_OBJS/ngx_auto_headers.h
NGX_AUTO_CONFIG_H=$NGX_OBJS/ngx_auto_config.h

NGX_AUTOTEST=$NGX_OBJS/autotest
NGX_AUTOCONF_ERR=$NGX_OBJS/autoconf.err

# STUBs
NGX_ERR=$NGX_OBJS/autoconf.err
MAKEFILE=$NGX_OBJS/Makefile


NGX_PCH=
NGX_USE_PCH=
```

測試當前環境shell顯示字符串-n和\c的能力，因為不同系統的shell對此存在兼容性差別。

```bash
// auto/init 24~40

# check the echo's "-n" option and "\c" capability

if echo "test\c" | grep c >/dev/null; then

    if echo -n test | grep n >/dev/null; then
        ngx_n=
        ngx_c=

    else
        ngx_n=-n
        ngx_c=
    fi

else
    ngx_n=
    ngx_c='\c'
fi
```

最後，創建Makefile文件，定義make指令和make clean指令的行為。

```bash
// auto/init 43~51

# create Makefile

cat << END > Makefile

default:	build

clean:
	rm -rf Makefile $NGX_OBJS
END
```

auto/sources，定義core模塊、event模塊等模塊的INCS（include path頭文件搜索路徑）、DEPS（depend headers依賴頭文件列表）、SRCS（source files源文件列表）等變量；以及UNIX、FREEBSD、LINUX等不同操作系統的INCS、DEPS、SRCS變量。

```bash
// auto/sources 6~44

CORE_MODULES="ngx_core_module ngx_errlog_module ngx_conf_module"

CORE_INCS="src/core"

CORE_DEPS="src/core/nginx.h \
           src/core/ngx_config.h \
           src/core/ngx_core.h \
           src/core/ngx_log.h \
           src/core/ngx_palloc.h \
...
           src/core/ngx_syslog.h"
```

回到configure文件，第14行，測試objs目錄是否存在，若不存在即創建objs目錄。第16～19行，在命令行上顯示之前配置好的參數。第22～24行，若為調試模式，則往objs/ngx\_auto\_config.h文件尾部加入NGX\_DEBUG為1的宏定義。

```bash
// configure 14~24

test -d $NGX_OBJS || mkdir -p $NGX_OBJS

echo > $NGX_AUTO_HEADERS_H
echo > $NGX_AUTOCONF_ERR

echo "#define NGX_CONFIGURE \"$NGX_CONFIGURE\"" > $NGX_AUTO_CONFIG_H


if [ $NGX_DEBUG = YES ]; then
    have=NGX_DEBUG . auto/have
fi
```

第27到47行，如果NGX\_PLATFORM未定義，則用uname命令獲取系統內核名稱、內核版本、機器架構信息（順帶一提，2>/dev/null的意思是把輸出的stderr流拋棄，/dev/null也被稱為“黑洞”），若獲取到的系統內核名稱為“MINGW32”，則把系統名重命名為win32。

```bash
// configure 27~47

if test -z "$NGX_PLATFORM"; then
    echo "checking for OS"

    NGX_SYSTEM=`uname -s 2>/dev/null`
    NGX_RELEASE=`uname -r 2>/dev/null`
    NGX_MACHINE=`uname -m 2>/dev/null`

    echo " + $NGX_SYSTEM $NGX_RELEASE $NGX_MACHINE"

    NGX_PLATFORM="$NGX_SYSTEM:$NGX_RELEASE:$NGX_MACHINE";

    case "$NGX_SYSTEM" in
        MINGW32_*)
            NGX_PLATFORM=win32
        ;;
    esac

else
    echo "building for $NGX_PLATFORM"
    NGX_SYSTEM=$NGX_PLATFORM
fi
```

第49行執行auto/cc/conf腳本，auto/cc目錄顧名思義就是C編譯器（c compiler）相關配置的目錄。conf腳本為主腳本，定義編譯器相關的變量，然後根據不同的編譯器調用不同的腳本；name腳本用於獲取編譯器名稱；auto/cc目錄下的其它腳本主要用於特定編譯器的配置。

auto/cc/conf腳本，對編譯器的相關變量進行聲明，再執行auto/cc/name腳本，通過判斷目前平台變量（$NGX\_PLATFORM）和編譯器類型變量（$CC）來對相關變量和$NGX\_CC\_NAME進行初始化，並打印當前使用的編譯器信息“using xxx C Compiler”。

```bash
// configure 49

. auto/cc/conf
```

第51到53行，如果NGX_PLATFORM不是定義為win32的平台，則執行auto/header腳本，而這個腳本則通過調用auto/include腳本來檢測unistd.h等腳本在該平台下的可運行性。

```bash
// configure 51~53

if [ "$NGX_PLATFORM" != win32 ]; then
    . auto/headers
fi
```

```bash
// auto/headers 6~13
ngx_include="unistd.h";      . auto/include
ngx_include="inttypes.h";    . auto/include
ngx_include="limits.h";      . auto/include
ngx_include="sys/filio.h";   . auto/include
ngx_include="sys/param.h";   . auto/include
ngx_include="sys/mount.h";   . auto/include
ngx_include="sys/statvfs.h"; . auto/include
ngx_include="crypt.h";       . auto/include
```

auto/include會為傳入的ngx\_include參數生成錯誤紀錄文件和自動測試文件，該測試文件實現簡單的引入頭文件功能並帶有一個main函數，然後用編譯器編譯該自動測試文件，並把編譯命令輸出的結果的錯誤信息重定向輸出到錯誤紀錄文件當中。

```bash
// auto/include 6~58

echo $ngx_n "checking for $ngx_include ...$ngx_c"

cat << END >> $NGX_AUTOCONF_ERR

----------------------------------------
checking for $ngx_include

END


ngx_found=no

cat << END > $NGX_AUTOTEST.c

$NGX_INCLUDE_SYS_PARAM_H
#include <$ngx_include>

int main(void) {
    return 0;
}

END


ngx_test="$CC -o $NGX_AUTOTEST $NGX_AUTOTEST.c"

eval "$ngx_test >> $NGX_AUTOCONF_ERR 2>&1"

if [ -x $NGX_AUTOTEST ]; then

    ngx_found=yes

    echo " found"

    ngx_name=`echo $ngx_include \
              | tr abcdefghijklmnopqrstuvwxyz/. ABCDEFGHIJKLMNOPQRSTUVWXYZ__`


    have=NGX_HAVE_$ngx_name . auto/have_headers

    eval "NGX_INCLUDE_$ngx_name='#include <$ngx_include>'"

else
    echo " not found"

    echo "----------"    >> $NGX_AUTOCONF_ERR
    cat $NGX_AUTOTEST.c  >> $NGX_AUTOCONF_ERR
    echo "----------"    >> $NGX_AUTOCONF_ERR
    echo $ngx_test       >> $NGX_AUTOCONF_ERR
    echo "----------"    >> $NGX_AUTOCONF_ERR
fi

rm -rf $NGX_AUTOTEST*
```

接下來用if [ -x ]來判斷編譯輸出的自動測試可執行文件是否存在且為可執行文件，若是，則表明找到了需要引入的頭文件，執行auto/have\_headers腳本把其加入到$NGX\_AUTO\_HEADERS\_H頭文件中，而這個文件則是objs/ngx\_auto\_config.h文件；否則，記錄到錯誤文件中。最後，清理自動測試相關的所有臨時文件。

```bash
// auto/have_headers 6~12

cat << END >> $NGX_AUTO_HEADERS_H

#ifndef $have
#define $have  1
#endif

END
```

回到configure script，第55行，執行auto/os/conf腳本，該腳本用於初始化不同系統的特性變量，如圖，根據NGX\_PLATFORM，執行不同系統的腳本，該特定腳本會檢測當前操作系統環境的相關特性是否可行，然後auto/have腳本把可用的特性加入到objs/ngx\_auto\_config.h文件中。

```bash
// configure 55

. auto/os/conf
```

```bash
// auto/os/conf 6~78

echo "checking for $NGX_SYSTEM specific features"

case "$NGX_PLATFORM" in

    FreeBSD:*)
        . auto/os/freebsd
    ;;

...

    GNU:*)
        # GNU Hurd
        have=NGX_GNU_HURD . auto/have_headers
        CORE_INCS="$UNIX_INCS"
        CORE_DEPS="$UNIX_DEPS $POSIX_DEPS"
        CORE_SRCS="$UNIX_SRCS"
        CC_AUX_FLAGS="$CC_AUX_FLAGS -D_GNU_SOURCE -D_FILE_OFFSET_BITS=64"
    ;;

    *)
        CORE_INCS="$UNIX_INCS"
        CORE_DEPS="$UNIX_DEPS $POSIX_DEPS"
        CORE_SRCS="$UNIX_SRCS"
    ;;

esac
```

另外，除了系統平台特徵的變量初始化，該腳本還會檢測當前CPU架構的Cache Line和內存對齊特性。


```bash
// auto/os/conf 81~116
case "$NGX_MACHINE" in

    i386 | i686 | i86pc)
        have=NGX_HAVE_NONALIGNED . auto/have
        NGX_MACH_CACHE_LINE=32
    ;;

...

    *)
        have=NGX_ALIGNMENT value=16 . auto/define
        NGX_MACH_CACHE_LINE=32
    ;;

esac

if test -z "$NGX_CPU_CACHE_LINE"; then
    NGX_CPU_CACHE_LINE=$NGX_MACH_CACHE_LINE
fi

have=NGX_CPU_CACHE_LINE value=$NGX_CPU_CACHE_LINE . auto/define
```

回到configure script，第57～59行，若非win32平台，則執行auto/unix腳本，該腳本首先分析了當前unix環境下的用戶和分組，並賦值給變量。

```bash
// configure 57~59

if [ "$NGX_PLATFORM" != win32 ]; then
    . auto/unix
fi
```

```bash
// auto/unix 6~27

NGX_USER=${NGX_USER:-nobody}

if [ -z "$NGX_GROUP" ]; then
    if [ $NGX_USER = nobody ]; then
        if grep nobody /etc/group 2>&1 >/dev/null; then
            echo "checking for nobody group ... found"
            NGX_GROUP=nobody
        else
            echo "checking for nobody group ... not found"

            if grep nogroup /etc/group 2>&1 >/dev/null; then
                echo "checking for nogroup group ... found"
                NGX_GROUP=nogroup
            else
                echo "checking for nogroup group ... not found"
                NGX_GROUP=nobody
            fi
        fi
    else
        NGX_GROUP=$NGX_USER
    fi
fi
```

接下來便是對當前操作系統的unix特徵的一系列的分析，如圖，先賦值給如下變量，然後調用auto/feature腳本，該腳本和上面分析的auto/include腳本同理，通過生成自動測試代碼並編譯之來檢測當前系統是否支持該特性，如果支持，則加入到objs/ngx_auto_config.h文件中，否則則寫入錯誤記錄中。

```bash
// auto/unix 30~46

ngx_feature="poll()"
ngx_feature_name=
ngx_feature_run=no
ngx_feature_incs="#include <poll.h>"
ngx_feature_path=
ngx_feature_libs=
ngx_feature_test="int  n; struct pollfd  pl;
                  pl.fd = 0;
                  pl.events = 0;
                  pl.revents = 0;
                  n = poll(&pl, 1, 0);
                  if (n == -1) return 1"
. auto/feature

if [ $ngx_found = no ]; then
    EVENT_POLL=NONE
fi
```

configure script第61行，執行auto/threads腳本，檢測當前環境對多線程的支持。第62行，執行auto/modules腳本，配置nginx模塊相關的變量並檢測模塊可用性。第63行，執行auto/lib/conf腳本，根據選項配置第三方庫如OpenSSL和zlib等。

```bash
// configure 61~63

. auto/threads
. auto/modules
. auto/lib/conf
```

```bash
// auto/threads 5~20

if [ $USE_THREADS = YES ]; then

    if [ "$NGX_PLATFORM" = win32 ]; then
        cat << END

$0: --with-threads is not supported on Windows

END
        exit 1
    fi

    have=NGX_THREADS . auto/have
    CORE_DEPS="$CORE_DEPS $THREAD_POOL_DEPS"
    CORE_SRCS="$CORE_SRCS $THREAD_POOL_SRCS"
    CORE_LIBS="$CORE_LIBS -lpthread"
fi
```

第65～100行，根據目前環境變量中的變量，調用auto/define腳本，把相關變量加入到objs/ngx_auto_config.h文件中。

```bash
// configure 65~100

case ".$NGX_PREFIX" in
    .)
        NGX_PREFIX=${NGX_PREFIX:-/usr/local/nginx}
        have=NGX_PREFIX value="\"$NGX_PREFIX/\"" . auto/define
    ;;

    .!)
        NGX_PREFIX=
    ;;

    *)
        have=NGX_PREFIX value="\"$NGX_PREFIX/\"" . auto/define
    ;;
esac

if [ ".$NGX_CONF_PREFIX" != "." ]; then
    have=NGX_CONF_PREFIX value="\"$NGX_CONF_PREFIX/\"" . auto/define
fi

have=NGX_SBIN_PATH value="\"$NGX_SBIN_PATH\"" . auto/define
have=NGX_CONF_PATH value="\"$NGX_CONF_PATH\"" . auto/define
have=NGX_PID_PATH value="\"$NGX_PID_PATH\"" . auto/define
have=NGX_LOCK_PATH value="\"$NGX_LOCK_PATH\"" . auto/define
have=NGX_ERROR_LOG_PATH value="\"$NGX_ERROR_LOG_PATH\"" . auto/define

have=NGX_HTTP_LOG_PATH value="\"$NGX_HTTP_LOG_PATH\"" . auto/define
have=NGX_HTTP_CLIENT_TEMP_PATH value="\"$NGX_HTTP_CLIENT_TEMP_PATH\""
. auto/define
have=NGX_HTTP_PROXY_TEMP_PATH value="\"$NGX_HTTP_PROXY_TEMP_PATH\""
. auto/define
have=NGX_HTTP_FASTCGI_TEMP_PATH value="\"$NGX_HTTP_FASTCGI_TEMP_PATH\""
. auto/define
have=NGX_HTTP_UWSGI_TEMP_PATH value="\"$NGX_HTTP_UWSGI_TEMP_PATH\""
. auto/define
have=NGX_HTTP_SCGI_TEMP_PATH value="\"$NGX_HTTP_SCGI_TEMP_PATH\""
. auto/define
```

第102～104行，auto/make腳本為生成Makefile文件的腳本，auto/lib/make為生成第三方庫Makefile文件的腳本，auto/install腳本生成Makefile文件中的install指令相關的操作。

```bash
// configure 102~104

. auto/make
. auto/lib/make
. auto/install
```

而在configure script最後，第116行，打印configure的狀態概況。

```bash
// configure 106~116

# STUB
. auto/stubs

have=NGX_USER value="\"$NGX_USER\"" . auto/define
have=NGX_GROUP value="\"$NGX_GROUP\"" . auto/define

if [ ".$NGX_BUILD" != "." ]; then
    have=NGX_BUILD value="\"$NGX_BUILD\"" . auto/define
fi

. auto/summary
```

```bash
// auto/summary 6~82

echo
echo "Configuration summary"


if [ $USE_THREADS = YES ]; then
    echo "  + using threads"
fi

...

cat << END
  nginx path prefix: "$NGX_PREFIX"
  nginx binary file: "$NGX_SBIN_PATH"
  nginx modules path: "$NGX_MODULES_PATH"
  nginx configuration prefix: "$NGX_CONF_PREFIX"
  nginx configuration file: "$NGX_CONF_PATH"
  nginx pid file: "$NGX_PID_PATH"
END

...

echo "$NGX_POST_CONF_MSG"
```