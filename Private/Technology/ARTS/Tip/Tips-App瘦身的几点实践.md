#Tips-App瘦身的几点实践

## 资源文件

- 使用Asset Catalog，充分利用Apple提供的App Thining（App Slicing、Bitcode、On-Demand Resources）
- 删除无用图片使用[LSUnusedResources](https://github.com/tinymind/LSUnusedResources)工具即可，原理就是find命令找到所有的资源文件，和代码中的资源文件，然后做差集，就是无用的了
- 图片压缩。小图片（100k以下）使用[tinypng](https://tinypng.com/)或者[imageoptim](https://imageoptim.com/mac)来压缩图片即可
- 图片压缩。大图片（100k以上）使用[webp](https://developers.google.com/speed/webp/docs/precompiled)将图片压缩成webp格式,然后用[libwep](https://github.com/carsonmcdonald/WebP-iOS-example)在显示的时候解压即可。
- 无用代码、类查找、删除. [AppCode](https://www.jetbrains.com/objc/?fromMenu) 静态分析（不是很准确，需要人工二次确认）。Clang开发代码检查工具（代码量百万级以上可以考虑）