---
title: "UI自动化图片相似度检测"
date: 2020-11-25T22:04:53+08:00
---

> 在不同分辨率手机上校验图标是否存在的方法


## Brute-Force匹配器基础

Brute-Force匹配器很简单，它取第一个集合里一个特征的描述子并用第二个集合里所有其他的特征和他通过一些距离计算进行匹配。最近的返回。

对于BF匹配器，首先我们得用cv2.BFMatcher()创建BF匹配器对象.它取两个可选参数，第一个是normType。它指定要使用的距离量度。默认是cv2.NORM_L2。对于SIFT,SURF很好。（还有cv2.NORM_L1）。对于二进制字符串的描述子，比如ORB，BRIEF，BRISK等，应该用cv2.NORM_HAMMING。使用Hamming距离度量，如果ORB使用VTA_K == 3或者4，应该用cv2.NORM_HAMMING2

第二个参数是布尔变量，crossCheck模式是false，如果它是true，匹配器返回那些和(i, j)匹配的，这样集合A里的第i个描述子和集合B里的第j个描述子最匹配。两个集合里的两个特征应该互相匹配，它提供了连续的结果，

当它创建以后，两个重要的方法是BFMatcher.match()和BFMatcher.knnMatch()。第一个返回最匹配的，第二个方法返回k个最匹配的，k由用户指定。当我们需要多个的时候很有用。

想我们用cv2.drawKeypoints()来画关键点一样，cv2.drawMatches()帮我们画匹配的结果，它把两个图像水平堆叠并且从第一个图像画线到第二个图像来显示匹配。还有一个cv2.drawMatchesKnn来画k个最匹配的。如果k=2，它会给每个关键点画两根匹配线。所以我们得传一个掩图，如果我们想选择性的画的话。

## SIFT实现

### 加载图像 找到描述子
```
def sift(filename):
    img = cv2.imread(filename) # 读取文件
    img = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY) # 转化为灰度图
    sift = cv2.SIFT_create()
    keyPoint, descriptor = sift.detectAndCompute(img, None) # 特征提取得到关键点以及对应的描述符（特征向量）
    return img, keyPoint, descriptor
```

### bf.match

接着我们用距离度量cv2.NORM_L2创建一个BFMatcher对象，crossCheck设为真。然后我们用Matcher.match()方法来获得两个图像里最匹配的。我们按他们距离升序排列，这样最匹配的（距离最小）在最前面。然后我们画出最开始的50个匹配

```
import cv2
from matplotlib import pyplot as plt
def match(filename1, filename2):
    img1, kp1, des1 = sift(filename1)
    img2, kp2, des2 = sift(filename2)
    bf = cv2.BFMatcher(cv2.NORM_L2, crossCheck=True)  # sift的normType应该使用NORM_L2或者NORM_L1
    matches = bf.match(des1, des2)
    matches = sorted(matches, key=lambda x: x.distance)
    len_before = len(matches)
    for m in matches:
        for n in matches:
            if(m != n and m.distance >= n.distance*0.75):
                matches.remove(m)
                break
    len_after = len(matches)
    img = cv2.drawMatches(img1, kp1, img2, kp2, matches[:20], img2, flags=2)
    plt.imshow(img), plt.show()
    val = round(len_after / len_before, 2)
    print(val)
    return val
```

### bf.knnMatch 

BFMatcher.knnMatch()方法会返回最佳匹配。而该方法为每个关键点返回 k 个最佳匹配(降序排列之后取前 k 个)，其中 k 是由用户设定的，设置k=2这样我们可以应用比率检测
```
import cv2
from matplotlib import pyplot as plt
def match(filename1, filename2):
    img1, kp1, des1 = sift(filename1)
    img2, kp2, des2 = sift(filename2)
    bf = cv2.BFMatcher()
    matches = bf.knnMatch(des1, des2, k=2)
    good = []
    for m, n in matches:
        if m.distance < 0.75 * n.distance:
            good.append([m])
    img = cv2.drawMatchesKnn(img1, kp1, img2, kp2, good, None, flags=2)
    plt.imshow(img), plt.show()
    val = round(len(good) / len(matches), 2)
    print(val)
    return val
```

### 基于FlannBasedMatcher的SIFT实现

FLANN(Fast_Library_for_Approximate_Nearest_Neighbors)快速最近邻搜索包，它是一个对大数据集和高维特征进行最近邻搜索的算法的集合,而且这些算法都已经被优化过了。在面对大数据集时它的效果要好于 BFMatcher。
经验证，FLANN比其他的最近邻搜索软件快10倍。使用 FLANN 匹配,我们需要传入两个字典作为参数。这两个用来确定要使用的算法和其他相关参数等。
第一个是 IndexParams。
index_params = dict(algorithm = FLANN_INDEX_KDTREE, trees = 5) 。
这里使用的是KTreeIndex配置索引，指定待处理核密度树的数量（理想的数量在1-16）。
第二个字典是SearchParams。
search_params = dict(checks=100)用它来指定递归遍历的次数。值越高结果越准确，但是消耗的时间也越多。实际上，匹配效果很大程度上取决于输入。
5kd-trees和50checks总能取得合理精度，而且短时间完成。在之下的代码中，丢弃任何距离大于0.7的值，则可以避免几乎90%的错误匹配，但是好的匹配结果也会很少。

```
import cv2
from matplotlib import pyplot as plt
def match(filename1, filename2):
    img1, kp1, des1 = sift(filename1)
    img2, kp2, des2 = sift(filename2)
    # FLANN 参数设计
    FLANN_INDEX_KDTREE = 0
    index_params = dict(algorithm=FLANN_INDEX_KDTREE, trees=5)
    search_params = dict(checks=50)
    flann = cv2.FlannBasedMatcher(index_params, search_params)
    matches = flann.knnMatch(des1, des2, k=2)
    good = []
    for m, n in matches:
        if m.distance < 0.75 * n.distance:
            good.append([m])
    img = cv2.drawMatchesKnn(img1, kp1, img2, kp2, good, None, flags=2)
    plt.imshow(img), plt.show()
    val = round(len(good) / len(matches), 2)
    print(val)
    return val
```

## SURF实现
module 'cv2.cv2' has no attribute 'xfeatures2d' 报错，版本太高，需安装opencv-contrib-python这个库

```
pip install opencv_python==3.4.2.16 
pip install opencv-contrib-python==3.4.2.16
```

```
def surf(filename):
    img = cv2.imread(filename) # 读取文件
    img = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY) # 转化为灰度图
    sift = cv2.xfeatures2d.SURF_create()
    keyPoint, descriptor = sift.detectAndCompute(img, None) # 特征提取得到关键点以及对应的描述符（特征向量）
    return img, keyPoint, descriptor
```
- bf.knnMatch
```
import cv2
from matplotlib import pyplot as plt
def match(filename1, filename2):
    img1, kp1, des1 = surf(filename1)
    img2, kp2, des2 = surf(filename2)
    bf = cv2.BFMatcher()
    matches = bf.knnMatch(des1, des2, k=2)
    good = []
    for m, n in matches:
        if m.distance < 0.75 * n.distance:
            good.append([m])
    img = cv2.drawMatchesKnn(img1, kp1, img2, kp2, good, None, flags=2)
    plt.imshow(img), plt.show()
    val = round(len(good) / len(matches), 2)
    print(val)
    return val
```
- flann.knnMatch
```
import cv2
from matplotlib import pyplot as plt
def match(filename1, filename2):
    img1, kp1, des1 = surf(filename1)
    img2, kp2, des2 = surf(filename2)
    # FLANN 参数设计
    FLANN_INDEX_KDTREE = 0
    index_params = dict(algorithm=FLANN_INDEX_KDTREE, trees=5)
    search_params = dict(checks=50)
    flann = cv2.FlannBasedMatcher(index_params, search_params)
    matches = flann.knnMatch(des1, des2, k=2)
    good = []
    for m, n in matches:
        if m.distance < 0.75 * n.distance:
            good.append([m])
    img = cv2.drawMatchesKnn(img1, kp1, img2, kp2, good, None, flags=2)
    plt.imshow(img), plt.show()
    val = round(len(good) / len(matches), 2)
    print(val)
    return val
```

## 基于BFMatcher的ORB实现

```
def orb(filename):
    img = cv2.imread(filename) # 读取文件
    img = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY) # 转化为灰度图
    sift = cv2.ORB_create()
    keyPoint, descriptor = sift.detectAndCompute(img, None) # 特征提取得到关键点以及对应的描述符（特征向量）
    return img, keyPoint, descriptor
```

```
import cv2
from matplotlib import pyplot as plt
def match(filename1, filename2):
    img1, kp1, des1 = orb(filename1)
    img2, kp2, des2 = orb(filename2)
    bf = cv2.BFMatcher(cv2.NORM_HAMMING) # orb的normType应该使用NORM_HAMMING
    matches = bf.knnMatch(des1, des2, k = 2) # drawMatchesKnn
    good = []
    for m, n in matches:
        if m.distance < 0.75 * n.distance:
            good.append([m])
    img = cv2.drawMatchesKnn(img1, kp1, img2, kp2, good, None, flags=2)
    plt.imshow(img), plt.show()
    val = round(len(good) / len(matches), 2)
    print(val)
    return val
```


——————
参考
https://testerhome.com/articles/26924
https://blog.csdn.net/zhangziju/article/details/79754652
https://www.jianshu.com/p/ed57ee1056ab
