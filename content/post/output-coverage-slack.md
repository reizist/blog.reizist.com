---
title: "test coverageをcircleciでoutputするようにした"
description: "test coverageをcircleciでoutputするようにした"
date: 2018-11-08T02:17:00+09:00
draft: false
---

test coverageをcircleciでoutputするようにした。
所属チームの他のコンポーネントでテスト的に無料枠のCodecovを使っていて、
自分がメインで見ているrailsプロダクトにもcoverageを意識的に見る文化を取り入れたいと思ったものの
Codecovを最初から有料枠にするのも気が引けたので無料でできる範囲から。

やったことはほぼ https://qiita.com/u-minor/items/c18f0f03a9255f9e5231 の通りで、

多少アレンジしている。
circleciでの設定はrun section追加するだけで、プロジェクトの `bin/` に放り込んだshell scriptを実行しているだけ。

```yml:config.yml
- run:
    name: Post coverage
    command: bin/post-coverage-to-slack
```

```sh
###
# original: https://gist.githubusercontent.com/u-minor/8cb27fa9c04163142ebd/raw/circleci-coverage-slack
# Post coverage rate to Slack
#
# Usage: bash circleci-coverage-slack.sh [cobertura|jacoco]
#
# Required environment variables:
#
# - CIRCLE_TOKEN: project-specific readonly API token (need to access build artifacts for others)
# - SLACK_ENDPOINT: Slack endpoint url
# - COVERAGE_FILE: coverage xml filename (default: coverage.xml)
# - MAX_BUILD_HISTORY: max history num to fetch old rate (default: 5)
#

function calcCoberturaRate() {
  local rate=$(ruby -r "rexml/document" -e '
node = REXML::XPath.first(REXML::Document.new(STDIN.read), "coverage")
puts "%.2f" % (node.attributes["line-rate"].to_f * 100)
')
  [ "$rate" == "100.00" ] && rate=100
  echo $rate
}

function calcJacocoRate() {
  local rate=$(ruby -r "rexml/document" -e '
node = REXML::XPath.first(REXML::Document.new(STDIN.read), "report/counter[@type=\"LINE\"]")
missed = node.attributes["missed"].to_f
covered = node.attributes["covered"].to_f
puts "%.2f" % (covered / (covered + missed) * 100)
')
  [ "$rate" == "100.00" ] && rate=100
  echo $rate
}

function calcRate() {
  case "$coverage_type" in
    "cobertura" )
      calcCoberturaRate
      ;;
    "jacoco" )
      calcJacocoRate
      ;;
  esac
}

coverage_type=${1:-cobertura}
coverage=$(find $CIRCLE_ARTIFACTS -type f | grep ${COVERAGE_FILE:-coverage.xml})
url_base="https://circleci.com/api/v1/project/${CIRCLE_PROJECT_USERNAME}/${CIRCLE_PROJECT_REPONAME}"

## calculate rates
rate_new=$(cat $coverage | calcRate)
rate_old=0
for build_num in $(echo $CIRCLE_PREVIOUS_BUILD_NUM; seq $(expr $CIRCLE_BUILD_NUM - 1) -1 1 | head -n ${MAX_BUILD_HISTORY:-5}); do
  echo "fetching coverage info for build:${build_num} ..."
  artifacts="${url_base}/${build_num}/artifacts"
  [ -n "$CIRCLE_TOKEN" ] && artifacts="${artifacts}?circle-token=${CIRCLE_TOKEN}"
  coverage=$(curl -s -H 'Accept: application/json' $artifacts | jq -r '.[].url' | grep ${COVERAGE_FILE:-coverage.xml})
  coverage_html=$(curl -s -H 'Accept: application/json' $artifacts | jq -r '.[].url' | grep index.html)
  [ -n "$coverage" ] && break
done
if [ -n "$coverage" ]; then
  [ -n "$CIRCLE_TOKEN" ] && coverage="${coverage}?circle-token=${CIRCLE_TOKEN}"
  rate_old=$(curl -s $coverage | calcRate)
fi

## construct messages
author=$(git log --format='%an <%ae>' -1 | sed 's/\\/\\\\/g;s/\"/\\"/g')
log=$(git log --format=%s -1 | sed 's/\\/\\\\/g;s/\"/\\"/g')
mode=$(ruby -e 'puts (ARGV[0].to_f <=> ARGV[1].to_f) + 1' -- $rate_new $rate_old)
mes=("DECREASED" "NOT CHANGED" "INCREASED")
color=("danger" "#a0a0a0" "good")
cat > .slack_payload <<_EOT_
{
  "attachments": [
    {
      "fallback": "Coverage ${mes[$mode]} (${rate_new}%)",
      "text": "*${mes[$mode]}* (${rate_old}% -> ${rate_new}%) <${coverage_html}>",
      "pretext": "Coverage report: <https://circleci.com/gh/${CIRCLE_PROJECT_USERNAME}/${CIRCLE_PROJECT_REPONAME}/${CIRCLE_BUILD_NUM}|#${CIRCLE_BUILD_NUM}> <https://circleci.com/gh/${CIRCLE_PROJECT_USERNAME}/${CIRCLE_PROJECT_REPONAME}|${CIRCLE_PROJECT_USERNAME}/${CIRCLE_PROJECT_REPONAME}> (<https://circleci.com/gh/${CIRCLE_PROJECT_USERNAME}/${CIRCLE_PROJECT_REPONAME}/tree/${CIRCLE_BRANCH}|${CIRCLE_BRANCH}>)",
      "color": "${color[$mode]}",
      "mrkdwn_in": ["pretext", "text", "fields"],
      "fields": [
        {
          "value": "Commit: <https://github.com/${CIRCLE_PROJECT_USERNAME}/${CIRCLE_PROJECT_REPONAME}/commit/${CIRCLE_SHA1}|${CIRCLE_SHA1}>",
          "short": false
        },
        {
          "value": "Author: ${author}",
          "short": false
        },
        {
          "value": "${log}",
          "short": false
        }
      ]
    }
  ]
}
_EOT_

## post to slack
curl -s --data-urlencode payload@.slack_payload ${SLACK_ENDPOINT}
rm .slack_payload
```

変更したのは1点で、corbetura formatとは別にhtml formatも出力するようにしておいて、circleciのattachmentにlinkを追加している。
今まではsimplecov というgemを使ってcircleciでartifactとしてuploadするのみだったので、実際にgithubで開発をしていると、

1. github pushするとintegrationによりcircleci buildが実行される
2. build完了時、PRにその結果が通知される
3. 通知されたpostからcircleciのリンクをたどり該当のbuild結果からartifactのタブを選択
4. 該当のhtmlをクリック

という手順が必要で、何が言いたいかというとcoverageの詳細を確認する手順が多かった。

少しでもアクセスしにくいと感じると人間は低きに流れるのでチェックする習慣は無くなってしまうと思う。

のでpercentage of coverageをpostしcoverageを意識させつつ詳細を見やすくした、というのが今回の対応です。

{{% img-responsive "/output-coverage-slack/simplecov.png" %}}


尚このためにsimplecovの設定としてはMultiFormatterを使っていて

```rb
require 'simplecov'
require 'simplecov-cobertura'

SimpleCov.formatters = SimpleCov::Formatter::MultiFormatter.new(
  [
    SimpleCov::Formatter::HTMLFormatter,
    SimpleCov::Formatter::CoberturaFormatter
  ]
)
```

によってcorbeturaとhtmlを `coverage/` に出力している。
