---
layout: post
title: "wrapping Unity C# coroutines for exception handling, value retrieval, and locking"
date: 2013-02-05 01:40
comments: true
categories: Codes 
tags: [unity3d, c#, coroutine]
---

Firstly, coroutines are awesome.  If you aren't familiar with them (particularly in the contenxt of Unity3d) then you should be.  In short, [coroutines](http://en.wikipedia.org/wiki/Coroutine) are methods that can suspend and resume execution.  In the context of Unity what this means is that you can have methods that appear to run concurrently. The Unity documentation has some [examples](http://docs.unity3d.com/Documentation/ScriptReference/index.Coroutines_26_Yield.html).

Coroutines are **the** way to script a lot of things in Unity, however there are a few problems that you may run into if you use them heavily: exception handling, return values, and locking. Especially if you use nested coroutines!

<!--more-->

###Return Values###

Coroutines in Unity, although built on iterators, don't handle return values. Even though it appears as if you can return a value, internally return values are used to keep track of where to resume.  But, "I want a coroutine to return something!", you say.  So you work around that using a callback:
``` c#
IEnumerator FooBar()
{
	yield return StartCoroutine(DoSomethingEverySecond(success=> {
		if (success)
			doMoreStuff;
	}));
}

IEnumerator DoSomethingEverySecond(System.Action<bool> success)
{
	bool result = false;
	foreach (Something s in somethings)
	{
		if (s.doStuff())
			result = true;
		yield return new WaitForSeconds(1f);
	}
	if (success != null) success(result);
}
```
Ok so this helps with getting a return value out of a coroutine.  It works well enough. Only caveat is in the callback code you can't do another yield, not a big problem, and solveable by setting some variables. We'll return to this topic. 

###Exception Handling###

There is no exception handling at the coroutine level.  You can't put a ```yield``` statement in a ```try…catch``` block.  This in particular can cause a problem if you have nested coroutines:
``` c# 
void Start()
{
	try { StartCoroutine(ParentCoroutine()); } //can try...catch if there's no yielding
	catch (Exception e) { Debug.LogError("oh noes we had a problem: " + e.Message); }
}

IEnumerator ParentCoroutine()
{
	yield return StartCoroutine(NestedCoroutineDoSomeStuff());  //can't try...catch these
	yield return StartCoroutine(NestedCoroutineFoo()); //if this has an exception nothing below executes
	if (stuff)
		yield return StartCoroutine(NestedCoroutineBar());
}
```

Logically, if any of the nested coroutines throw an exception then the parent will stop execution.  Now, this is only fair, but since you can't ```try…catch``` the nested coroutines it forces you to have to do error handling at a much more obnoxious nested level.  You have to make sure to handle all exceptions in each subroutine, even if it doesn't really make sense to do that.  For instance you have a MovePiece coroutine,  if it fails (say the piece was attacked mid move) what you'd like to do in the parent coroutine (which is something like AttackState) is notice the failed move and handle it there.  If the MovePiece coroutine does a bunch of stuff (set some flags, run a few different animations) the parent probably only cares whether it was successfull or not.  You could use a callback like above to indicate success but, this doesn't help if MovePiece threw an exception.  In that case the parent coroutine exits and we're stuck handling the exception at too high a level (say in a non-coroutine above the parent). Not ideal.  

Ideally we'd be able to catch any exception per coroutine (nested or not) allowing us to think of each coroutine as a logical unit as it should be.  A great solution for this [exists](http://twistedoakstudios.com/blog/Post83_coroutines-more-than-you-want-to-know). Part of this solution is also a better way to handle return values without having to use a callback.  This uses a parametrized wrapper to Coroutine and allows us to handle exceptions and return values like so:
``` c#
Coroutine<bool> dostuff = StartCoroutine<bool>(DoStuff()); // declare a return value type
yield return dostuff.coroutine;
try
{ 
	if (dostuff.Value)  //attempt to access the return value
		NextStuff();
}
catch
{
	//and handle any exceptions here
}
``` 
 
That is much better!  Now the child exits without interrupting the parent and the parent can deal with it appropriately (exceptions and return values).  Now, what's this about locking?


###Locking###

The other problem is locking.  Coroutines (although technically not multi-threaded in Unity) do cause concurrency problems!  What if, based on user input, you start a Coroutine multiple times and it operates on the same objects?  A mess can ensue.  One solution in Unity is that you can start a coroutine using a string name of the IEnumerator method.  This uses reflection and is yucky.  Who want to use strings when you're using a nice strongly typed language like C#?  But, if you use strings you can do ```StopCoroutine("DoStuff"); StartCoroutine("DoStuff")```.  Another option is to just have some instance variable bool and appropriate logic for every coroutine.  This too gets messy quickly with lots of coroutines.

These techniques get you part of the way but what if you want a coroutine to actually just block/wait for a previous instance to finish running like traditional locking but without any undue messiness?  Well using ```lock(someobject)``` doesn't work, because it would only be evaluated once and isn't checked on the coroutine resume.  Some people have written large coroutine managers that help with all these problems but those seemed like overkill so I built on the exception handling stuff and added locking support. The idea is that you start the coroutine with a string identifier (if none is given no locking is performed) and an optional timeout period.  With the string identifier each MonoBehavior keeps the currently running coroutines in a queue and either yields (analogous to blocking on a lock) until it's turn comes up or bails on a timeout.  Like so:
```
StartCoroutine(DoStuff(), "DoStuff", 10f); // will yield null
                                           // if any previous "DoStuff"s are running up to 10seconds
```

The easiest way to use this is to extend MonoBehaviour and use the new class as the base class for all your MonoBehaviours.  Here is mine:

{% include_code My MonoBehaviour Base Class lang:csharp  csharp/TTMonoBehaviour.cs %}

What do you think?


