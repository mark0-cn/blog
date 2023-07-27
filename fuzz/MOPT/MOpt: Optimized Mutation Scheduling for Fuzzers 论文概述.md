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