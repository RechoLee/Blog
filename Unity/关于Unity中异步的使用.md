# 关于Unity中异步的使用小结
> 一般在Unity中我们实现异步调用的话，最多使用的是Coroutine,然而在C#的语法中提供了另一种解决方案，那就是async加await

## 认识一下async/await
> 知乎上的解释：[async/await](https://www.zhihu.com/question/56651792)

这两个关键词，async和await是c#新的语法糖，用来方便我们编写高并发的代码

## 异步与多线程
首先说说我自己的理解，异步和多线程一般用于做一件这样的事：“你现在正在干一件很重要的事，这时候突然需要你去执行另一个任务，然而你现在做的工作又不能停止，所以我们需要分配另一个人去干那个突如其来的任务”

## Unity中的异步和多线程
> 首先说明一下，由于Unity的脚本程序是单线程执行的，所以在另一个线程中不能调用继承MonoBehaviour的组件


看下面的一个例子
######Thread实现
```C#
//通过Thread实现
Thread thread = new Thread(() =>
{
	GetComponent<Transform>().gameObject.SetActive(false);
});
thread.Start();
```
######Task
```c#
//通过Task
Task task = new Task(() =>
{
	GetComponent<Transform>().gameObject.SetActive(false);
});
task.Start();
```
######TaskFactory
```c#
//TaskFactory
TaskFactory factory = new TaskFactory();
Task task = factory.StartNew(() =>
{
    GetComponent<Transform>().gameObject.SetActive(false);
});
```
#### 以上都会出现下面的错误
从下图中我们可以看出，在Unity中其他线程的程序是不能访问Unity的组件的。
<div align="center">
<img src="./img/asyncAwait1.png" width="728" height="450"/>
</div>

#### 使用async/await访问
###### 调用部分
```c#
Debug.Log("before Func");
Func();
Debug.Log("after Func");
```
###### Func函数
```c#
public async void Func()
{

    await Task.Run(()=> { Thread.Sleep(2000);});
    GetComponent<Transform>().gameObject.SetActive(false);
    Debug.Log("inactive");
    await Task.Run(() => { Thread.Sleep(2000); });
    GetComponent<Transform>().gameObject.SetActive(true);
    Debug.Log("active");
    Debug.Log("Func over");
}
```
###### 执行结果
<div align="center">
<img src="./img/asyncAwait2.png" width="728" height="450"/>
</div>

#### 对比Coroutine实现
###### 调用
```c#
Debug.Log("before Func1");
StartCoroutine(Func1());
Debug.Log("after Func1");
```
###### Func1函数
```c#
public IEnumerator Func1()
{
    yield return new WaitForSeconds(2f);
    GetComponent<Transform>().gameObject.SetActive(false);
    Debug.Log("inactive");
    yield return new WaitForSeconds(2f);
    GetComponent<Transform>().gameObject.SetActive(true);
    Debug.Log("active");
    Debug.Log("Func1 over");
}
```

###### 执行结果
**从下图我们会发现一个奇怪的现象，怎么inactive后没有任何输出了呢，检查代码没有错误，我们在看看上面的图，使用async/await却能正常的执行后面的代码，这就是两者的却别所在。从本质上来说，在unity中，当一个物体变成了inactive，它上的代码也是不执行的，也就是GetComponent是不执行的（这里我们的代码就是放在这个测试物体上的），Unity中的Coroutine并没有错，而是我们需要在使用的时候注意这一点，而async/await为什么能够执行后面的代码呢，我猜测可能和async/await的语法糖背后的实现机制有关，查看了资料，最后大致了解到，当我们将一个方法标记成async方法的时候，它在执行到await的时候就返回调用了，然而await后面的逻辑是封装到一个代码块中的（语法糖背后的实现机制，有的解释是封装到IL的状态机中作为一个状态，所以相关的变量都被保存下来），等待await执行完成之后，便接着执行后续的代码块。**
<div align="center">
<img src="./img/asyncAwait3.png" width="728" height="450"/>
</div>

## 外文参考
> Unity确实为我们提供了一件重要的事情。 正如你在上面的例子中看到的，我们的异步方法默认情况下将在主unity线程上运行。 在非Unity的C＃应用程序中，异步方法通常会在单独的线程中自动运行，这在Unity中将是一个大问题，因为在这些情况下，我们并不总是能够与Unity API进行交互。 没有Unity引擎的支持，我们对Unity方法/对象的调用有时会失败，因为它们将在单独的线程上执行。 在引擎框架下，它的工作原理是因为Unity提供了一个名为UnitySynchronizationContext的默认SynchronizationContext，它会自动收集每个帧排队的任何异步代码，并在主要的Unity线程上继续运行它们。

> 不过事实证明，这足以令我们开始。 我们只需要一些帮助代码，让我们做一些更有趣的事情，而不仅仅是简单的时间延迟。

>> 地址：[来自游戏蛮牛](http://www.manew.com/thread-108589-1-1.html)
>> [原文地址](www.stevevermeulen.com/index.php/2017/09/23/using-async-await-in-unity3d-2017/)

### 自定义 Awaiters
目前，我们可以编写很多有趣的异步代码。 我们可以调用其他异步方法，我们可以使用Task.Delay，就像上面的例子中的那样，但不是很多。
作为一个简单的例子，让我们添加直接在TimeSpan上等待的能力，而不是每次像上面的例子一样总是要调用Task.Delay。 如下：

```c#
public class AsyncExample : MonoBehaviour
{
    async void Start()
    {
        await TimeSpan.FromSeconds(1);
    }
}
```
我们需要做的只是为TimeSpan类添加一个自定义GetAwaiter扩展方法：
```C#
public static class AwaitExtensions
{
    public static TaskAwaiter GetAwaiter(this TimeSpan timeSpan)
    {
        return Task.Delay(timeSpan).GetAwaiter();
    }
}
```
这是因为为了支持在较新版本的C＃中“等待”一个给定的对象，所需要的只是对象有一个名为GetAwaiter的方法返回Awaiter对象。 这是伟大的，因为它允许我们等待我们想要的任何东西，像上面的TimeSpan对象，而不需要改变实际的TimeSpan类。
我们可以使用同样的方法来支持等待其他类型的对象，包括Unity用于协程指令的所有类; 我们可以使WaitForSeconds，WaitForFixedUpdate，WWW等都等同于在协同程序中可以实现的方式。 我们还可以向IEnumerator添加一个GetAwaiter方法，以支持等待协程来允许使用旧的IEnumerator代码来互换异步代码。

```c#
public class AsyncExample : MonoBehaviour
{
    public async void Start()
    {
        // Wait one second
        await new WaitForSeconds(1.0f);
        // Wait for IEnumerator to complete
        await CustomCoroutineAsync();
        await LoadModelAsync();
        // You can also get the final yielded value from the coroutine
        var value = (string)(await CustomCoroutineWithReturnValue());
        // value is equal to "asdf" here
        // Open notepad and wait for the user to exit
        var returnCode = await Process.Start("notepad.exe");
        // Load another scene and wait for it to finish loading
        await SceneManager.LoadSceneAsync("scene2");
    }
    async Task LoadModelAsync()
    {
        var assetBundle = await GetAssetBundle("www.my-server.com/myfile");
        var prefab = await assetBundle.LoadAssetAsync<GameObject>("myasset");
        GameObject.Instantiate(prefab);
        assetBundle.Unload(false);
    }
    async Task<AssetBundle> GetAssetBundle(string url)
    {
        return (await new WWW(url)).assetBundle
    }
    IEnumerator CustomCoroutineAsync()
    {
        yield return new WaitForSeconds(1.0f);
    }
    IEnumerator CustomCoroutineWithReturnValue()
    {
        yield return new WaitForSeconds(1.0f);
        yield return "asdf";
    }
}
```

> 以上为复制的内容 后续会自己理解性的更改

## 结尾
瞎琢磨\^O^,自己的一些理解


<div align="right">
随便写写，记录生活
<p>via Recho</p>
</div>