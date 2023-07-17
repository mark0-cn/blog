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
