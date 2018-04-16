# 关于Unity中异步的使用小结
> 一般在Unity中我们实现异步调用的话，最多使用的是Coroutine,然而在C#的语法中提供了另一种解决方案，那就是async加await

## 认识一下async/await
> 知乎上的解释：[async/await](https://www.zhihu.com/question/56651792)
这两个关键词，async和await是c#新的语法糖，用来方便我们编写高并发的代码