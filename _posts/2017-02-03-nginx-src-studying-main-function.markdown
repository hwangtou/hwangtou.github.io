#運行文件和啟動流程（草稿）

nginx執行的main函數位於src/core/nginx.c源文件中。
##1.	函數變量的統一聲明```c
// src/core/nginx.c 187~195

int ngx_cdecl
main(int argc, char *const *argv)
{
    ngx_buf_t        *b;
    ngx_log_t        *log;
    ngx_uint_t        i;
    ngx_cycle_t      *cycle, init_cycle;
    ngx_conf_dump_t  *cd;
    ngx_core_conf_t  *ccf;
```
第190～195行，先是定義一系列的變量，因為為了兼容舊版本C編譯器的標準，避免for循環初始條件中聲明變量等問題，nginx在將在函數中使用到的所有變量都先在函數定義的開始進行聲明。

##2. ngx\_debug\_init()```c
// src/core/nginx.c 197

    ngx_debug_init();
```

第197行，調用ngx\_debug\_init()函數，這是一個非常有趣的內存debug函數，其會為malloc申請到的和free釋放了的堆空間填充特定的字節，那麼在內存調試的時候，就可以通過內存內容知道該內存空間是剛被申請的，或者已被使用過的，還是已經被銷毀的。

而在不同的平台，有不同的實現方法，如圖為os/unix/ngx\_darwin\_init.c即darwin內核操作系統的實現方法，通過設置環境變量MallocScribble為字符串1的方法來實現，而其註釋中也說明在不同的Mac OS X版本中對內存的填充效果也有所不同。然後根據是否為內存填充debug模式來設定ngx\_debug\_malloc的值。

```c
// src/os/unix/ngx_darwin_init.c 63~88

void
ngx_debug_init(void)
{
#if (NGX_DEBUG_MALLOC)

    /*
     * MacOSX 10.6, 10.7:  MallocScribble fills freed memory with 0x55
     *                     and fills allocated memory with 0xAA.
     * MacOSX 10.4, 10.5:  MallocScribble fills freed memory with 0x55,
     *                     MallocPreScribble fills allocated memory with 0xAA.
     * MacOSX 10.3:        MallocScribble fills freed memory with 0x55,
     *                     and no way to fill allocated memory.
     */

    setenv("MallocScribble", "1", 0);

    ngx_debug_malloc = 1;

#else

    if (getenv("MallocScribble")) {
        ngx_debug_malloc = 1;
    }

#endif
}
```

ngx\_debug\_init()函數之於FreeBSD操作系統如圖，代碼的精神大同小異，而FreeBSD操作系統提供了特殊的全局變量malloc\_option，根據不同的版本而不同，而字符串J則是malloc\_option的flag，不同的flag有不同的意義，而J則會令所有通過malloc和realloc申請的內存的每一個字節都會被初始化為0xA5，free釋放的內存也是如此。

而在Linux或其它平台的定義中，則是用宏定義直接把ngx\_debug\_init()直接丟棄，或許是這類型的系統沒有類似的實現方法吧。

```c
// src/os/unix/ngx_freebsd_init.c 76~98

void
ngx_debug_init(void)
{
#if (NGX_DEBUG_MALLOC)

#if __FreeBSD_version >= 500014 && __FreeBSD_version < 1000011
    _malloc_options = "J";
#elif __FreeBSD_version < 500014
    malloc_options = "J";
#endif

    ngx_debug_malloc = 1;

#else
    char  *mo;

    mo = getenv("MALLOC_OPTIONS");

    if (mo && ngx_strchr(mo, 'J')) {
        ngx_debug_malloc = 1;
    }
#endif
}
```

而在Linux或其它平台的定義中，則是用宏定義直接把ngx\_debug\_init()直接丟棄，或許是這類型的系統沒有類似的實現方法吧。
```c
// src/os/unix/ngx_linux_config.h 117

#define ngx_debug_init()
```

##3. ngx\_strerror\_init()與ngx\_errno```c
// src/core/nginx.c 199~201

    if (ngx_strerror_init() != NGX_OK) {
        return 1;
    }
```

第198～200行，nginx調用ngx\_strerror\_init()函數，其作用是初始化nginx的errno描述功能，當ngx\_strerrno\_init()函數成功調用之後，使用者就可以通過ngx\_strerror()方法，獲得errno對應的錯誤描述字符串。
在ngx\_errno.h中，有一系列的errno宏定義，把系統的errno變成nginx的內部表達，同時也用條件編譯的方法來解決不同平台的errno差異，如HP Unix平台下的EAGAIN問題。```c
// src/os/unix/ngx_errno.h 16~56

#define NGX_EPERM         EPERM
#define NGX_ENOENT        ENOENT
#define NGX_ENOPATH       ENOENT
#define NGX_ESRCH         ESRCH
#define NGX_EINTR         EINTR
#define NGX_ECHILD        ECHILD
#define NGX_ENOMEM        ENOMEM
#define NGX_EACCES        EACCES
#define NGX_EBUSY         EBUSY
#define NGX_EEXIST        EEXIST
#define NGX_EEXIST_FILE   EEXIST
#define NGX_EXDEV         EXDEV
#define NGX_ENOTDIR       ENOTDIR
#define NGX_EISDIR        EISDIR
#define NGX_EINVAL        EINVAL
#define NGX_ENFILE        ENFILE
#define NGX_EMFILE        EMFILE
#define NGX_ENOSPC        ENOSPC
#define NGX_EPIPE         EPIPE
#define NGX_EINPROGRESS   EINPROGRESS
#define NGX_ENOPROTOOPT   ENOPROTOOPT
#define NGX_EOPNOTSUPP    EOPNOTSUPP
#define NGX_EADDRINUSE    EADDRINUSE
#define NGX_ECONNABORTED  ECONNABORTED
#define NGX_ECONNRESET    ECONNRESET
#define NGX_ENOTCONN      ENOTCONN
#define NGX_ETIMEDOUT     ETIMEDOUT
#define NGX_ECONNREFUSED  ECONNREFUSED
#define NGX_ENAMETOOLONG  ENAMETOOLONG
#define NGX_ENETDOWN      ENETDOWN
#define NGX_ENETUNREACH   ENETUNREACH
#define NGX_EHOSTDOWN     EHOSTDOWN
#define NGX_EHOSTUNREACH  EHOSTUNREACH
#define NGX_ENOSYS        ENOSYS
#define NGX_ECANCELED     ECANCELED
#define NGX_EILSEQ        EILSEQ
#define NGX_ENOMOREFILES  0
#define NGX_ELOOP         ELOOP
#define NGX_EBADF         EBADF

#if (NGX_HAVE_OPENAT)
#define NGX_EMLINK        EMLINK
#endif

#if (__hpux__)
#define NGX_EAGAIN        EWOULDBLOCK
#else
#define NGX_EAGAIN        EAGAIN
#endif
```在ngx\_strerror\_init函數中，NGX\_SYS\_NERR是在objs/ngx\_auto\_config.h中定義的系統errno最大數目，ngx\_sys\_errlist是源文件中的靜態變量，根據errno最大數目為其創建堆空間，作為ngx\_str數組之用。迭代errno，然後用<string.h>中的strerror函數來獲得對應的錯誤提示，把其複製到新創建的字符串堆空間中，並把該字符串賦值到ngx\_sys\_errlist數組對應的errno位置中。```c
// src/os/unix/ngx_errno.h 16~56

#define NGX_EPERM         EPERM
#define NGX_ENOENT        ENOENT
#define NGX_ENOPATH       ENOENT
#define NGX_ESRCH         ESRCH
#define NGX_EINTR         EINTR
#define NGX_ECHILD        ECHILD
#define NGX_ENOMEM        ENOMEM
#define NGX_EACCES        EACCES
#define NGX_EBUSY         EBUSY
#define NGX_EEXIST        EEXIST
#define NGX_EEXIST_FILE   EEXIST
#define NGX_EXDEV         EXDEV
#define NGX_ENOTDIR       ENOTDIR
#define NGX_EISDIR        EISDIR
#define NGX_EINVAL        EINVAL
#define NGX_ENFILE        ENFILE
#define NGX_EMFILE        EMFILE
#define NGX_ENOSPC        ENOSPC
#define NGX_EPIPE         EPIPE
#define NGX_EINPROGRESS   EINPROGRESS
#define NGX_ENOPROTOOPT   ENOPROTOOPT
#define NGX_EOPNOTSUPP    EOPNOTSUPP
#define NGX_EADDRINUSE    EADDRINUSE
#define NGX_ECONNABORTED  ECONNABORTED
#define NGX_ECONNRESET    ECONNRESET
#define NGX_ENOTCONN      ENOTCONN
#define NGX_ETIMEDOUT     ETIMEDOUT
#define NGX_ECONNREFUSED  ECONNREFUSED
#define NGX_ENAMETOOLONG  ENAMETOOLONG
#define NGX_ENETDOWN      ENETDOWN
#define NGX_ENETUNREACH   ENETUNREACH
#define NGX_EHOSTDOWN     EHOSTDOWN
#define NGX_EHOSTUNREACH  EHOSTUNREACH
#define NGX_ENOSYS        ENOSYS
#define NGX_ECANCELED     ECANCELED
#define NGX_EILSEQ        EILSEQ
#define NGX_ENOMOREFILES  0
#define NGX_ELOOP         ELOOP
#define NGX_EBADF         EBADF

#if (NGX_HAVE_OPENAT)
#define NGX_EMLINK        EMLINK
#endif

#if (__hpux__)
#define NGX_EAGAIN        EWOULDBLOCK
#else
#define NGX_EAGAIN        EAGAIN
#endif
```

```c
// src/os/unix/ngx_errno.c 45~87

ngx_int_t
ngx_strerror_init(void)
{
    char       *msg;
    u_char     *p;
    size_t      len;
    ngx_err_t   err;

    /*
     * ngx_strerror() is not ready to work at this stage, therefore,
     * malloc() is used and possible errors are logged using strerror().
     */

    len = NGX_SYS_NERR * sizeof(ngx_str_t);

    ngx_sys_errlist = malloc(len);
    if (ngx_sys_errlist == NULL) {
        goto failed;
    }

    for (err = 0; err < NGX_SYS_NERR; err++) {
        msg = strerror(err);
        len = ngx_strlen(msg);

        p = malloc(len);
        if (p == NULL) {
            goto failed;
        }

        ngx_memcpy(p, msg, len);
        ngx_sys_errlist[err].len = len;
        ngx_sys_errlist[err].data = p;
    }

    return NGX_OK;

failed:

    err = errno;
    ngx_log_stderr(0, "malloc(%uz) failed (%d: %s)", len, err, strerror(err));

    return NGX_ERROR;
}
```

而當調用ngx\_strerror時，則把傳入的errno當作索引，直接從ngx\_sys\_errlist數組中獲取該錯誤字符串，並防止長度溢出地複製到傳入的字符串指針中。```c
// src/os/unix/ngx_errno.c 32~42

u_char *
ngx_strerror(ngx_err_t err, u_char *errstr, size_t size)
{
    ngx_str_t  *msg;

    msg = ((ngx_uint_t) err < NGX_SYS_NERR) ? &ngx_sys_errlist[err]:
                                              &ngx_unknown_error;
    size = ngx_min(size, msg->len);

    return ngx_cpymem(errstr, msg->data, size);
}
```這樣做除了大大地簡化了調用strerror的流程，減少出錯機率，尤其是字符串複製函數strcpy使用錯誤的概率，而且還減少了庫函數的調用。##4. ngx\_get\_options()與命令行參數獲取```c
// src/core/nginx.c 203~215

    if (ngx_get_options(argc, argv) != NGX_OK) {
        return 1;
    }

    if (ngx_show_version) {
        ngx_show_version_info();

        if (!ngx_test_config) {
            return 0;
        }
    }

    /* TODO */ ngx_max_sockets = -1;
```
第202～212行，執行了ngx\_get\_options()。ngx\_get\_options函數和程序main函數的函數簽名相同，可見該函數就是直接接受main函數的參數為參數，是專門處理命令行參數的函數。

該函數通過for循環迭代傳入的參數，若參數的第一個字符不為“-”，則不合法，返回錯誤；若合法，則用switch語句來執行對應的流程。如圖，部分的命令行參數會改變一些nginx的全局變量，而這些全局變量會在後續的運行中影響執行流程。```c
// src/os/unix/ngx_errno.c 727~852

static ngx_int_t
ngx_get_options(int argc, char *const *argv)
{
    u_char     *p;
    ngx_int_t   i;

    for (i = 1; i < argc; i++) {

        p = (u_char *) argv[i];

        if (*p++ != '-') {
            ngx_log_stderr(0, "invalid option: \"%s\"", argv[i]);
            return NGX_ERROR;
        }

        while (*p) {

            switch (*p++) {

            case '?':
            case 'h':
                ngx_show_version = 1;
                ngx_show_help = 1;
                break;

            case 'v':
                ngx_show_version = 1;
                break;

            case 'V':
                ngx_show_version = 1;
                ngx_show_configure = 1;
                break;

            case 't':
                ngx_test_config = 1;
                break;

            case 'T':
                ngx_test_config = 1;
                ngx_dump_config = 1;
                break;

            case 'q':
                ngx_quiet_mode = 1;
                break;

            case 'p':
                if (*p) {
                    ngx_prefix = p;
                    goto next;
                }

                if (argv[++i]) {
                    ngx_prefix = (u_char *) argv[i];
                    goto next;
                }

                ngx_log_stderr(0, "option \"-p\" requires directory name");
                return NGX_ERROR;

            case 'c':
                if (*p) {
                    ngx_conf_file = p;
                    goto next;
                }

                if (argv[++i]) {
                    ngx_conf_file = (u_char *) argv[i];
                    goto next;
                }

                ngx_log_stderr(0, "option \"-c\" requires file name");
                return NGX_ERROR;

            case 'g':
                if (*p) {
                    ngx_conf_params = p;
                    goto next;
                }

                if (argv[++i]) {
                    ngx_conf_params = (u_char *) argv[i];
                    goto next;
                }

                ngx_log_stderr(0, "option \"-g\" requires parameter");
                return NGX_ERROR;

            case 's':
                if (*p) {
                    ngx_signal = (char *) p;

                } else if (argv[++i]) {
                    ngx_signal = argv[i];

                } else {
                    ngx_log_stderr(0, "option \"-s\" requires parameter");
                    return NGX_ERROR;
                }

                if (ngx_strcmp(ngx_signal, "stop") == 0
                    || ngx_strcmp(ngx_signal, "quit") == 0
                    || ngx_strcmp(ngx_signal, "reopen") == 0
                    || ngx_strcmp(ngx_signal, "reload") == 0)
                {
                    ngx_process = NGX_PROCESS_SIGNALLER;
                    goto next;
                }

                ngx_log_stderr(0, "invalid option: \"-s %s\"", ngx_signal);
                return NGX_ERROR;

            default:
                ngx_log_stderr(0, "invalid option: \"%c\"", *(p - 1));
                return NGX_ERROR;
            }
        }

    next:

        continue;
    }

    return NGX_OK;
}
```

這裡有一個很巧妙的處理。因為nginx的命令除了以“-”開頭之外，還有可能是沒有“-”開頭的參數，但根據每次for循環開始時的判斷，命令的參數會被視為錯誤的指令，然而nginx在可以帶參數的命令做了一個小處理，當命令時需要帶參數的時候，把for循環迭代的指針遞增，把參數讀取到相關的地方，然後跳到下一個循環，而這是for循環就會迭代下一個指令，而不是當前指令的參數。獲取了nginx的命令後，即刻檢查當前是否僅僅為顯示版本的模式，如果是，則直接顯示nginx版本信息然後退出程序。##5. ngx\_time\_init()與nginx時間處理```c
// src/core/nginx.c 217

    ngx_time_init();
```
第217行，調用src/core/ngx\_times.h下的ngx\_time\_init()函數，對nginx的時間相關實現進行了初始化。nginx內置了五種時間格式，因此全局聲明了如圖五個ngx\_str\_t變量。ngx\_time\_init()函數，為這些ngx\_str\_t類型賦值了其長度，而在最後調用ngx\_time\_update()函數，該函數把當前的時間的字符串指賦值到這五個ngx\_str\_t時間變量中。```c
// src/core/ngx_times.h 39~43

extern volatile ngx_str_t    ngx_cached_err_log_time;
extern volatile ngx_str_t    ngx_cached_http_time;
extern volatile ngx_str_t    ngx_cached_http_log_time;
extern volatile ngx_str_t    ngx_cached_http_log_iso8601;
extern volatile ngx_str_t    ngx_cached_syslog_time;
``````c
// src/core/ngx_times.c 62~74

void
ngx_time_init(void)
{
    ngx_cached_err_log_time.len = sizeof("1970/09/28 12:00:00") - 1;
    ngx_cached_http_time.len = sizeof("Mon, 28 Sep 1970 06:00:00 GMT") - 1;
    ngx_cached_http_log_time.len = sizeof("28/Sep/1970:12:00:00 +0600") - 1;
    ngx_cached_http_log_iso8601.len = sizeof("1970-09-28T12:00:00+06:00") - 1;
    ngx_cached_syslog_time.len = sizeof("Sep 28 12:00:00") - 1;

    ngx_cached_time = &cached_time[0];

    ngx_time_update();
}
```nginx使用事件驅動或者多線程來更新當前時間，而每次時間的更新都會調用ngx\_time\_update()函數，該函數會用ngx\_gettimeofday（即gettimeofday系統函數）來獲取當前的時間，然後通過格式化字符串複製，把當前時間的字符串按照格式地複製到這個nginx表達時間的全局ngx\_str\_t變量中。```c
// src/core/ngx_times.c 77~189

void
ngx_time_update(void)
{
    u_char          *p0, *p1, *p2, *p3, *p4;
    ngx_tm_t         tm, gmt;
    time_t           sec;
    ngx_uint_t       msec;
    ngx_time_t      *tp;
    struct timeval   tv;

    if (!ngx_trylock(&ngx_time_lock)) {
        return;
    }

    ngx_gettimeofday(&tv);

    sec = tv.tv_sec;
    msec = tv.tv_usec / 1000;

    ngx_current_msec = (ngx_msec_t) sec * 1000 + msec;

    tp = &cached_time[slot];

    if (tp->sec == sec) {
        tp->msec = msec;
        ngx_unlock(&ngx_time_lock);
        return;
    }

    if (slot == NGX_TIME_SLOTS - 1) {
        slot = 0;
    } else {
        slot++;
    }

    tp = &cached_time[slot];

    tp->sec = sec;
    tp->msec = msec;

    ngx_gmtime(sec, &gmt);


    p0 = &cached_http_time[slot][0];

    (void) ngx_sprintf(p0, "%s, %02d %s %4d %02d:%02d:%02d GMT",
                       week[gmt.ngx_tm_wday], gmt.ngx_tm_mday,
                       months[gmt.ngx_tm_mon - 1], gmt.ngx_tm_year,
                       gmt.ngx_tm_hour, gmt.ngx_tm_min, gmt.ngx_tm_sec);

#if (NGX_HAVE_GETTIMEZONE)

    tp->gmtoff = ngx_gettimezone();
    ngx_gmtime(sec + tp->gmtoff * 60, &tm);

#elif (NGX_HAVE_GMTOFF)

    ngx_localtime(sec, &tm);
    cached_gmtoff = (ngx_int_t) (tm.ngx_tm_gmtoff / 60);
    tp->gmtoff = cached_gmtoff;

#else

    ngx_localtime(sec, &tm);
    cached_gmtoff = ngx_timezone(tm.ngx_tm_isdst);
    tp->gmtoff = cached_gmtoff;

#endif


    p1 = &cached_err_log_time[slot][0];

    (void) ngx_sprintf(p1, "%4d/%02d/%02d %02d:%02d:%02d",
                       tm.ngx_tm_year, tm.ngx_tm_mon,
                       tm.ngx_tm_mday, tm.ngx_tm_hour,
                       tm.ngx_tm_min, tm.ngx_tm_sec);


    p2 = &cached_http_log_time[slot][0];

    (void) ngx_sprintf(p2, "%02d/%s/%d:%02d:%02d:%02d %c%02i%02i",
                       tm.ngx_tm_mday, months[tm.ngx_tm_mon - 1],
                       tm.ngx_tm_year, tm.ngx_tm_hour,
                       tm.ngx_tm_min, tm.ngx_tm_sec,
                       tp->gmtoff < 0 ? '-' : '+',
                       ngx_abs(tp->gmtoff / 60), ngx_abs(tp->gmtoff % 60));

    p3 = &cached_http_log_iso8601[slot][0];

    (void) ngx_sprintf(p3, "%4d-%02d-%02dT%02d:%02d:%02d%c%02i:%02i",
                       tm.ngx_tm_year, tm.ngx_tm_mon,
                       tm.ngx_tm_mday, tm.ngx_tm_hour,
                       tm.ngx_tm_min, tm.ngx_tm_sec,
                       tp->gmtoff < 0 ? '-' : '+',
                       ngx_abs(tp->gmtoff / 60), ngx_abs(tp->gmtoff % 60));

    p4 = &cached_syslog_time[slot][0];

    (void) ngx_sprintf(p4, "%s %2d %02d:%02d:%02d",
                       months[tm.ngx_tm_mon - 1], tm.ngx_tm_mday,
                       tm.ngx_tm_hour, tm.ngx_tm_min, tm.ngx_tm_sec);

    ngx_memory_barrier();

    ngx_cached_time = tp;
    ngx_cached_http_time.data = p0;
    ngx_cached_err_log_time.data = p1;
    ngx_cached_http_log_time.data = p2;
    ngx_cached_http_log_iso8601.data = p3;
    ngx_cached_syslog_time.data = p4;

    ngx_unlock(&ngx_time_lock);
}
```因為幾種時間格式需要顯示當前時區的時間，因此nginx會根據不同的時區配置來計算時間，而時間的格式化則通過ngx\_sprintf函數（即sprintf庫函數）實現格式化。順帶一提，nginx認為固定的開銷比隨著用戶數量增長而增長的開銷要來的實際，就如獲取時間，定期把時間更新到字符串，然後直接從字符串複製當前時間，比每次用戶需要獲取時間再從系統獲取時間要好。即使每毫秒更新一次時間，獲取系統時間的頻率也只不過一秒一千次，而如果有一千個用戶同時在一秒內獲取數次時間，這個頻率已經遠超一千次。而當系統面臨壓力的時候，這種調用對系統性能的影響簡直就是呈正反饋趨勢，很容易就會把系統壓垮。##6. getpid和ngx\_log\_init()```c
// src/core/nginx.c 223~228

    ngx_pid = ngx_getpid();

    log = ngx_log_init(ngx_prefix);
    if (log == NULL) {
        return 1;
    }
```第223行，調用ngx\_getpid()函數，獲取當前進程的ID。其實這是一個位於os/unix/ngx\_proccess.h的宏定義，在預處理時將會被替代成unistd.h庫下的getpid函數。

第225～228行，對ngx\_log模塊進行初始化。傳入的ngx\_prefix為字符指針，指的是nginx當前配置的目錄前綴，但當nginx初次啟動時其指針值為NULL。

ngx\_log\_init()主要是對ngx\_log.c中定義的ngx\_log和ngx\_log\_file變量進行初始化，並返回ngx\_log的指針。ngx\_log\_file變量為ngx\_open\_file\_t類型，亦即是存儲當前打開的文件信息的結構體，在這裡，ngx\_log\_file的指針當即賦值給ngx\_log的file變量中。若NGX\_ERROR\_LOG\_PATH未定義，則無法使用error\_log，因此直接把ngx\_log\_file的fd賦值為ngx\_stderr標準錯誤輸出流，直接返回當前的ngx\_log指針。而後把函數參數prefix和NGX\_ERROR\_LOG\_PATH連結成錯誤日誌文件路徑，若傳入的prefix為NULL，則使用nginx默認的NGX\_PREFIX路徑，然後創建或者打開該錯誤日誌文件，若沒有錯誤，把fd賦值到ngx\_log\_file中，然後返回ngx\_log指針。

```c
// src/core/ngx_log.c 317~400

ngx_log_t *
ngx_log_init(u_char *prefix)
{
    u_char  *p, *name;
    size_t   nlen, plen;

    ngx_log.file = &ngx_log_file;
    ngx_log.log_level = NGX_LOG_NOTICE;

    name = (u_char *) NGX_ERROR_LOG_PATH;

    /*
     * we use ngx_strlen() here since BCC warns about
     * condition is always false and unreachable code
     */

    nlen = ngx_strlen(name);

    if (nlen == 0) {
        ngx_log_file.fd = ngx_stderr;
        return &ngx_log;
    }

    p = NULL;

#if (NGX_WIN32)
    if (name[1] != ':') {
#else
    if (name[0] != '/') {
#endif

        if (prefix) {
            plen = ngx_strlen(prefix);

        } else {
#ifdef NGX_PREFIX
            prefix = (u_char *) NGX_PREFIX;
            plen = ngx_strlen(prefix);
#else
            plen = 0;
#endif
        }

        if (plen) {
            name = malloc(plen + nlen + 2);
            if (name == NULL) {
                return NULL;
            }

            p = ngx_cpymem(name, prefix, plen);

            if (!ngx_path_separator(*(p - 1))) {
                *p++ = '/';
            }

            ngx_cpystrn(p, (u_char *) NGX_ERROR_LOG_PATH, nlen + 1);

            p = name;
        }
    }

    ngx_log_file.fd = ngx_open_file(name, NGX_FILE_APPEND,
                                    NGX_FILE_CREATE_OR_OPEN,
                                    NGX_FILE_DEFAULT_ACCESS);

    if (ngx_log_file.fd == NGX_INVALID_FILE) {
        ngx_log_stderr(ngx_errno,
                       "[alert] could not open error log file: "
                       ngx_open_file_n " \"%s\" failed", name);
#if (NGX_WIN32)
        ngx_event_log(ngx_errno,
                       "could not open error log file: "
                       ngx_open_file_n " \"%s\" failed", name);
#endif

        ngx_log_file.fd = ngx_stderr;
    }

    if (p) {
        ngx_free(p);
    }

    return &ngx_log;
}
```

##7. init_cycle的初始化
第235～255行，對init\_cycle變量進行初步的初始化。
init\_cycle變量在函數棧空間上，對其內存空間進行清零操作，然後把上文獲得的日誌log指針和新創建的內存池pool指針賦值給init\_cycle的變量。

```c
// src/core/nginx.c 235~255

    /*
     * init_cycle->log is required for signal handlers and
     * ngx_process_options()
     */

    ngx_memzero(&init_cycle, sizeof(ngx_cycle_t));
    init_cycle.log = log;
    ngx_cycle = &init_cycle;

    init_cycle.pool = ngx_create_pool(1024, log);
    if (init_cycle.pool == NULL) {
        return 1;
    }

    if (ngx_save_argv(&init_cycle, argc, argv) != NGX_OK) {
        return 1;
    }

    if (ngx_process_options(&init_cycle) != NGX_OK) {
        return 1;
    }
```

ngx\_save\_argv()函數，用於保存main函數傳入的參數。有趣的是，nginx通過條件編譯的方法，區分開FreeBSD和其它平台的實現，FreeBSD僅需直接把參數的指針賦值到變量即可，而其它平台則需要創建堆空間來存放參數變量，這到底是為什麼呢？嗯，有待思考。 ```c
// src/core/nginx.c 855~894

static ngx_int_t
ngx_save_argv(ngx_cycle_t *cycle, int argc, char *const *argv)
{
#if (NGX_FREEBSD)

    ngx_os_argv = (char **) argv;
    ngx_argc = argc;
    ngx_argv = (char **) argv;

#else
    size_t     len;
    ngx_int_t  i;

    ngx_os_argv = (char **) argv;
    ngx_argc = argc;

    ngx_argv = ngx_alloc((argc + 1) * sizeof(char *), cycle->log);
    if (ngx_argv == NULL) {
        return NGX_ERROR;
    }

    for (i = 0; i < argc; i++) {
        len = ngx_strlen(argv[i]) + 1;

        ngx_argv[i] = ngx_alloc(len, cycle->log);
        if (ngx_argv[i] == NULL) {
            return NGX_ERROR;
        }

        (void) ngx_cpystrn((u_char *) ngx_argv[i], (u_char *) argv[i], len);
    }

    ngx_argv[i] = NULL;

#endif

    ngx_os_environ = environ;

    return NGX_OK;
}
```

ngx\_process\_options()函數，與上文ngx\_get\_options()函數相呼應，用於把從命令中獲得的參數賦值到init\_cycle的變量。
在ngx\_get\_process()解析命令時，若用戶用p指令配置了nginx的prefix，那麼ngx\_prefix變量不為NULL，此時把該值賦值到cycle的conf\_prefix和prefix變量中；若用戶沒有使用該命令，則根據objs/ngx\_auto\_config.h頭文件中預設的值來配置這兩個變量。```c
// src/core/nginx.c 897~990

static ngx_int_t
ngx_process_options(ngx_cycle_t *cycle)
{
    u_char  *p;
    size_t   len;

    if (ngx_prefix) {
        len = ngx_strlen(ngx_prefix);
        p = ngx_prefix;

        if (len && !ngx_path_separator(p[len - 1])) {
            p = ngx_pnalloc(cycle->pool, len + 1);
            if (p == NULL) {
                return NGX_ERROR;
            }

            ngx_memcpy(p, ngx_prefix, len);
            p[len++] = '/';
        }

        cycle->conf_prefix.len = len;
        cycle->conf_prefix.data = p;
        cycle->prefix.len = len;
        cycle->prefix.data = p;

    } else {

#ifndef NGX_PREFIX

        p = ngx_pnalloc(cycle->pool, NGX_MAX_PATH);
        if (p == NULL) {
            return NGX_ERROR;
        }

        if (ngx_getcwd(p, NGX_MAX_PATH) == 0) {
            ngx_log_stderr(ngx_errno, "[emerg]: " ngx_getcwd_n " failed");
            return NGX_ERROR;
        }

        len = ngx_strlen(p);

        p[len++] = '/';

        cycle->conf_prefix.len = len;
        cycle->conf_prefix.data = p;
        cycle->prefix.len = len;
        cycle->prefix.data = p;

#else

#ifdef NGX_CONF_PREFIX
        ngx_str_set(&cycle->conf_prefix, NGX_CONF_PREFIX);
#else
        ngx_str_set(&cycle->conf_prefix, NGX_PREFIX);
#endif
        ngx_str_set(&cycle->prefix, NGX_PREFIX);

#endif
    }

    if (ngx_conf_file) {
        cycle->conf_file.len = ngx_strlen(ngx_conf_file);
        cycle->conf_file.data = ngx_conf_file;

    } else {
        ngx_str_set(&cycle->conf_file, NGX_CONF_PATH);
    }

    if (ngx_conf_full_name(cycle, &cycle->conf_file, 0) != NGX_OK) {
        return NGX_ERROR;
    }

    for (p = cycle->conf_file.data + cycle->conf_file.len - 1;
         p > cycle->conf_file.data;
         p--)
    {
        if (ngx_path_separator(*p)) {
            cycle->conf_prefix.len = p - ngx_cycle->conf_file.data + 1;
            cycle->conf_prefix.data = ngx_cycle->conf_file.data;
            break;
        }
    }

    if (ngx_conf_params) {
        cycle->conf_param.len = ngx_strlen(ngx_conf_params);
        cycle->conf_param.data = ngx_conf_params;
    }

    if (ngx_test_config) {
        cycle->log->log_level = NGX_LOG_INFO;
    }

    return NGX_OK;
}```

同理，ngx\_conf\_file、ngx\_conf\_params和ngx\_test\_config分別為-c、-g和-t命令的配置。

##8. 調用ngx\_os\_init()進行平台相關初始化```c
// src/core/nginx.c 257~267

    if (ngx_os_init(log) != NGX_OK) {
        return 1;
    }

    /*
     * ngx_crc32_table_init() requires ngx_cacheline_size set in ngx_os_init()
     */

    if (ngx_crc32_table_init() != NGX_OK) {
        return 1;
    }
```

ngx\_os\_init()函數，用於進行平台相關的初始化動作，用條件編譯的方式來決定是否進行平台差異的初始化，如在darwin內核下即根據os/unix/ngx\_darwin\_init.c來進行初始化，而條件編譯的宏定義與各自平台的os/unix/ngx\_<os>\_config.h頭文件中，這個頭文件在configure期間被決定。而ngx\_init\_setproctitle，則用於設定當前進程的標題。接下來即是獲取ngx\_pagesize系統分頁大小以及CPU相關特徵的代碼。```c
// src/os/unix/ngx_posix_init.c 34~85

ngx_int_t
ngx_os_init(ngx_log_t *log)
{
    ngx_time_t  *tp;
    ngx_uint_t   n;

#if (NGX_HAVE_OS_SPECIFIC_INIT)
    if (ngx_os_specific_init(log) != NGX_OK) {
        return NGX_ERROR;
    }
#endif

    if (ngx_init_setproctitle(log) != NGX_OK) {
        return NGX_ERROR;
    }

    ngx_pagesize = getpagesize();
    ngx_cacheline_size = NGX_CPU_CACHE_LINE;

    for (n = ngx_pagesize; n >>= 1; ngx_pagesize_shift++) { /* void */ }

#if (NGX_HAVE_SC_NPROCESSORS_ONLN)
    if (ngx_ncpu == 0) {
        ngx_ncpu = sysconf(_SC_NPROCESSORS_ONLN);
    }
#endif

    if (ngx_ncpu < 1) {
        ngx_ncpu = 1;
    }

    ngx_cpuinfo();

    if (getrlimit(RLIMIT_NOFILE, &rlmt) == -1) {
        ngx_log_error(NGX_LOG_ALERT, log, errno,
                      "getrlimit(RLIMIT_NOFILE) failed");
        return NGX_ERROR;
    }

    ngx_max_sockets = (ngx_int_t) rlmt.rlim_cur;

#if (NGX_HAVE_INHERITED_NONBLOCK || NGX_HAVE_ACCEPT4)
    ngx_inherited_nonblocking = 1;
#else
    ngx_inherited_nonblocking = 0;
#endif

    tp = ngx_timeofday();
    srandom(((unsigned) ngx_pid << 16) ^ tp->sec ^ tp->msec);

    return NGX_OK;
}
```

在ngx\_os\_init()函數執行之後，nginx才可以執行平台依賴相關的代碼，例如接下來要執行的依賴cacheline\_size的ngx\_crc32\_table\_init()函數。##9. ngx\_add\_inherited\_sockets()與平滑升降級```c
// src/core/nginx.c 269~271

    if (ngx_add_inherited_sockets(&init_cycle) != NGX_OK) {
        return 1;
    }
```首先我們要搞清楚為什麼有添加繼承socket這一步：因為nginx支持熱重啓，那麼在熱重啓之後，nginx想要直接通過監聽socket的fd來實現繼續監聽，而不是關閉本來的fd然後重新創建一個。其實，ngx\_add\_inherited\_sockets還有一個對應的函數——ngx\_exec\_new\_binary，顧名思義，是運行新的二進制文件時調用的函數，也就是平滑升降級nginx時會用到的函數。此函數會把當前監聽的socket文件描述符格式化為以分號為分割的字符串，並把其賦值到以NGINX\_VAR為key的環境變量中。而ngx\_add\_inherited\_sockets則是通過環境變量```c
// src/core/nginx.c 437~497

static ngx_int_t
ngx_add_inherited_sockets(ngx_cycle_t *cycle)
{
    u_char           *p, *v, *inherited;
    ngx_int_t         s;
    ngx_listening_t  *ls;

    inherited = (u_char *) getenv(NGINX_VAR);

    if (inherited == NULL) {
        return NGX_OK;
    }

    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0,
                  "using inherited sockets from \"%s\"", inherited);

    if (ngx_array_init(&cycle->listening, cycle->pool, 10,
                       sizeof(ngx_listening_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    for (p = inherited, v = p; *p; p++) {
        if (*p == ':' || *p == ';') {
            s = ngx_atoi(v, p - v);
            if (s == NGX_ERROR) {
                ngx_log_error(NGX_LOG_EMERG, cycle->log, 0,
                              "invalid socket number \"%s\" in " NGINX_VAR
                              " environment variable, ignoring the rest"
                              " of the variable", v);
                break;
            }

            v = p + 1;

            ls = ngx_array_push(&cycle->listening);
            if (ls == NULL) {
                return NGX_ERROR;
            }

            ngx_memzero(ls, sizeof(ngx_listening_t));

            ls->fd = (ngx_socket_t) s;
        }
    }

    if (v != p) {
        ngx_log_error(NGX_LOG_EMERG, cycle->log, 0,
                      "invalid socket number \"%s\" in " NGINX_VAR
                      " environment variable, ignoring", v);
    }

    ngx_inherited = 1;

    return ngx_set_inherited_sockets(cycle);
}
``````c
// src/core/nginx.c 626~724

ngx_pid_t
ngx_exec_new_binary(ngx_cycle_t *cycle, char *const *argv)
{
    char             **env, *var;
    u_char            *p;
    ngx_uint_t         i, n;
    ngx_pid_t          pid;
    ngx_exec_ctx_t     ctx;
    ngx_core_conf_t   *ccf;
    ngx_listening_t   *ls;

    ngx_memzero(&ctx, sizeof(ngx_exec_ctx_t));

    ctx.path = argv[0];
    ctx.name = "new binary process";
    ctx.argv = argv;

    n = 2;
    env = ngx_set_environment(cycle, &n);
    if (env == NULL) {
        return NGX_INVALID_PID;
    }

    var = ngx_alloc(sizeof(NGINX_VAR)
                    + cycle->listening.nelts * (NGX_INT32_LEN + 1) + 2,
                    cycle->log);
    if (var == NULL) {
        ngx_free(env);
        return NGX_INVALID_PID;
    }

    p = ngx_cpymem(var, NGINX_VAR "=", sizeof(NGINX_VAR));

    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {
        p = ngx_sprintf(p, "%ud;", ls[i].fd);
    }

    *p = '\0';

    env[n++] = var;

#if (NGX_SETPROCTITLE_USES_ENV)

    /* allocate the spare 300 bytes for the new binary process title */

    env[n++] = "SPARE=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
               "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
               "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
               "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
               "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX";

#endif

    env[n] = NULL;

#if (NGX_DEBUG)
    {
    char  **e;
    for (e = env; *e; e++) {
        ngx_log_debug1(NGX_LOG_DEBUG_CORE, cycle->log, 0, "env: %s", *e);
    }
    }
#endif

    ctx.envp = (char *const *) env;

    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

    if (ngx_rename_file(ccf->pid.data, ccf->oldpid.data) == NGX_FILE_ERROR) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      ngx_rename_file_n " %s to %s failed "
                      "before executing new binary process \"%s\"",
                      ccf->pid.data, ccf->oldpid.data, argv[0]);

        ngx_free(env);
        ngx_free(var);

        return NGX_INVALID_PID;
    }

    pid = ngx_execute(cycle, &ctx);

    if (pid == NGX_INVALID_PID) {
        if (ngx_rename_file(ccf->oldpid.data, ccf->pid.data)
            == NGX_FILE_ERROR)
        {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          ngx_rename_file_n " %s back to %s failed after "
                          "an attempt to execute new binary process \"%s\"",
                          ccf->oldpid.data, ccf->pid.data, argv[0]);
        }
    }

    ngx_free(env);
    ngx_free(var);

    return pid;
}
``````c
// auto/install 210~217

upgrade:
	$NGX_SBIN_PATH -t

	kill -USR2 \`cat $NGX_PID_PATH\`
	sleep 1
	test -f $NGX_PID_PATH.oldbin

	kill -QUIT \`cat $NGX_PID_PATH.oldbin\`
```

##10. modules的預加載與init\_cycle的初始化

```c
// src/core/nginx.c 273~285

    if (ngx_preinit_modules() != NGX_OK) {
        return 1;
    }

    cycle = ngx_init_cycle(&init_cycle);
    if (cycle == NULL) {
        if (ngx_test_config) {
            ngx_log_stderr(0, "configuration file %s test failed",
                           init_cycle.conf_file.data);
        }

        return 1;
    }
```
接下來將調用ngx\_init\_cycle()來對init\_cycle變量進行初始化，但是在此之前，先要調用ngx\_preinit\_modules()預初始化所有的模塊。

ngx\_preinit\_modules()函數的定義位於core/ngx\_module.c中，通過遍歷ngx\_modules數組來調用其模塊的初始化函數的函數指針。而ngx\_modules變量位於objs/ngx\_modules.c源文件中，也就是說需要加載的模塊是在configure script執行的時候決定的。

```c
// src/core/ngx_module.c 25~39

ngx_int_t
ngx_preinit_modules(void)
{
    ngx_uint_t  i;

    for (i = 0; ngx_modules[i]; i++) {
        ngx_modules[i]->index = i;
        ngx_modules[i]->name = ngx_module_names[i];
    }

    ngx_modules_n = i;
    ngx_max_module = ngx_modules_n + NGX_MAX_DYNAMIC_MODULES;

    return NGX_OK;
}
```

接下來是ngx\_init\_cycle()函數，其定義於src/core/ngx\_cycle.c，在這個函數的執行過程中，切切實實地對init\_cycle中的大部份變量進行了初始化和賦值，如cycle的log日誌、pool內存池、conf\_*配置文件相關、open\_files正在打開的文件和shared\_memory共享內存等。

第39～53行，函數變量的聲明。

第55～62行，通過環境變量設定的時區，更新當前的時區，同時再用ngx\_timezone\_update()更新一次當前時間，因為當前還沒開始用handler來更新時間，為了獲取當前的最新時間，需要手動調用一次。然後創建內存池。

第73～81行，通過剛才創建的pool內存池來創建cycle的空間，並把pool、log、old\_cycle等變量賦值回cycle結構體中。至於為什麼要把在main函數棧上的init\_cycle變量作為old\_cycle，然後創建新的cycle變量，大概是因為nginx想把cycle從main函數棧空間上改為在內存池中。

第83～111行，把old\_cycle的prefix和conf\_*複製到pool中，使用了pool字符串賦值函數ngx\_pstrdup()函數，即pool string duplicate的意思。

```c
// src/core/ngx_cycle.c 36~111

ngx_cycle_t *
ngx_init_cycle(ngx_cycle_t *old_cycle)
{
    void                *rv;
    char               **senv;
    ngx_uint_t           i, n;
    ngx_log_t           *log;
    ngx_time_t          *tp;
    ngx_conf_t           conf;
    ngx_pool_t          *pool;
    ngx_cycle_t         *cycle, **old;
    ngx_shm_zone_t      *shm_zone, *oshm_zone;
    ngx_list_part_t     *part, *opart;
    ngx_open_file_t     *file;
    ngx_listening_t     *ls, *nls;
    ngx_core_conf_t     *ccf, *old_ccf;
    ngx_core_module_t   *module;
    char                 hostname[NGX_MAXHOSTNAMELEN];

    ngx_timezone_update();

    /* force localtime update with a new timezone */

    tp = ngx_timeofday();
    tp->sec = 0;

    ngx_time_update();


    log = old_cycle->log;

    pool = ngx_create_pool(NGX_CYCLE_POOL_SIZE, log);
    if (pool == NULL) {
        return NULL;
    }
    pool->log = log;

    cycle = ngx_pcalloc(pool, sizeof(ngx_cycle_t));
    if (cycle == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->pool = pool;
    cycle->log = log;
    cycle->old_cycle = old_cycle;

    cycle->conf_prefix.len = old_cycle->conf_prefix.len;
    cycle->conf_prefix.data = ngx_pstrdup(pool, &old_cycle->conf_prefix);
    if (cycle->conf_prefix.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->prefix.len = old_cycle->prefix.len;
    cycle->prefix.data = ngx_pstrdup(pool, &old_cycle->prefix);
    if (cycle->prefix.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    cycle->conf_file.len = old_cycle->conf_file.len;
    cycle->conf_file.data = ngx_pnalloc(pool, old_cycle->conf_file.len + 1);
    if (cycle->conf_file.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }
    ngx_cpystrn(cycle->conf_file.data, old_cycle->conf_file.data,
                old_cycle->conf_file.len + 1);

    cycle->conf_param.len = old_cycle->conf_param.len;
    cycle->conf_param.data = ngx_pstrdup(pool, &old_cycle->conf_param);
    if (cycle->conf_param.data == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }
```

##11. 測試模式與信號處理相關

```c
// src/core/nginx.c 287~311

    if (ngx_test_config) {
        if (!ngx_quiet_mode) {
            ngx_log_stderr(0, "configuration file %s test is successful",
                           cycle->conf_file.data);
        }

        if (ngx_dump_config) {
            cd = cycle->config_dump.elts;

            for (i = 0; i < cycle->config_dump.nelts; i++) {

                ngx_write_stdout("# configuration file ");
                (void) ngx_write_fd(ngx_stdout, cd[i].name.data,
                                    cd[i].name.len);
                ngx_write_stdout(":" NGX_LINEFEED);

                b = cd[i].buffer;

                (void) ngx_write_fd(ngx_stdout, b->pos, b->last - b->pos);
                ngx_write_stdout(NGX_LINEFEED);
            }
        }

        return 0;
    }
```第287～311行，當為測試模式時進入該段代碼，因為之前在初始化cycle的時候已經對配置文件進行了解析，所以運行到這裡時，配置文件必定是沒有問題的。而如果配置了ngx\_dump\_config，證明配置文件需要被打印，用for循環把所有配置文件路徑打印到標準輸出流中。最後退出整個應用，因為測試模式的進程的任務已經完成。

第313～315行，如果有信號需要處理，則調用定義於ngx\_cycle.c源文件的的ngx\_signal\_process()函數，該函數通過ccf配置文件變量結構體中的pid文件路徑字符串變量，嘗試打開紀錄了pid的文件並把紀錄的pid反序列化成int類型，如果失敗，則返回1；如果成功，則把pid和信號字符串作為參數，調用ngx\_os\_signal\_process()函數。
```c
// src/core/nginx.c 313~315

    if (ngx_signal) {
        return ngx_signal_process(cycle, ngx_signal);
    }
```
```c
// src/core/ngx_cycle.c 997~1048

ngx_int_t
ngx_signal_process(ngx_cycle_t *cycle, char *sig)
{
    ssize_t           n;
    ngx_pid_t         pid;
    ngx_file_t        file;
    ngx_core_conf_t  *ccf;
    u_char            buf[NGX_INT64_LEN + 2];

    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "signal process started");

    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

    ngx_memzero(&file, sizeof(ngx_file_t));

    file.name = ccf->pid;
    file.log = cycle->log;

    file.fd = ngx_open_file(file.name.data, NGX_FILE_RDONLY,
                            NGX_FILE_OPEN, NGX_FILE_DEFAULT_ACCESS);

    if (file.fd == NGX_INVALID_FILE) {
        ngx_log_error(NGX_LOG_ERR, cycle->log, ngx_errno,
                      ngx_open_file_n " \"%s\" failed", file.name.data);
        return 1;
    }

    n = ngx_read_file(&file, buf, NGX_INT64_LEN + 2, 0);

    if (ngx_close_file(file.fd) == NGX_FILE_ERROR) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      ngx_close_file_n " \"%s\" failed", file.name.data);
    }

    if (n == NGX_ERROR) {
        return 1;
    }

    while (n-- && (buf[n] == CR || buf[n] == LF)) { /* void */ }

    pid = ngx_atoi(buf, ++n);

    if (pid == (ngx_pid_t) NGX_ERROR) {
        ngx_log_error(NGX_LOG_ERR, cycle->log, 0,
                      "invalid PID number \"%*s\" in \"%s\"",
                      n, buf, file.name.data);
        return 1;
    }

    return ngx_os_signal_process(cycle, sig, pid);

}
```

而ngx\_os\_signal\_process函數，則會迭代signals數組，找出和傳入的信號字符串一樣的信號，然後用kill對該pid的進程發出對應的終端信號，然後返回1。無論如何，回到main函數後都會馬上退出，因為這種用於處理信號的模式將作用於原有的程序。
```c
// src/core/nginx.c 313~315

ngx_int_t
ngx_os_signal_process(ngx_cycle_t *cycle, char *name, ngx_pid_t pid)
{
    ngx_signal_t  *sig;

    for (sig = signals; sig->signo != 0; sig++) {
        if (ngx_strcmp(name, sig->name) == 0) {
            if (kill(pid, sig->signo) != -1) {
                return 0;
            }

            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "kill(%P, %d) failed", pid, sig->signo);
        }
    }

    return 1;
}
```接下來分析：
```c
// src/core/nginx.c 317~345

    ngx_os_status(cycle->log);

    ngx_cycle = cycle;

    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

    if (ccf->master && ngx_process == NGX_PROCESS_SINGLE) {
        ngx_process = NGX_PROCESS_MASTER;
    }

#if !(NGX_WIN32)

    if (ngx_init_signals(cycle->log) != NGX_OK) {
        return 1;
    }

    if (!ngx_inherited && ccf->daemon) {
        if (ngx_daemon(cycle->log) != NGX_OK) {
            return 1;
        }

        ngx_daemonized = 1;
    }

    if (ngx_inherited) {
        ngx_daemonized = 1;
    }

#endif
```

第317行，ngx\_os\_status()，顯示編譯器和系統特徵的相關信息。

第319行，把初始化得到的在內存池上的cycle指針賦值到ngx\_cycle全局變量中。

第321～325行，檢查是否為master模式，如果是則賦值給ngx\_process變量。

第329～331行，調用在src/core/ngx\_process.c上定義的ngx\_init\_signals()函數，使用signals信號結構體數組的變量來對信號進行初始化，signals是在該源文件聲明和定義的內部變量，用於存放不同的信號的名稱和執行函數指針等。

```c
// src/os/unix/ngx_process.c 283~306

ngx_int_t
ngx_init_signals(ngx_log_t *log)
{
    ngx_signal_t      *sig;
    struct sigaction   sa;

    for (sig = signals; sig->signo != 0; sig++) {
        ngx_memzero(&sa, sizeof(struct sigaction));
        sa.sa_handler = sig->handler;
        sigemptyset(&sa.sa_mask);
        if (sigaction(sig->signo, &sa, NULL) == -1) {
#if (NGX_VALGRIND)
            ngx_log_error(NGX_LOG_ALERT, log, ngx_errno,
                          "sigaction(%s) failed, ignored", sig->signame);
#else
            ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                          "sigaction(%s) failed", sig->signame);
            return NGX_ERROR;
#endif
        }
    }

    return NGX_OK;
}
```

第333～343行，如果不是inherited繼承而來的而且需要daemon守護，則調用src/os/unix/ngx_daemon.c中定義的ngx_daemon()函數。```c
// src/os/unix/ngx_process.c 333~343

    if (!ngx_inherited && ccf->daemon) {
        if (ngx_daemon(cycle->log) != NGX_OK) {
            return 1;
        }

        ngx_daemonized = 1;
    }

    if (ngx_inherited) {
        ngx_daemonized = 1;
    }
```

守護的是通過fork來實現的，調用fork()函數，如果返回了-1，則fork失敗，返回NGX_ERROR；如果返回0，則fork成功，跳出switch；如果是其它值，必然是操作系統出錯，直接退出程序。```c
// src/os/unix/ngx_daemon.c 12~70

ngx_int_t
ngx_daemon(ngx_log_t *log)
{
    int  fd;

    switch (fork()) {
    case -1:
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "fork() failed");
        return NGX_ERROR;

    case 0:
        break;

    default:
        exit(0);
    }

    ngx_pid = ngx_getpid();

    if (setsid() == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "setsid() failed");
        return NGX_ERROR;
    }

    umask(0);

    fd = open("/dev/null", O_RDWR);
    if (fd == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                      "open(\"/dev/null\") failed");
        return NGX_ERROR;
    }

    if (dup2(fd, STDIN_FILENO) == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "dup2(STDIN) failed");
        return NGX_ERROR;
    }

    if (dup2(fd, STDOUT_FILENO) == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "dup2(STDOUT) failed");
        return NGX_ERROR;
    }

#if 0
    if (dup2(fd, STDERR_FILENO) == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "dup2(STDERR) failed");
        return NGX_ERROR;
    }
#endif

    if (fd > STDERR_FILENO) {
        if (close(fd) == -1) {
            ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "close() failed");
            return NGX_ERROR;
        }
    }

    return NGX_OK;
}
```