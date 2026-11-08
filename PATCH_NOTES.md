# LDDC QQ 音乐搜索修复补丁（2026-11-07）

## 问题
QQ 于 2026-11-07 关闭免登录正式搜索：`DoSearchForQQMusicLite` 恒返回
`APIRequestError: qm API请求错误,错误码:2001`，LDDC 的 QQ 源搜索整体失效
（取词接口本身没事，坏的只是搜索）。

## 改法（只动一个文件：LDDC/core/api/lyrics/qm.py）
- `QMAPI.search()` 里单曲（SONG）搜索分流到新方法 `_search_song_smartbox`，
  不再调用已死的 DoSearchForQQMusicLite。
- 新方法走 QQ 搜索框的 smartbox 联想接口
  （c.y.qq.com/splcloud/fcgi-bin/smartbox_new.fcg，免登录）拿候选，
  再用 `get_song_detail_yqq` 逐首补全时长/专辑/歌手/语种，
  组装成原 `format_songinfos` 期望的结构复用其转换，返回契约不变。
- 取词（get_lyrics）、专辑曲目（get_songlist）等其余逻辑一行未动。

## 实测（2026-11-07，在源码里无头跑通）
- 搜「泡沫」→ 首条 id 1530858、专辑 Xposed、时长 258000ms，与 QQ 数据一致；
  接着走 LDDC 自己的 get_lyrics → 逐字（VERBATIM）39 行带词级时间戳。
- 搜韩文「불꽃」→ 命中高耀太原版。

## 已知边界
- smartbox 是联想接口：候选数比正式搜索少（一页约 4~10 条）、没有第二页
  （page > 1 返回空）。手动找歌足够，批量自动匹配的召回略逊于当年。
- 专辑搜索与歌单搜索仍走旧接口（歌单搜索用的是同一个已死接口、仍不可用；
  专辑搜索走 Desktop 版接口、未实测），本次只修单曲搜索。
- QQ 若再关 smartbox，此补丁同样失效，届时只能上登录态方案。

## 用法（Windows，从源码运行）
```
pip install -r requirements.txt
python -m LDDC
```
在本目录下执行即可（补丁已包含在源码里）。首次运行的设置与旧 exe 独立，
可能需要重设一遍本地匹配选项。

## 追加修复：双行同时高亮（LDDC/core/parser/qrc.py）
QQ 部分歌曲的 QRC 行时长自带约 +300ms 尾巴（如《该死的温柔》57 行中 51 行），
行结束时间越进下一行，按区间高亮的播放器会同时点亮两行。已在 qrc2data
返回前钳制：越界行的词起始/词结束/行结束一律不越过下一行行首。实测该歌
越界 51 → 0；无重叠的歌（如高耀太《불꽃》）解析结果与补丁前逐值一致，
行首时间与歌词内容不动。LDDC 导出增强 LRC 已验零越界。

## 追加修复：保存歌词时补齐艺术家标签（LDDC/core/song_info.py）
LDDC 原生 write_lyrics 只写歌词、不碰艺术家标签。现改为：有匹配结果时
顺带补齐——ARTIST 缺失用匹配歌手补，专辑艺术家（aART/TPE2/ALBUMARTIST）
缺失才填（优先文件自带 ARTIST、其次匹配歌手），已有值绝不覆盖。
m4a/mp3/flac 三格式实测通过，WMA(ASF) 不在覆盖范围。
