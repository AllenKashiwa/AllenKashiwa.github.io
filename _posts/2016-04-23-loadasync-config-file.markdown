---
layout:     post
title:      "Unity开发中异步加载配置文件，像读取数据库一样读取配置信息"
date:       2016-04-23 20:17:00
---

# 数据驱动

![图片来源于国产单机游戏古剑奇谭](/assets/images/in-post/gujian_cover.png)

数据驱动是软件设计与开发中不可忽视的内容，开发电子游戏更是如此。电子游戏世界是由逻辑与数据构建的。在开发过程中，我们基本上会将逻辑与数据分离开来。游戏开发完成后，逻辑部分相对改动较小，而数据的改动则相对频繁。我们可能需要不断修改来让游戏世界达到平衡。因此，在游戏开发按需加载配置是一项很重要的任务。

# CSV文件

使用**逗号分隔值**（Comma-Separated Values，**CSV**，有时也称为字符分隔值，因为分隔字符也可以不是逗号）作为配置数据的手段是较为常见的。CSV读取方便又能使用Excel进行编辑，作为游戏中的配置文件十分合适。随着游戏项目的进展，配置表将越来越多，表的内容也会越来越多，合理的加载这些文件显得尤为重要。本文将介绍如何在Unity中异步加载CSV文件并方便读取其中的数据。

# 实践

在开始之前，我想指出笔者使用的Unity版本是5.3.4。对本文的内容来说这可能无关紧要，但若有读者想要参考全部代码，并运行demo，可以前往我的[gihub](https://github.com/AllenKashiwa/StudyUnity)（请切换到csvhelper这个分支）。如果你克隆了我的库，知道我使用哪个版本的Unity可能对你有所帮助。

我们首先来设计存储表中数据的类。

```
public class CSVLine : IEnumerable
{
	private Dictionary<string,string> dataContainer = new Dictionary<string,string>();

	private void AddItem(string key,string value){
		if(dataContainer.ContainsKey(key)){
			Debug.LogError(string.Format("CSVLine AddItem: there is a same key you want to add. key = {0}", key));
		}else{
			dataContainer.Add(key,value);
		}
	}

	public string this[string key]{
		get { return dataContainer[key]; }
		set { AddItem(key, value); }
	}

	public IEnumerator GetEnumerator()
	{
		foreach (KeyValuePair<string,string> item in dataContainer)
		{
			yield return item;
		}
	}
}
```

CSVLine类将用来存储表中每一行的数据。

```
public class CSVTable : IEnumerable
{
	private Dictionary<string, CSVLine> dataContainer = new Dictionary<string, CSVLine>();

	private void AddLine(string key, CSVLine line)
	{
		if(dataContainer.ContainsKey(key)){
			Debug.LogError(string.Format("CSVTable AddLine: there is a same key you want to add. key = {0}", key));
		}else{
			dataContainer.Add(key, line);
		}
	}
	public CSVLine this[string key]
	{
		get { return dataContainer[key]; }
		set { AddLine(key, value); }
	}

	public IEnumerator GetEnumerator()
	{
		foreach (var item in dataContainer)
		{
			yield return item.Value;
		}
	}

	public CSVLine WhereIDEquals(int id)
	{
		CSVLine result = null;
		if (!dataContainer.TryGetValue(id.ToString(), out result))
		{
			Debug.LogError(string.Format("CSVTable WhereIDEquals: The line you want to get data from is not found. id:{0}", id));
		}
		return result;
	}
}
```

CSVTable用来存储每行的主键及行内容的引用。

```
public delegate void ReadCSVFinished(CSVTable result);

public class CSVHelper : MonoBehaviour
{
	#region singleton
	private static GameObject container = null;
	private static CSVHelper instance = null;
	public static CSVHelper Instance()
	{
		if (instance == null)
		{
			container = new GameObject("CSVHelper");
			instance = container.AddComponent<CSVHelper>();
		}
		return instance;
	}
	#endregion

	#region mono
	void Awake()
	{
		DontDestroyOnLoad(container);
	}
	#endregion

	#region private members
	//不同平台下StreamingAssets的路径是不同的，这里需要注意一下。
	public static readonly string csvFilePath =
	#if UNITY_ANDROID
			"jar:file://" + Application.dataPath + "!/assets/";
	#elif UNITY_IPHONE
			Application.dataPath + "/Raw/";
	#elif UNITY_STANDALONE_WIN || UNITY_EDITOR
			"file://" + Application.dataPath + "/StreamingAssets/";
	#else
			string.Empty;
	#endif
	private Dictionary<string, CSVTable> readedTable = null;
	#endregion

	#region public interfaces
	public void ReadCSVFile(string fileName, ReadCSVFinished callback)
	{
		
		if (readedTable == null)
			readedTable = new Dictionary<string, CSVTable>();
		CSVTable result;
		if (readedTable.TryGetValue(fileName, out result))
		{
			Debug.LogWarning(string.Format("CSVHelper ReadCSVFile: You already read the file:{0}", fileName));
			return;
		}
		StartCoroutine(LoadCSVCoroutine(fileName, callback));
	}

	public CSVTable SelectFrom(string tableName)
	{
		CSVTable result = null;
		if (!readedTable.TryGetValue(tableName, out result))
		{
			Debug.LogError(string.Format("CSVHelper SelectFrom: The table you want to get data from is not readed. table name:{0}",tableName));
		}
		return result;
	}
	#endregion

	#region private imp
	private IEnumerator LoadCSVCoroutine(string fileName, ReadCSVFinished callback)
	{
		string fileFullName = csvFilePath + fileName + ".csv";
		using (WWW www = new WWW(fileFullName))
		{
			yield return www;
			string text = string.Empty;
			if (!string.IsNullOrEmpty(www.error))
			{
				Debug.LogError(string.Format("CSVHelper LoadCSVCoroutine:Load file failed file = {0}, error message = {1}", fileFullName, www.error));
				yield break;
			}
			text = www.text;
			if (string.IsNullOrEmpty(text))
			{
				Debug.LogError(string.Format("CSVHelper LoadCSVCoroutine:Loaded file is empty file = {0}", fileFullName));
				yield break;
			}
			CSVTable table = ReadTextToCSVTable(text);
			readedTable.Add(fileName, table);
			if (callback != null)
			{
				callback.Invoke(table);
			}
		}
	}

	private CSVTable ReadTextToCSVTable(string text)
	{
		CSVTable result = new CSVTable();
		text = text.Replace("\r", "");
		string[] lines = text.Split('\n');
		if (lines.Length < 2)
		{
			Debug.LogError("CSVHelper ReadTextToCSVData: Loaded text is not csv format");//必需包含一行键，一行值，至少两行
		}
		string[] keys = lines[0].Split(',');//第一行是键
		for (int i = 1; i < lines.Length; i++)//第二行开始是值
		{
			CSVLine curLine = new CSVLine();
			string line = lines[i];
			if (string.IsNullOrEmpty(line.Trim()))//略过空行
			{
				break;
			}
			string[] items = line.Split(',');
			string key = items[0].Trim();//每一行的第一个值是唯一标识符
			for (int j = 0; j < items.Length; j++)
			{
				string item = items[j].Trim();
				curLine[keys[j]] = item;
			}
			result[key] = curLine;
		}
		return result;
	}
	#endregion
}
```

接着是我们的CSVReader类。这是一个mono的单例类，因为我使用了Unity实现的协程来做异步。通过ReadCSVFile接口来加载文件，加载完成后使用一个ReadCSVFinished 的回调函数获取加载好的数据。解析文件的细节在ReadTextToCSVTable函数中。
下面我们来看具体的使用：

在StreamAssets文件夹下准备一个测试用的csv文件，我的文件如下：

![](/assets/images/in-post/csv_template.png)

在场景中任意游戏物体中挂载以下脚本：

```
using UnityEngine;
using System.Collections;
using System.Collections.Generic;

public class ReadTest : MonoBehaviour
{
	private bool readFinish = false;
	void Start()
	{
		CSVHelper.Instance().ReadCSVFile("csv_test", (table) => {
			readFinish = true;
			// 可以遍历整张表
			foreach (CSVLine line in table)
			{
				foreach (KeyValuePair<string,string> item in line)
				{
					Debug.Log(string.Format("item key = {0} item value = {1}", item.Key, item.Value));
				}
			}
			//可以拿到表中任意一项数据
			Debug.Log(table["10011"]["id"]);
		});
	}

	void Update()
	{
		if (readFinish)
		{
			// 可以类似访问数据库一样访问配置表中的数据
			CSVLine line = CSVHelper.Instance().SelectFrom("csv_test").WhereIDEquals(10011);
			Debug.Log(line["name"]);
			readFinish = false;
		}
	}
}
```

运行游戏可以查看效果：

![](/assets/images/in-post/run_log.png)

# 结语

本文介绍的方法实现了异步加载配置文件，并且可以像读取数据库一样读取数据。当然，要完全像使用SQL语句那样简便并且实现数据的各种组合比较困难，这里是夸张的说法。但对于读取配置信息已经足够。我希望读者可以实现自己的扩展，使读取数据更容易。如果有兴趣的话，也可以实现一个CSVWriter类用于数据写入。