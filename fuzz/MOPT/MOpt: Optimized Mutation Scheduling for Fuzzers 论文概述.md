### 3.2.2 Local Best Position L_best

Lbest is the position of the particle where the corresponding operator yields the most interesting test cases (given the same amount of invocations).

for each particle (i.e., operator), we measure its local efficiency e f fnow, i.e., the number of interesting test cases contributed by this operator divided by the number of invocations of this operator during one iteration. We denote the largest e f fnow as e f fbest . Thus, Lbestis the position where the operator obtains e f fbest in history.

Lbest是粒子的位置，其对应的操作符在给定相同数量的调用时产生了最多有趣的测试用例。为了实现这种比较，对于每个粒子（即操作符），我们测量其局部效率effnow，即在一次迭代中该操作符贡献的有趣测试用例数量除以该操作符的调用次数。我们用effbest表示最大的effnow。因此，Lbest是操作符在历史记录中获得effbest的位置。

```c
for (i = 0; i < operator_num; i++)
{
    double temp_eff = 0.0;

    if (stage_cycles_puppet_v2[swarm_now][i] > stage_cycles_puppet[swarm_now][i])
        temp_eff = (double)(stage_finds_puppet_v2[swarm_now][i] - stage_finds_puppet[swarm_now][i]) /
        (double)(stage_cycles_puppet_v2[swarm_now][i] - stage_cycles_puppet[swarm_now][i]);

    if (eff_best[swarm_now][i] < temp_eff)
    {
        eff_best[swarm_now][i] = temp_eff;
        L_best[swarm_now][i] = x_now[swarm_now][i];
    }

    stage_finds_puppet[swarm_now][i] = stage_finds_puppet_v2[swarm_now][i];
    stage_cycles_puppet[swarm_now][i] = stage_cycles_puppet_v2[swarm_now][i];
    temp_stage_finds_puppet += stage_finds_puppet[swarm_now][i];
}
```

### 3.2.3 Global Best Position G_best

we measure the number of interesting test cases contributed by each operator till now in all swarms, and use it as the particle's global efficiency globale f f . Then we compute the distribution of all particles' global efficiency. 

For each operator (i.e., particle), its global best position Gbestis defined as the proportion of its globale f f in this distribution. With this distribution, particles (i.e., operators) with higher efficiency can get higher probability to be selected.

与原始的PSO不同，MOPT在不同的概率空间（具有相同的形状和大小）中移动粒子。因此，并不存在适用于所有粒子的唯一全局最佳位置。相反，在这里，不同的粒子在不同的空间中有不同的全局最佳位置。在PSO中，全局最佳位置取决于不同粒子之间的关系。因此，在这里，我们同时评估多个群集的粒子的效率，称为全局效率（globaleff），以从全局的角度评估每个粒子的效率。

更具体地说，我们测量迄今为止在所有群集中由每个操作符贡献的有趣测试用例的数量，并将其用作粒子的全局效率（globaleff）。然后，我们计算所有粒子全局效率的分布。对于每个操作符（即粒子），其全局最佳位置Gbest被定义为其globaleff在该分布中的比例。通过这个分布，效率更高的粒子（即操作符）被选中的概率更高。

```c
u64 temp_operator_finds_puppet = 0;
for (i = 0; i < operator_num; i++)
{
    operator_finds_puppet[i] = core_operator_finds_puppet[i];
    
    for (j = 0; j < swarm_num; j++)
    {
        operator_finds_puppet[i] = operator_finds_puppet[i] + stage_finds_puppet[j][i];
    }
    temp_operator_finds_puppet = temp_operator_finds_puppet + operator_finds_puppet[i];
}

for (i = 0; i < operator_num; i++)
{
    if (operator_finds_puppet[i])
        G_best[i] = (double)((double)(operator_finds_puppet[i]) / (double)(temp_operator_finds_puppet)); 
}
```

### 3.3.4 Multiple Swarms

与原始的PSO群体有多个粒子在解空间中探索不同，MOPT定义的群体实际上只在解空间中探索一个候选解（即概率分布），因此可能会陷入局部最优解。因此，MOPT采用多个群体，并对每个群体应用定制的PSO算法，如图7所示，以避免局部最优解。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202307301355990.png)

这些群集之间需要进行同步。MOPT简单地将效率最高的群集作为最佳群集，并使用其概率分布在模糊测试期间调度变异。在这里，我们将群集的效率（表示为swarm_eff）定义为该群集在一次迭代中贡献的有趣测试用例数量除以生成的新测试用例数量。

```c
double swarm_eff = 0.0;
swarm_now = 0;
for (i = 0; i < swarm_num; i++)
{
    if (swarm_fitness[i] > swarm_eff)
    {
        swarm_eff = swarm_fitness[i];
        swarm_now = i;
    }
}
```

概述：总结而言，MOPT采用多个群集，并对每个群集应用定制的PSO算法。在模糊测试过程中，在每次PSO迭代中执行以下三个额外任务。

1. 在每个群集中为所有粒子定位局部最佳位置。在每个群集中，通过模糊测试期间评估每个粒子在一次迭代中的局部效率effnow。对于每个粒子，其历史上具有最高效率effbest的位置被标记为其局部最佳位置Lbest。

2. 在所有群集中为所有粒子定位全局最佳位置。在每个粒子中，评估其在所有群集中的全局效率globaleff。然后，计算所有粒子全局效率的分布。将每个粒子的globaleff在此分布中的比例用作其全局最佳位置Gbest。

3. 选择最佳群集来指导模糊测试。在每次迭代中，评估每个群集的效率swarm_eff。选择效率最高的群集，并将其在当前迭代中的概率分布应用于进一步的模糊测试。

然后，在每次迭代结束时，MOPT以与PSO类似的方式移动每个群集中的粒子。更具体地说，对于群集Si中的粒子Pj，我们按照以下方式更新其位置。

```c
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
```

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202307272205469.png)

# 4 Implementation of MOPT

## 4.1 MOPT Main Framework

如图8所示，MOPT包括四个核心模块，即PSO初始化模块、PSO更新模块、试点模糊测试模块和核心模糊测试模块。PSO初始化模块只执行一次，用于设置PSO算法的初始参数。其他三个模块形成一个迭代循环，并共同协作以持续进行目标程序的模糊测试。

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202307301413670.png)

+ 试点模糊测试模块采用多个群集，即多个概率分布，来选择变异操作符并进行模糊测试。在模糊测试过程中，测量每个粒子在每个群集中的局部效率。因此，我们可以找到每个粒子在每个群集中的局部最佳位置。

+ 此外，在试点模糊测试期间，还会评估每个群集的效率。然后，选择效率最高的群集，并且核心模糊测试模块将使用其探索的概率分布来调度变异操作符。

+ 在核心模糊测试模块完成后，可以评估迄今为止由每个操作符贡献的有趣测试用例的总数。因此，可以评估每个粒子的全局效率（即全局最佳位置）。

```c
for (i = 0; i < operator_num; i++)
{
    x_now[tmp_swarm][i] = x_now[tmp_swarm][i] / x_temp;
    if (likely(i != 0))
        probability_now[tmp_swarm][i] = probability_now[tmp_swarm][i - 1] + x_now[tmp_swarm][i];
    else
        probability_now[tmp_swarm][i] = x_now[tmp_swarm][i];
}
```

### 4.1.1 MOpt: Optimized Mutation Scheduling for Fuzzers

initializes parameters for the PSO algorithm

1. sets the initial location x_now of each particle in each swarm with a random value, and normalizes the sum of x_now of all the particles in one swarm to 1

2. sets the displacement of particle movement v_now of each particle in each swarm to 0.1

3. sets the initial local efficiency eff_best of each particle in each swarm to 0

4. sets the initial local best position L_best of each particle in each swarm to 0.5

5. sets the initial global best position G_best of each particle across swarms to 0.5

```c
for (j = 0; j < operator_num; ++j) {

    afl->stage_finds_puppet[tmp_swarm][j] = 0;
    afl->probability_now[tmp_swarm][j] = 0.0;
    // 随机初始化 x_now
    afl->x_now[tmp_swarm][j] =
        ((double)(random() % 7000) * 0.0001 + 0.1);
    total_puppet_temp += afl->x_now[tmp_swarm][j];
    afl->v_now[tmp_swarm][j] = 0.1;
    afl->L_best[tmp_swarm][j] = 0.5;
    afl->G_best[j] = 0.5;
    afl->eff_best[tmp_swarm][j] = 0.0;

}

for (j = 0; j < operator_num; ++j) {

    afl->stage_cycles_puppet_v2[tmp_swarm][j] =
        afl->stage_cycles_puppet[tmp_swarm][j];
    afl->stage_finds_puppet_v2[tmp_swarm][j] =
        afl->stage_finds_puppet[tmp_swarm][j];
    // 归一化
    afl->x_now[tmp_swarm][j] =
        afl->x_now[tmp_swarm][j] / total_puppet_temp;

}
```

### 4.1.2 Pilot Fuzzing Module

此模块利用多个群集来执行模糊测试，其中每个群集探索不同的概率分布。模块按顺序评估每个群集，并在生成可配置数量的新测试用例（记为periodpilot）后停止测试该群集。

```c
if (unlikely(tmp_pilot_time > period_pilot))
{
    // 记录结果
    ...
}
```

使用特定群集进行模糊测试的过程如下。对于每个群集，其概率分布用于调度选择变异操作符，并模糊目标程序。在模糊过程中，模块通过插装目标程序来测量三个参数：

1. 由特定粒子（即操作符）贡献的有趣测试用例的数量
2. 特定粒子的调用次数
3. 该群集发现的有趣测试用例的数量。

每个粒子（在当前群集中）的局部效率是第一个测量值除以第二个测量值。因此，我们可以定位每个粒子的局部最佳位置。

```c
if (stage_cycles_puppet_v2[swarm_now][i] > stage_cycles_puppet[swarm_now][i])
temp_eff = (double)(stage_finds_puppet_v2[swarm_now][i] - stage_finds_puppet[swarm_now][i]) / (double)(stage_cycles_puppet_v2[swarm_now][i] - stage_cycles_puppet[swarm_now][i]);
```

当前群集的效率是第三个测量值除以测试用例计数periodpilot。因此，我们可以找到效率最高的群集。通过这种方式，

```c
swarm_fitness[swarm_now] = (double)(total_puppet_find - temp_puppet_find) / ((double)(tmp_pilot_time)/ period_pilot_tmp);
temp_puppet_find = total_puppet_find;
```

试点模糊测试模块能够评估每个群集的性能，找到效果最好的群集，从而为核心模糊测试模块提供更好的指导和概率分布选择。

### 4.1.3 Core Fuzzing Module

该模块将使用试点模糊测试模块选出的最佳群集，并利用其概率分布进行模糊测试。在生成可配置数量的新测试用例（记为periodcore）后，核心模糊测试模块将停止。

一旦停止，我们可以从PSO初始化开始至今，测量每个粒子贡献的有趣测试用例的数量，不论它属于哪个群集。然后，我们可以计算粒子之间的分布，并确定每个粒子的全局最佳位置。

```c
if (unlikely(tmp_core_time > period_core))
{
    total_pacemaker_time += tmp_core_time;
    tmp_core_time = 0;
    temp_puppet_find = total_puppet_find;
    new_hit_cnt = queued_paths + unique_crashes;

    u64 temp_stage_finds_puppet = 0;
    for (i = 0; i < operator_num; i++)
    {

        core_operator_finds_puppet[i] = core_operator_finds_puppet_v2[i];
        core_operator_cycles_puppet[i] = core_operator_cycles_puppet_v2[i];
        temp_stage_finds_puppet += core_operator_finds_puppet[i];
    }

    key_module = 2;

    old_hit_count = new_hit_cnt;
}
```

需要注意的是，如果在试点模糊测试模块中只使用一个群集，则核心模块可以与试点模块合并。通过这个过程，核心模块能够从全局的角度评估每个粒子的表现，并找到全局最优的概率分布，以指导后续的模糊测试。

### 4.1.4 PSO Updating Module

根据试点模糊测试模块和核心模糊测试模块提供的信息，此模块按照方程式3和4更新每个群集中的粒子。在更新每个粒子后，我们将进入下一次PSO迭代更新。

因此，我们可以逐步接近一个最优的群集（即操作符的概率分布），并将其用于指导核心模糊测试模块，从而帮助提高模糊测试的效率。

通过这个过程，模糊测试器能够优化粒子的位置，以尽可能高效地探索测试用例空间，并逐步找到最优的概率分布，以指导后续的模糊测试。

## 4.2 Pacemaker Fuzzing Mode

虽然将MOPT应用于基于变异的模糊测试器是通用的，但我们意识到当应用于特定的模糊测试器，如AFL时，MOPT的性能可以进一步优化。

根据广泛的实证分析，我们发现AFL及其后继版本在deterministic阶段花费的时间要比在havoc和splicing阶段花费的时间多得多，而混乱和拼接阶段可以发现更多独特的崩溃和路径。

因此，MOPT针对基于AFL的模糊测试器提供了一个优化，称为节拍器模糊测试模式，它有选择地避免耗时的确定性阶段。具体来说，当MOPT完成对一个种子测试用例的变异后，如果在很长时间内（由用户设置的T）没有发现任何新的独特崩溃或路径，它将选择性地禁用后续测试用例的deterministic阶段。节拍器模糊测试模式具有以下优点。

+ deterministic性阶段花费了太多时间，会减慢整体效率。另一方面，MOPT只在混乱阶段更新概率分布，与deterministic阶段无关。因此，通过节拍器模糊测试模式禁用确定性阶段可以加速MOPT的收敛速度。

+ 在这种模式下，模糊测试器可以跳过deterministic阶段，而无需在一个测试用例上花费太多时间。相反，它会从模糊测试队列中选择更多的种子进行变异，因此更有可能更快地找到漏洞。

+ deterministic阶段在模糊测试的开始阶段可能表现良好，但一段时间后变得低效。这种模式在效率下降后有选择性地禁用此阶段，从而在避免浪费太多时间的同时仍然从此阶段受益。


更具体地说，MOPT为AFL提供了两种类型的节拍器模糊测试模式，取决于是否重新启用确定性阶段：

+ MOPT-AFL-tmp：当新的有趣测试用例数量超过预定义的阈值时，它将再次启用确定性阶段。

+ MOPT-AFL-ever：在后续的模糊测试过程中，它将永远不会重新启用确定性阶段。

这两种模式让用户可以根据具体的需求和性能要求来选择适合的节拍器模糊测试方式。

<br>`limit_time_bound:` control how many interesting test cases need to be found before MOpt-AFL quits the pacemaker fuzzing mode and reuses the deterministic stage. 
0 < `limit_time_bound` < 1, MOpt-AFL-tmp.  `limit_time_bound` >= 1, MOpt-AFL-ever. 

```c
#define limit_time_bound 1.1

if (unlikely(queued_paths + unique_crashes > ((queued_paths + unique_crashes)*limit_time_bound + orig_hit_cnt_puppet)))
{
    key_puppet = 0;
    cur_ms_lv = get_cur_time();
    new_hit_cnt = queued_paths + unique_crashes;
    orig_hit_cnt_puppet = 0;
    last_limit_time_start = 0;
}
```
