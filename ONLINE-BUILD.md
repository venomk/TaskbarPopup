# 在线编译（GitHub Actions → EXE）

这个仓库自带 GitHub Actions 工作流 `.github/workflows/build.yml`，不需要本地装任何编译环境，
在 GitHub 网页上就能编出可以直接运行的 `TaskbarPopup.exe`。

## 怎么编

1. 把这个目录推到 GitHub 仓库（默认分支建议叫 `main` 或 `master`，两者都会触发）。
   推上去以后 **Actions** 标签页里会自动跑一次 **Build**。
2. 想手动触发：**Actions → 左侧 Build → 右侧 Run workflow → 选分支 → Run workflow**
   （工作流里已经配了 `workflow_dispatch`）。
3. 等两个 job 跑完（x64 大约 2–4 分钟，arm64 慢一些），点进这次运行，页面最下方
   **Artifacts** 里下载：
   * `TaskbarPopup-zip` —— **要的就是这个**，里面是 `TaskbarPopup/TaskbarPopup.exe` + README + LICENSE；
   * `TaskbarPopup` —— 只含解压后的那几个文件；
   * `TaskbarPopup-arm64-zip` —— ARM64 版（Windows on ARM 用）。

## 怎么发版本

打一个 `v` 开头的标签（例如 `v1.2.4`）并推送：

```
git tag v1.2.4
git push origin v1.2.4
```

工作流会用标签里的版本号编译（`TP_VERSION`），自动创建/补全 Release 并上传两个 zip。
代码签名（SignPath）只在仓库里配了 `SIGNPATH_API_TOKEN` 密钥和 `SIGNPATH_ORGANIZATION_ID` 变量时才启用，
没配就跳过，不影响出包。

## 编译方式说明

* **x64**：在 `ubuntu-24.04` 上用 MinGW-w64 交叉编译（`sh build-mingw.sh`），
  静态链接，出来的 exe 不依赖运行库；同时把 `src/app.rc`（清单、图标、版本号）用 windres 编进去。
* **arm64**：在 `windows-latest` 上用 CMake + MSVC 编（`cmake -B build -A ARM64`）。
  这一步失败不影响 x64 出包，release 会只用 x64 版继续。

## 本地编译（可选）

* MinGW / Linux 交叉：`sh build-mingw.sh`，产物 `TaskbarPopup.exe`；
* MSVC：`cmake -B build && cmake --build build --config Release`。

## 这次相对上游多出来的东西

1. 迷你任务栏里多了一个**设置按钮（齿轮）**：左键点它收起迷你任务栏并打开设置窗口；
   F2 编辑模式里可以拖位置、改大小、拖出去隐藏、拖到另一条边上；
   设置窗口「状态和控制 → 显示」里可以开关它；界面文字中英双语。
2. **系统托盘里的常驻图标也一起列出来**：Windows 10 从任务栏托盘区的工具栏读，Windows 11 读注册表。
   默认把「一直显示在任务栏托盘区」的图标和「隐藏的图标」一起排进迷你任务栏；
   点这些图标和点系统托盘一样（左键打开、右键菜单）；
   任务栏藏着时会先临时显示一下再点，点完再藏回去；
   设置窗口「状态和控制 → 显示 → 显示托盘里隐藏的图标」展开后可以关掉常驻图标那一项
   （ini 里是 `ShowAllTrayIcons=1`，关掉只剩隐藏的图标）。
3. **右键托盘图标直接弹出和系统托盘一样的菜单**：在迷你任务栏里右键一个托盘图标，
   不再弹迷你任务栏自己那份菜单，而是直接到系统托盘里右键，弹出的菜单和系统托盘里右键
   这个图标时一模一样（打开 / 打开文件所在的位置 / 结束程序等，由系统决定）；
   原来的「程序菜单」项因为不再需要，已经整条删掉（无死代码）。

相对上游的完整改动补丁见根目录的 `taskbar-popup-changes.patch`（6 个文件，+235/−72；在仓库根目录
`git apply -p1 taskbar-popup-changes.patch` 或 `patch -p1` 即可应用，已验证能精确应用、也能精确反向还原）。
