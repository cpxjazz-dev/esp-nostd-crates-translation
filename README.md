# ESP32-C3 Rust `no_std` 常用 crate 文档翻译（v2 · C3 专属查阅版）

ESP32-C3 Rust `no_std` 常用 crate（esp-hal / esp-radio / esp-rtos / esp-nvs / embassy-net 等）文档的中文翻译，只保留 C3 相关内容，可直接当查阅手册用。

- 主文档：[`esp-nostd-crates-translation.md`](esp-nostd-crates-translation.md)（Markdown，v2）
- 网页版：[`esp-nostd-crates-translation.html`](esp-nostd-crates-translation.html)（内容与 Markdown 一致，浏览器直接打开）
- Obsidian 版：[`esp-nostd-crates-translation-obsidian.md`](esp-nostd-crates-translation-obsidian.md)（带 YAML frontmatter，放进 vault 即用）

翻译原则：说明性文字译为中文；crate 名、函数名、宏名、feature 名、代码与 Cargo.toml 保持原文。版本号以 crates.io 为准，落项目前请核对。

内容包括：C3 硬件速查与引脚陷阱、esp-hal 外设 API 表、异步专章（esp-rtos + Embassy）、esp-alloc / esp-backtrace / esp-println / esp-storage / esp-nvs、esp-radio 与 embassy-net（Wi-Fi + TCP）、已核实文件名的示例索引与未核实清单。
