# Patches carried by this fork

Base: `OutSystems/WebView` `master` at `f582423` (3.120.11, built against Avalonia 11.0.10 and
CefGlue 120.6099.207). Upstream has since moved only its CefGlue dependency (3.120.12 → 120.6099.214,
3.120.13 → 120.6099.215); there is no source change to take.

This fork exists because the Avalonia 12 port of CefGlue (`github.com/Quantower/CefGlue`, which carries
its own `PATCHES.md`) is useless without a WebView built against Avalonia 12. Everything below is on top
of the base.

One commit per patch — moving to a newer upstream should be a rebase with cherry-picks, not a merge.

| Patch | Files | Why |
|---|---|---|
| Build against Avalonia 12.1.0 | `WebViewControl.Avalonia/WebView.Avalonia.cs`, `Directory.Build.props` | Avalonia 12 renamed `GotFocusEventArgs` to `FocusChangedEventArgs`. The binding indexer in `ExtraInitialize` needed no source change, but its signature moved from `IBinding` to `BindingBase`, so the 11-built binary threw `MissingMethodException` at runtime — a rebuild is mandatory. Grepping the binary for `.Bind(` does not find indexer bindings; only a rebuild proves compatibility. |
| Resolve `CefGlue.*` from the Quantower feed | `Nuget.config` | The CefGlue fork keeps the upstream package ids and publishes its own `120.7000.x` range to `nuget.deployment.me`. Source mapping makes `CefGlue.*` come only from there. |
| Bind the browser's `AllowDrop` to the `WebView` | `WebViewControl.Avalonia/WebView.Avalonia.cs` | The CefGlue fork made the off-screen frame follow its control's `AllowDrop` instead of a hardcoded `true`. `AllowDrop` inherits along the logical tree and the browser was a visual-only child, so the host's value stopped at the `WebView`. The binding mirrors the `FocusableProperty` one next to it. |
| Make the browser a logical child | `WebViewControl.Avalonia/WebView.Avalonia.cs` | `ExtraInitialize` added the browser to `VisualChildren` only, so it had no logical parent. Its tooltip's styling chain (`ToolTip` → `Popup` → browser) ended there and never reached `Application.Styles`: browser tooltips fell back to Avalonia's default template, black text on `PopupRoot`'s white background, while every other control got the app's tooltip style. |
| Bind the browser's tooltip delay to the `WebView` | `WebViewControl.Avalonia/WebView.Avalonia.cs` | The CefGlue fork opens the browser tooltip after `ToolTip.ShowDelay` read from the browser control, which the host cannot reach. The binding lets the host set the delay on the `WebView`, as on any other control. |
| Dispose a JS executor only when its own frame is released | `WebViewControl/WebView.cs`, `WebViewControl/WebView.JavascriptExecutor.cs`, `tests/Tests.WebView/JavascriptEvaluation.cs` | Executors are keyed by frame name, and the main frame's name is empty. CEF also releases transitional frames under a name a live frame holds: the temporary sub-frame a cross-site iframe starts in (an unnamed iframe reports it with an empty name), and the speculative main frame of a cross-site navigation that never commits, such as a 204. Either release disposed the live page's executor. Its replacement starts flushing only when that frame's JS context is created again, so until the page reloaded every `ExecuteScript` sat in the queue and every `EvaluateScript` without a timeout never completed — with no error anywhere. In the terminal, the FinancialJuice widget's unnamed iframe froze the panel on a blocking `EvaluateScript(...).Result`. A release now disposes the executor only if it is bound to that very frame (`CefFrame.Identifier`); each release carries the identifier of the frame that actually went away. |

Commits named "Require the CefGlue fork with …" or "Bump the version …" change only `CefGlueVersion`
and `Version` in `Directory.Build.props`; they are releases, not patches.

## Invariants

- Package id stays `WebViewControl-Avalonia`, the upstream one. Version range `3.121.x` is ours.
- `CefGlueVersion` becomes the nuspec dependency `CefGlue.Avalonia >= …`, a **lower bound**. Publishing a
  new `CefGlue.Avalonia` alone changes nothing downstream: NuGet keeps the older build that still
  satisfies the bound. Raising `CefGlueVersion` and publishing a new `WebViewControl-Avalonia` is what
  moves consumers.
- `WebViewControl.Avalonia.dll` references `Xilium.CefGlue.Avalonia` by assembly name and nothing is
  strong-named — see the CefGlue fork's invariants before renaming anything there.
- To build against an unpublished CefGlue, point the `CefGlueFork` source in `Nuget.config` at the local
  `cefglue/Nuget/output` folder. Never commit that change.

## Building and publishing

```
dotnet build WebViewControl.Avalonia/WebViewControl.Avalonia.csproj -c Release
```

`GeneratePackageOnBuild` packs `nuget/WebViewControl-Avalonia.<Version>.nupkg` (git-ignored). The
nuspec `repository commit` is `HEAD` at build time, so commit the version bump first. Then:

```
dotnet nuget push nuget/WebViewControl-Avalonia.<Version>.nupkg -s https://nuget.deployment.me -k quantower
git push origin quantower/av12
```

Push the branch right after the package, so the commit the nuspec names exists on GitHub. A published
version is immutable — every fix ships as a new version.

## Tests

```
dotnet build tests/TestResourceAssembly.V1.0.0.0/TestResourceAssembly.csproj -c Release
dotnet build tests/TestResourceAssembly.V2.0.0.0/TestResourceAssembly.csproj -c Release
DOTNET_ROLL_FORWARD=Major dotnet test tests/Tests.WebView/Tests.WebView.csproj -c Release -p:Platform=x64 -p:GeneratePackageOnBuild=false
```

- `-p:GeneratePackageOnBuild=false` is not optional: the test project references
  `WebViewControl.Avalonia`, and without it the build repacks the current version's nupkg from the
  working tree.
- `DynamicallyLoadedAssemblyFile` loads the two `TestResourceAssembly` builds from their `bin` folders.
- The tests target `net8.0`; `DOTNET_ROLL_FORWARD=Major` runs them on a newer runtime.
- They open real windows and run Chromium windowed, not off-screen. External sites are faked through
  `BeforeResourceLoad`, which sees `http(s)` requests too, so no test needs the network.
