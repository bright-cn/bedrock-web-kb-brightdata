# 使用 Bright Data 将实时网页数据导入 Amazon Bedrock 知识库

[![推广](https://github.com/bright-cn/Awesome-Web-Scraping/blob/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner%20CN.png)](https://www.bright.cn/products/web-scraper/google-hotels)

一个批处理流水线，可将公开网页转化为持久、可搜索的 Amazon Bedrock 知识库。Bright Data 负责抓取并清理页面，S3 负责交接，Bedrock 则对内容进行分块、嵌入、索引，并提供带引用的查询回答。

```
public URLs
  ─▶ Bright Data Web Unlocker        unblocks, geo-routes, renders JS when needed
  ─▶ S3 (.md + .metadata.json)       one clean doc + sidecar per page
  ─▶ Bedrock KB                      chunks, embeds, indexes (S3 Vectors)
  ─▶ Retrieve / RetrieveAndGenerate  cited answers
```

## 为何需要它

目前，Bedrock 原生的网页爬虫数据源仅支持静态网站，将缺少 `robots.txt` 视为禁止抓取，页面上限为 25,000 个，并且必须使用 OpenSearch Serverless。一些公开页面由客户端渲染、受到反爬虫保护，或需要清洁的数据提取。使用 Bright Data 获取这些页面，再让 Bedrock 从 S3 摄取结果。

## 基于实时试点的测量结果

6 个文档页面、us-east-2、Titan Text Embeddings V2、S3 Vectors 存储。

- 每页抓取延迟为 24.6 至 36.7 秒，中位数为 30.6 秒。使用 6 个工作进程时，全部页面共耗时 43.1 秒。
- 批次级页面框架清理移除了 21% 的字符，并将重复行数从 63 降至 0。
- 摄取耗时约 25 秒。在包含 6 个问题的黄金集上，检索 hit@5 为 100%，MRR 为 0.88。
- 对仅支持 JS 的页面（quotes.toscrape.com/js）执行普通 GET，只得到 96 个可索引文本字符。网络解锁器的快速路径返回了一个 191 字符的页面外壳。`render=True` 返回了完整页面。

请将这些数据视为单个测量点，而不是基准测试。请在你自己的目标网站上复现这些结果。

## 项目结构

```
config.py                    settings from environment (.env)
src/web_kb/
  brightdata.py              Web Unlocker (URL -> Markdown, optional render) + Crawl API
  clean.py                   native .md twin preference + cross-page chrome stripping
  dedup.py                   MinHash + LSH near-duplicate detection (opt-in)
  s3_loader.py               docs + sidecars to S3: content hashing, dedup, reconcile, last-seen
  knowledge_base.py          create KB (S3 Vectors or OpenSearch) + data source + ingestion
  retrieve.py                Retrieve / RetrieveAndGenerate
  evaluate.py                retrieval quality against a golden set (hit@k, MRR, misses)
  live_tool.py               hybrid routing: KB first, live Bright Data lookup as fallback
scripts/
  1_scrape_to_s3.py          scrape a URL list -> S3 (incremental, reports failures)
  2_create_kb.py             create the knowledge base + S3 data source
  3_sync.py                  run an ingestion job and read its statistics
  4_query.py                 query the knowledge base (--section filter, --generate)
  5_evaluate.py              score retrieval against a golden set (--min-hit-rate CI gate)
  6_inspect_cleaning.py      audit what chrome stripping would remove, before trusting it
  7_reconcile.py             retire objects whose URL left the list (dry-run by default)
  8_render_check.py          does a target need JS rendering? plain vs fast path vs rendered
eval/
  golden.example.json        golden-set format: {question, expect: [source urls]}
  golden-v1-mislabeled.json  the first version, kept on purpose. See eval/README.md
lambda/
  normalize_crawl_delivery.py  S3-triggered: split Crawl API delivery into per-page docs
infra/
  terraform/                   provisions bucket, vector store, IAM role, KB (see its README)
  kb-service-role-trust.json   trust policy (bedrock.amazonaws.com)
  kb-service-role-policy.json  least-privilege permissions for the KB role
  operator-policy.json         permissions for the human or CI job that runs the scripts
```

## 前提条件

- Python 3.10+。
- 一个拥有 API 令牌和网络解锁器区域的 [Bright Data](https://www.bright.cn/) 账户。
- 一个 AWS 账户，并已在你的区域中启用 Amazon Bedrock 和 Titan Text Embeddings V2。
- 一个用于语料库的 S3 存储桶，以及一个向量存储（默认使用 S3 Vectors）。

**首先检查你的嵌入配额。** 该配额按区域设置，新账户的配额可能为零。零配额允许你创建所有资源，但之后每个摄取作业都会失败。某个账户在同一天测得 us-east-1 为每分钟 0 个请求，而 us-east-2 为每分钟 60 个请求。

```bash
aws service-quotas list-service-quotas --service-code bedrock --region YOUR_REGION \
  --query "Quotas[?contains(QuotaName,'Titan Text Embeddings V2') \
           && contains(QuotaName,'requests per minute')].[QuotaName,Value]" --output text
```

## 设置

1. **安装。**
   ```bash
   python -m venv .venv && source .venv/bin/activate
   pip install -r requirements.txt
   cp .env.example .env      # then fill it in. AWS_PROFILE works from here too
   ```

2. **配置 AWS 端。** 最快捷的方式是使用 Terraform。它会创建语料库存储桶、带有正确不可筛选元数据键的 S3 Vectors 存储、服务角色，并可选择创建知识库本身。
   ```bash
   cd infra/terraform && tofu init && tofu apply    # or terraform
   tofu output -raw env >> ../../.env               # then add your Bright Data token
   ```
   请参阅 `infra/terraform/README.md`。若要改为手动操作，请通过 `infra/kb-service-role-trust.json` 和 `infra/kb-service-role-policy.json` 创建知识库服务角色。请先替换占位的账户 ID、区域和 ARN，然后将角色 ARN 填入 `.env`。

3. **填写 `.env`**，提供 Bright Data 令牌、存储桶、区域和 ARN。你无需手动创建 S3 Vectors 存储桶或索引。`2_create_kb.py` 会创建它们，并将 `AMAZON_BEDROCK_TEXT` 和 `AMAZON_BEDROCK_METADATA` 声明为不可筛选。该设置决定 Bedrock 能否摄取你的文档。如果已有索引但未配置此项，脚本会拒绝继续并说明原因。否则，同步可能会报告 COMPLETE，但 Bedrock 会静默地无法嵌入语料库中的大部分内容。如果你自行配置，请传入 `--skip-vector-store`。

## 运行

```bash
# 0. Does your target even need scraping or rendering? Measure before you spend.
python scripts/8_render_check.py https://your-target/page --probe "text only on the rendered page"
python scripts/6_inspect_cleaning.py urls.txt          # audit what stripping would remove
python scripts/6_inspect_cleaning.py urls.txt --verify # and score what survived vs each .md twin

# 1. Scrape a URL list to clean Markdown in S3 (one .md + sidecar per page)
python scripts/1_scrape_to_s3.py urls.txt --section docs
python scripts/1_scrape_to_s3.py urls.txt --section docs --near-dup 0.85   # also drop near-dupes

# 2. Create the knowledge base and connect the bucket as an S3 data source
python scripts/2_create_kb.py --name web-kb --store s3vectors --chunking DEFAULT
#    creates the vector bucket and index with the right metadata config, then the KB
#    -> prints KNOWLEDGE_BASE_ID and DATA_SOURCE_ID, so paste both into .env

# 3. Ingest (sync). Re-run after every re-scrape. Syncs are incremental.
python scripts/3_sync.py

# 4. Query. Chunks by default, a written answer with --generate.
python scripts/4_query.py "what changed in pricing?" --section docs
python scripts/4_query.py "summarize the latest release" --generate

# 5. Measure retrieval BEFORE scaling (chunking is fixed when you create the data source)
python scripts/5_evaluate.py eval/golden.json --k 5
python scripts/5_evaluate.py eval/golden.json --min-hit-rate 0.8    # CI gate
```

第 1 步中值得了解的行为：

- **优先使用原生 Markdown 对应版本。** 遵循 llms.txt 约定的网站会提供 `<page>.md`。如果该文件存在，它通常更干净、更快且免费，因此加载器会跳过抓取。
- **页面框架会根据整个批次进行清理。** 加载器会删除在大多数页面上重复出现的行。在新网站上使用前，请通过 `6_inspect_cleaning.py` 审核结果。使用 `--verify` 为结果评分，它会将保留下来的内容与每个页面自己的 `.md` 对应版本进行比较。该语料库的测量结果：每页移除一行，每次都是相同的 llms.txt 样板内容，未丢失任何页面独有内容。
- **完全重复的页面始终会被丢弃，近似重复的页面仅在你要求时才会丢弃。** 内容哈希可识别字节完全相同的镜像。在真实网站上，这包括 `?utm_source=` 与 `?print=1` 别名以及末尾斜杠变体，它们都返回相同的字节。对于采用不同形式的同一文档副本，需要使用 `--near-dup`。它会对清理后的文本运行 MinHash，并报告丢弃的每个页面及其匹配页面。此功能默认关闭，因为错误匹配会导致页面丢失。报告会列出保留下来的最相近页面对，便于你根据证据设置阈值。该语料库的测量结果：不同页面之间的分数最高为 0.02，而一个页面及其 `.md` 对应版本的分数为 0.75。默认阈值 0.85 会保留两者，而阈值 0.75 会丢弃对应版本。
- **未更改的页面不会被重写。** 变更检测对原始抓取结果进行哈希，而不是对清理后的正文进行哈希。即使页面来自不同批次，只要没有变化，Bedrock 就会跳过它。若没有这道检查，每次同步都会重新嵌入整个语料库。
- **空响应和返回 200 的 HTML 响应均视为失败。** 错误的区域名称会返回空的 200 响应；如果某个区域的 IP 允许列表不再匹配你的当前地址，也会如此。加载器会报告这两种响应，而不是将其作为内容写入 S3。
- **失败的抓取会明确列出名称，并且脚本会以非零状态退出**，因为无法重新抓取的页面仍会将其旧版本作为当前版本提供。
- **最后发现日期与内容时间分开跟踪。** 每次成功抓取都会将日期记录到摄取前缀之外的台账中。`--stale-after DAYS` 会报告停止刷新的页面，而 `scraped_date` 无法提供这项信息。

第 3 步完成后，**请查看统计信息，而不是状态**。即使大多数文档失败，摄取作业也可能报告 `COMPLETE`。`3_sync.py` 会输出计数，并在发生失败时以非零状态退出；作业上的 `failureReasons` 则包含实际错误。

## 保持内容最新

按照与你的数据源更新频率相匹配的计划，重新运行第 1 步和第 3 步。`slugify` 键是确定性的，因此重新抓取会覆盖同一个对象，增量同步只会重新嵌入你重写的页面。当 URL 从列表中移除时，请运行 `7_reconcile.py urls.txt --delete`，下次同步会停用其向量。它会与完整列表进行比较，因此绝不会将仅仅抓取失败的页面误认为已被你移除。生产环境中，请通过 Amazon EventBridge 计划连接第 1、7 和 3 步。

## 混合检索

知识库是一个快照，因此无法回答有关快速变化事实的问题，也无法回答语料库外页面的问题。`src/web_kb/live_tool.py` 会按查询进行路由。它首先查询知识库。当最佳分块低于相关性下限时，它会改为运行 Bright Data 实时搜索和抓取。请使用你的黄金集以及一些语料库未覆盖的问题来调整下限。`5_evaluate.py` 会自动输出该区间：命中项的最低分和未命中项的最高分。请将这些额外问题保存在单独文件中，以免计入 `--min-hit-rate` 检查。在 AWS 端，将实时路径连接为 Bedrock 智能体操作组或 AgentCore Gateway 工具。Gateway 的原生知识库目标适用于 Bedrock 托管知识库。像本项目这样的客户自主管理构建，需要通过一个调用 `Retrieve` 的小型 Gateway 工具接入。

## 使用爬虫 API 扩展

若要覆盖整个网站而不是固定 URL 列表，`brightdata.start_crawl(...)` 会触发一次爬取，发现内部页面并返回 Markdown 记录。交付是单独的步骤：使用 `wait_for_crawl(...)` 轮询，直到快照变为 `ready`，然后调用 `deliver_crawl(...)` 将记录写入 S3 中的落地前缀。使用该前缀上的 S3 触发器部署 `lambda/normalize_crawl_delivery.py`。它会将每条记录拆分为源前缀下的一个 `<slug>.md` 文档和伴随文件，后续同步会摄取它们。建议优先使用 IAM 角色交付凭证，而不是访问密钥对。

## 测试

离线单元测试覆盖纯逻辑，无需任何凭证：

```bash
python -m pytest tests/ -q      # 39 tests
```
