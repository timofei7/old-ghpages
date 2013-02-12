---
layout: post
title: "generic singleton MonoBehavior part 2: singleton self loading prefab"
date: 2013-01-11 01:40
comments: true
categories:  Work
tags: [unity3d, c#, singleton]
---

I had a [previous post](blog/2012/04/26/unity-singletons/) about singletons in [Unity3D](http://www.unity3d.com) and have since added a useful functionality to that class.  One of the useful features of a singleton is that it is self instantiating.  As a MonoBehaviour in Unity you may have a class that has serializable public variables available within the Unity editor to allow easy tweaking of variables / for artists to modify / to hook up to other Unity compenents easily. Additionally you are probably using [prefabs](http://docs.unity3d.com/Documentation/Manual/Prefabs.html) to manage game components in your scenes.

What I wanted to have was a prefab with a singleton on it that I could dump in the Resources folder and have it be self-loading.  Just as singletons allow for really nice static access methods, this would allow me to really easily organize my project as a bunch of prefabs and when a level needed a particular component it would just use it, no need to instantiate and keep a reference within some big main class, nor keep it in the scene heirarchy at all to begin with. 

<!--more--> 

Below is my updated singleton class. What I ended up doing was adding an Attribute so you could just define the Prefab attribute for any singletons that did have associated prefabs and off you go.  I have left in some debug statements commented out for clarity.  The only caveat is that of course you could accidentally have added multiple instances of the prefab in your scene -- what I do now is to just throw an exception if that is the case (an alternative may be to just choose one and self-destruct the others but deciding which is correct is unclear). 

{% codeblock Singleton.cs lang:c# %}
using UnityEngine;
using System.Collections;
using System;

/// <summary>
/// Prefab attribute. Use this on child classes
/// to define if they have a prefab associated or not
/// By default will attempt to load a prefab
/// that has the same name as the class,
/// otherwise [Prefab("path/to/prefab")] to define it specifically. 
/// </summary>
[AttributeUsage(AttributeTargets.Class,Inherited = true)]
public class PrefabAttribute : Attribute
{
    string _name;
	public string Name { get { return this._name; } }
	public PrefabAttribute() { this._name = ""; }
    public PrefabAttribute(string name) { this._name = name; }
}

/// <summary>
/// MONOBEHAVIOR PSEUDO SINGLETON ABSTRACT CLASS
/// usage		: can be attached to a gameobject and if not
/// 			: this will create one on first access
/// example		: '''public sealed class MyClass : Singleton<MyClass> {'''
/// references	: http://tinyurl.com/d498g8c
/// 			: http://tinyurl.com/cc73a9h
/// 			: http://unifycommunity.com/wiki/index.php?title=Singleton
/// </summary>
public abstract class Singleton<T> : MonoBehaviour where T : MonoBehaviour
{
	
	private static T _instance = null;
	public static bool IsAwake { get { return (_instance != null); } }
			
	/// <summary>
	/// gets the instance of this Singleton
	/// use this for all instance calls:
	/// MyClass.Instance.MyMethod();
	/// or make your public methods static
	/// and have them use Instance internally
	/// for a nice clean interface
	/// </summary>
	public static T Instance {
		get {
			Type mytype = typeof(T);
			if (_instance == null) {
				_instance = (T)FindObjectOfType (mytype);
				if (_instance == null) {
					//Debug.Log("initializing instance of: " + mytype.Name);
					string goName = mytype.ToString ();
					GameObject go = GameObject.Find (goName);
					if (go == null) // try again searching for a cloned object
					{
						go = GameObject.Find (goName+"(Clone)");
						if (go != null)
						{
							//Debug.Log("found clone of object using it!"); 
						}
					}
					
					if (go == null) //if still not found try prefab or create
					{
			            bool hasPrefab = Attribute.IsDefined(mytype, typeof(PrefabAttribute));
						// checks if the [Prefab] attribute is set and pulls that if it can
						if (hasPrefab)
						{
							PrefabAttribute attr = (PrefabAttribute)Attribute.GetCustomAttribute(mytype,typeof(PrefabAttribute));
							string prefabname = attr.Name;
							//Debug.LogWarning(goName + " not found attempting to instantiate prefab... either: " + goName + " or: " + prefabname);
							try
							{
								if (prefabname != "")
								{
									go = (GameObject)Instantiate(Resources.Load(prefabname, typeof(GameObject)));
								}
								else
								{
									go = (GameObject)Instantiate(Resources.Load(goName, typeof(GameObject)));
								}
							} catch (Exception e)
							{
								Debug.LogError("could not instantiate prefab even though prefab attribute was set: " + e.Message + "\n" + e.StackTrace);
							}
						}
						if (go == null)
						{
							//Debug.LogWarning(goName + " not found creating...");
							go = new GameObject ();
							go.name = goName;
						}
					}
					_instance = go.GetComponent<T>();
					if (_instance == null)
					{
						_instance = go.AddComponent<T>();
					}
				}
				else
				{ 
					//Debug.Log(mytype.Name + " had to be searched for but was found"); 
					int count = FindObjectsOfType(mytype).Length;
					if (count > 1)
					{
						Debug.LogError("Singleton: there are " + count.ToString() + " of " + mytype.Name);
						throw new Exception("Too many (" + count.ToString() + ") prefab singletons of type: " + mytype.Name);
					}
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
	// 	and add any initialization code you want such as
	// 	DontDestroyOnLoad(this.gameObject);
	// 	if you want this to persist across loads
	//  or if you want to set a parent object with SetParent()
	
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
				parentGO.transform.parent = null;
			}
			this.transform.parent = parentGO.transform;
		}
	}
	
}

{% endcodeblock %}

What do you think?  Only other caveats are those associated with using the [Resources folder](http://docs.unity3d.com/Documentation/ScriptReference/Resources.html) in general such as not being able to strip out components that aren't used.  Since my games so far have been small this hasn't been an issue and I know that I should clean out any resources that I'm definitely not using. 


