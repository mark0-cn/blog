# afl-fuzz分析

## main函数分析

首先通过getopt获取参数

**usage** 输出帮助信息

**setup_signal_handlers** 注册signel

**check_asan_opts** 检测并设置相关环境变量

**fix_up_sync** 同步分布式运行时一些参数

**save_cmdline** 保存命令行参数

**fix_up_banner** 创建一个banner

**check_if_tty** 检测是否在tty上运行

**get_core_count** 获取cpu core数量

**check_crash_handling** 确保核心转储不会进入程序

**check_cpu_governor** 检测cpu属性

**setup_post** 从AFL_POST_LIBRARY中加载afl_postprocess并执行post_handler("hello", &tlen);

**setup_shm** 申请共享内存，用于virgin_bits

**init_count_class16** 初始化count_class_lookup8

**setup_dirs_fds** 创建out_dir和相关文件夹

**read_testcases** 从in_dir中读取测试用例，并处理
+ 当测试用例大于1时，调用**shuffle_ptrs**函数重组
+ **add_to_queue** 将测试用例加入到队列中

**load_auto** 加载额外的一些数据

**pivot_inputs** 为测试用例在out_dir/queue中创建硬链接

**load_extras** 从extras目录中读取extras并按大小排序

**find_timeout** 从fuzzer_stats文件中读取exec_timeout选项（-t 选项）

**detect_file_args** 从命令行中寻找@@符号，用 **.cur_input** 路径代替

**setup_stdio_file** 打开 **.cur_input** 文件

**check_binary** 通过检查文件头和memmem查找magic number方法，检查binary是否是脚本、是否是ELF、是否被插桩、是否使用afl-gcc编译等

**get_cur_time** 获取时间

**get_qemu_argv**

**perform_dry_run**
+ 从queue中读取文件，执行 **calibrate_case**
+ 根据 **calibrate_case** 返回值判断运行结果
``` c
/* Execution status fault codes */

enum {
  /* 00 */ FAULT_NONE,
  /* 01 */ FAULT_TMOUT,
  /* 02 */ FAULT_CRASH,
  /* 03 */ FAULT_ERROR,
  /* 04 */ FAULT_NOINST,
  /* 05 */ FAULT_NOBITS
};
```
**calibrate_case**

1. **init_forkserver** fork 用于分布式通信
2. 将 trace_bits 拷贝到 first_trace
3. 调用 **has_new_bits**
4. **get_cur_time_us** 获取运行开始时间
5. 运行 stage_max 次
   1. **write_to_testcase** 把文件写入到 **.cur_input**
   2. **run_target** 判断是否是分布式运行
   + 否：fork，子进程执行 **execv(target_path, argv)**
   + 是：通过pipe通信
6. 调用 **hash32** 查看是否有新的路径
   + 是：调用 **has_new_bits**
   + 否：将 trace_bits 保存到 first_trace
7. **get_cur_time_us** 获取运行结束时间
8. **update_bitmap_score** 根据 queue 中的每个对象的 exec_us 和 len 计算分数
9. **show_stats** 显示运行相关信息

**cull_queue** 精简队列，该算法选择一个较小的测试用例子集，该子集仍覆盖到目前为止所看到的每个元组，并且其特征使它们对Fuzzing特别有利。该算法通过为每个队列条目分配与其执行延迟和文件大小成正比的分数来工作;然后为每个tuples选择最低得分候选者。

这里需要结合update_bitmap_score()进行理解。update_bitmap_score在trim_case和calibrate_case中被调用，用来维护一个最小(favored)的测试用例集合(top_rated[i])。这里会比较执行时间*种子大小，如果当前用例更小，则会更新top_rated。结合以下事例更容易理解

```
tuple t0,t1,t2,t3,t4；seed s0,s1,s2 初始化temp_v=[1,1,1,1,1]
s1可覆盖t2,t3 | s2覆盖t0,t1,t4，并且top_rated[0]=s2，top_rated[2]=s1
开始后判断temp_v[0]=1，说明t0没有被访问
top_rated[0]存在(s2) -> 判断s2可以覆盖的范围 -> trace_mini=[1,1,0,0,1]
更新temp_v=[0,0,1,1,0]
标记s2为favored
继续判断temp_v[1]=0，说明t1此时已经被访问过了，跳过
继续判断temp_v[2]=1，说明t2没有被访问
top_rated[2]存在(s1) -> 判断s1可以覆盖的范围 -> trace_mini=[0,0,1,1,0]
更新temp_v=[0,0,0,0,0]
标记s1为favored
此时所有tuple都被覆盖，favored为s1,s2
```

**mark_as_redundant** 将非 favored 文件写入到/queue/.state/redundant_edges/

**show_init_stats** 显示一些统计信息

**find_start_position** 恢复上一次执行，只有找到 fuzzer_stats 文件时才生效

**write_stats_file** 创建 outdir/fuzzer_stats，并往里写入运行状态

**save_auto** 更新token

### 主循环

1. **cull_queue** 首先精简队列

2. 根据**find_start_position**函数返回值，做样本偏移

3. **show_stats** 显示运行状态

4. 如果分布式运行，通过**sync_fuzzers**同步状态

5. **fuzz_one** 对testcase进行fuzz

6. 根据一定的频率，调用**sync_fuzzers**同步状态

**write_bitmap** 向 outdir/fuzz_bitmap 写入 virgin_bits

**write_stats_file** 向 outdir/fuzzer_stats 文件中写入运行状态

**save_auto** 保存token

一些销毁函数
``` c
  fclose(plot_file);
  destroy_queue();
  destroy_extras();
  ck_free(target_path);
  ck_free(sync_id);
```

## fuzz_one 函数分析

首先根据 pending_favored 值判断是否有一定几率跳过

``` c
  if (pending_favored) {

    /* If we have any favored, non-fuzzed new arrivals in the queue,
       possibly skip to them at the expense of already-fuzzed or non-favored
       cases. */

    if ((queue_cur->was_fuzzed || !queue_cur->favored) &&
        UR(100) < SKIP_TO_NEW_PROB) return 1;

  } else if (!dumb_mode && !queue_cur->favored && queued_paths > 10) {

    /* Otherwise, still possibly skip non-favored cases, albeit less often.
       The odds of skipping stuff are higher for already-fuzzed inputs and
       lower for never-fuzzed entries. */

    if (queue_cycle > 1 && !queue_cur->was_fuzzed) {

      if (UR(100) < SKIP_NFAV_NEW_PROB) return 1;

    } else {

      if (UR(100) < SKIP_NFAV_OLD_PROB) return 1;

    }

  }
```

之后根据 queue_cur->cal_failed 去执行 **calibrate_case**函数

``` c
// 当 ctrl+c 时，会执行 handle_stop_sig 函数，stop_soon 会置为1
    if (stop_soon || res != crash_mode) {
      cur_skipped_paths++;
      goto abandon_entry;
    }

```

如果测试用例没被裁减过，执行 **trim_case** 函数进行裁剪

``` c
  if (!dumb_mode && !queue_cur->trim_done) {

    u8 res = trim_case(argv, queue_cur, in_buf);

    if (res == FAULT_ERROR)
      FATAL("Unable to execute target application");

    if (stop_soon) {
      cur_skipped_paths++;
      goto abandon_entry;
    }

    /* Don't retry trimming, even if it failed. */

    queue_cur->trim_done = 1;

    if (len != queue_cur->len) len = queue_cur->len;

  }
```

**calculate_score**： 根据当前当测试用例计算得分

根据一些状态，直接跳到 havoc_stage

``` c
  /* Skip right away if -d is given, if we have done deterministic fuzzing on
     this entry ourselves (was_fuzzed), or if it has gone through deterministic
     testing in earlier, resumed runs (passed_det). */

  if (skip_deterministic || queue_cur->was_fuzzed || queue_cur->passed_det)
    goto havoc_stage;

  /* Skip deterministic fuzzing if exec path checksum puts this out of scope
     for this master instance. */

  if (master_max && (queue_cur->exec_cksum % master_max) != master_id - 1)
    goto havoc_stage;
```

### SIMPLE BITFLIP 阶段分析

1. bitflip 1/1(flip1)，执行次数：len << 3。
<br>通过调用 FLIP_BIT 宏实现步长为1的单bit反转

## trim_case 函数分析
先省略，以后再补充

## 相关结构体

### queue_entry
queue_entry 是 queue 中的元素，用于保存测试用例
``` c
struct queue_entry {
/*
fname：
    测试用例名称
len：
    文件长度
*/
  u8* fname;                          /* File name for the test case      */
  u32 len;                            /* Input length                     */

/*
cal_failed:
    每执行一次 calibrate_case 函数，cal_failed++。当遇到其他情况时，cal_failed 会等于 CAL_CHANCES (3)
trim_done:
    每执行一次 trim_case 函数，trim_done 会置为一。表示测试用例被裁减过
was_fuzzed：
    是否被fuzz过，在 fuzz_one 函数末尾会置为1
passed_det:
    通过 out_dir/queue/.state/deterministic_done/ 下是否有该文件，表示是否确定性
    在 mark_as_det_done 函数内置为1，或者通过 add_to_queue 函数的第三个参数 u8 passed_det 赋值保存
*/
  u8  cal_failed,                     /* Calibration failed?              */
      trim_done,                      /* Trimmed?                         */
      was_fuzzed,                     /* Had any fuzzing done yet?        */
      passed_det,                     /* Deterministic stages passed?     */
      has_new_cov,                    /* Triggers new coverage?           */
      var_behavior,                   /* Variable behavior?               */
      favored,                        /* Currently favored?               */
      fs_redundant;                   /* Marked as redundant in the fs?   */
/*
bitmap_size:
    由 count_bytes(trace_bits) 函数计算得来
exec_cksum:
    用于保存 hash32(trace_bits) 的值
*/
  u32 bitmap_size,                    /* Number of bits set in bitmap     */
      exec_cksum;                     /* Checksum of the execution trace  */
/*
exec_us:
    记录执行用时，在 calibrate_case 函数中记录保存
handicap：
    保存该测试用例执行多少轮，在 calibrate_case 函数的第四个参数 u32 handicap 获取保存
depth:
    每当执行一次 add_to_queue 函数时，就会 depth = cur_depth + 1，cur_depth 的值由先前 fuzz_one 函数内 cur_depth = queue_cur->depth 获取保存
*/
  u64 exec_us,                        /* Execution time (us)              */
      handicap,                       /* Number of queue cycles behind    */
      depth;                          /* Path depth                       */

  u8* trace_mini;                     /* Trace bytes, if kept             */
  u32 tc_ref;                         /* Trace bytes ref count            */

  struct queue_entry *next,           /* Next element, if any             */
                     *next_100;       /* 100 elements ahead               */

};
```