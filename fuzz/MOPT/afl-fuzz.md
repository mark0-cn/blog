## pso_updating 函数分析

``` c
for (tmp_swarm = 0; tmp_swarm < swarm_num; tmp_swarm++)
{
    double x_temp = 0.0;
    for (i = 0; i < operator_num; i++)
    {
        probability_now[tmp_swarm][i] = 0.0;
        v_now[tmp_swarm][i] = w_now * v_now[tmp_swarm][i] + RAND_C * (L_best[tmp_swarm][i] - x_now[tmp_swarm][i]) + RAND_C * (G_best[i] - x_now[tmp_swarm][i]);
        x_now[tmp_swarm][i] += v_now[tmp_swarm][i];
        if (x_now[tmp_swarm][i] > v_max)
            x_now[tmp_swarm][i] = v_max;
        else if (x_now[tmp_swarm][i] < v_min)
            x_now[tmp_swarm][i] = v_min;
        x_temp += x_now[tmp_swarm][i];
    }

    for (i = 0; i < operator_num; i++)
    {
        x_now[tmp_swarm][i] = x_now[tmp_swarm][i] / x_temp;
        if (likely(i != 0))
            probability_now[tmp_swarm][i] = probability_now[tmp_swarm][i - 1] + x_now[tmp_swarm][i];
        else
            probability_now[tmp_swarm][i] = x_now[tmp_swarm][i];
    }
    if (probability_now[tmp_swarm][operator_num - 1] < 0.99 || probability_now[tmp_swarm][operator_num - 1] > 1.01) FATAL("ERROR probability");
}
```

主要是对应着以下公式，计算每一个点的速度。当速度超过最大最小值时，用最大最小值代替（0.05-1.0）。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202307272205469.png)

```c
double x_now[swarm_num][operator_num],
	   L_best[swarm_num][operator_num],
	   eff_best[swarm_num][operator_num],
	   G_best[operator_num],
	   v_now[swarm_num][operator_num],
       probability_now[swarm_num][operator_num],
	   swarm_fitness[swarm_num];

 static u64 stage_finds_puppet[swarm_num][operator_num],           /* Patterns found per fuzz stage    */
            stage_finds_puppet_v2[swarm_num][operator_num],
            stage_cycles_puppet_v2[swarm_num][operator_num],
            stage_cycles_puppet_v3[swarm_num][operator_num],
            stage_cycles_puppet[swarm_num][operator_num],
			operator_finds_puppet[operator_num],
			core_operator_finds_puppet[operator_num],
			core_operator_finds_puppet_v2[operator_num],
			core_operator_cycles_puppet[operator_num],
			core_operator_cycles_puppet_v2[operator_num], 
			core_operator_cycles_puppet_v3[operator_num];          /* Execs per fuzz stage             */
```