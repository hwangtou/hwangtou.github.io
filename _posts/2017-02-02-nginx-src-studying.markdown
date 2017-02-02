#Nginx源代碼分析
>If I have been able to see further, it was only because I stood on the shoulder of giants.
最近我花了不少時間研究nginx的源代碼，確實讓我受益匪淺。果然，當攀上巨人的肩膀去看這個世界時，風光真的和地上的截然不同。回想起之前我研究過的同為server應用框架的skynet，我有點明白為何skynet被認為是輕量級項目，現在再次讀skynet源代碼時，有種豁然開朗、分外輕鬆的感覺。當然，skynet的代碼也很厲害，其在代碼盡量少的前提下，已經很好地解決她要解決的問題了。事實上，Igor Sysoev和他的團隊真的很厲害，通過對比2004年0.1.0版和2016年最新版1.1.18版的源代碼，我們就會發現nginx的整個項目基本上沒有很大的改動，而且在0.1.0版就已經實現了BSD系kqueue和Linux系epoll的支持，而且在後來的版本還能在不影響項目結構的前提下添加Windows等系統的支持，這是需要多好的設計和遠見才能達成的目標！而我認為我可以把從nginx中學習到的好好總結吸收，從而提升自己的編碼水平。以下是我的學習總結和分享：
##1 代碼風格與編程技巧###1.1	半自文檔式的命名風格大學時，我曾用Objective-C開發iOS應用，行內的人喜歡把Objective-C稱為“自文檔語言”，意思是開發者不需要看開發接口文檔，而是直接看函數命名，就能知道這個函數的作用和調用方式。而nginx也給我這種“自文檔”的感覺，其在很多函數命名都是有很嚴格的規範，讀者和用接口的人可以很容易地知道這個函數會出現在哪個頭文件中，是要達到什麼效果的，例如：ngx\_log\_init()這個函數，ngx說明這是nginx為了和其它第三方模塊劃分開命名空間用的，log表明這是一個ngx\_log.h/c中實現的日誌相關的函數，init則說明這是一個初始化時會用到的函數；而對於類型聲明，如ngx\_list\_t，最後的t表明這是一個類型，ngx\_connection\_log\_error\_e，最後的e表明這是一個枚舉，這可以避免類型和函數引用的混亂，要知道nginx大量地使用函數指針。
```c
// src/core/ngx_log.h 231~238
ngx_log_t *ngx_log_init(u_char *prefix);
void ngx_cdecl ngx_log_abort(ngx_err_t err, const char *fmt, ...);
void ngx_cdecl ngx_log_stderr(ngx_err_t err, const char *fmt, ...);
u_char *ngx_log_errno(u_char *buf, u_char *last, ngx_err_t err);
ngx_int_t ngx_log_open_default(ngx_cycle_t *cycle);
ngx_int_t ngx_log_redirect_stderr(ngx_cycle_t *cycle);
ngx_log_t *ngx_log_get_file_log(ngx_log_t *head);
char *ngx_log_set_log(ngx_conf_t *cf, ngx_log_t **head);
```nginx內部大部分的函數和變量都是很容易明白的，不過也有部分的命名需要一些思考，才能明白作者命名時的想法，例如結構體ngx\_file\_s中的name，需要結合上下文才能發現其實這是文件的整個路徑，而不僅僅是文件名。因此，我認為nginx的命名風格是“半自文檔式”的。
自我反思：代碼命名一定要做到讀者不需要查開發文檔也能直接看懂，而nginx的方法是用一個統一的前綴ngx\_來和第三方代碼（尤其是用戶自己的代碼）劃分開命名空間，用第二個單詞來區分代碼不同的功能，用第三和之後的命名來表示行為。如果是函數命名則不需要後綴，但如果是結構體和枚舉等類型，則用諸如\_s和\_e等後綴結尾。###1.2 讓內存對象的生命週期更好管理類似Objective-C內存管理的代碼風格，alloc函數簇用於申請對象的內存空間，init函數簇用於初始化對象的成員變量，dealloc函數用於釋放內存，同樣，nginx的命名風格也有類似的規範：
在ngx\_array、ngx\_connection等功能實現中，create簇函數後綴意味著創建堆上的空間並調用init函數對空間進行初始化，而destroy函數則用於釋放申請到的空間。```c
// ngx/core/ngx_array.h 25~28

ngx_array_t *ngx_array_create(ngx_pool_t *p, ngx_uint_t n, size_t size);
void ngx_array_destroy(ngx_array_t *a);
void *ngx_array_push(ngx_array_t *a);
void *ngx_array_push_n(ngx_array_t *a, ngx_uint_t n);
```

不過，nginx在這方面的規範也沒有到“強迫症”的程度，就像ngx\_list中就沒有對應的釋放方法，這種情況很可能是因為這類功能都是基於ngx\_pool來創建的，ngx\_pool是nginx中用得比較廣泛的內存管理工具，設計目標是為減少內存管理所帶來的痛苦，因為nginx中通常都會為了一個特定的業務創建一個ngx\_pool內存池，並把這個業務中所需要的內存都從這個內存池中申請，當這個任務完成了之後，就直接調用ngx\_pfree函數把整個內存池回收掉，這就不用再為內存洩漏操心了！相比引入GC和引用計數，這個做法雖然看起來不夠geek，但卻十分實用，nginx的代碼哲學中必定有一條：純粹，功能不需要做得面面俱到，該要的接口必須要有，但不該要的功能也一個不加。```c
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
自我反思：誠如nginx的內存管理風格，代碼命名不僅要做到一看即懂，而且還要儘量做到統一，每個功能的類似接口都儘量使用同樣的命名方法和函數簽名，當然接口執行效果也可以類比，這樣的話讀者或者使用者就能減少學習成本和降低出錯的機率了。再往深思考，其實這和面向對象中的面向接口編程的思想有相近之處，C語言也可以很“面向對象”地編程。###1.3 大量的防禦式編程代碼在我看來，nginx的防禦式代碼不僅多，而且是有層次，並且會伴隨著日誌紀錄。多，是因為每當要執行有可能出錯的操作時都需要作出防禦，而nginx尤其關注內存空間申請失敗，畢竟nginx在要挑戰的是系統性能極限，內存空間申請失敗是必須關注的點，因此nginx在所有涉及到內存申請的地方都做了判斷，另外，涉及系統接口調用方面，也必須對返回的結果做檢查；層次，是因為會發生的問題不一定都是嚴重的，如果防禦沒有層次，一個小小的問題就退出了整個程序，或者一個嚴重的問題也不退出程序，都是不合理的，根據ngx\_log中的定義和代碼，常用的級別有EMERG、ALERT、CRIT、ERR、WARN等等；日誌，是因為當防禦式代碼生效的時候，意味著有正常狀況意外的事情發生，這可能會導致業務的錯誤，如果不把其記錄下來，那麼也許開發人員一直都不會知道有這個錯誤發生，也無法修補錯誤了。```c
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
```c
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
順帶一提，雖然不屬於防禦式編程，但nginx也為防止日後有可能發生的系統接口改變而作出防備，在nginx中有很多會直接替換到系統接口的宏定義，如圖，而我認為其目的是添加一層抽象層，讓代碼調用不直接指向系統接口，那麼即使日後該系統接口發生了改變，或者需要兼容特殊操作系統的特殊系統接口，也只需要對抽象層的實現作出修改，那就不會影響到所有相關以來的調用了。```c
// src/core/ngx_string.h 53~63

#define ngx_strncmp(s1, s2, n)  strncmp((const char *) s1, (const char *) s2, n)


/* msvc and icc7 compile strcmp() to inline loop */
#define ngx_strcmp(s1, s2)  strcmp((const char *) s1, (const char *) s2)


#define ngx_strstr(s1, s2)  strstr((const char *) s1, (const char *) s2)
#define ngx_strlen(s)       strlen((const char *) s)

#define ngx_strchr(s1, c)   strchr((const char *) s1, (int) c)

```
自我反思：今後的編碼的時候要更嚴謹，必須要認真思考接口的行為及其可能會出現的問題，只要有出現問題的概率，就要進行防禦。###1.4 用宏解決環境差異nginx對環境差異問題非常嚴謹。從nginx的源代碼目錄結構來看，os/unix目錄下的代碼是nginx的平台相關代碼，而編譯的配置腳本(configure script)和條件編譯(conditional compile)，則是nginx解決環境差異的主要方式。
第一種方式：通過configure script，決定需要編譯的代碼。如圖，os/unix目錄下有不少的代碼文件是以平台作為區分，如darwin/freebsd/linux/solaris，在auto/source腳本下，nginx定義了不同的環境編譯所需要的代碼文件，當運行configure script的時候，腳本根據所在的系統決定到底要把哪些代碼文件加入到編譯參數中。```
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