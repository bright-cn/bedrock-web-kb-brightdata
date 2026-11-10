# AWS 端的 Terraform 配置

在 AWS 上配置流水线所需的全部资源，这样你就无需手动输入那个可能静默毁掉整个语料库的设置。

```bash
cd infra/terraform
tofu init            # or terraform init
tofu plan            # creates nothing, shows what would happen
tofu apply
tofu output -raw env >> ../../.env    # then add your Bright Data token
```

它会创建：

- 语料库 S3 存储桶，并启用加密和阻止公共访问
- 一个 S3 Vectors 存储桶和索引，其中 `AMAZON_BEDROCK_TEXT` 与 `AMAZON_BEDROCK_METADATA` 被声明为不可筛选；此设置决定 Bedrock 能否摄取你的文档，并且事后无法更改
- 知识库服务角色，权限范围限定为此语料库存储桶、此向量存储和唯一的嵌入模型，信任策略中带有 `aws:SourceAccount` 条件
- 知识库及其 S3 数据源，除非你设置 `create_knowledge_base = false`

如果你更愿意使用 `scripts/2_create_kb.py` 创建这些资源，请设置 `create_knowledge_base = false`；该命令会输出相同的 ID。其他所有资源仍会在此处完成配置。

## 应用之前

**首先检查你的嵌入配额。** 该配额按区域设置，新账户的配额可能为零，这会让所有资源顺利应用，但之后每个摄取作业都会失败。

```bash
aws service-quotas list-service-quotas --service-code bedrock --region us-east-2 \
  --query "Quotas[?contains(QuotaName,'Titan Text Embeddings V2') \
           && contains(QuotaName,'requests per minute')].[QuotaName,Value]" --output text
```

## 备注

- 需要 AWS provider 6.24 或更高版本，该版本开始提供 `aws_s3vectors_*` 资源。已使用 6.60.0 完成验证和规划。
- `force_destroy` 默认为 false，因此 `tofu destroy` 会拒绝删除非空存储桶。对于可随时丢弃的试点项目可将其设为 true；对于任何重要项目，请保持为 false。
- `embedding_dimension` 会根据 Titan Text Embeddings V2 接受的值进行验证，因为模型与索引不匹配时，会在写入阶段而不是规划阶段失败。
