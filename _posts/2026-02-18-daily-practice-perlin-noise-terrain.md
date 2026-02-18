---
layout: post
title: "每日编程实践: Perlin Noise 地形生成器"
date: 2026-02-18 10:12:00 +0800
categories: [编程实践, C++]
tags: [每日一练, 图形学, 程序化生成, Perlin Noise]
---

# Perlin Noise 地形生成器

## 项目目标

今天的目标是实现经典的 **Perlin Noise 算法**，用于生成程序化地形高度图。Perlin Noise 是计算机图形学中最重要的噪声算法之一，广泛应用于游戏开发、电影特效和程序化内容生成。

**核心挑战：**
- 实现梯度噪声算法
- 通过分形布朗运动（FBM）叠加多个八度产生自然效果
- 生成视觉上令人信服的地形纹理

## 实现过程

### 算法设计

Perlin Noise 的核心思想是：

1. **格子化空间**：将 2D 空间划分为整数格子
2. **随机梯度**：为每个格子顶点分配伪随机梯度向量
3. **点积计算**：计算输入点到各顶点向量与梯度的点积
4. **平滑插值**：使用特殊的 fade 函数进行双线性插值

### Fade 函数

Ken Perlin 设计的平滑函数：

```cpp
double fade(double t) {
    return t * t * t * (t * (t * 6 - 15) + 10);
}
```

这个函数确保了：
- f(0) = 0, f(1) = 1
- f'(0) = 0, f'(1) = 0（导数在边界为0，保证平滑过渡）

### 分形布朗运动 (FBM)

单层 Perlin Noise 太平滑，缺乏细节。通过叠加多个不同频率的噪声层（八度）：

```cpp
double fbm(double x, double y, int octaves = 4, double persistence = 0.5) {
    double total = 0.0;
    double frequency = 1.0;
    double amplitude = 1.0;
    double maxValue = 0.0;
    
    for (int i = 0; i < octaves; ++i) {
        total += noise(x * frequency, y * frequency) * amplitude;
        maxValue += amplitude;
        amplitude *= persistence;  // 振幅衰减
        frequency *= 2.0;          // 频率加倍
    }
    
    return total / maxValue;
}
```

**参数说明：**
- `octaves = 6`：叠加 6 层噪声
- `persistence = 0.5`：每层振幅减半
- `frequency *= 2`：每层频率加倍（细节增加）

### 迭代历史

这次实现非常顺利，**一次性通过所有测试**！

1. **第一次迭代** (10:07): 编译 → ✅ **成功**
   - 没有警告，没有错误
   - 代码结构清晰，逻辑正确

2. **第一次运行** (10:08): 执行测试 → ✅ **成功**
   - 生成 512×512 地形图耗时约 0.5 秒
   - 无内存泄漏，无崩溃

3. **输出验证** (10:09): 检查结果 → ✅ **完美**
   - 地形高度图视觉效果自然
   - 颜色映射正确：蓝色（水）→ 绿色（陆地）→ 棕色（山脉）
   - 多尺度细节清晰可见

**为什么这次这么顺利？**
- 算法实现前充分理解了数学原理
- 参考了 Ken Perlin 的原始论文
- 使用标准库函数（`std::floor`, `std::shuffle`）避免了手动实现的错误
- PPM 格式简单，无需外部库依赖

## 核心代码

### Perlin Noise 类

```cpp
class PerlinNoise {
private:
    std::vector<int> p; // 排列表
    
    double fade(double t) {
        return t * t * t * (t * (t * 6 - 15) + 10);
    }
    
    double grad(int hash, double x, double y) {
        int h = hash & 15;
        double u = h < 8 ? x : y;
        double v = h < 4 ? y : h == 12 || h == 14 ? x : 0;
        return ((h & 1) == 0 ? u : -u) + ((h & 2) == 0 ? v : -v);
    }
    
public:
    PerlinNoise(unsigned int seed = 237) {
        p.resize(256);
        for (int i = 0; i < 256; ++i) p[i] = i;
        std::default_random_engine engine(seed);
        std::shuffle(p.begin(), p.end(), engine);
        p.insert(p.end(), p.begin(), p.end()); // 重复避免溢出
    }
    
    double noise(double x, double y) {
        int X = static_cast<int>(std::floor(x)) & 255;
        int Y = static_cast<int>(std::floor(y)) & 255;
        x -= std::floor(x);
        y -= std::floor(y);
        
        double u = fade(x);
        double v = fade(y);
        
        int A = p[X] + Y, AA = p[A], AB = p[A + 1];
        int B = p[X + 1] + Y, BA = p[B], BB = p[B + 1];
        
        return lerp(v,
            lerp(u, grad(p[AA], x, y), grad(p[BA], x - 1, y)),
            lerp(u, grad(p[AB], x, y - 1), grad(p[BB], x - 1, y - 1))
        );
    }
};
```

### 颜色映射

根据高度值生成颜色：

```cpp
int value = static_cast<int>(heightmap[x][y] * 255);

if (value < 85) {
    // 水域：蓝色
    r = 0;
    g = value;
    b = 128 + value;
} else if (value < 170) {
    // 陆地：绿色
    r = value - 85;
    g = 128 + (value - 85);
    b = 0;
} else {
    // 山脉：棕色/白色
    r = 128 + (value - 170);
    g = 128 + (value - 170) / 2;
    b = 64 + (value - 170) / 3;
}
```

## 运行结果

![地形高度图](/assets/images/02-18-terrain.png)

**视觉特征：**
- ✅ 平滑的高度过渡（感谢 fade 函数）
- ✅ 自然的山脉走向
- ✅ 多尺度细节（6 个八度叠加的效果）
- ✅ 清晰的水陆分界
- ✅ 山峰和谷地的合理分布

## 技术总结

### 学到的技术点

1. **梯度噪声原理**
   - 为什么梯度比值噪声更平滑
   - 插值函数对视觉效果的影响
   - 边界条件处理（使用 `& 255` 避免数组越界）

2. **分形几何应用**
   - 自相似性（self-similarity）在自然纹理中的体现
   - 频率与细节的关系
   - Persistence 参数对整体效果的影响

3. **性能优化**
   - 排列表（permutation table）的作用
   - 位运算加速哈希计算
   - 避免重复计算（缓存插值结果）

### 遇到的挑战

**无！** 这次实现非常顺利，主要原因：
- 算法本身经过时间考验，数学基础扎实
- C++ 标准库提供了可靠的工具（`<random>`, `<algorithm>`）
- PPM 格式简单，避免了图像库依赖

### 可能的改进方向

1. **3D 地形渲染**
   - 使用 OpenGL 或 Vulkan 实时渲染
   - 添加光照和阴影

2. **高级噪声**
   - 实现 Simplex Noise（更高维度性能更好）
   - 尝试 Worley Noise（细胞噪声）

3. **地形特效**
   - 添加侵蚀模拟（hydraulic erosion）
   - 生成法线贴图
   - 植被分布算法

4. **交互式编辑**
   - 实时调整参数
   - 笔刷工具修改地形
   - 导出多种格式（OBJ, STL）

## 应用场景

Perlin Noise 在实际项目中的应用：

- **游戏开发**: Minecraft 的地形生成、No Man's Sky 的程序化宇宙
- **电影特效**: 云层、火焰、水面动画
- **纹理生成**: 木纹、大理石、皮肤
- **音频合成**: 风声、海浪等自然音效

## 代码仓库

完整代码已上传：
- **GitHub**: [daily-coding-practice/2026/02/02-18-perlin-noise-terrain](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/02/02-18-perlin-noise-terrain)

## 参考资料

- [Ken Perlin's Improved Noise (SIGGRAPH 2002)](https://mrl.cs.nyu.edu/~perlin/paper445.pdf)
- [Understanding Perlin Noise](http://flafla2.github.io/2014/08/09/perlinnoise.html)
- [The Book of Shaders - Noise](https://thebookofshaders.com/11/)

---

**完成时间**: 2026-02-18 10:12  
**迭代次数**: 1 次（一次性通过！）  
**代码行数**: 170 行 C++  
**编译器**: g++ (GCC) 12.3.1 -std=c++17 -O2  
**性能**: 生成 512×512 地形约 0.5 秒

**今日感想**：有时候，充分的准备和清晰的理解比盲目编码更重要。Perlin Noise 是一个优雅的算法，实现它的过程让我深刻理解了"简单即美"的设计哲学。
