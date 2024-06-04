



# ConvertHullAlgorithm （凸包算法）



## 引用

> 《计算几何》-导言：凸包的例子

## 前言

1. 算法的基本逻辑与理念来自于《计算几何》这本书，后面其他几章的演示也都会在Revit中实现调试，希望能够每个算法都找一个合适的实现方向在Revit中实现
2. 所有的代码都是c#编写并在Revit中调试，因为部分接口与判定使用了Revit API，如果想单纯运行算法可以修改部分API到常用库

## 算法逻辑

算法比较简单，主要是获取一个点集合的外围边界，通过对点与向量相比较从而获取边界区域。


![ConvertHull](https://github.com/imkcrevit/bimath.github.io/assets/44009852/9be1a343-d8ef-466a-b586-c9a7c5e25f63)


1. 算法将所有点集按照点的X值从小到大排序，排序完成后连接最左侧的点与最右侧的点可以看到会将整个子集分割为两个子集：上子集，下子集

 ![ConvertHull2](https://github.com/imkcrevit/bimath.github.io/assets/44009852/ef42ede1-c82d-4b9a-8fbf-1e7bf72c2e04)


2.  如果找到边界则代表边界点将会包围所有的点，故边界点两两连接后将会是最大的向量，保证向量右侧没有其他的点，视为边界点

![ConvertHull3](https://github.com/imkcrevit/bimath.github.io/assets/44009852/4e315dc8-efb7-4cfe-845e-ab7ac2b58303)


3. 通过上面的步骤，可以创建一个遍历，获得最外侧向量并逐步向最大值位置推进，从而描绘出边界区域

![ConvertHull4](https://github.com/imkcrevit/bimath.github.io/assets/44009852/bee8f356-717c-481f-abac-8d4590bb7049)


4. 在完成上子集遍历后(及点推进到了X值最大点)，将顺时针继续遍历子集，此时需要将整个数组的集合按照**从大到小**转置，并将参照的向量Dir修改为(TheRight - TheLeft)![ConvertHull5](https://github.com/imkcrevit/bimath.github.io/assets/44009852/89e25ab4-9555-4648-91c8-74691c0cfc37)


5. 最左侧断点与最右侧断点相互成为起点，终点后，即可完成所有的边界遍历。



### 代码

1. 创建一个判定，两个向量的方向性，最开始以两个起始点的向量为比较向量，首先遍历上子集，则需要判定目标向量是否在次向量的上方或是左侧.

**在判定位置使用了向量叉乘，并判断向量的Z值，原理是：通过计算法向量根据左手法则可以判定向量的旋转方向，从而判定两个向量的位置关系**

![ConvertHull6](https://github.com/imkcrevit/bimath.github.io/assets/44009852/cfc62f8c-46bb-4c2b-9157-657b234f2c8d)


```
	/// <summary>
    /// Compare the vector and if value locate right of original , return true , else false
    /// In the upper hull and lower hull , the vector dot angle will -90 ~ 0 
    /// </summary>
    /// <param name="original"></param>
    /// <param name="value"></param>
    /// <returns></returns>
    public bool IsRight(XYZ original, XYZ value)
    {
        //Dot Product
        var angleValue = original.CrossProduct(value);
        //90°
        if (angleValue.Z >= 0) return true;
        
        return false;
    }

```

2. 创建遍历的函数，此处放上书中的伪代码与c#代码解释。

```
算法 CONVEXHULL(P)
输入：平面点集P
输出：由CH(P)的所有顶点沿顺时针方向组成的一个列表
1. 根据x-坐标，对所有点进行排序，得到序列p1, …, pn
2. 在Lupper 中加入p1 和p2（p1 在前）
3. for (i← 3 to n)
4. do 在Lupper 中加入pi
5. while (Lupper 中至少还有三个点，而且最末尾的三个点所构成的不是一个右拐)
6. do 将倒数第二个顶点从Lupper 中删去
7. 在Llower 中加入pn 和pn-1（pn 在前）
8. for (i← n-2 downto 1)
9. do 在Llower 中加入pi
10. while (Llower 中至少还有三个点，而且最末尾的三个点所构成的不是一个右拐)
11. do 将倒数第二个顶点从Llower 中删去
12. 将第一个和最后一个点从Llower 中删去
（以免在上凸包与下凸包联接之后，出现重复顶点）
13. 将Llower 联接到Lupper 后面（将由此得到的列表记为L）
14. return(L)

```

- 定义起始点`TheLeft`,终止点`TheLast`和点集集合`vertices`
- 集合与`TheLeft`的向量并与`fakeDir = TheLast - TheLeft`判定是否在右侧，如果点在向量右侧，则将点`Point`的数据迭代，替换掉`fakeDir`，并进入下一次循环，查找是否在`fakeDir`的右侧
- 循环结束后将最右侧点添加到集合中，从而查找出所有的边界点

```
private List<XYZ> GetHullVertices(bool isUp = true)
    {
        var vertices = new List<XYZ>();
        vertices.AddRange(_vertices);
        if (!isUp)
        {
            vertices =  vertices.OrderByDescending(x=>x.X).ToList();
        }
        var theLeft = vertices.First();
        vertices.Remove(theLeft);
        var theLast = vertices.Last();
        var fakeDir = (theLast - theLeft).Normalize();
        var upHullVertices = new List<XYZ>(){theLeft};
        var fakeIndex = 1;
        while (true)
        {
            if (theLeft.IsAlmostEqualWithoutZ(theLast))
            {
                //upHullVertices.Add(theLast);
                break;
            }
            var fakePoint = XYZ.Zero;
            for (int i = 0; i < vertices.Count ; i++)
            {
                var point = vertices[i];
                var dir = (point - theLeft).Normalize();
                //if is dir right to fakeDir , replace value to dir and fakePoint
                if (IsRight(fakeDir, dir))
                {
                    fakeDir = dir;
                    fakePoint = point;
                }
            }
            //remove the theLeft point and replace theLeft to new Point;
            theLeft = fakePoint;
            if (vertices.Remove(theLeft))
            {
                upHullVertices.Add(theLeft);
                fakeDir = (theLast - theLeft).Normalize();
            }
            fakeIndex++;
        }

        return upHullVertices;
    }
   
```



## 运行结果

![ConvertHull7](https://github.com/imkcrevit/bimath.github.io/assets/44009852/bc143a6b-f309-4825-8bb2-2fc12ad3b8be)

![ConvertHull8](https://github.com/imkcrevit/bimath.github.io/assets/44009852/d9239d25-71a2-4e7d-ae6b-b401332c5817)

