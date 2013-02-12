---
layout: post
title: "generic singleton MonoBehavior class"
date: 2012-04-26 01:40
comments: true
categories:  Work
tags: [unity3d, c#, singleton]
---

In [Unity3D](http://www.unity3d.com) having a [singleton](http://en.wikipedia.org/wiki/Singleton_pattern) class is very useful, whether for "global" state or simply for the convenience of having a static accessor so you don't have to have lots of: <code>FindObjectOfType(typeof(Builder)) as Builder;</code>

So you code up a C# singleton and then realize that you actually need it to be a [MonoBehavior](http://unity3d.com/support/documentation/ScriptReference/MonoBehaviour.html), not just a [ScriptableObject](http://unity3d.com/support/documentation/ScriptReference/ScriptableObject.html) -- say you want the singleton to run [coroutines](http://unity3d.com/support/documentation/ScriptReference/index.Coroutines_26_Yield.html), or have a transform, or any other MonoBehavior feature.  But monobehaviors [can't](http://unity3d.com/support/documentation/ScriptReference/MonoBehaviour.Awake.html) be initialized with a constructor. 

So what you want is a monobehavior pseudo singleton pattern.  The unity community wiki has some [good info](http://unifycommunity.com/wiki/index.php?title=Singleton)
 about the various options but all require boilerplate. 

So I wrote a parametrized generic pseudo singleton monobehavior class.  I attempted to compile all the best advice out there, so it is threadsafe, has some extra stuff for setting a parent, doesn't implement an Awake, and initializes lazily.

<!--more--> 

The key was the C# [where](http://msdn.microsoft.com/en-us/library/d5x73970.aspx) keyword, used to constrain the parameterizing type. So in this case both the class is a MonoBehavior and its parameterization type is a MonoBehavior.

So now this blog has its first code section: 

{% codeblock Singleton.cs lang:c# %}
using UnityEngine;
using System.Collections;

/// <summary>
/// MONOBEHAVIOR PSEUDO SINGLETON ABSTRACT CLASS
/// usage	: best is to be attached to a gameobject but if not that is ok,
/// 		: this will create one on first access
/// example	: '''public sealed class MyClass : Singleton<MyClass> {'''
/// references	: http://tinyurl.com/d498g8c
/// 		: http://tinyurl.com/cc73a9h
/// 		: http://unifycommunity.com/wiki/index.php?title=Singleton
/// </summary>
public abstract class Singleton<T> : MonoBehaviour where T : MonoBehaviour
{

	private static T _instance = null;
			
	/// <summary>
	/// gets the instance of this Singleton
	/// use this for all instance calls:
	/// MyClass.Instance.MyMethod();
	/// or make your public methods static
	/// and have them use Instance
	/// </summary>
	public static T Instance {
		get {
			if (_instance == null) {
				_instance = (T)FindObjectOfType (typeof(T));
				if (_instance == null) {
					
					string goName = typeof(T).ToString ();			
					
					GameObject go = GameObject.Find (goName);
					if (go == null) {
						go = new GameObject ();
						go.name = goName;
					}
					
					_instance = go.AddComponent<T> ();					
				}
			}
			return _instance;
		}
	}
	
	/// <summary>
	/// for garbage collection
	/// </summary>
	public virtual void OnApplicationQuit ()
	{
		// release reference on exit
		_instance = null;
	}
	
	// in your child class you can implement Awake()
	// and add any initialization code you want such as
	// DontDestroyOnLoad(go);
	// if you want this to persist across loads
	// or if you want to set a parent object with SetParent()
	
	/// <summary>
	/// parent this to another gameobject by string
	/// call from Awake if you so desire
	/// </summary>
	protected void SetParent (string parentGOName)
	{
		if (parentGOName != null) {
			GameObject parentGO = GameObject.Find (parentGOName);
			if (parentGO == null) {
				parentGO = new GameObject ();
				parentGO.name = parentGOName;
			}
			this.transform.parent = parentGO.transform;
		}
	}
}
{% endcodeblock %}
