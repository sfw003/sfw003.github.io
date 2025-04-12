---
title: 【Qt Widgets】Qt 布局管理
description: Qt 布局管理
date: 2025-04-04
slug: qt-widgets_layout_management
categories:
    - Qt
tags:
    - Qt Widgets
---

## Qt 布局管理


### 基本布局

Qt 的布局管理系统提供了多种布局管理器类，用于自动排列和管理控件的位置与大小，确保界面在不同分辨率下保持美观和功能性。以下是其核心布局类及其特点的总结：




#### `QHBoxLayout`（水平布局）
• **功能**：将控件从左到右水平排列，适用于工具栏、按钮组等横向布局场景。
• **特点**：自动调整控件间距，支持拉伸系数（`stretch`）控制控件宽度占比。
• **代码示例**：
  ```cpp
  QHBoxLayout *layout = new QHBoxLayout;
  layout->addWidget(button1);
  layout->addWidget(button2);
  ```



####  `QVBoxLayout`（垂直布局）
• **功能**：将控件从上到下垂直排列，常用于表单、列表等纵向布局。
• **特点**：支持设置空白区域（`addStretch()`）和固定间距（`addSpacing()`），灵活分配空间。

```cpp
QVBoxLayout *layout = new QVBoxLayout;
layout->addWidget(label1);
layout->addWidget(lineEdit);
```



####  `QGridLayout`（网格布局）
• **功能**：将控件按行和列排列成网格，支持控件跨行或跨列。
• **特点**：适用于复杂界面（如计算器、表格），通过 `addWidget(widget, row, column, rowSpan, columnSpan)` 指定位置。

```cpp
QGridLayout *layout = new QGridLayout;
layout->addWidget(button1, 0, 0); // 第0行第0列
layout->addWidget(button2, 0, 1, 1, 2); // 跨2列
```



####  `QFormLayout`（表单布局）
• **功能**：专为表单设计，每行包含标签（如 `QLabel`）和输入控件（如 `QLineEdit`）。
• **特点**：自动对齐标签和输入框，简化登录界面、设置对话框的开发。


```cpp
QFormLayout *layout = new QFormLayout;
layout->addRow("用户名:", lineEditUser);
layout->addRow("密码:", lineEditPass);
```

