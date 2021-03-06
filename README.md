# MOVIELENS
# 这个项目用于展示通过训练movielens数据集对用户进行电影推荐
_项目初期，需要用到很多包，例如numpy、pandas、sklearn、os、math等等，本人是缺什么包就在cmd命令行下面pip什么包，后来在
实际运行中发现很多问题：pip自己安装的包，只安装该包所依赖的包，并非该包完整的包，在实际导入中会出现诸多can't find module这类的问题，很让人蛋疼。_

_笔者总结后给出一个这类问题的解决方案：强烈建议不要偷懒使用pip，应当去官网上自己手动下载相应的安装包后缀名为.whl
给出一个网址，该网址包含了python目前几乎所有的版本的安装包  
[Python各安装包](https://www.lfd.uci.edu/~gohlke/pythonlibs/#numpy "Python安装包")
下载完之后再手动pip install 路径\文件名.whl_
##  一、使用pandas对数据集进行读取查看操作（查看数据集）
下载完movielens数据集后
我们先导入numpy,os,pandas这三个包对数据进行读取和查看
代码如下：
```
import numpy
import os
import pandas as pd
rating = pd.read_csv(r'F:\pythontest\movielens\ratings.csv', index_col=None)
rating.head(10)
print(rating)#读取rating文件数据
movie = pd.read_csv(r'F:\pythontest\movielens\movies.csv', index_col=None)
movie.head(10)
print(movie)#读取movie文件数据

```
代码运行截图如下：  
![pandas读取数据集](https://github.com/Gaoshiguo/MOVIELENS/blob/master/movielens/1.png)  
代码运行效果图如下：
![image](https://github.com/Gaoshiguo/MOVIELENS/blob/master/movielens/2.png)  
我们可以看到读取的rating文件数据的前五项包括了`<userId>` `<movieId>` `<rating>` `<timestap>`  


## 二、对数据进行预处理
我们使用`<merge>`函数对ratings和movies两个文件进行合并，关联关系使用userId，合并完之后，再将数据按比率分成训练集和测试集
具体代码如下：
```
rating_count_by_movie = data.groupby(['movieId', 'title'], as_index=False)['rating'].count()
rating_count_by_movie.columns = ['movieId', 'title', 'rating_count']
rating_count_by_movie.sort_values(by=['rating_count'], ascending=False, inplace=True)
print(rating_count_by_movie[:10])#将合并后的文件按照降序排列打印

```
![image](https://github.com/Gaoshiguo/MOVIELENS/blob/master/movielens/3.png)
![image](https://github.com/Gaoshiguo/MOVIELENS/blob/master/movielens/4.png)

**我们可以看到数据被分成了训练集和测试集两个部分，但是数据非常多。笔者在后期实验中发现，如此之多的数据会在矩阵运算中发生溢出同时也为了运算简便，展示效果，所以我们对数据采取抽样化处理具体操作如下：**  

_我们使用pandas包中的sample函数对整个数据集中的文件随机抽样提取其**0.001%**的数据_

`<ratingData =ratingsData.sample(n=None,frac=0.0001,replace=False,weights=None,random_state=None,axis=None)
movieData =moviessData.sample(n=None,frac=0.0001,replace=False,weights=None,random_state=None,axis=None)>`  


![image](https://github.com/Gaoshiguo/MOVIELENS/blob/master/movielens/5.png)
![image](https://github.com/Gaoshiguo/MOVIELENS/blob/master/movielens/6.png)

_可以看到抽样化处理后数据变得很少，训练集只有1303条，测试集只有671条。方便运算_

## 三、建立用户、电影评分矩阵  

建立用户、电影评分矩阵,建立电影Id和用户Id的对应关系  
```
#建立用户电影评分矩阵，建立电影ID和用户ID的对应关系

trainRatingsPivotdata =pd.pivot_table(trainratingData[['userId','movieId','rating']],columns=['movieId'],index=['userId'],values='rating',fill_value=0)
moviesMap = dict(enumerate(list(trainRatingsPivotdata.columns)))
usersMap = dict(enumerate(list(trainRatingsPivotdata.index)))
ratingValues = trainRatingsPivotdata.values.tolist()

```
![image](https://github.com/Gaoshiguo/MOVIELENS/blob/master/movielens/7.png)


## **四、计算用户相似度**

```

#定义函数计算用户相似度
def calCosineSimilarity(list1,list2):
    res = 0
    denominator1 = 0
    denominator2 = 0
    for (val1,val2) in zip(list1,list2):
        res += (val1 * val2)
        denominator1 += val1 ** 2
        denominator2 += val2 ** 2
    return res / (math.sqrt(denominator1 * denominator2))  

```

```

用户相似度矩阵是一个上三角矩阵对角线元素为0
userSimMatrix = np.zeros((len(ratingValues),len(ratingValues)),dtype=np.float32)
for i in range(len(ratingValues)-1):
    for j in range(i+1,len(ratingValues)):
        userSimMatrix[i,j] = calCosineSimilarity(ratingValues[i],ratingValues[j])
        userSimMatrix[j,i] = userSimMatrix[i,j]

userMostSimDict = dict()
for i in range(len(ratingValues)):
    userMostSimDict[i] = sorted(enumerate(list(userSimMatrix[0])),key = lambda x:x[1],reverse=True)[:10]

```


![image](https://github.com/Gaoshiguo/MOVIELENS/blob/master/movielens/8.png)

## 五、计算推荐系数前十的用户

```
#计算推荐权值
userRecommendValues = np.zeros((len(ratingValues),len(ratingValues[0])),dtype=np.float32)
for i in range(len(ratingValues)):
    for j in range(len(ratingValues[i])):
        if ratingValues[i][j] == 0:
            val = 0
            for (user,sim) in userMostSimDict[I]:
                val += (ratingValues[user][j] * sim)
            userRecommendValues[i,j] = val

#为每个用户推荐十部电影
userRecommendDict = dict()
for i in range(len(ratingValues)):
    userRecommendDict[i] = sorted(enumerate(list(userRecommendValues[i])),key = lambda x:x[1],reverse=True)[:10]

#为索引的用户id和电影id转换为真正的用户id和电影id
userRecommendList = []
for key,value in userRecommendDict.items():
    user = usersMap[key]
    for (movieId,val) in value:
        userRecommendList.append([user,moviesMap[movieId]])
        
```
![image](https://github.com/Gaoshiguo/MOVIELENS/blob/master/movielens/9.png)
## 六、打印结果

```

#我们将推荐结果的电影id转换成对应的电影名，并打印结果
recommendDF = pd.DataFrame(userRecommendList,columns=['userId','movieId'])
recommendDF = pd.merge(recommendDF,moviesDF[['movieId','title']],on='movieId',how='inner')
recommendDF.tail(10)

```













