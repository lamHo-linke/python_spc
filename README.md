# 用Python实现SPC统计过程控制

## SPC与六西格玛

SPC (Statistical Process Control) 统计过程控制，是六西格玛工业管理理论的其中一个重要模块。SPC的控制图 (control chart) 是数据可视化的一个重要手段。而控制图的选择应该根据实际需求来，这里不展开讲控制图，关于控制图的细节可以查找其他资料。（7 种控制图，8 个判异准则。）

<img src='https://pic.downk.cc/item/5eb7b506c2a9a83be53b8e1e.png'>

简单介绍一下六西格玛，就是 6 sigma 的音译，sigma 是什么？接触过统计的人应该会有印象，$\sigma$ 这个符号就是 sigma，一般代表偏差。

先介绍一下生产质量控制中常用的一个概念 DPMO (Defects Per Million Opportunities)，就是在生产过程中每 100 万个机会中出现的缺陷数。比如在 100 万个焊点里，出现了 1 个缺陷，那么 DPMO 就等于 1。可见 DPMO 是越小越好，最好是为 0。当然在大量的生产过程中这种理想状态出现的几率是非常非常小的。DPMO 的取值会作为衡量生产质量的一个重要指标，下面是 DPMO 对应 sigma 水平的对照表。

|sigma level|DPMO|yield|
|:-:|:-:|:-:|
|6|3.4|99.99966%|
|5|230|99.977%|
|4|6210|99.38%|
|3|66800|93.32%|
|2|308000|69.15%|
|1|690000|30.85%|

如果 DPMO 为 3.4 ，对应的 sigma 水平是 6，也就是可以达到 99.99966% 的合格率。所以 sigma 水平就可以作为生产质量的一个评判标准，达到 6 sigma 水平的生产就是非常牛逼的水准了！

<img src='https://pic.downk.cc/item/5eb7cc68c2a9a83be56d120e.jpg' width=600>

6 个 $\sigma$ 的介绍就到这里，不过六西格玛管理理论远不止于此，除了统计学的知识，大部分是关于精益生产管理。


## U Chart 案例

这是当时工作中遇到的实际需求，要观察生产产品的单位缺陷数。拿到检测机器的测试记录，包括每天检测的产品编号、检测时间、残次件的维修时间、维修人员等等。冗余信息非常多，用 Excel 处理，只提取检测的产品数量和缺陷件数量，按月份统计整理。（现在看回头当时为什么会弄了两个表示时间的列也是百思不得其解。。。。）n 列表示当月检测的产品数量，c 列表示当月的缺陷数。

<img src='https://pic.downk.cc/item/5eb81213c2a9a83be51b1811.png'>

因为每个月的产品数量是不同的，所以采用用于可变样本量的 U-chart。如果每个月的产品数量相同，用 C-chart。

Defects per unit: $$u=\frac{c}{n}$$

Central Limit: $$CL=\bar{u}=\frac{\sum c_i}{\sum n_i}$$

Upper Central Limit: $$UCL=\bar{u}+3\sqrt{\frac{\bar{u}}{n_i}}$$

（开方里的是 $\bar{u}$，好像叠起来看不见了）本来还应该算下界的，但是因为 u 肯定是大于 0，且越小越好，所以下界不需要考虑。

用 pandas 导入整理好的数据命名为 defect_test。计算好3个需要展示的数据：u，CL，UCL。

```python
time = defect_test['Month']
u = defect_test['c']/defect_test['n']
cl = sum(defect_test['c'])/sum(defect_test['n'])
ucl = cl + 3*np.sqrt(cl/defect_test['n'])
```

用 matplotlib 可视化

```python
cl_s = np.ones(12)*cl
plt.style.use('ggplot')
plt.figure(figsize=(15,6))
plt.plot(time,u,'k',marker='o',markersize=10,lw=3,label='u=c/n')
plt.plot(time,cl_s,'r',label='CL')
plt.plot(time,ucl,'b--',label='UCL')
plt.legend()
plt.title('U chart');
```

<img src='https://pic.downk.cc/item/5eb81fc3c2a9a83be53e6989.png'>

从上图可以看出，缺陷数u一直在CL附近波动。大部分的u都在UCL以下，但是有两个月是在控制以外 (out of control)，二月和三月。二月份的u值是0.005277，三月份的是0.006198。对应的DPMO值5277和6198，对应了4 sigma的水平，对应Cpk值（工程能力，要求达到1.33以上）是1.33。总而来说，在二月和三月，缺陷数在控制以外，但Cpk还是达到了应有的期望值1.33。整一年生产过程是达到要求的，但是还是需要注意质量控制，需要保持所有的缺陷数在控制范围内。

对于超出上界限UCL的点判断为异常，在图表上显示出来：

```python
def uchart_test(c,n,time):
    import numpy as np
    import matplotlib.pyplot as plt
    
    u = c/n
    cl = sum(c)/sum(n)
    ucl = cl + 3 * np.sqrt(cl/n)
    cl_s = np.ones(len(time))*cl
    
    # out of control points
    ofc = u[u >= ucl]
    ofc_ind = list(ofc.index)
    ofc_time = time[ofc_ind]
    print("Out of Control:")
    if len(ofc) == 0:
        print("All under control.")
    else:
        for i,j in zip(ofc_ind,range(len(ofc))):
            print(str(j+1) + " - " + str(ofc_time[i]) + ", " + str(ofc[i]))

    plt.style.use('ggplot')
    plt.figure(figsize=(15,6))

    # plot out of control points
    for i in ofc_ind:
        plt.scatter(ofc_time[i],ofc[i],c='r',s=200)

    # plot u chart
    plt.plot(time,u,'k',marker='o',markersize=8,lw=3,label='u=c/n')
    plt.plot(time,cl_s,'g',label='CL')
    plt.plot(time,ucl,'b--',label='UCL')
    plt.legend()
    plt.ylim((0,max(u)+0.001))
    plt.title('U chart');
```

用定义好的函数 `uchart_test`，输入残缺件数c，总件数n，时间time，就可以画出 U chart，并且找出异常点：

<img src='https://pic.downk.cc/item/5eb8c0d7c2a9a83be577e9e3.png'>

## I-MR Chart 案例

使用 I-MR 控制图 可以在拥有连续数据且这些数据是不属于子组的单个观测值的情况下监视过程的均值和变异。使用此控制图可以监视过程在一段时间内的稳定性，以便您可以标识和更正过程中的不稳定性。

例如，一家医院的管理员想确定门诊疝气手术所用的时间是否稳定，以及手术时间的变异性是否稳定。由于数据不是以子组形式收集的，因此管理员使用 I-MR 控制图监视手术时间的均值和变异性。

实际看看下面这个数据案例：
<img src='https://pic.downk.cc/item/5eb8c4fec2a9a83be57efa77.png'>

I-MR Chart 是有两个图：一个是 I Chart (Individual Chart)，一个是 MR Chart (Moving Range Chart)。
<img src='https://pic.downk.cc/item/5eb8c67fc2a9a83be5822731.png'>

和所有控制图一样，重点是找到 CL，UCL，LCL。

**Tabular values for X and range charts**

|Subgroup Size|E2|D4|
|:-:|:-:|:-:|
|1|2.660|3.268|
|2|2.660|3.268|
|3|1.772|2.574|
|4|1.457|2.282|
|5|1.290|2.114|
|6|1.184|2.004|
|7|1.109|1.924|
|8|1.054|1.864|
|9|1.010|1.816|
|10|0.975|1.777|

subgroup 的大小可以调整，这里我们默认选 1。

I Chart：$X$

$$CL_X=\bar{X}=\frac{1}{n}\sum_{i=1}^n X_i$$

$$UCL_X=\bar{X}+E_2\times \bar{MR}$$

$$LCL_X=\bar{X}-E_2\times \bar{MR}$$

MR Chart：$MR=|X_i-X_{i-1}|$ （维度比 X 少 1）

$$CL_{MR}=\frac{1}{n-1}\sum_{i=2}^n (X_i-X_{i-1})$$

$$UCL_{MR}=D_4\times \bar{MR}$$

$$LCL_{MR}=0$$

```python
xs = data.iloc[:,[1,3,5,7,9]]
observation = data.iloc[:,[0,2,4,6,8]]
x = []
for i in range(xs.shape[1]):
    x.extend(xs.iloc[:,i])
x = np.array(x)
    
obs = []
for i in range(observation.shape[1]):
    obs.extend(observation.iloc[:,i])
obs = np.array(obs)

x_bar = x.mean()

# move range
x_1 = x[:-1]
x_2 = x[1:]
mr = np.array(np.array(list(abs(v - u) for u, v in zip(x, x[1:]))))

mr_bar = mr.mean()

# E2 = 2.660
x_ucl = x_bar + 2.660 * mr_bar
x_ucl_c = x_bar + 2.660 * mr_bar * (1/3)
x_ucl_b = x_bar + 2.660 * mr_bar * (2/3)
x_lcl = x_bar - 2.660 * mr_bar
x_lcl_c = x_bar - 2.660 * mr_bar * (1/3)
x_lcl_b = x_bar - 2.660 * mr_bar * (2/3)

# D4 = 3.268
mr_ucl = 3.268 * mr_bar
mr_ucl_c = mr_bar + (mr_ucl - mr_bar) * (1/3)
mr_ucl_b = mr_bar + (mr_ucl - mr_bar) * (2/3)
mr_lcl = 0
mr_lcl_c = mr_bar - (mr_ucl - mr_bar) * (1/3)
mr_lcl_b = mr_bar - (mr_ucl - mr_bar) * (2/3)   # <0

n = len(x)

x_ucl = np.ones(n)*x_ucl
x_ucl_c = np.ones(n)*x_ucl_c
x_ucl_b = np.ones(n)*x_ucl_b
x_lcl = np.ones(n)*x_lcl
x_lcl_c = np.ones(n)*x_lcl_c
x_lcl_b = np.ones(n)*x_lcl_b

plt.style.use('ggplot')
plt.figure(figsize=(15,8))

plt.subplot(2,1,1)
plt.plot(obs,x, 'k',marker='o',markersize=5,lw=2,label='X')
plt.plot(obs,np.ones(n)*x_bar,'r',label='CL')
plt.plot(obs,x_ucl,'b',lw=1)
plt.plot(obs,x_ucl_c,'b-.',lw=1)
plt.plot(obs,x_ucl_b,'b-.',lw=1)
plt.plot(obs,x_lcl,'b',lw=1)
plt.plot(obs,x_lcl_c,'b-.',lw=1)
plt.plot(obs,x_lcl_b,'b-.',lw=1)
plt.legend()
plt.title('I chart')

mr_ucl = np.ones(n-1)*mr_ucl
mr_ucl_c = np.ones(n-1)*mr_ucl_c
mr_ucl_b = np.ones(n-1)*mr_ucl_b
mr_lcl = np.ones(n-1)*mr_lcl
mr_lcl_c = np.ones(n-1)*mr_lcl_c
mr_lcl_b = np.ones(n-1)*mr_lcl_b

plt.subplot(2,1,2)
plt.plot(obs[1:],mr, 'k',marker='o',markersize=5,lw=2,label='MR')
plt.plot(obs[1:],np.ones(n-1)*mr_bar,'r',label='CL')
plt.plot(obs[1:],mr_ucl,'b',lw=1)
plt.plot(obs[1:],mr_ucl_c,'b-.',lw=1)
plt.plot(obs[1:],mr_ucl_b,'b-.',lw=1)
plt.plot(obs[1:],mr_lcl,'b',lw=1)
plt.plot(obs[1:],mr_lcl_c,'b-.',lw=1)
plt.legend()
plt.title('MR chart');
```

<img src='https://pic.downk.cc/item/5eb9122fc2a9a83be5183c86.png'>

判异的规则有 8 条：

|Rule|Rule Name|Pattern|&emsp; &emsp; &emsp; &emsp; &emsp; &emsp; Example&emsp; &emsp; &emsp; &emsp; &emsp; &emsp; |
|:-:|:-:|:-|-|
|1|Beyond Limits|1个点落在A区外|<img src='https://upload.wikimedia.org/wikipedia/commons/a/a0/Rule_1_-_Control_Charts_for_Nelson_Rules.svg'>|
|2|Zone A|连续3点中有2点落在中心线同一侧的Zone B以外|<img src='https://upload.wikimedia.org/wikipedia/commons/2/20/Rule_5_-_Control_Charts_for_Nelson_Rules.svg'>|
|3|Zone B|连续5点有4点落在中心线同一侧的Zone C以外|<img src='https://upload.wikimedia.org/wikipedia/commons/7/7b/Rule_6_-_Control_Charts_for_Nelson_Rules.svg'>|
|4|Zone C|连续9个以上的点落在中心线同一侧（Zone C或以外）|<img src='https://upload.wikimedia.org/wikipedia/commons/0/0e/Rule_2_-_Control_Charts_for_Nelson_Rules.svg'>|
|5|Trend|连续7点递增或递减|<img src='https://upload.wikimedia.org/wikipedia/commons/e/e0/Rule_3_-_Control_Charts_for_Nelson_Rules.svg'>|
|6|Mixture|连续8点无一点落在Zone C|<img src='https://upload.wikimedia.org/wikipedia/commons/b/b7/Rule_8_-_Control_Charts_for_Nelson_Rules.svg'>|
|7|Stratification|连续15点落在中心线两侧的Zone C内|<img src='https://upload.wikimedia.org/wikipedia/commons/a/ae/Rule_7_-_Control_Charts_for_Nelson_Rules.svg'>|
|8|Over-control|连续14点相邻交替上下|<img src='https://upload.wikimedia.org/wikipedia/commons/3/35/Rule_4_-_Control_Charts_for_Nelson_Rules.svg'>|

加入8条判异规则：

```python
def rules(data, obs, cl, ucl, ucl_b, ucl_c, lcl, lcl_b, lcl_c):
    n = len(data)
    ind = np.array(range(n))
    
    # rule 1
    ofc1 = data[(data > ucl) | (data < lcl)]
    ofc1_obs = obs[(data > ucl) | (data < lcl)]    
    
    # rule 2
    ofc2_ind = []
    for i in range(n-2):
        d = data[i:i+3]
        index = ind[i:i+3]
        if ((d>ucl_b).sum()==2) | ((d<lcl_b).sum()==2):
            ofc2_ind.extend(index[(d>ucl_b) | (d<lcl_b)])
    ofc2_ind = list(sorted(set(ofc2_ind)))
    ofc2 = data[ofc2_ind]
    ofc2_obs = obs[ofc2_ind]
    
    # rule 3
    ofc3_ind = []
    for i in range(n-4):
        d = data[i:i+5]
        index = ind[i:i+5]
        if ((d>ucl_c).sum()==4) | ((d<lcl_c).sum()==4):
            ofc3_ind.extend(index[(d>ucl_c) | (d<lcl_c)])
    ofc3_ind = list(sorted(set(ofc3_ind)))
    ofc3 = data[ofc3_ind]
    ofc3_obs = obs[ofc3_ind]
    
    # rule 4
    ofc4_ind = []
    for i in range(n-8):
        d = data[i:i+9]
        index = ind[i:i+9]
        if ((d>cl).sum()==9) | ((d<cl).sum()==9):
            ofc4_ind.extend(index)
    ofc4_ind = list(sorted(set(ofc4_ind)))
    ofc4 = data[ofc4_ind]
    ofc4_obs = obs[ofc4_ind]
    
    # rule 5
    ofc5_ind = []
    for i in range(n-6):
        d = data[i:i+7]
        index = ind[i:i+7]
        if all(u <= v for u, v in zip(d, d[1:])) | all(u >= v for u, v in zip(d, d[1:])):
            ofc5_ind.extend(index)
    ofc5_ind = list(sorted(set(ofc5_ind)))
    ofc5 = data[ofc5_ind]
    ofc5_obs = obs[ofc5_ind]
    
    # rule 6
    ofc6_ind = []
    for i in range(n-7):
        d = data[i:i+8]
        index = ind[i:i+8]
        if (all(d>ucl_c) | all(d<lcl_c)):
            ofc6_ind.extend(index)
    ofc6_ind = list(sorted(set(ofc6_ind)))
    ofc6 = data[ofc6_ind]
    ofc6_obs = obs[ofc6_ind]
    
    # rule 7
    ofc7_ind = []
    for i in range(n-14):
        d = data[i:i+15]
        index = ind[i:i+15]
        if all(lcl_c<d) and all(d<ucl_c):
            ofc7_ind.extend(index)
    ofc7_ind = list(sorted(set(ofc7_ind)))
    ofc7 = data[ofc7_ind]
    ofc7_obs = obs[ofc7_ind]
    
    # rule 8
    ofc8_ind = []
    for i in range(n-13):
        d = data[i:i+14]
        index = ind[i:i+14]
        diff = list(v - u for u, v in zip(d, d[1:]))
        if all(u*v<0 for u,v in zip(diff,diff[1:])):
            ofc8_ind.extend(index)
    ofc8_ind = list(sorted(set(ofc8_ind)))
    ofc8 = data[ofc8_ind]
    ofc8_obs = obs[ofc8_ind]

    return ofc1, ofc1_obs, ofc2, ofc2_obs, ofc3, ofc3_obs, ofc4, ofc4_obs, ofc5, ofc5_obs,ofc6, ofc6_obs, ofc7, ofc7_obs, ofc8, ofc8_obs

def imrchart_test(obs,x,subgroup=1):
    import numpy as np
    import matplotlib.pyplot as plt
    
    # depend on subgroup value
    E2 = [2.660, 2.660, 1.772, 1.457, 1.290, 1.184, 1.109, 1.054, 1.010, 0.975]
    D4 = [3.268, 3.268, 2.574, 2.282, 2.114, 2.004, 1.924, 1.864, 1.816, 1.777]
    e2 = E2[subgroup-1]
    d4 = D4[subgroup-1]
    
    n = len(x)
    
    x_bar = x.mean()
    mr = np.array(list(abs(v - u) for u, v in zip(x, x[1:])))
    mr_bar = mr.mean()
    
    # I chart
    x_ucl = x_bar + e2 * mr_bar
    x_ucl_c = x_bar + e2 * mr_bar * (1/3)
    x_ucl_b = x_bar + e2 * mr_bar * (2/3)
    x_lcl = x_bar - e2 * mr_bar
    x_lcl_c = x_bar - e2 * mr_bar * (1/3)
    x_lcl_b = x_bar - e2 * mr_bar * (2/3) 
    
    # MR chart
    mr_ucl = d4 * mr_bar
    mr_ucl_c = mr_bar + (mr_ucl - mr_bar) * (1/3)
    mr_ucl_b = mr_bar + (mr_ucl - mr_bar) * (2/3)
    mr_lcl = 0
    mr_lcl_c = mr_bar - (mr_ucl - mr_bar) * (1/3)
    mr_lcl_b = mr_bar - (mr_ucl - mr_bar) * (2/3)
    if mr_lcl_c < 0: mr_lcl_c = 0
    if mr_lcl_b < 0: mr_lcl_b = 0
    
    # 8 rules
    x_ofc = rules(x, obs, x_bar, x_ucl, x_ucl_b, x_ucl_c, x_lcl, x_lcl_b, x_lcl_c)
    print("========== I chart test ==========")
    for r in range(8):
        print("Against Rule %d:" %(r+1))
        if len(x_ofc[r*2]) == 0:
            print("None.")
        else: 
            for num,i,j in zip(range(len(x_ofc[r*2])), x_ofc[r*2+1], x_ofc[r*2]):
                print("\t%d -> %s, %f" %(num+1, str(i), j))
    
    mr_ofc = rules(mr, obs[1:], mr_bar, mr_ucl, mr_ucl_b, mr_ucl_c, mr_lcl, mr_lcl_b, mr_lcl_c)
    print("========== MR chart test ==========")
    for r in range(8):
        print("Against Rule %d:" %(r+1))
        if len(mr_ofc[r*2]) == 0:
            print("None.")
        else: 
            for num,i,j in zip(range(len(mr_ofc[r*2])), mr_ofc[r*2+1], mr_ofc[r*2]):
                print("\t%d -> %s, %f" %(num+1, str(i), j))      
    
    # plot
    plt.style.use('ggplot')
    plt.figure(figsize=(15,8))

    plt.subplot(2,1,1)
    plt.plot(obs,x, 'k',marker='o',markersize=5,lw=2,label='X')
    plt.plot(obs,np.ones(n)*x_bar,'r',label='CL')
    plt.plot(obs,np.ones(n)*x_ucl,'b',lw=1)
    plt.plot(obs,np.ones(n)*x_ucl_c,'b-.',lw=1)
    plt.plot(obs,np.ones(n)*x_ucl_b,'b-.',lw=1)
    plt.plot(obs,np.ones(n)*x_lcl,'b',lw=1)
    plt.plot(obs,np.ones(n)*x_lcl_c,'b-.',lw=1)
    plt.plot(obs,np.ones(n)*x_lcl_b,'b-.',lw=1)
    for r in range(8):
        for i in range(len(x_ofc[r*2])):
            plt.scatter(x_ofc[r*2+1][i],x_ofc[r*2][i],c='r',s=70)
    plt.legend()
    plt.title('I chart')

    plt.subplot(2,1,2)
    plt.plot(obs[1:],mr, 'k',marker='o',markersize=5,lw=2,label='MR')
    plt.plot(obs[1:],np.ones(n-1)*mr_bar,'r',label='CL')
    plt.plot(obs[1:],np.ones(n-1)*mr_ucl,'b',lw=1)
    plt.plot(obs[1:],np.ones(n-1)*mr_ucl_c,'b-.',lw=1)
    plt.plot(obs[1:],np.ones(n-1)*mr_ucl_b,'b-.',lw=1)
    plt.plot(obs[1:],np.ones(n-1)*mr_lcl,'b',lw=1)
    plt.plot(obs[1:],np.ones(n-1)*mr_lcl_c,'b-.',lw=1)
    for r in range(8):
        for i in range(len(mr_ofc[r*2])):
            plt.scatter(mr_ofc[r*2+1][i],mr_ofc[r*2][i],c='r',s=70)
    plt.legend()
    plt.title('MR chart');

imrchart_test(obs,x)
```

<img src='https://pic.downk.cc/item/5eb9699dc2a9a83be5dcf194.png'>
<img src='https://pic.downk.cc/item/5eb96a11c2a9a83be5de1c15.png'>
<img src='https://pic.downk.cc/item/5eb9693cc2a9a83be5dbf8ef.png'>

观察 I chart，对照输出的具体异常点：

- Rule 1 - 第76个观测值32，在A区以外，比UCL大；
- Rule 2 - 第19,20,64,65,76,77，连续三个点中有两个落在同一侧的B区以外；（76已经是Rule1异常）
- Rule 3 - 第13,14,15,17,18,19,20,76,77,78,79，连续五个点中有四个落在同一侧的C区外；（19,20,76,77已经是Rule1和Rule2异常）
- Rule 4 - 第8至20，连续九个点落在中心线同一侧；（13,14,15,17,18,19,20已经在前面的Rule检测到）
- Rule 5 至 8 均未发现异常。

观察 MR chart，对照输出的具体异常点：

- Rule 1 - 第46,66,71,76,91异常，在UCL以外；
- 其余Rule均未发现异常。

---

针对不同的场景选择不同的控制图。每个控制图，按照计算公式编写对应的代码，确定上下界和判异规则，将异常点在图上用红点表示出来。
