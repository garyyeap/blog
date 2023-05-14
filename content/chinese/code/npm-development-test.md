---
title: "NPM 開發測試筆記"
date: 2023-05-14
draft: false
categories:
- 寫扣
tags:
- javascript
- npm
- node
---

開發 NPM 套件時的測試方式，在開發的套件目錄下

```bash
# yarn
yarn link

# npm
npm link
```

這樣會建立一個 symbolic link 到套件的所有套件的目錄，要查看的話

yarn

```bash
cd ~/.config/yarn/link
```

npm，列出設定，找 prefix

```bash
npm config ls -l
```

拿到 prefix 後

```bash
cd {prefix}/lib/node_modules
```

在想測試該套件的應用/套件目錄下

```bash
# yarn
yarn link <package name>

# npm
npm link <package name>
```

基本上就等於“安裝”完成了，可以開始測試，在開發的套件裡面做的改變會即時反應到想用來測試的應用/套件上

測試完畢後，在測試的應用/套件上“解除安裝”

```bash
# yarn
yarn unlink <package name>

# npm
npm unlink <package name>
```

從 npm 主目錄上移除開發完畢的套件

```bash
# yarn
yarn unlink

# npm
npm unlink
```

如果想要在 global 環境測試，可以先確認套件是不是已經在 global 目錄

```bash
# yarn
yarn global list

# npm
npm ls -g --depth=0 --link=true
```

然後，yarn 的話，需要有到開發中套件目錄的絕對位址

```bash
yarn global add {full path to package}
```

移除的話就單純一些，知道名字就可以了

```bash
yarn global remove “package name
```

npm 的話跟上面本地的一樣，略過 `npm link <package name>` 指令即可
