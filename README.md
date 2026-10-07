# 手绘花园 · Three.js + Vue

[在线演示](https://evedensity.github.io/minedensity-flowers/) · [原网站](https://minedensity.top/) · [Apache-2.0](LICENSE)

从个人网站独立提取的花朵与花园场景演示。保留花瓣展开、弯曲花茎、点击种花、向上拖动生长、草地阴影、分层灌木与蕨类。底层是 Three.js 几何体，使用手绘材质和轮廓线呈现插画风格。

## 运行

需要 Node.js 22.12+（推荐 24）和支持 WebGL 的浏览器。

```bash
npm install
npm run dev
```

打开终端显示的地址。点击草地种花，按住并向上拖动控制生长，移动鼠标查看轻微视差。右下角按钮清除新种的花，初始示例花保留。新增花最多保留 35 朵。

```bash
npm run build
npm run preview
```

## 代码位置

- `src/components/scene/FlowerGardenThree.vue`：花朵几何、动画、交互与渲染循环。
- `src/composables/flowerGrowth.ts`：生长曲线与花瓣折叠。
- `src/components/scene/gardenWorld.ts`：地形、光照和阴影。
- `src/components/scene/botanicalBackdrop.ts`：分层灌木、叶片与蕨类。
- `src/components/scene/distantGarden.ts`：远景。
- `src/components/scene/illustrationMaterial.ts`：手绘材质。

## 公开范围

只提取程序生成的花朵和场景，并保留天空中的 Density 装饰字样；不包含原站文章、截图、音乐、个人联系账号、邮箱、备案号、统计脚本、服务器配置或原网站 Git 历史。手绘文字位于 `src/components/scene/LayerSky.vue`。本仓库包含独立的 GitHub Pages 部署工作流，不需要服务器密钥。

视觉方向参考 makemepulse 2019 年新年网站；本演示未复制其图片或模型素材。参考不等于官方合作或授权。

## 许可证

原创代码采用 **Apache License 2.0**，见 [LICENSE](LICENSE) 和 [NOTICE](NOTICE)。第三方依赖遵循各自许可证。

## GitHub Pages

这是无需后端的静态应用，适合部署到 GitHub Pages。Vite 使用相对资源路径，支持仓库子路径。仓库 Settings → Pages → Source 选择 GitHub Actions 后，推送到 main 会自动构建并发布 `dist`。

WebGL 不可用时会显示提示。页面隐藏时暂停绘制；高分屏像素比限制为 2。复杂场景在低端设备上仍可能需要进一步降低阴影分辨率或像素比。
