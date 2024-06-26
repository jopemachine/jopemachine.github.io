---
layout: post
title: vscode 터미널 키 바인딩 (Mac)
subtitle: 개발환경 셋팅
author: jopemachine
header-img: img/header-img/vscode.png
header-mask: 0.3
tags:
  - vscode
last-update: November 19, 2021
---

## 문제

vscode 터미널에서 cmd +left, cmd + right 키를 사용하여 단어 사이를 오갈 수 있게 키를 바인딩 하고 싶다.

## 방법

1. 커맨드 팔레트를 연다 (<kbd>cmd</kbd> + <kbd>shift</kbd> + <kbd>p</kbd>)
2. Open Keyboard Shortcuts (JSON) 를 검색
3. 아래 텍스트를 복사 붙여넣기 한다.

```json
    ...
    {
        "key": "cmd+left",
        "command": "workbench.action.terminal.sendSequence",
        "args": {
            "text": "\u001bb"
        },
        "when": "terminalFocus && !terminalTextSelected"
    },
    {
        "key": "cmd+right",
        "command": "workbench.action.terminal.sendSequence",
        "args": {
            "text": "\u001bf"
        },
        "when": "terminalFocus && !terminalTextSelected"
    }
    ...
```

## 출처

1. [Integrated Terminal](https://code.visualstudio.com/docs/editor/integrated-terminal)
2. [Forward Backward Words in the Vscode Terminal](https://stackoverflow.com/questions/58476611/forward-backward-words-in-the-vscode-terminal)
