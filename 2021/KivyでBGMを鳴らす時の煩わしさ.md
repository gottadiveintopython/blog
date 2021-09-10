# KivyでBGMを鳴らす時の面倒臭さ

使ったことがある人ならわかると思うのですがKivyの音を鳴らす機能はかなり使いづらいです。
いや厳密には使っているaudio providerに依ると思うのですが、とりあえず私がdesktop Linuxで主に使っているproviderである`audio_gstplayer`は以下の点が煩わしいです。

```
(有効になっているaudio providerはlogを見ると分かる)
[INFO   ] [AudioGstplayer] Using Gstreamer 1.16.2.0
[INFO   ] [Audio       ] Providers: audio_gstplayer, audio_sdl2 (audio_ffpyplayer ignored)
```

**#1 再生した直後に音の長さは得られない**

`Sound.length`をただ参照すれば良いと思ったら間違いです。
このpropertyに有効な値が入るのは音を鳴らし始めて少し経ってからです。
つまり以下のようなcodeを書かないといけません。

```python
from kivy.clock import Clock
from kivy.core.audio import SoundLoader

sound = SoundLoader.load("sample.ogg")
sound.play()

# 駄目
# print(sound.length)

# 機能するがcodeが醜くなるのが不満
Clock.schedule_once(lambda __: print(f"音の長さは{sound.length}秒です"), .5)  
```

**#2 再生直後に再生位置を動かせない**

これも #1 と同じで少し待ってあげないといけません。

```python
from kivy.clock import Clock
from kivy.core.audio import SoundLoader

sound = SoundLoader.load("sample.ogg")
sound.play()

# 駄目
# print(sound.seek(30))

# 機能するがcodeが醜くなるのが不満
Clock.schedule_once(lambda __: sound.seek(30), .5)
```

## 原因と対策

このような"待ち"いちいち要る理由は音声の処理が別のthreadで行われているからだと私は思っています。
つまりこちらが`sound`を操作してから実際の処理が行われるまで時間差があるのです。
そしてそれが結果として上記のようにcodeが醜くなることに繋がっています。
というわけでそういった煩わしさはできるだけlibraryの中に隠しました。  
[BgmPlayer](https://github.com/gottadiveintopython/kivyx.utils.bgmplayer)

```python
from kivy.core.audio import SoundLoader
from kivyx.utils.bgmplayer import Bgm

bgm = Bgm(SoundLoader.load("sample1.ogg"))

# 再生位置の変更は再生する前からできる
bgm.pos = 90  # 00:01:30

# 再生
bgm.play()

# 長さの取得はまだ少し煩わしい。もしasync関数内であれば多少簡潔に書ける。
import asynckivy
async def async_func():
    if bgm.length is None:
        await asynckivy.event(bgm, 'length')
    print(f"音の長さは{bgm.length}秒です")
```
