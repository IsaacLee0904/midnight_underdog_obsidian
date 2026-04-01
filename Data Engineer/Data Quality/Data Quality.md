tags: #data-quality-check #etl #data-engineering
source: [DataExport](https://www.youtube.com/watch?v=JiedBnTFCeg&list=PLwUdL9DpGWU0lhwp3WCxRsb1385KFTLYE&index=13)

---
* How to build high trust in the data sets that you build
* How to build good data docs
* Data quality checks and how thet diff between facts and dims 
## Data Quality Basic

>Data quality refers to the development and implementation of activities that apply quality management techniques to data in order to ensure the data is fit to serve the specific needs of an organization in a particular context.

資料品質 (Data Quality) 是指資料的準確性、一致性、完整性和可靠性，高品質的資料能夠讓企業基於準確資訊做出有效的決策，而資料品質包含數個面向：

* **可發現性 (Discoverability)**：指的是當 user 想要做決策，他們能輕鬆知道資料是否存在並且要去哪裡取用它 -> 應該算是定義更嚴格的的資料可用性 (Data Availability)

* **時間完整性** :
* **屬性完整性**：
* **指標定義一致性**：
* **運用準確性**：
* **來源可靠性**：


- **可發現性 (Discoverability)**：這點常被低估且困難。意味著如果有人想做決策，他們能輕鬆知道資料是否存在並取得它。
- **對資料品質的誤解與不完整定義**：例如 Zillow 曾以為他們有完美的機器學習模型可以預測房地產市場，結果損失了上億美元，因為他們的資料品質不夠高，沒有考慮到模型可能遺漏了哪些重要的資料點。有時候甚至會遇到難以預測的『黑天鵝事件』（例如 COVID-19 疫情）。
- **基本規範**：不該為 null 的欄位永遠不能為 null，且特別是維度表 (Dimension tables) 中不能有重複值。
- **從資料中獲得商業價值**：這通常指的是金錢。每一條你寫的資料管道要麼能產生更多收入，要麼能節省成本（例如測量在 AWS 上的花費，好讓我們少給 Jeff Bezos 幾十億美元）。唯一的例外是提供戰略價值的資料集，這類資料主要用於重大高層決策。
- **資料易於使用**：欄位名稱必須明顯且合理。例如 Airbnb 常見的命名慣例是維度欄位以 `dim` 開頭，指標欄位以 `m_` 開頭。
- **資料及時到達**：與分析團隊達成一致的資料到達時間非常強大，可以讓他們的工作更順暢，並減少資料工程師收到的無謂請求。

簡而言之，**資料品質 = 資料信任 + 資料影響力**。如果你能持續建立具備這兩個屬性的資料集，你很快就會獲得晉升


[[Data Quality Patterns]]

## Reference
[《資料與程式碼的交鋒》Day 24 — 資料需求金字塔](https://shu-ting.medium.com/data-feat-programming-day-24-5f691450323f)
