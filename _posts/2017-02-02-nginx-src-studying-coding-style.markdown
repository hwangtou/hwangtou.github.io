> If I have been able to see further, it was only because I stood on the shoulder of giants.


# 代碼風格與編程技巧（草稿）


// src/core/ngx_log.h 231~238

void ngx_cdecl ngx_log_abort(ngx_err_t err, const char *fmt, ...);
void ngx_cdecl ngx_log_stderr(ngx_err_t err, const char *fmt, ...);
u_char *ngx_log_errno(u_char *buf, u_char *last, ngx_err_t err);
ngx_int_t ngx_log_open_default(ngx_cycle_t *cycle);
ngx_int_t ngx_log_redirect_stderr(ngx_cycle_t *cycle);
ngx_log_t *ngx_log_get_file_log(ngx_log_t *head);
char *ngx_log_set_log(ngx_conf_t *cf, ngx_log_t **head);
```





// ngx/core/ngx_array.h 25~28

ngx_array_t *ngx_array_create(ngx_pool_t *p, ngx_uint_t n, size_t size);
void ngx_array_destroy(ngx_array_t *a);
void *ngx_array_push(ngx_array_t *a);
void *ngx_array_push_n(ngx_array_t *a, ngx_uint_t n);
```

不過，nginx在這方面的規範也沒有到“強迫症”的程度，就像ngx\_list中就沒有對應的釋放方法，這種情況很可能是因為這類功能都是基於ngx\_pool來創建的，ngx\_pool是nginx中用得比較廣泛的內存管理工具，設計目標是為減少內存管理所帶來的痛苦，因為nginx中通常都會為了一個特定的業務創建一個ngx\_pool內存池，並把這個業務中所需要的內存都從這個內存池中申請，當這個任務完成了之後，就直接調用ngx\_pfree函數把整個內存池回收掉，這就不用再為內存洩漏操心了！相比引入GC和引用計數，這個做法雖然看起來不夠geek，但卻十分實用，nginx的代碼哲學中必定有一條：純粹，功能不需要做得面面俱到，該要的接口必須要有，但不該要的功能也一個不加。
// src/core/ngx_palloc.h 75~86

void *ngx_alloc(size_t size, ngx_log_t *log);
void *ngx_calloc(size_t size, ngx_log_t *log);

ngx_pool_t *ngx_create_pool(size_t size, ngx_log_t *log);
void ngx_destroy_pool(ngx_pool_t *pool);
void ngx_reset_pool(ngx_pool_t *pool);

void *ngx_palloc(ngx_pool_t *pool, size_t size);
void *ngx_pnalloc(ngx_pool_t *pool, size_t size);
void *ngx_pcalloc(ngx_pool_t *pool, size_t size);
void *ngx_pmemalign(ngx_pool_t *pool, size_t size, size_t alignment);
ngx_int_t ngx_pfree(ngx_pool_t *pool, void *p);
```

自我反思：誠如nginx的內存管理風格，代碼命名不僅要做到一看即懂，而且還要儘量做到統一，每個功能的類似接口都儘量使用同樣的命名方法和函數簽名，當然接口執行效果也可以類比，這樣的話讀者或者使用者就能減少學習成本和降低出錯的機率了。再往深思考，其實這和面向對象中的面向接口編程的思想有相近之處，C語言也可以很“面向對象”地編程。


// src/os/unix/ngx_files.c 30~75

ssize_t
ngx_read_file(ngx_file_t *file, u_char *buf, size_t size, off_t offset)
{
    ssize_t  n;

    ngx_log_debug4(NGX_LOG_DEBUG_CORE, file->log, 0,
                   "read: %d, %p, %uz, %O", file->fd, buf, size, offset);

#if (NGX_HAVE_PREAD)

    n = pread(file->fd, buf, size, offset);

    if (n == -1) {
        ngx_log_error(NGX_LOG_CRIT, file->log, ngx_errno,
                      "pread() \"%s\" failed", file->name.data);
        return NGX_ERROR;
    }

#else

    if (file->sys_offset != offset) {
        if (lseek(file->fd, offset, SEEK_SET) == -1) {
            ngx_log_error(NGX_LOG_CRIT, file->log, ngx_errno,
                          "lseek() \"%s\" failed", file->name.data);
            return NGX_ERROR;
        }

        file->sys_offset = offset;
    }

    n = read(file->fd, buf, size);

    if (n == -1) {
        ngx_log_error(NGX_LOG_CRIT, file->log, ngx_errno,
                      "read() \"%s\" failed", file->name.data);
        return NGX_ERROR;
    }

    file->sys_offset += n;

#endif

    file->offset += n;

    return n;
}
```
簡而言之，其哲學大概是：不相信調用（尤其是系統調用）必然能返回想要的結果，因此一定要做足夠嚴謹周密的防禦，還要把問題按重要性分別記錄下來，方便開發人員發現、重現和解決問題。

// src/core/ngx_log.h 8~32
#define NGX_LOG_STDERR            0
#define NGX_LOG_EMERG             1
#define NGX_LOG_ALERT             2
#define NGX_LOG_CRIT              3
#define NGX_LOG_ERR               4
#define NGX_LOG_WARN              5
#define NGX_LOG_NOTICE            6
#define NGX_LOG_INFO              7
#define NGX_LOG_DEBUG             8

#define NGX_LOG_DEBUG_CORE        0x010
#define NGX_LOG_DEBUG_ALLOC       0x020
#define NGX_LOG_DEBUG_MUTEX       0x040
#define NGX_LOG_DEBUG_EVENT       0x080
#define NGX_LOG_DEBUG_HTTP        0x100
#define NGX_LOG_DEBUG_MAIL        0x200
#define NGX_LOG_DEBUG_STREAM 
```
順帶一提，雖然不屬於防禦式編程，但nginx也為防止日後有可能發生的系統接口改變而作出防備，在nginx中有很多會直接替換到系統接口的宏定義，如圖，而我認為其目的是添加一層抽象層，讓代碼調用不直接指向系統接口，那麼即使日後該系統接口發生了改變，或者需要兼容特殊操作系統的特殊系統接口，也只需要對抽象層的實現作出修改，那就不會影響到所有相關以來的調用了。
// src/core/ngx_string.h 53~63

#define ngx_strncmp(s1, s2, n)  strncmp((const char *) s1, (const char *) s2, n)


/* msvc and icc7 compile strcmp() to inline loop */
#define ngx_strcmp(s1, s2)  strcmp((const char *) s1, (const char *) s2)


#define ngx_strstr(s1, s2)  strstr((const char *) s1, (const char *) s2)
#define ngx_strlen(s)       strlen((const char *) s)

#define ngx_strchr(s1, c)   strchr((const char *) s1, (int) c)

```
自我反思：今後的編碼的時候要更嚴謹，必須要認真思考接口的行為及其可能會出現的問題，只要有出現問題的概率，就要進行防禦。



// src/os/unix
ngx_darwin.h
ngx_darwin_config.h
ngx_darwin_init.c
ngx_freebsd.h
ngx_freebsd_config.h
ngx_freebsd_init.c
ngx_freebsd_sendfile_chain.c
ngx_linux.h
ngx_linux_aio_read.c
ngx_linux_config.h
ngx_linux_init.c
ngx_linux_sendfile_chain.c
ngx_solaris.h
ngx_solaris_config.h
ngx_solaris_init.c
ngx_solaris_sendfilev_chain.c
...
```

```bash
// auto/sources 189~205

FREEBSD_DEPS="src/os/unix/ngx_freebsd_config.h src/os/unix/ngx_freebsd.h"
FREEBSD_SRCS=src/os/unix/ngx_freebsd_init.c
FREEBSD_SENDFILE_SRCS=src/os/unix/ngx_freebsd_sendfile_chain.c

LINUX_DEPS="src/os/unix/ngx_linux_config.h src/os/unix/ngx_linux.h"
LINUX_SRCS=src/os/unix/ngx_linux_init.c
LINUX_SENDFILE_SRCS=src/os/unix/ngx_linux_sendfile_chain.c


SOLARIS_DEPS="src/os/unix/ngx_solaris_config.h src/os/unix/ngx_solaris.h"
SOLARIS_SRCS=src/os/unix/ngx_solaris_init.c
SOLARIS_SENDFILEV_SRCS=src/os/unix/ngx_solaris_sendfilev_chain.c


DARWIN_DEPS="src/os/unix/ngx_darwin_config.h src/os/unix/ngx_darwin.h"
DARWIN_SRCS=src/os/unix/ngx_darwin_init.c
DARWIN_SENDFILE_SRCS=src/os/unix/ngx_darwin_sendfile_chain.c
```

如圖，nginx也用了同樣的方法來兼容不同平台的事件驅動介面。
SELECT_MODULE=ngx_select_module
SELECT_SRCS=src/event/modules/ngx_select_module.c
WIN32_SELECT_SRCS=src/event/modules/ngx_win32_select_module.c

POLL_MODULE=ngx_poll_module
POLL_SRCS=src/event/modules/ngx_poll_module.c

KQUEUE_MODULE=ngx_kqueue_module
KQUEUE_SRCS=src/event/modules/ngx_kqueue_module.c

DEVPOLL_MODULE=ngx_devpoll_module
DEVPOLL_SRCS=src/event/modules/ngx_devpoll_module.c

EVENTPORT_MODULE=ngx_eventport_module
EVENTPORT_SRCS=src/event/modules/ngx_eventport_module.c

EPOLL_MODULE=ngx_epoll_module
EPOLL_SRCS=src/event/modules/ngx_epoll_module.c

IOCP_MODULE=ngx_iocp_module
IOCP_SRCS=src/event/modules/ngx_iocp_module.c
```
```
// src/event/modules

ngx_devpoll_module.c
ngx_epoll_module.c
ngx_eventport_module.c
ngx_kqueue_module.c
ngx_poll_module.c
ngx_select_module.c
ngx_win32_select_module.c
```
順便一提，nginx還有一種十分厲害的編程方法，巧妙地利用函數指針，實現了類似面向對象語言的“面向接口編程”，思路是把統一同一行為的函數的函數指針，函數指針在初始化的時候賦值到結構體中，而在調用函數時不是通過直接調用函數，而是通過調用結構體中的函數指針，那麼就可以做到底層實現對上層用戶透明了。
例如ngx\_event.h中的ngx\_event\_module\_t結構體，其定義了事件驅動介面所需要的函數指針變量，而在ngx\_kqueue\_module.c文件中則聲明全局變量ngx\_kqueue\_module\_ctx，並對其變量賦以對應的函數指針。在要使用的時候，用戶只需直接用結構體的函數指針變量傳遞正確的參數，即可獲得正確的行為，而無需關心實際調用的函數是什麼，內部的運作原理是怎樣的。也提高了代碼的靈活性，降低了耦合度，即使需要改變或擴展實現，也無需修改本來的函數，只需要增加一個函數簽名相同的函數，把結構體中的函數指針替換掉即可。
// src/event/ngx_event.h 446~453

typedef struct {
    ngx_str_t              *name;

    void                 *(*create_conf)(ngx_cycle_t *cycle);
    char                 *(*init_conf)(ngx_cycle_t *cycle, void *conf);

    ngx_event_actions_t     actions;
} ngx_event_module_t;
```

```c
// src/event/modules/ngx_kqueue_module.c 76~98
ngx_event_module_t  ngx_kqueue_module_ctx = {
    &kqueue_name,
    ngx_kqueue_create_conf,                /* create configuration */
    ngx_kqueue_init_conf,                  /* init configuration */

    {
        ngx_kqueue_add_event,              /* add an event */
        ngx_kqueue_del_event,              /* delete an event */
        ngx_kqueue_add_event,              /* enable an event */
        ngx_kqueue_del_event,              /* disable an event */
        NULL,                              /* add an connection */
        NULL,                              /* delete an connection */
#ifdef EVFILT_USER
        ngx_kqueue_notify,                 /* trigger a notify */
#else
        NULL,                              /* trigger a notify */
#endif
        ngx_kqueue_process_events,         /* process the events */
        ngx_kqueue_init,                   /* init the events */
        ngx_kqueue_done                    /* done the events */
    }

};

```
第二種方式：通過條件編譯，決定不同環境所要編譯的代碼。在nginx成功配置後，會生成objs目錄以及ngx\_auto\_config.h頭文件，在該頭文件中定義了當前操作系統環境的宏定義，如#define NGX\_HAVE\_KQUEUE  1表示使用kqueue作為事件驅動介面，#define NGX\_DARWIN  1表示目前的操作系統內核基於Darwin。而在nginx有大量根據環境相關的宏定義來進行條件編譯的代碼，如圖，在ngx\_thread\_id.c這個實現文件中，nginx定義了不同的系統環境的ngx\_thread\_tid函數的實現，而在預處理的時候通過宏定義來決定哪段代碼才是真正用於編譯的代碼。

// src/os/unix/ngx_thread_id.c 13~70

#if (NGX_LINUX)

/*
 * Linux thread id is a pid of thread created by clone(2),
 * glibc does not provide a wrapper for gettid().
 */

ngx_tid_t
ngx_thread_tid(void)
{
    return syscall(SYS_gettid);
}

#elif (NGX_FREEBSD) && (__FreeBSD_version >= 900031)

#include <pthread_np.h>

ngx_tid_t
ngx_thread_tid(void)
{
    return pthread_getthreadid_np();
}

#elif (NGX_DARWIN)

/*
 * MacOSX thread has two thread ids:
 *
 * 1) MacOSX 10.6 (Snow Leoprad) has pthread_threadid_np() returning
 *    an uint64_t value, which is obtained using the __thread_selfid()
 *    syscall.  It is a number above 300,000.
 */

ngx_tid_t
ngx_thread_tid(void)
{
    uint64_t  tid;

    (void) pthread_threadid_np(NULL, &tid);
    return tid;
}

/*
 * 2) Kernel thread mach_port_t returned by pthread_mach_thread_np().
 *    It is a number in range 100-100,000.
 *
 * return pthread_mach_thread_np(pthread_self());
 */

#else

ngx_tid_t
ngx_thread_tid(void)
{
    return (uint64_t) (uintptr_t) pthread_self();
}

#endif
```

順帶一提，nginx的條件編譯不僅用於解決環境差異問題，還用於debug模式的開關。我曾經思考nginx的宏是否過於氾濫，甚至是否有必要因為宏的危險而不使用宏，但實際上有很多情況我們都無法避免使用宏，例如在ngx\_log中，ngx\_log\_debug簇調試日誌接口，在調試模式的時候ngx\_log\_debug會被替換為ngx\_log\_debug\_core函數，而非調試模式時則被替換為空白，也就是在非調試的情況下，可以減少了一次函數的調用，而不需要進入函數後判斷是否為調試模式，減少不必要的判斷和函數調用棧。

自我反思：不要盡信教科書和網友說的，我們要用理性的方式對待macro，只要不要濫用，使用得當，其實macro是個好東西。很多情況下不能避免要使用到macro，畢竟macro是預處理階段的強大工具，在很多情況下我們並不需要運行時的動態性，只需要編譯的時候確定下來，這個時候我們就應該使用macro，通過降低運行時的動態性來換取更好的運行性能。

## 5. 解決循環引用問題

誰不曾陷入循環引用的苦惱當中，即使儘量把#include放到實現文件中，但也很難避免在頭文件的結構體定義時需要互相引用，如果多次重複的前置聲明（forward declaration）又有可能導致編譯器警告。而nginx的解決方案是我看過的最好的解決方案，既能解決重複引用，又能解決重複前置聲明：

首先我們來思考一下預處理器的處理方式，預處理器會把頭文件按照引用順序整理成一個文件，而循環引用就是因為這個過程中頭文件多次互相引入，導致預處理器拋出異常中斷編譯。Forward declaration是解決這個問題的最佳辦法，而要解決的問題就是多次forward declaration的問題了。Nginx的解決辦法則是用一個總頭文件來進行頭文件（/src/core/ngx\_core.h）引入，有點類似objc項目的pch文件，把所有項目的頭文件都加到了這個預編譯頭文件中，至於如何解決多次前置聲明的問題，nginx在這個總頭文件的開頭為有可能會被不同頭文件使用的結構體統一做了一次前置聲明，那麼就不再需要在其它頭文件中做重複的前置聲明了。

```c
// src/core/ngx_core.h 12~92

#include <ngx_config.h>


typedef struct ngx_module_s      ngx_module_t;
typedef struct ngx_conf_s        ngx_conf_t;
typedef struct ngx_cycle_s       ngx_cycle_t;
typedef struct ngx_pool_s        ngx_pool_t;
typedef struct ngx_chain_s       ngx_chain_t;
typedef struct ngx_log_s         ngx_log_t;
typedef struct ngx_open_file_s   ngx_open_file_t;
typedef struct ngx_command_s     ngx_command_t;
typedef struct ngx_file_s        ngx_file_t;
typedef struct ngx_event_s       ngx_event_t;
typedef struct ngx_event_aio_s   ngx_event_aio_t;
typedef struct ngx_connection_s  ngx_connection_t;

#if (NGX_THREADS)
typedef struct ngx_thread_task_s  ngx_thread_task_t;
#endif

typedef void (*ngx_event_handler_pt)(ngx_event_t *ev);
typedef void (*ngx_connection_handler_pt)(ngx_connection_t *c);


#define  NGX_OK          0
#define  NGX_ERROR      -1
#define  NGX_AGAIN      -2
#define  NGX_BUSY       -3
#define  NGX_DONE       -4
#define  NGX_DECLINED   -5
#define  NGX_ABORT      -6


#include <ngx_errno.h>
#include <ngx_atomic.h>
#include <ngx_thread.h>
#include <ngx_rbtree.h>
#include <ngx_time.h>
#include <ngx_socket.h>
#include <ngx_string.h>
#include <ngx_files.h>
#include <ngx_shmem.h>
#include <ngx_process.h>
#include <ngx_user.h>
#include <ngx_dlopen.h>
#include <ngx_parse.h>
#include <ngx_parse_time.h>
#include <ngx_log.h>
#include <ngx_alloc.h>
#include <ngx_palloc.h>
#include <ngx_buf.h>
#include <ngx_queue.h>
#include <ngx_array.h>
#include <ngx_list.h>
#include <ngx_hash.h>
#include <ngx_file.h>
#include <ngx_crc.h>
#include <ngx_crc32.h>
#include <ngx_murmurhash.h>
#if (NGX_PCRE)
#include <ngx_regex.h>
#endif
#include <ngx_radix_tree.h>
#include <ngx_times.h>
#include <ngx_rwlock.h>
#include <ngx_shmtx.h>
#include <ngx_slab.h>
#include <ngx_inet.h>
#include <ngx_cycle.h>
#include <ngx_resolver.h>
#if (NGX_OPENSSL)
#include <ngx_event_openssl.h>
#endif
#include <ngx_process_cycle.h>
#include <ngx_conf_file.h>
#include <ngx_module.h>
#include <ngx_open_file_cache.h>
#include <ngx_os.h>
#include <ngx_connection.h>
#include <ngx_syslog.h>
#include <ngx_proxy_protocol.h>
```

還有一點比較特別的是，nginx的頭文件引入使用的是尖括號，而不是雙引號。通常，大家在引入頭文件時都習慣直接用雙括號引入，為什麼nginx需要這樣做呢？個人認為可能是因為nginx需要支持第三方擴展開發，第三方開發者需要在代碼中可以引入nginx內部的頭文件，還可能是因為用尖括號的方式引入頭文件，看起來比較好看，但這只是個人的猜想，進一步的原因還有待研究查實。
// auto/sources 8

CORE_INCS="src/core"

```

```bash
// auto/make 241~245

if test -n "$NGX_PCH"; then
    ngx_cc="\$(CC) $ngx_compile_opt \$(CFLAGS) $ngx_use_pch \$(ALL_INCS)"
else
    ngx_cc="\$(CC) $ngx_compile_opt \$(CFLAGS) \$(CORE_INCS)"
fi

```

自我反思：想讓自己的項目代碼結構更加整潔，同時解決循環引用的問題，就應該參考nginx的這種項目結構，同時結合一些編譯指令的技巧，讓項目結構更美觀。