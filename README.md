# Multi YouTube

YouTubeの複数の配信・動画を一つのウィンドウに表示して視聴できるWebアプリです。

このWebアプリは1つのプレイヤーのみ再生可能です。  
プレイヤーが再生中のときに他のプレイヤーの再生を始めると、再生中のプレイヤーは一時停止します。

YouTubeでは、1つのWebページ内に複数のYouTubeプレーヤーを配置する場合は常に再生されるのは1つのプレイヤーに限定しなければならないと[利用規約](https://developers.google.com/youtube/terms/developer-policies)で定められています。  
このWebアプリを使用して複数あるプレイヤーのうち1つのプレイヤーだけを再生することに関しては問題ありませんが、2つ以上のプレイヤーの同時再生は規約違反になります。

[MIT License](https://opensource.org/licenses/MIT) です。  
ライセンスの通り、使用は自己責任です。

## Project setup
```
yarn install
```

### Compiles and hot-reloads for development
```
yarn serve
```

### Compiles and minifies for production
```
yarn build
```

### Lints and fixes files
```
yarn lint
```
