---
layout: post
title: "在 Unity 编辑器标题栏显示 Git 分支名"
categories: unity
---

{: .note}
> 开发中经常会需要同时开多个项目，虽然可以通过【Preferences/General/Use Project Path in Window Title】选项让Unity在标题栏展示路径来区分当前是哪个项目，但是不同项目可能在不同分支，如果能直接看到当前项目所在的分支就更好了。
>
> 调查后发现了相关资料：https://zhuanlan.zhihu.com/p/462453808
>
> 基于此文章的思路，提供一个在Unity标题栏追加项目分支名的脚本，仅支持Windows系统。
>
> 不过这个问题只是在Unity里解决了，在使用IDE打开脚本的时候，依旧无法通过标题栏得知这些信息。

# 效果

应用前：

```
MyProject - SampleScene - PC, Mac & Linux Standalone - Unity 2022
```

应用后：

```
MyProject - SampleScene - PC, Mac & Linux Standalone - Unity 2022 [Master]
```

标题末尾会追加当前 git 分支名，方便在多个 Unity 编辑器窗口之间切换时快速辨别分支。

# 代码

将代码保存为脚本 `UpdateUnityEditorProcess` 放到项目的任意 `Editor` 文件夹下即可。脚本会在编辑器启动后自动运行。

仅限 Windows 编辑器可用。


```csharp
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Text;
using UnityEditor;

#if UNITY_EDITOR_WIN

// 通过 Win32 API 修改 Unity 编辑器窗口标题，在标题末尾追加当前 git 分支名
public class UpdateUnityEditorProcess
{
    // ── Win32 P/Invoke ──────────────────────────────────────────────────────

    public delegate bool EnumThreadWindowsCallback(IntPtr hWnd, IntPtr lParam);

    [DllImport("user32.dll", CharSet = CharSet.Auto, SetLastError = true)]
    public static extern bool EnumWindows(EnumThreadWindowsCallback callback, IntPtr extraData);
    [DllImport("user32.dll", CharSet = CharSet.Auto, SetLastError = true)]
    public static extern int GetWindowThreadProcessId(HandleRef handle, out int processId);
    [DllImport("user32.dll", CharSet = CharSet.Auto, ExactSpelling = true)]
    public static extern IntPtr GetWindow(HandleRef hWnd, int uCmd);
    [DllImport("user32.dll", CharSet = CharSet.Auto)]
    public static extern bool IsWindowVisible(HandleRef hWnd);
    [DllImport("user32.dll", CharSet = CharSet.Auto)]
    public static extern IntPtr GetParent(IntPtr hWnd);
    [DllImport("user32.dll")]
    private static extern bool GetWindowText(int hWnd, StringBuilder title, int maxBufSize);
    [DllImport("user32.dll", EntryPoint = "SetWindowText", CharSet = CharSet.Auto)]
    public extern static int SetWindowText(int hwnd, string lpString);

    // ── 字段 ────────────────────────────────────────────────────────────────

    public IntPtr hwnd = IntPtr.Zero;
    private bool haveMainWindow = false;
    private IntPtr mainWindowHandle = IntPtr.Zero;
    private int processId = 0;
    private static readonly StringBuilder sbtitle = new(255);
    private static string GitBranch = "";

    // ── 单例 ────────────────────────────────────────────────────────────────

    private static UpdateUnityEditorProcess _instance;
    public static UpdateUnityEditorProcess Instance
    {
        get
        {
            if (_instance == null)
            {
                _instance = new UpdateUnityEditorProcess();
                _instance.hwnd = _instance.GetMainWindowHandle(Process.GetCurrentProcess().Id);
            }
            return _instance;
        }
    }

    // ── Git 分支 ────────────────────────────────────────────────────────────

    // 重新获取当前 git 分支名并缓存，供 SetTitle 使用
    // 在域重载时（InitializeOnLoad）和资源导入后（AssetPostprocessor）各调用一次
    public static void RefreshGitBranch()
    {
        GitBranch = GetCurrentGitBranch();
    }

    private static string GetCurrentGitBranch()
    {
        try
        {
            var process = new Process();
            process.StartInfo.FileName = "git";
            process.StartInfo.Arguments = "rev-parse --abbrev-ref HEAD";
            process.StartInfo.RedirectStandardOutput = true;
            process.StartInfo.UseShellExecute = false;
            process.StartInfo.CreateNoWindow = true;
            process.Start();
            string branch = process.StandardOutput.ReadToEnd().Trim();
            process.WaitForExit();
            return branch;
        }
        catch
        {
            return "";
        }
    }

    // ── 标题写入 ────────────────────────────────────────────────────────────

    // 读取当前窗口标题，若尚未包含分支标签则追加 " [分支名]"
    // 注意：调用方需在 Unity 完成自身标题更新后再调用（用 delayCall 延迟），
    //       否则 Unity 会在我们写入后再次覆盖标题，导致首次追加丢失
    public void SetTitle()
    {
        sbtitle.Length = 0;
        GetWindowText(hwnd.ToInt32(), sbtitle, 255);
        string strTitle = sbtitle.ToString();
        string branchTag = "[" + GitBranch + "]";
        // 已含标签时跳过，避免重复追加
        if (!string.IsNullOrEmpty(GitBranch) && !strTitle.Contains(branchTag))
        {
            SetWindowText(hwnd.ToInt32(), strTitle + " " + branchTag);
        }
    }

    // ── 主窗口句柄查找 ──────────────────────────────────────────────────────

    public IntPtr GetMainWindowHandle(int processId)
    {
        if (!this.haveMainWindow)
        {
            this.mainWindowHandle = IntPtr.Zero;
            this.processId = processId;
            EnumThreadWindowsCallback callback = new EnumThreadWindowsCallback(this.EnumWindowsCallback);
            EnumWindows(callback, IntPtr.Zero);
            GC.KeepAlive(callback);
            this.haveMainWindow = true;
        }
        return this.mainWindowHandle;
    }

    private bool EnumWindowsCallback(IntPtr handle, IntPtr extraParameter)
    {
        int num;
        GetWindowThreadProcessId(new HandleRef(this, handle), out num);
        if ((num == this.processId) && this.IsMainWindow(handle))
        {
            this.mainWindowHandle = handle;
        }
        return true;
    }

    // 主窗口判定：无父窗口、无 owner、且可见
    private bool IsMainWindow(IntPtr handle)
    {
        return (GetParent(handle) == IntPtr.Zero
            && !(GetWindow(new HandleRef(this, handle), 4) != IntPtr.Zero)
            && IsWindowVisible(new HandleRef(this, handle)));
    }
}

// 域重载时（编辑器启动、脚本编译完成）触发标题更新
[InitializeOnLoad]
class UpdateUnityEditorTitle
{
    private static bool isInGame = false;

    [System.Obsolete]
    static UpdateUnityEditorTitle()
    {
        // 域重载时立即获取分支名，确保后续 delayCall 执行时已有值
        UpdateUnityEditorProcess.RefreshGitBranch();

        // delayCall 在当前帧末尾执行，此时 Unity 已完成自身标题初始化
        EditorApplication.delayCall += DoUpdateTitleFunc;

        // hierarchyChanged 早于 Unity 写标题触发，需延迟到 delayCall 执行
        EditorApplication.hierarchyChanged += () => EditorApplication.delayCall += DoUpdateTitleFunc;

        EditorApplication.playmodeStateChanged += OnPlaymodeStateChanged;
    }

    static void OnPlaymodeStateChanged()
    {
        if (EditorApplication.isPlaying == isInGame) return;
        isInGame = EditorApplication.isPlaying;
        DoUpdateTitleFunc();
    }

    static void DoUpdateTitleFunc()
    {
        UpdateUnityEditorProcess.Instance.SetTitle();
    }
}

// 资源导入完成后刷新分支名并更新标题
// 切换 git 分支时 Unity 会检测到文件变化并触发重导入，借此实现分支名自动刷新
class UpdateUnityEditorTitlePostprocessor : AssetPostprocessor
{
    static void OnPostprocessAllAssets(string[] importedAssets, string[] deletedAssets,
        string[] movedAssets, string[] movedFromAssetPaths)
    {
        UpdateUnityEditorProcess.RefreshGitBranch();
        UpdateUnityEditorProcess.Instance.SetTitle();
    }
}

#endif
```