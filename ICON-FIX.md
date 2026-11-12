# 任务栏图标修复（录制时显示 Electron 默认图标）

## 现象
- 桌面快捷方式图标正常（安装程序用 `resources\recordly.ico` 创建）
- 但**录制时任务栏出现的是 Electron 默认图标**

## 原因
打包时跳过了 `signAndEditExecutable`（本机 rcedit 处理不了这个 215 MB 的 exe），
所以 `Recordly.exe` 内部没有写入 Recordly 图标。Windows 的任务栏按钮取的是 **exe 内嵌图标**，
而桌面快捷方式取的是 `.lnk` 里指定的图标，所以两者表现不一致。

## 修复
1. 从 Release 下载 `Recordly.exe`（已注入图标，215 MB）；
2. 关闭 Recordly；
3. 用它覆盖安装目录里的同名文件：
   `C:\Users\<你的用户名>\AppData\Local\Programs\Recordly\Recordly.exe`
4. 重新启动即可；录制时任务栏也是 Recordly 图标了。

> 已验证：替换后应用正常启动（asar 完整性校验不受影响），exe 内含 1 个图标组 / 7 个尺寸。
