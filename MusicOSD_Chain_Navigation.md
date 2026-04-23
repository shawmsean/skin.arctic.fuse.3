# MusicOSD 链式导航实现文档

## 概述

为 Kodi 皮肤 `skin.arctic.fuse.3` 的音乐OSD实现了链式导航功能，模仿VideoOSD的按"下键"循环切换不同功能面板的体验。

**导航链**: MusicOSD → 歌曲评论(1142) → 播放列表(1140) → 歌手信息(1141)

同时为 `plugin.audio.music` 插件添加了评论SQLite缓存和歌手ID暴露功能。

---

## 一、MusicOSD 链式导航

### 1.1 导航变量定义

**文件**: `1080i/Includes_Actions.xml`

```xml
<!-- 音乐OSD链式导航变量 -->
<variable name="Action_MusicOSD_Main_OnDown">
    <value condition="!Skin.HasSetting(MusicOSD.OnDown.Disable1142)">1142</value>
    <value>$VAR[Action_MusicOSD_1142_OnDown]</value>
</variable>
<variable name="Action_MusicOSD_1142_OnDown">
    <value condition="!Skin.HasSetting(MusicOSD.OnDown.Disable1140)">1140</value>
    <value>$VAR[Action_MusicOSD_1140_OnDown]</value>
</variable>
<variable name="Action_MusicOSD_1140_OnDown">
    <value condition="!Skin.HasSetting(MusicOSD.OnDown.Disable1141)">1141</value>
</variable>
```

**设计要点**:
- 每个变量先检查对应的 `Skin.HasSetting` 开关，允许用户禁用某一层
- 如果当前层被禁用，自动跳到下一层的变量，实现"跳过"效果
- 变量优先级由上到下，第一个匹配的 `value` 生效

### 1.2 MusicOSD 入口

**文件**: `1080i/MusicOSD.xml`

将硬编码的 `ActivateWindow(1142)` 改为变量驱动：

```xml
<ondown condition="!Window.IsVisible(script-cu-lrclyrics-main.xml)">
    SetProperty(bili_comment_content_url,plugin://plugin.audio.music/current_song_comments/0,10000)
</ondown>
<ondown condition="!Window.IsVisible(script-cu-lrclyrics-main.xml)">
    ActivateWindow($VAR[Action_MusicOSD_Main_OnDown])
</ondown>
```

**关键**: 在激活评论窗口前，先通过 `SetProperty` 设置评论内容URL到Window(10000)，供1142的 `<content>` 动态读取。

### 1.3 窗口切换模式 — Close + AlarmClock

所有窗口切换统一使用 `Close` + `AlarmClock(00:00, ActivateWindow(...), silent)` 模式：

```xml
<!-- 从1142(评论)向下到1140(播放列表) -->
<ondown condition="!Skin.HasSetting(MusicOSD.OnDown.Disable1140)">Close</ondown>
<ondown condition="!Skin.HasSetting(MusicOSD.OnDown.Disable1140)">
    AlarmClock(musicosd_nav,ActivateWindow($VAR[Action_MusicOSD_1142_OnDown]),00:00,silent)
</ondown>

<!-- 从1140(播放列表)向上回到1142(评论) -->
<onup condition="!Skin.HasSetting(MusicOSD.OnDown.Disable1142)">
    SetProperty(bili_comment_content_url,plugin://plugin.audio.music/current_song_comments/0,10000)
</onup>
<onup condition="!Skin.HasSetting(MusicOSD.OnDown.Disable1142)">Close</onup>
<onup condition="!Skin.HasSetting(MusicOSD.OnDown.Disable1142)">
    AlarmClock(musicosd_nav,ActivateWindow(1142),00:00,silent)
</onup>
```

**为什么不用 `DialogClose` + `ActivateWindow`**:
- `DialogClose` 是**异步**的，关闭动画未完成时 `ActivateWindow` 就执行了
- 导致新窗口在旧窗口关闭动画期间打开，产生焦点问题
- `Close` 是**同步**的，立即关闭当前dialog
- `AlarmClock(00:00)` 在下一个事件循环执行，确保时序正确

---

## 二、播放列表(1140)音乐模式

### 2.1 方形样式 + 音乐内容源

**文件**: `1080i/Custom_1140_OSD_Playlist.xml`

将原来的单一 `List_Landscape_Row` 拆分为两个条件分支：

```xml
<!-- 音乐模式：方形样式 + 音乐播放列表 -->
<include content="List_Square_Row" condition="Window.IsActive(musicosd)">
    <param name="orientation">horizontal</param>
    <param name="control">fixedlist</param>
    <param name="id">7000</param>
    <include content="Object_ContentDynamic">
        <param name="content">playlistmusic://</param>
    </include>
    <!-- 音乐模式导航 -->
    <onup condition="!Skin.HasSetting(MusicOSD.OnDown.Disable1142)">
        SetProperty(bili_comment_content_url,plugin://plugin.audio.music/current_song_comments/0,10000)
    </onup>
    <onup condition="!Skin.HasSetting(MusicOSD.OnDown.Disable1142)">Close</onup>
    <onup condition="!Skin.HasSetting(MusicOSD.OnDown.Disable1142)">
        AlarmClock(musicosd_nav,ActivateWindow(1142),00:00,silent)
    </onup>
    <onup>Close</onup>
    <ondown>Close</ondown>
    <ondown>AlarmClock(musicosd_nav,ActivateWindow($VAR[Action_MusicOSD_1140_OnDown]),00:00,silent)</ondown>
</include>

<!-- 视频模式：横向样式 + 视频内容 -->
<include content="List_Landscape_Row" condition="!Window.IsActive(musicosd)">
    <!-- 原有视频逻辑 -->
</include>
```

**为什么需要拆分**:
- `$VAR[Path_OSD_Episodes]` 在音乐播放时返回空（所有条件基于VideoPlayer属性）
- 空内容导致列表控件无法获得焦点：`Control 7000 in window 11140 has been asked to focus, but it can't`
- 音乐模式需要 `playlistmusic://` 作为内容源
- 音乐模式需要方形样式（`List_Square_Row`），视频模式保持横向样式

---

## 三、歌手信息(1141)音乐模式

### 3.1 内容源替换

**文件**: `1080i/Custom_1141_OSD_Cast.xml`

将TMDbHelper搜索替换为插件自身的歌手详情路由：

```xml
<!-- 旧：TMDbHelper搜索（第三方，依赖歌手名匹配） -->
<content condition="Window.IsActive(musicosd)">
    plugin://plugin.video.themoviedb.helper/?info=...&tmdb_type=artist&query=歌手名
</content>

<!-- 新：插件直接路由（精确，使用歌手ID） -->
<content condition="Window.IsActive(musicosd)">
    plugin://plugin.audio.music/artist/$INFO[Window(10000).Property(nc_current_artist_id)]/
</content>
```

### 3.2 插件端暴露歌手ID

**文件**: `plugin.audio.music/addon.py` — `play()` 函数

在记录播放历史后，将歌手ID设置到Window Property：

```python
# 将歌手ID设置到Window Property，供皮肤端OSD歌手信息使用
if artist_id:
    xbmcgui.Window(10000).setProperty('nc_current_artist_id', str(artist_id))
    xbmcgui.Window(10000).setProperty('nc_current_artist_name', artist_name)
```

### 3.3 点击导航 — 从Dialog跳转到全屏窗口

**核心难点**: 1141是dialog窗口，点击歌手子菜单项需要跳转到全屏音乐窗口(10502)，但Kodi禁止在modal dialog存在时激活其他窗口。

**最终方案**:

```xml
<onclick condition="Window.IsActive(musicosd)">
    SetProperty(musicosd_nav_path,$INFO[Container(6501).ListItem.FileNameAndPath],Home)
</onclick>
<onclick condition="Window.IsActive(musicosd)">Close</onclick>
<onclick condition="Window.IsActive(musicosd)">
    AlarmClock(musicosd_back1,Action(Back),00:01,silent)
</onclick>
<onclick condition="Window.IsActive(musicosd)">
    AlarmClock(musicosd_back2,Action(Back),00:02,silent)
</onclick>
<onclick condition="Window.IsActive(musicosd)">
    AlarmClock(musicosd_nav,ActivateWindow(10502,$INFO[Window(Home).Property(musicosd_nav_path)]),00:03,silent)
</onclick>
<onclick condition="!Window.IsActive(musicosd)">Action(Info)</onclick>
```

**时间线**:
| 时间 | 动作 | 说明 |
|------|------|------|
| 0s | `SetProperty` | 保存选中项路径到Window Property |
| 0s | `Close` | 关闭1141 dialog |
| 1s | `Action(Back)` | 1141关闭动画完成，第一次Back退出MusicOSD子层 |
| 2s | `Action(Back)` | 第二次Back退出MusicOSD |
| 3s | `ActivateWindow(10502,path)` | 所有modal dialog已关闭，激活音乐窗口并导航 |

---

## 四、评论SQLite缓存

### 4.1 实现

**文件**: `plugin.audio.music/addon.py` — `song_comments()` 函数

```python
# 尝试从SQLite缓存读取评论API数据
cache_db = get_cache_db()
cache_key = cache_db.generate_cache_key('song_comments', song_id, offset, limit)
resp = cache_db.get(cache_key)

if resp is not None:
    xbmc.log(f'[Music Comments] Cache HIT: song_id={song_id}, offset={offset}, limit={limit}', xbmc.LOGINFO)
else:
    xbmc.log(f'[Music Comments] Cache MISS: song_id={song_id}, offset={offset}, limit={limit}', xbmc.LOGINFO)
    resp = music.song_comments(music_id=song_id, offset=offset, limit=limit)
    if resp:
        cache_db.set(cache_key, resp, cache_type='song_comments', expire_seconds=6*60*60)
        xbmc.log(f'[Music Comments] Cache written: song_id={song_id}, offset={offset}, TTL=6h', xbmc.LOGINFO)
```

**缓存策略**:
- 缓存键: `MD5(song_comments_{song_id}_{offset}_{limit})`
- TTL: 6小时
- 缓存内容: API返回的原始 `resp` 数据
- 每个offset独立缓存，增量加载同样受益

---

## 五、踩坑记录

### 5.1 DialogClose是异步的

**问题**: `DialogClose(1142)` + `ActivateWindow(1140)` 导致1140在1142关闭动画期间打开。

**解决**: 改用 `Close`（同步）+ `AlarmClock(00:00, ActivateWindow(...), silent)`。

### 5.2 共享窗口需要模式特定内容源

**问题**: 1140和1141在视频/音乐模式下共享，但 `$VAR[Path_OSD_Episodes]` 和 `$VAR[Path_OSD_Cast]` 在音乐播放时返回空，导致列表控件无法获得焦点。

**解决**: 用 `Window.IsActive(musicosd)` 条件分支，音乐模式使用 `playlistmusic://` 和 `plugin://plugin.audio.music/artist/<id>/`。

### 5.3 Action(Info)对plugin内容源无效

**问题**: 1141的 `<onclick>Action(Info)</onclick>` 对TMDbHelper内容有效，但对 `plugin.audio.music` 的plugin内容无效，点击无反应。

**解决**: 音乐模式使用 `SetProperty` + `Close` + `AlarmClock` 链式操作跳转到全屏窗口。

### 5.4 Close后$INFO丢失

**问题**: `Close` 关闭dialog后，`Container(6501)` 不再存在，`$INFO[Container(6501).ListItem.FileNameAndPath]` 返回空。

**解决**: 在 `Close` 之前用 `SetProperty(musicosd_nav_path, ...)` 保存路径到Window Property，后续用 `Window(Home).Property(musicosd_nav_path)` 读取。

### 5.5 ActivateWindow被modal dialog拒绝

**问题**: `Activate of window '10502' refused because there are active modal dialogs`。MusicOSD是 `<window>` 类型（非dialog），`DialogClose` 对它无效。

**解决**: 用 `Action(Back)` 关闭MusicOSD。MusicOSD需要两次Back才能完全退出，所以用两个 `AlarmClock` 分别在1秒和2秒后发送Back。

### 5.6 Action(Back)在关闭动画期间被忽略

**问题**: `ignoring action 92, because topmost modal dialog closing animation is running`。

**解决**: 给足够延迟（1秒），让1141的关闭动画完成后再发送 `Action(Back)`。

### 5.7 Skin.SetString是持久化的

**问题**: 用 `Skin.SetString` 做防重复锁，重启Kodi后锁仍然存在，功能永久失效。

**解决**: 改用 `SetProperty(key,value,Home)` + `ClearProperty(key,Home)`，窗口属性是临时的。

---

## 六、涉及文件清单

| 文件 | 修改内容 |
|------|---------|
| `skin/1080i/Includes_Actions.xml` | 添加 `Action_MusicOSD_*_OnDown` 导航变量 |
| `skin/1080i/MusicOSD.xml` | ondown改为变量驱动 |
| `skin/1080i/Custom_1142_OSD_MusicComments.xml` | 添加ondown链式导航 |
| `skin/1080i/Custom_1140_OSD_Playlist.xml` | 拆分为音乐/视频两个条件分支，方形样式，音乐内容源 |
| `skin/1080i/Custom_1141_OSD_Cast.xml` | 音乐模式内容源改为插件路由，onclick链式跳转 |
| `skin/1080i/Includes_Labels.xml` | 添加 `Label_MusicOSD_HintText` 系列变量 |
| `skin/1080i/DialogSeekBar.xml` | 音乐OSD提示文本改为动态变量 |
| `skin/1080i/Dialog_DialogCustom.xml` | 添加音乐OSD OnDown设置项 |
| `skin/language/*/strings.po` | 添加字符串 #31367 |
| `plugin.audio.music/addon.py` | play()中暴露歌手ID；song_comments()添加SQLite缓存 |

---

## 七、调试方法

1. 查看Kodi日志中 `[Music Comments]` 前缀的缓存日志（LOGINFO级别）
2. 查看Kodi日志中 `Activate of window '10502' refused` 确认dialog关闭问题
3. 查看Kodi日志中 `ignoring action 92` 确认动画时序问题
4. Kodi日志位置: `%APPDATA%\Kodi\kodi.log`
5. 皮肤修改后需重启Kodi生效（KEEP_IN_MEMORY缓存）
