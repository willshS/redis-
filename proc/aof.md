# 持久化
[redis持久化](https://redis.io/topics/persistence)官网说明
## aof
首先看一下aof相关变量（server中关于aof的变量）
```
int aof_enabled;                // aof开关
int aof_state;                  /* 状态 AOF_(ON|OFF|WAIT_REWRITE) */
int aof_fsync;                  // 策略 每秒 每query
char *aof_filename;             // aof文件名
int aof_no_fsync_on_rewrite;    /* Don't fsync if a rewrite is in prog. */
int aof_rewrite_perc;           /* Rewrite AOF if % growth is > M and... */
off_t aof_rewrite_min_size;     /* the AOF file is at least N bytes. */
off_t aof_rewrite_base_size;    /* AOF size on latest startup or rewrite. */
off_t aof_current_size;         /* AOF current size. */
off_t aof_fsync_offset;         /* AOF offset which is already synced to disk. */
int aof_flush_sleep;            /* Micros to sleep before flush. (used by tests) */
int aof_rewrite_scheduled;      /* Rewrite once BGSAVE terminates. */
list *aof_rewrite_buf_blocks;   /* Hold changes during an AOF rewrite. */
sds aof_buf;      /* AOF buffer, written before entering the event loop */
int aof_fd;       /* File descriptor of currently selected AOF file */
int aof_selected_db; /* Currently selected DB in AOF */
time_t aof_flush_postponed_start; /* UNIX time of postponed AOF flush */
time_t aof_last_fsync;            /* UNIX time of last fsync() */
time_t aof_rewrite_time_last;   /* Time used by last AOF rewrite run. */
time_t aof_rewrite_time_start;  /* Current AOF rewrite start time. */
int aof_lastbgrewrite_status;   /* C_OK or C_ERR */
unsigned long aof_delayed_fsync;  /* delayed AOF fsync() counter */
int aof_rewrite_incremental_fsync;/* fsync incrementally while aof rewriting? */
int rdb_save_incremental_fsync;   /* fsync incrementally while rdb saving? */
int aof_last_write_status;      /* C_OK or C_ERR */
int aof_last_write_errno;       /* Valid if aof write/fsync status is ERR */
int aof_load_truncated;         /* Don't stop on unexpected AOF EOF. */
int aof_use_rdb_preamble;       /* Use RDB preamble on AOF rewrites. */
redisAtomic int aof_bio_fsync_status; /* Status of AOF fsync in bio job. */
redisAtomic int aof_bio_fsync_errno;  /* Errno of AOF fsync in bio job. */
/* AOF pipes used to communicate between parent and child during rewrite. */
int aof_pipe_write_data_to_child;
int aof_pipe_read_data_from_parent;
int aof_pipe_write_ack_to_parent;
int aof_pipe_read_ack_from_child;
int aof_pipe_write_ack_to_child;
int aof_pipe_read_ack_from_parent;
int aof_stop_sending_diff;     /* If true stop sending accumulated diffs
                                  to child process. */
sds aof_child_diff;             /* AOF diff accumulator child side. */
```
