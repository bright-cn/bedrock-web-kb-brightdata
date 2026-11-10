# 黄金集

`golden.json` 是当前使用的集合。包含 6 个问题，每个问题都与应回答该问题的页面配对。

`golden-v1-mislabeled.json` 是第一个版本，特意保留。对于“How do I target a specific country for my request?”这个问题，它预期返回配置页面。这是错误的：说明国家/地区定位功能的是特性页面。运行后得到 83% 的 hit@5，并将该问题列为未命中。

保留它是因为重点在于这个教训。未命中意味着语料库与你的预期不一致，但它不会告诉你究竟哪一方有误。在更改流水线的任何部分之前，请先阅读返回的分块。

    python scripts/5_evaluate.py eval/golden-v1-mislabeled.json --k 5   # 83%, one miss
    python scripts/5_evaluate.py eval/golden.json --k 5                  # 100%
