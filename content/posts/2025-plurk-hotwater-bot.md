+++
title = "多喝熱水2.0"
date = "2025-03-22T00:00:00+08:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
keywords = ["噗浪機器人", "噗浪聊天機器人", "噗浪"]
showFullContent = false
type = "posts"
+++

- [噗浪機器人傳送門](https://www.plurk.com/hotwaterbot)
- [github](https://github.com/kinako890419/plurk-hotwater-bot)

## 待辦事項
- [x] 換平台部署因為AWS額度用完了
- [ ] 改時區
- [ ] 不要回應匿名噗
- [ ] 修好不知道為什麼有時候一天會發兩次噗的問題
- [ ] 加標註關鍵字回應(之類的功能)
- [ ] 可能把code修正常一點

> 機器人回應不代表本人立場，請小心服用

<!--more-->

## 自動回覆
- 機率性回覆
![image](https://hackmd.io/_uploads/rJUZNFPd1e.png)
- 關鍵字隨機回覆，例如：
![image](https://hackmd.io/_uploads/BJeuVtw_yx.png)

## 關鍵字回覆

| qualifier | 關鍵字         | 備註          |
| --------- | ----------- | ----------- |
| 希望        | `!抽`        | 抽三張塔羅牌      |
| 想要        | `!抱怨`       |             |
| 問         | `!為什麼`      |             |
| 問         | `!要不要`      | 隨機決定選項+解釋理由 |
| -         | `要不要`、`好不好` | 只有回覆隨機的決定選項 |
| -         | 其他關鍵字       |             |

### 抽塔羅牌 (三張)

希望 + !抽
![image](https://hackmd.io/_uploads/HJ99kTNuJx.png)

- 回應不太穩，但大致上分三段內容做回覆


### 嗆你 (請謹慎使用)

想要 + !抱怨
![image](https://hackmd.io/_uploads/rJSpbaN_1l.png)

### ??

問 + !為什麼
![image](https://hackmd.io/_uploads/Hkxsm6Vdke.png)

