# 个人信息和功能配置恢复总结

## 已恢复的配置项 ✅

### 1. 社交链接

已恢复到 `themes/matery/_config.yml` 的 `socialLink` 部分：

```yaml
socialLink:
  github: https://github.com/RickDamon
  email: 719196134@qq.com
  qq: 719196134
```

这些链接会在首页 banner 中显示，访客可以通过点击这些链接联系你。

### 2. GitHub Fork Me 按钮

已恢复到 `themes/matery/_config.yml` 的 `githubLink` 部分：

```yaml
githubLink:
  enable: true
  url: https://github.com/RickDamon/RickDamon.github.io
  title: Fork Me
```

这会在网站右上角显示一个"Fork Me on GitHub"的按钮。

### 3. 音乐播放器

已恢复到 `themes/matery/_config.yml` 的 `music` 部分：

```yaml
music:
  enable: true
  server: netease
  type: playlist
  id: 6891570000          # 网易云音乐的个人收藏歌单ID
  fixed: true            # 吸底模式（始终显示在页面底部）
  autoplay: false        # 不自动播放
  theme: '#42b983'       # 主题颜色
  volume: 0.7            # 默认音量
  listFolded: true       # 歌单列表默认折叠
  hideLrc: true          # 隐藏歌词
```

音乐播放器会在首页显示，使用网易云音乐 ID `6891570000` 的播放列表。

## 配置位置

所有配置都在以下文件中：

- **社交链接和 GitHub 链接**: `themes/matery/_config.yml` (第 162-167 行, 第 409-412 行)
- **音乐播放器**: `themes/matery/_config.yml` (第 79-98 行)

## 后续调整

如需修改这些配置，直接编辑 `themes/matery/_config.yml` 文件，然后：

```bash
npx hexo generate
npx hexo deploy
```

### 音乐播放器可调整的选项

```yaml
enable: false          # 禁用音乐播放器
fixed: false           # 改为非吸底模式
autoplay: true         # 启用自动播放
volume: 0.5            # 调整默认音量（0-1）
listFolded: false      # 歌单列表默认展开
hideLrc: false         # 显示歌词
theme: '#FF0000'       # 修改主题颜色
```

### 社交链接可调整的选项

在 `socialLink` 中添加其他社交平台：

```yaml
socialLink:
  github: https://github.com/RickDamon
  email: 719196134@qq.com
  qq: 719196134
  weibo: https://weibo.com/xxx        # 微博
  zhihu: https://www.zhihu.com/xxx    # 知乎
  twitter: https://twitter.com/xxx    # Twitter
  facebook: https://www.facebook.com/xxx  # Facebook
```

## 验证

已验证配置已正确应用：
- ✅ 社交链接信息已在首页生成的 HTML 中
- ✅ GitHub Fork Me 按钮已配置
- ✅ 音乐播放器（meting-js）已在首页生成的 HTML 中
- ✅ 网易云音乐播放列表 ID 已配置

你的个人博客配置已完全恢复！
