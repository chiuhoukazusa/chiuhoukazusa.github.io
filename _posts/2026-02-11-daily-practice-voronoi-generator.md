---
layout: post
title: "每日编程实践: Voronoi图生成器"
date: 2026-02-11 10:10:00 +0800
categories: [编程实践, C++]
tags: [每日一练, 图形学, 计算几何, Voronoi]
---

# Voronoi 图生成器

今天实现了一个经典的计算几何项目：**Voronoi 图生成器**。这是一个在图形学、游戏开发和程序化生成中非常实用的算法。

## 项目目标

生成一个美观的 Voronoi 图（泰森多边形），要求：
- ✅ 随机生成种子点
- ✅ 计算每个像素到最近种子点的距离
- ✅ 为每个 Voronoi 单元着色
- ✅ 标记种子点位置
- ✅ 输出高质量 PNG 图像

## 实现过程

### 算法设计

Voronoi 图的核心思想是**空间划分**：给定平面上的一组种子点，将平面划分为多个区域，每个区域内的所有点到某个特定种子点的距离最近。

我使用了暴力法实现：

```cpp
for (每个像素 (x, y)) {
    最近种子 = null
    最小距离 = ∞
    
    for (每个种子点 s) {
        距离 = distance(x, y, s.x, s.y)
        if (距离 < 最小距离) {
            最小距离 = 距离
            最近种子 = s
        }
    }
    
    像素颜色 = 最近种子的颜色
}
```

**时间复杂度**：O(N × W × H)  
对于 50 个种子点和 800×600 图像，需要计算约 2400 万次距离。

### 核心代码

```cpp
struct Point {
    float x, y;           // 种子点坐标
    unsigned char r, g, b; // RGB 颜色
};

// 欧式距离计算
float distance(float x1, float y1, float x2, float y2) {
    float dx = x1 - x2;
    float dy = y1 - y2;
    return std::sqrt(dx * dx + dy * dy);
}

// 生成 Voronoi 图
void generateVoronoi(unsigned char* image, int width, int height, 
                     const std::vector<Point>& seeds) {
    for (int y = 0; y < height; ++y) {
        for (int x = 0; x < width; ++x) {
            float minDist = std::numeric_limits<float>::max();
            int closestSeed = 0;
            
            // 找到最近的种子点
            for (size_t i = 0; i < seeds.size(); ++i) {
                float dist = distance(x, y, seeds[i].x, seeds[i].y);
                if (dist < minDist) {
                    minDist = dist;
                    closestSeed = i;
                }
            }
            
            // 设置像素颜色
            int idx = (y * width + x) * 3;
            image[idx + 0] = seeds[closestSeed].r;
            image[idx + 1] = seeds[closestSeed].g;
            image[idx + 2] = seeds[closestSeed].b;
        }
    }
}
```

### 随机数生成

使用 C++11 的 `<random>` 库生成高质量的随机数：

```cpp
std::random_device rd;
std::mt19937 gen(rd());
std::uniform_int_distribution<> disX(0, width - 1);
std::uniform_int_distribution<> disY(0, height - 1);
std::uniform_int_distribution<> disColor(50, 255);
```

- **Mersenne Twister** 引擎：周期长、分布均匀
- 颜色范围 50-255：避免过暗的颜色

### 图像保存

使用 **stb_image_write.h** 单头文件库：

```cpp
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "stb_image_write.h"

stbi_write_png("voronoi.png", WIDTH, HEIGHT, 3, image.data(), WIDTH * 3);
```

优点：
- 单头文件，零依赖
- 支持 PNG, BMP, TGA, JPG
- 简单易用

## 运行结果

![Voronoi图]({{ site.baseurl }}/assets/images/2026-02-11-voronoi.png)

生成的 Voronoi 图展示了 50 个随机种子点的空间划分。每个彩色区域代表一个 Voronoi 单元，白点标记种子位置。可以看到：
- 单元边界是垂直平分线
- 单元形状接近多边形
- 颜色过渡清晰

## 技术总结

### 学到的知识点

1. **Voronoi 图算法**
   - 暴力法实现：简单直观，适合小规模
   - 优化方向：Fortune 算法（O(N log N)）、跳点算法

2. **距离度量**
   - 欧式距离：圆形扩展
   - 曼哈顿距离：菱形扩展
   - 切比雪夫距离：方形扩展

3. **图像处理技术**
   - RGB 颜色空间
   - 像素遍历和索引计算
   - PNG 图像编码

### 遇到的坑

**编译警告**：stb 库内部有 `-Wmissing-field-initializers` 警告
- **原因**：结构体初始化 `{0}` 在 C++11 下不完全
- **解决**：警告来自库内部，不影响功能，可忽略
- **改进**：可以用 `-Wno-missing-field-initializers` 抑制

### 性能分析

| 参数 | 值 | 说明 |
|------|-----|------|
| 图像尺寸 | 800×600 | 48 万像素 |
| 种子点数 | 50 | 适中密度 |
| 距离计算 | 2400 万次 | 暴力法 |
| 运行时间 | <1 秒 | 可接受 |
| 输出文件 | 25KB | PNG 压缩 |

## 应用场景

Voronoi 图在实际开发中有很多应用：

### 游戏开发 🎮
- **程序化地图生成**：Roguelike 地图、岛屿生成
- **区域划分**：势力范围、领地边界
- **AI 寻路**：空间分割、导航网格

### 图形学 🎨
- **纹理合成**：有机图案、细胞纹理
- **点云处理**：曲面重建、法线估计
- **噪声生成**：Worley Noise（细胞噪声）

### 计算几何 📐
- **最近邻查询**：KNN 可视化
- **空间索引**：加速碰撞检测
- **插值算法**：自然邻点插值

## 扩展方向

### 近期计划
- [ ] 实现 **Lloyd 松弛算法**：生成更均匀的分布
- [ ] 绘制 **边界线**：提取 Voronoi 边
- [ ] 尝试不同 **距离度量**：曼哈顿、切比雪夫

### 进阶目标
- [ ] 实现 **Fortune 算法**：O(N log N) 扫描线算法
- [ ] **3D Voronoi**：扩展到三维空间
- [ ] **动画生成**：种子点移动的动态演示
- [ ] **交互式编辑**：允许用户添加/删除种子点

## 代码仓库

GitHub: [chiuhoukazusa/daily-coding-practice](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/02/02-11-voronoi-generator)

完整代码包含：
- `voronoi.cpp` - 主程序
- `stb_image_write.h` - 图像库
- `voronoi.png` - 输出示例
- `README.md` - 详细文档

---

**完成时间**: 2026-02-11 10:10  
**开发时长**: 10 分钟  
**迭代次数**: 1 次（一次成功）  
**编译器**: g++ 12.3.1  
**标准**: C++11

这是一个非常有成就感的项目！从零开始 10 分钟完成编译、运行、输出验证全流程，代码简洁高效。Voronoi 图是图形学的基础算法，掌握它对理解更复杂的空间划分算法（如 Delaunay 三角化）很有帮助。

下一步计划实现 Lloyd 松弛算法，生成更自然的分布效果。敬请期待！
