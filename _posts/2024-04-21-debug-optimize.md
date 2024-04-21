---
title: Debug Classを最適化してみましょう。
author: haku
date: 2024-04-21 23:29:00 +0900
categories: [Unity, Tips]
tags: [Unity, Tip, optimize]
---
---
## Debug.Logとは？
[**Debug.Log**](https://docs.unity3d.com/2023.2/Documentation/ScriptReference/Debug.Log.html)はUnityで開発中に、初心者でもベテランでも最も頻繁に使用する機能の一つだと思います。

実行中に特定の情報をConsoleに出力し、変数の値を確認したり、特定のイベントが発生したことを確認する際に主に使用されます。

したがって、開発時に便利な情報を出力したり、Debugging時に主に利用されます。

#### よく使うDebugの例
```c#
using UnityEngine;

public class DebugTest : MonoBehaviour
{
    private int _playerScore = 100;
    private float _playerHealth = 75.5f;
    private string _playerName = "Player1";
    private bool _isPlayerAlive = true;

    private void Start()
    {
        // 各変数を個別に出力
        Debug.Log("Player score: " + _playerScore);
        Debug.Log("Player health: " + _playerHealth);
        Debug.Log("Player name: " + _playerName);
        Debug.Log("Is player alive: " + _isPlayerAlive);

        // 変数を一行で出力
        Debug.Log($"Player: {_playerName}, Score: {_playerScore}, Health: {_playerHealth}, Alive: {_isPlayerAlive}");

        // フォーマット指定子を使用して変数を出力
        Debug.LogFormat("Player: {0}, Score: {1}, Health: {2}, Alive: {3}", _playerName, _playerScore, _playerHealth, _isPlayerAlive);

        // その他
        Debug.Log($"{nameof(_playerName)}: {_playerName}, {nameof(_playerScore)}: {_playerScore}, {nameof(_playerHealth)}: {_playerHealth}, {nameof(_isPlayerAlive)}: {_isPlayerAlive}");
        Debug.Log($"Player: {_playerName}, Score: {_playerScore}, Health: {_playerHealth}, Alive: {_isPlayerAlive}");
    }
}
```

![image-20240421174457665](../assets/img/post/2024-04-20-hello-world (copy)/image-20240421174457665.png){: width="500"}
_Script実行結果_

---

## BuildされたApplicationにDebug.Logは影響を与えるか？
![image-20240421174807506](../assets/img/post/2024-04-20-hello-world (copy)/image-20240421174807506.png){: width="500"}

Game Viewでは、`Debug.Log`がRenderingされないのに、BuildされたApplicationに影響があるかな？

テストしてみましょう。👨‍💻


```c#
using UnityEngine;
using UnityEngine.UI;

public class DebugTest : MonoBehaviour
{
    public Text logText;
    private void Start()
    {
        int i = 0;
        
        // Debug.Logで、iを増減させる。
        Debug.Log(++i);
        // Debug.Logで、増減させたiをGameViewに出力する。
        logText.text = i.ToString();
    }
}
```
![image-20240421180000396](../assets/img/post/2024-04-20-hello-world (copy)/image-20240421180000396.png){: width="500"}

![image-20240421180050329](../assets/img/post/2024-04-20-hello-world (copy)/image-20240421180050329.png){: width="500"}
_Buid後、実行結果_

はい🙋🏻、GameViewに`1`が出力されました。

これは`Debug.Log(++i);`が、BuildされたApplicationでも動作しているということです。

ため、開発に便利たけど多量に乱発される場合、性能にも影響が発生します。

これを解決するため、本ポストではDebug Classをラッピングし、BuildされたApplicationでは`Debug.Log`が動作しないようにし、影響を与えない方法を説明します。

---

## 条件付きコンパイル
```c#
#if UNITY_STANDALONE
// Logic
#endif

#if UNITY_ANDROID
// Logic
#endif
```
Debug Classをラッピングする前に[**条件付きコンパイル**](https://docs.unity3d.com/2022.3/Documentation/Manual/PlatformDependentCompilation.html)について調べてみましょう。

![image-20240421182637683](../assets/img/post/2024-04-20-hello-world (copy)/image-20240421182637683.png){: width="500"}

IDEで確認してみると、現在のBuild Targetが`Window`, `Mac`, `Linux`であるため、`UNITY_ANDROID`, `UNITY_IOS`の場合、Compile自体がされていないことが確認できます。

これを利用すると簡単に`Unity Editor`のみ、`Debug.Log`が表示されるようにラッピングすること思います。

---

## System.Diagnostics.Conditional Attribute
```c#
// 条件付きコンパイル
#if UNITY_EDITOR
Debug.Log("Unity Editor");
#endif

// System.Diagnostics.Conditional
[Conditional("UNITY_EDITOR")]
private void UnityEditor() => Debug.Log("Unity Editor");
```

役割は、`条件付きコンパイル`と同じですが、`Attribute`の形を持っているので、関数の上に書いて関数自体を`条件付きコンパイル`することができます。

Debug Classをラッピングする場合、関連関数が多いため、よりきれいにScriptを管理するためConditionalを使おうと思います。

---

## Debug Class ラッピング

![image-20240421183626785](../assets/img/post/2024-04-20-hello-world (copy)/image-20240421183626785.png)

Debug機能を使用する場合、より便利に使用するため、新しいScriptの名前を`Debug.cs`に作成します。

```c#
using System.Diagnostics;

public static class Debug
{
    [Conditional("UNITY_EDITOR")]
    public static void Log(object message) => UnityEngine.Debug.Log(message);
}
```

上で調べた`Conditional`を利用して`UnityEngine.Debug.Log`をラッピングしました。

ラッピングした`Debug.Log`を利用してもう一度テストしてみましょう。

```c#
using UnityEngine;
using UnityEngine.UI;

public class DebugTest : MonoBehaviour
{
    public Text logText;
    private void Start()
    {
        int i = 0;
        
        // ラッピングした、Debug.Logで、iを増減させる。
        Debug.Log(++i);
        // ラッピングした、Debug.Logで、増減させたiをGameViewに出力する。
        logText.text = i.ToString();
    }
}
```
> Debugを利用時、`in UnityEngine`ではなく、ラッピングした`Debug Class`を利用。
> ![image-20240421184201546](../assets/img/post/2024-04-20-hello-world (copy)/image-20240421184201546.png)
{: .prompt-warning }

![image-20240421184810586](../assets/img/post/2024-04-20-hello-world (copy)/image-20240421184810586.png)
_(左)Unity Editer実行、(右)Build後実行_

テスト結果を確認してみると、Unity Editorでは`1`、BuildされたApplicationでは`0`が表示され、これは希望通りUnity Editorでは、Debug.Logが動作したが、BuildされたApplicationではDebug.Logが動作しなかったことを確認しました。


### ラッピングされた、Debug Class Script
上で調べたことをもとに、開発中その時その時必要な機能を追加しながら、自分だけのDebug Classを作ってみてください。 

```c#
using UnityEngine;
using System.Diagnostics;
using Object = UnityEngine.Object;

/// <summary>
/// カラーリスト
/// </summary>
public enum DColor
{
    white,
    grey,
    black,
    red,
    green,
    blue,
    yellow,
    cyan,
    brown,
    
}

/// <summary>
/// Debugラッピング
/// Unityでのみ表示する。
/// </summary>
public static class Debug
{
    /******************************************
    *                roperties                *
    ******************************************/
    #region
    public static ILogger logger => UnityEngine.Debug.unityLogger;
    public static ILogger unityLogger => UnityEngine.Debug.unityLogger;
    public static bool developerConsoleVisible
    {
        get => UnityEngine.Debug.developerConsoleVisible;
        set => UnityEngine.Debug.developerConsoleVisible = value;
    }
    public static bool isDebugBuild => UnityEngine.Debug.isDebugBuild;
    #endregion
    
    /******************************************
    *                   Log                   *
    ******************************************/
    #region
    [Conditional("UNITY_EDITOR")]
    public static void Log(object message, UnityEngine.Object context) => UnityEngine.Debug.Log(message, context);
    [Conditional("UNITY_EDITOR")]
    public static void Log(object message) => UnityEngine.Debug.Log(message);
    #endregion

    /******************************************
    *                LogFormat                *
    ******************************************/
    #region
    [Conditional("UNITY_EDITOR")]
    public static void LogFormat(string format, params object[] args) => UnityEngine.Debug.LogFormat(format, args);
    [Conditional("UNITY_EDITOR")]
    public static void LogFormat(string text, DColor color) => UnityEngine.Debug.LogFormat("<color={0}>{1}</color>",color.ToString(), text);
    [Conditional("UNITY_EDITOR")]
    public static void LogFormat(Object context, string format, params object[] args) => UnityEngine.Debug.LogFormat(context, format, args);
    #endregion
    
    /******************************************
    *                 LogError                *
    ******************************************/
    #region
    [Conditional("UNITY_EDITOR")]
    public static void LogError(object message) => UnityEngine.Debug.LogError(message);
    [Conditional("UNITY_EDITOR")]
    public static void LogErrorFormat(string format, params object[] args) => UnityEngine.Debug.LogErrorFormat(format, args);
    #endregion

    /******************************************
    *               LogWarning                *
    ******************************************/
    #region
    [Conditional("UNITY_EDITOR")]
    public static void LogWarning(object message) => UnityEngine.Debug.LogWarning(message);
    [Conditional("UNITY_EDITOR")]
    public static void LogWarning(object message, UnityEngine.Object context) => UnityEngine.Debug.LogWarning(message, context);
    [Conditional("UNITY_EDITOR")]
    public static void LogWarningFormat(string format, params object[] args) => UnityEngine.Debug.LogWarningFormat(format, args);
    #endregion

    /******************************************
    *              LogAssertion               *
    ******************************************/
    #region
    [Conditional("UNITY_EDITOR")]
    public static void LogAssertion(object message, Object context) => UnityEngine.Debug.LogAssertion(message, context);
    [Conditional("UNITY_EDITOR")]
    public static void LogAssertion(object message) => UnityEngine.Debug.LogAssertion(message);
    [Conditional("UNITY_EDITOR")]
    public static void LogAssertionFormat(Object context, string format, params object[] args) => UnityEngine.Debug.LogAssertionFormat(context, format, args);
    [Conditional("UNITY_EDITOR")]
    public static void LogAssertionFormat(string format, params object[] args) => UnityEngine.Debug.LogAssertionFormat(format, args);
    #endregion
    
    /******************************************
    *                  Assert                 *
    ******************************************/
    #region
    [Conditional("UNITY_EDITOR")]
    public static void Assert(bool condition, string message, Object context) => UnityEngine.Debug.Assert(condition, message, context);
    [Conditional("UNITY_EDITOR")]
    public static void Assert(bool condition) => UnityEngine.Debug.Assert(condition);
    [Conditional("UNITY_EDITOR")]
    public static void Assert(bool condition, object message, Object context) => UnityEngine.Debug.Assert(condition, message, context);
    [Conditional("UNITY_EDITOR")]
    public static void Assert(bool condition, string message) => UnityEngine.Debug.Assert(condition, message);
    [Conditional("UNITY_EDITOR")]
    public static void Assert(bool condition, object message) => UnityEngine.Debug.Assert(condition, message);
    [Conditional("UNITY_EDITOR")]
    public static void Assert(bool condition, Object context) => UnityEngine.Debug.Assert(condition, context);
    [Conditional("UNITY_EDITOR")]
    public static void AssertFormat(bool condition, Object context, string format, params object[] args) => UnityEngine.Debug.AssertFormat(condition, context, format, args);
    [Conditional("UNITY_EDITOR")]
    public static void AssertFormat(bool condition, string format, params object[] args) => UnityEngine.Debug.AssertFormat(condition, format, args);
    #endregion
    
    /******************************************
    *               WriteLine                 *
    ******************************************/
    #region
    [Conditional("UNITY_EDITOR")]
    public static void WriteLine(string? message, string? category) => System.Diagnostics.Debug.WriteLine(message, category);
    [Conditional("UNITY_EDITOR")]
    public static void WriteLine(string format, params object?[] args) => System.Diagnostics.Debug.WriteLine(format, args);
    [Conditional("UNITY_EDITOR")]
    public static void WriteLine(string? message) => System.Diagnostics.Debug.WriteLine(message);
    [Conditional("UNITY_EDITOR")]
    public static void WriteLine(object? value) => System.Diagnostics.Debug.WriteLine(value);
    [Conditional("UNITY_EDITOR")]
    public static void WriteLine(object? value, string? category) => System.Diagnostics.Debug.WriteLine(value, category);
    #endregion
}
```