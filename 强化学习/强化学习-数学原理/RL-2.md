# 1、States Value 状态价值
## 1.1. return 
### 1.1.1、为什么 return 是重要的
return ： The (discounted) sum of the rewards obtained along a trajectory.

return : 沿着一条轨迹获的折扣奖励总和

下面3个策略中那个是最好的
![alt text](./img2/return-example-1.png)
![alt text](./img2/return-example-2.png)

其中策略3 求的；两条轨迹的平均值，因为我们的return是求的一条轨迹

### 1.1.2、求解 return
![alt text](./img2/return-calculate.png)
- 方法 1
![alt text](./img2/return-calculate-1.png)
- 方法 2
![alt text](./img2/return-calculate-2.png)
![alt text](./img2/return-calculate-3.png)

## 1.2. State-Value 状态价值函数
![alt text](./img2/State-Value-1.png)
![alt text](./img2/State-Value-2.png)
从当前状态 s 出发，得到的所有 trajectory(轨迹)的return值，在对这些值求平均值，就是 State-Value，用 v 表示，v 依赖策略函数 π 和 状态 s
![alt text](./img2/State-Value-3.png)

## 1.3 Bellman 贝尔曼公式
![alt text](./img2/bellman-1.png)

对第一个表达式的理解：

在当前状态 s 下，以概率π(a|s)选择动作 a，给定 s 和 a 之后，奖励 R_t+1以概率 p(r|s,a)取值为r，先计算每个动作下的期望奖励 SUM_r(p(r|s,a) * r),再对这些动作的期望按 π(a|s)加权平均

![alt text](./img2/bellman-2.png)

对第二个表达式的理解： 按照图中的编号来逐步解释

1、从当前状态 s 出来，下一个时刻的State-Value值

2、从当前状态 s 到下一个时刻状态s'以概率 p(s'|s)取值，先计算每个s'下的State-Value

3、马尔科夫决策中历史无关性，当前State-value之和当前状态有关，和历史状态无关，因此可以去除 S_t = s的条件

4、中间的State-Value表达式是 下个状态s'的

5、状态转移需要 策略函数 π 来求

![alt text](./img2/bellman-3.png)

最终公式
![alt text](./img2/bellman-4.png)
![alt text](./img2/bellman-5.png)

## 1.4 BellMan公式 矩阵形式
![alt text](./img2/bellman-m-1.png)
![alt text](./img2/bellman-m-2.png)
![alt text](./img2/bellman-m-3.png)

# 2、求解 State-Value
![alt text](./img2/solve-bellman-1.png)
![alt text](./img2/solve-bellman-2.png)

# 3. Action Value 状态价值函数
![alt text](./img2/Action-Value-1.png)
![alt text](./img2/Action-Value-2.png)
![alt text](./img2/Action-Value-3.png)

# 4、总结
![alt text](./img2/summary.png)
