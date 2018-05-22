# Unity WWW不得不知的秘密
## WWW加载图片
## WWW加载视频
### Handheld的运用
> [参考网址](https://blog.csdn.net/hasion/article/details/45170317)

```C#
using UnityEngine;
using System.Collections;
using System.IO;
public class PlayerMovie : MonoBehaviour {
	//网络视频地址
	private string Url_movie;
	//视频下载本地存储地址
	private string Url_save;
	//文件
	FileInfo file;
	void Awake()
	{
		Url_movie="http://xxx.../business_work/we.avi";
		Url_save=Application.persistentDataPath+"/test.avi";
		//初始化文件
		file=new FileInfo (Url_save);
	}
	void Start()
	{
		//Handheld.PlayFullScreenMovie(Url_movie, Color.black, FullScreenMovieControlMode.Hidden);
		//判断文件是否下载过
		if(!file.Exists)
		{
			StartCoroutine("downmovie");
		}else
		{
			//文件存在 直接播放视频
			print ("文件存在 直接播放视频");
			Handheld.PlayFullScreenMovie(Url_save,Color.black,FullScreenMovieControlMode.Full);
		}
	}
	IEnumerator downmovie()
	{
		//加载www
		WWW _www=new WWW(Url_movie);
		yield return _www;

		if(_www.isDone)
		{
			print("视频加载完成");
			//获取www的字节
			byte[] bytes=_www.bytes;
			creat(bytes);
		}

	}
	//文件的流写入
	void creat(byte [] bytes)
	{
		Stream str;
		//文件创建
		str=file.Create();
		//文件写入
		str.Write(bytes,0,bytes.Length);
		//关闭并销毁流
		str.Close();
		str.Dispose();

		//播放视频
		Playermov();
	}
	void Playermov()
	{		Handheld.PlayFullScreenMovie(Url_save,Color.black,FullScreenMovieControlMode.Full);
	}
}
```
### 通过www
> [参考网址]()

```C#
public void OnButtonClick()
{
   string videourl = "";
    StartCoroutine(downloadVideo(videourl));
}
IEnumerator downloadVideo(string videourl)
{
    WWW www = new WWW(videourl);
    yield return www;
    if(www.isDone)
    {
        //下载视频到（视频格式：.mp4）
       //Application.persistentData路径
    }
}

//通过文件流将字节数组转换为视频
FileStream fs = new FileStream(Application.persistentData,FileMode.Creat,FileAccess.Write);
fs.Write(www.bytes,0,[url]www.bytes.length[/url]);
fs.Close();
```
### 通过 UnityWebRequest