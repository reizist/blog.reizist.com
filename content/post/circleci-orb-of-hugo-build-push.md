---
title: "Hugoのbuild & pushを行うcircleci orb作った"
date: 2018-11-25T15:55:24-08:00
draft: false
---

## モチベーション
以前から自分用のドメインを持っていたが、プライベート用の何かしら（e.g. 転職活動に使う各アセット、具体的には職務経歴書などのホストなど）に使う程度だったのだけどもったいないからポートフォリオ作るか -> せっかくだからブログも改めて独自ドメイン運用するか -> この際新しく作ろう -> hugo -> テンプレ作ろう

## 使い方

```
hugo new site blog.yourdomain
hugo new post/great-post.md
```

以下の環境変数をセット

### 必須
- GIT_USER: git user name
- GIT_EMAIL: git user email address

### オプション
- GIT_PUBILSH_BRANCH: publish branch (default: gh-pages)
- GIT_BUILD_BRANCH: build branch (default: master)

`~/.circleci/config.yml` を以下のように定義

```yml
version: 2.1

orbs:
  build-push: reizist/hugo-build-push@dev:8

workflows:
  "build-push":
    jobs:
      - build-push/build:
          filters:
            branches:
              only:
                - $GIT_BUILD_BRANCH
```

これによって有効なgit accountが設定されていれば `GIT_BUILD_BRANCH` の内容をもとに `GIT_PUBLISH_BRANCH` に静的ファイルがpublishされるのでそれをgh-pagesなりでホストすればよい。

orbのソースは[githubにある](https://github.com/reizist/orb-hugo-build-push)のでみていただくとよいが、
alpineベースのイメージにhugoやgitなどの最低限のライブラリを突っ込んでいて軽量なのでビルドも速く、
github pushから30sec程度でブログが更新されるので体験としては結構気に入っている。
まだorbのタグ付とかちゃんとしてない感があるのでもうちょい改善の余地がある。
