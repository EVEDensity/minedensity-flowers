# 手绘花园

从 [minedensity.top](https://minedensity.top/) 提取的花朵与花园场景，使用 Three.js 和 Vue 3。

[在线演示](https://evedensity.github.io/minedensity-flowers/)
![Uploading image.png…]()

## 功能

- 点击草地种花，向上拖动控制生长。
- 手绘花瓣、分层植物、地形和阴影。
- 移动鼠标产生轻微视差，支持清除新种的花。

## 本地运行

推荐 Node.js 24，需要支持 WebGL 的浏览器。

```bash
npm install
npm run dev
```

## 构建与部署

```bash
npm run build
npm run preview
```

GitHub Pages：在 Settings → Pages 中选择 GitHub Actions，推送到 main 后自动部署。

## 许可证

[Apache License 2.0](LICENSE)。第三方依赖保留各自许可证，参见 [NOTICE](NOTICE)。

视觉风格参考 [makemepulse 2019](https://2019.makemepulse.com/)，未使用其图片或模型素材。
