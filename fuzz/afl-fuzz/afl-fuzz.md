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
