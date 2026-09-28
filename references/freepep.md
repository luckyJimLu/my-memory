# FreePEP

- Repository: https://github.com/siknet/FreePEP
- License: MIT
- Category: Education / Python / WebUI / CLI / PDF tooling
- Added: 2026-09-28

## Overview

FreePEP 是一个面向人民教育出版社中小学电子教材平台的 Python 工具，提供 WebUI 与 CLI，用于教材目录筛选、批量下载、图片页面处理和 PDF 合成。

## Key capabilities

- WebUI：按学段、年级、学科筛选并批量处理教材。
- CLI：支持交互式菜单及参数化调用，便于脚本化。
- PDF pipeline：下载教材页面图片并合成为 PDF。
- Batch mode：支持按“学段/年级”目录归档、断点续传与多线程处理。
- Windows packaging：可使用 PyInstaller 打包为独立可执行版本。

## Architecture notes

核心实现以 Python 为主，主要模块包括：

- `pep_core.py`：目录解析、页面处理与 PDF 合成等核心逻辑。
- `webui.py`：WebUI / 本地服务。
- `cli.py`：交互式与参数化 CLI。
- `download_all.py`：批量下载与目录归档。
- `build_exe.py`：Windows 独立发行包构建。
- Playwright / Chromium：浏览器自动化相关能力。
- Pillow：图像与 PDF 处理。

## Potential reuse in my projects

1. **HomeTutor / 教育内容系统**
   - 可参考其教材目录结构、学段/年级/学科三级筛选模型。
   - 可参考批量教材本地化、PDF 归档和断点续传设计。

2. **AIHome / 自动化工具**
   - 可参考 WebUI + CLI 双入口设计。
   - 可提取“任务队列 + 进度 + 本地文件输出”的通用模式。

3. **通用资源下载器**
   - 可参考目录缓存、批处理、失败恢复、并发下载与最终文件合成流程。

## Notes

- 教材内容具有版权，建议仅在合法授权、个人学习和平台允许的范围内使用。
- 自动化访问第三方网站时，应遵守目标网站条款、访问频率限制及相关法律法规。
- 如果后续实际复用代码，应单独核对当前版本的 LICENSE、依赖许可和上游接口变化。

## Source

https://github.com/siknet/FreePEP
