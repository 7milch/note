はい、ご質問の状況、承知いたしました。
Terraformで管理されていない既存のS3バケットをオリジンとして、TerraformでCloudFrontを構築する、という構成は非常によくあるパターンです。

現在の課題は「S3バケットがIPアドレスでアクセスを制限している」点ですね。このままではCloudFrontからのアクセスも拒否されてしまいます。

これを解決し、Terraformでセキュアに構築するための手順と注意点を解説します。

---

### 全体的な流れ

1.  **Terraformで既存S3バケットの情報を取得する**: Terraform管理外のリソースを`data`ソースとして参照します。
2.  **CloudFrontからS3へのアクセスを設定する (OACの作成)**: 最新の推奨方式であるOrigin Access Control (OAC) を利用して、CloudFront専用のアクセス許可を作成します。
3.  **TerraformでCloudFrontディストリビューションを構築する**: 取得したS3情報とOACを使ってCloudFrontを定義します。
4.  **S3バケットポリシーを更新する**: **ここが最も重要です。** 既存のIP制限に加えて、CloudFrontからのアクセスを許可するステートメントを追加します。

### Step 1: Terraformで既存S3バケットの情報を取得する

まず、Terraformのコード内で、管理外のS3バケットの情報を参照できるようにします。これには`data`ソースを使います。`resource`ではないため、このコードがS3バケット自体を作成・変更・削除することはありません。

```terraform
# main.tf

# Terraform管理外の既存S3バケットの情報を名前で取得する
data "aws_s3_bucket" "existing_s3" {
  bucket = "your-existing-bucket-name" # ここに既存のバケット名を入力
}
```

### Step 2 & 3: OACを作成し、CloudFrontを構築する

次に、CloudFrontのディストリビューションと、S3にアクセスするためのOACをTerraformで定義します。

```terraform
# main.tf (続き)

# 1. CloudFront Origin Access Control (OAC) を作成
resource "aws_cloudfront_origin_access_control" "s3_oac" {
  name                              = "OAC for my-s3-bucket"
  description                       = "Origin Access Control for S3"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# 2. CloudFrontディストリビューションを作成
resource "aws_cloudfront_distribution" "s3_distribution" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Distribution for ${data.aws_s3_bucket.existing_s3.bucket}"

  origin {
    # S3バケットのリージョナルエンドポイントを指定 (推奨)
    domain_name = data.aws_s3_bucket.existing_s3.bucket_regional_domain_name
    origin_id   = "S3-${data.aws_s3_bucket.existing_s3.id}"

    # 作成したOACをオリジンに紐付ける
    origin_access_control_id = aws_cloudfront_origin_access_control.s3_oac.id
  }

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-${data.aws_s3_bucket.existing_s3.id}"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
  }

  # ... その他の設定 (ログ、証明書など) ...
  viewer_certificate {
    cloudfront_default_certificate = true
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
}
```

### Step 4: S3バケットポリシーを更新する

ここが最も重要なポイントです。現在のバケットポリシーには、特定のIPアドレスからのアクセスを許可するルール（`Allow` Effectと`IpAddress` Condition）が設定されているはずです。

このポリシーに、**「指定したCloudFrontディストリビューションからのアクセスを許可する」** というルールを追加する必要があります。

#### 方法1：ポリシー全体をTerraformで管理する（推奨）

既存のポリシーもTerraformのコードに取り込み、CloudFront用の許可を追加して、まとめて管理する方法です。これにより、設定の全体像がコードで把握でき、将来の変更も容易になります。

```terraform
# policy.tf

# 既存のIP制限とCloudFrontからの許可をマージしたポリシーを定義
data "aws_iam_policy_document" "s3_policy_doc" {
  # Statement 1: 既存のIPアドレス制限 (ご自身のポリシーに合わせて修正してください)
  statement {
    sid    = "IPAllow"
    effect = "Allow"
    principals {
      type        = "*"
      identifiers = ["*"]
    }
    actions   = ["s3:GetObject"] # 必要なアクションに絞る
    resources = ["${data.aws_s3_bucket.existing_s3.arn}/*"]

    condition {
      test     = "IpAddress"
      variable = "aws:SourceIp"
      values   = ["192.0.2.1/32", "203.0.113.10/32"] # ここに許可したいIPアドレスを記述
    }
  }

  # Statement 2: CloudFront (OAC) からのアクセス許可 (この部分を追加)
  statement {
    sid    = "AllowCloudFrontOAC"
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["cloudfront.amazonaws.com"]
    }
    actions   = ["s3:GetObject"]
    resources = ["${data.aws_s3_bucket.existing_s3.arn}/*"]

    # 重要: このCloudFrontディストリビューションからのみ許可する
    condition {
      test     = "ArnLike"
      variable = "AWS:SourceArn"
      values   = [aws_cloudfront_distribution.s3_distribution.arn]
    }
  }
}

# S3バケットにポリシーを適用する
resource "aws_s3_bucket_policy" "combined_policy" {
  bucket = data.aws_s3_bucket.existing_s3.id
  policy = data.aws_iam_policy_document.s3_policy_doc.json
}
```

**この方法の利点:**
*   バケットポリシー全体がコード化（IaC）される。
*   `terraform apply` を実行するだけで、既存のIP制限を維持しつつ、CloudFrontのアクセス許可が追加される。

#### 方法2：手動でポリシーを更新する

もし既存のバケットポリシーをTerraformで管理したくない場合、以下の手順で手動更新します。

1.  `terraform apply` を実行して、CloudFrontディストリビューションを作成します。
2.  Terraformの出力などから、作成されたCloudFrontディストリビューションのARN（`aws_cloudfront_distribution.s3_distribution.arn`）を確認します。
3.  AWSマネジメントコンソールでS3バケットの「アクセス許可」タブを開き、バケットポリシーを編集します。
4.  既存のポリシーに、以下の`Statement`を追加します。

```json
{
    "Sid": "AllowCloudFrontOAC",
    "Effect": "Allow",
    "Principal": {
        "Service": "cloudfront.amazonaws.com"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::your-existing-bucket-name/*",
    "Condition": {
        "ArnLike": {
            "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT_ID:distribution/DISTRIBUTION_ID"
        }
    }
}
```
*   `your-existing-bucket-name` と `AWS:SourceArn` の値は、ご自身の環境に合わせて書き換えてください。

### まとめと注意点

*   **OAC vs OAI**: OAC（Origin Access Control）はOAI（Origin Access Identity）の後継であり、よりセキュアで柔軟なため、新規構築ではOACを強く推奨します。
*   **S3エンドポイント**: `bucket_regional_domain_name` を使うことで、CloudFrontとS3間の通信がAWSのバックボーンネットワーク内で完結しやすくなり、パフォーマンス上有利です。
*   **ポリシーの`Condition`**: CloudFrontからのアクセスを許可する際、`AWS:SourceArn` を指定することが非常に重要です。これを指定しないと、他のAWSアカウントを含む**すべての**CloudFrontディストリビューションからS3バケットにアクセスできてしまい、セキュリティリスクとなります。
*   **Terraformの実行**: `terraform plan` を実行して、`aws_s3_bucket_policy` が意図通りに変更されることを必ず確認してから `terraform apply` を実行してください。

この手順で、既存のIP制限を壊すことなく、安全にCloudFrontを導入できます。
ご不明な点があれば、お気軽に追加でご質問ください。
