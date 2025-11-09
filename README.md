# Kyverno Config Repository

このリポジトリはKyvernoポリシーを管理し、CI/CDパイプラインで自動検証を行うためのものです。

## 📁 ディレクトリ構造

```
.
├── policies/                    # Kyvernoポリシーファイル
│   ├── require-labels.yaml
│   ├── disallow-latest-tag.yaml
│   ├── require-hpa.yaml
│   └── check-hpa-exists.yaml
├── resources/
│   └── test-resources/         # テスト用Kubernetesリソース
│       ├── good-pod.yaml
│       ├── good-deployment.yaml
│       └── good-hpa.yaml
├── kyverno-test.yaml           # Kyverno testコマンド用テスト定義
└── .github/
    └── workflows/
        └── kyverno-test.yml    # GitHub Actions CI/CDワークフロー
```

## 🚀 GitHub Actions CI/CD

このリポジトリでは、GitHub Actionsを使用してKyverno CLIによる自動検証を実行しています。

### ワークフローの内容

1. **ポリシーテスト**: `kyverno apply` コマンドで全ポリシーをテストリソースに適用
   - ポリシーをリソースに適用して検証
   - 基本的なバリデーションポリシーをテスト
   - 注: `check-hpa-exists.yaml`はKubernetes API必要のためスキップされます
2. **テストケース実行**: `kyverno test` コマンドで定義されたテストケースを実行
   - 構造化されたテストケースで期待結果を検証
   - 全てのポリシーとリソースの組み合わせをテスト

### トリガー条件

- mainブランチへのpush
- `claude/**` ブランチへのpush
- mainブランチへのPull Request作成時 (opened)
- mainブランチへのPull Request更新時 (synchronize)

## 🔧 ローカルでのテスト

### Kyverno CLIのインストール

```bash
# Linux
wget -qO- https://github.com/kyverno/kyverno/releases/download/v1.12.0/kyverno-cli_v1.12.0_linux_x86_64.tar.gz | tar -xz
sudo mv kyverno /usr/local/bin/

# macOS (Homebrew)
brew install kyverno
```

### ポリシーのテスト

```bash
# 全ポリシーをリソースに適用してテスト
kyverno apply policies/ --resource resources/test-resources/

# 特定のポリシーとリソースでテスト
kyverno apply policies/require-labels.yaml \
  --resource resources/test-resources/good-pod.yaml
```

### テストケースの実行

```bash
# kyverno-test.yamlで定義されたテストを実行
kyverno test .
```

## 📝 サンプルポリシー

### 1. require-labels.yaml

Kubernetes リソースに必須ラベル `app.kubernetes.io/name` が設定されていることを検証します。

**対象リソース**: Pod, Deployment, StatefulSet, DaemonSet

### 2. disallow-latest-tag.yaml

コンテナイメージに `latest` タグの使用を禁止し、明示的なバージョンタグを要求します。

**対象リソース**: Pod

### 3. require-hpa.yaml

HPA（HorizontalPodAutoscaler）リソースが適切に設定されていることを検証します。

**ポリシー内容**:
- `validate-hpa-configuration`: HPAリソースが適切に設定されていることを確認
  - `scaleTargetRef`: Deploymentへの参照が正しく設定されているか
  - `minReplicas`: 1以上であること
  - `maxReplicas`: 2以上であり、minReplicasより大きいこと

**対象リソース**: HorizontalPodAutoscaler

**動作モード**: Audit（検知のみ、ブロックしない）

### 4. check-hpa-exists.yaml

Deployment/StatefulSetに対応するHPAが実際に存在するかをKubernetes APIを使って検証します。

**ポリシー内容**:
- `check-hpa-exists`: API経由で同じnamespace内のHPAを取得し、対応するHPAが存在するか確認
  - `context.apiCall`を使用して実際のクラスター状態を参照
  - HPAの`scaleTargetRef.name`にリソース名が含まれているかチェック

**対象リソース**: Deployment, StatefulSet

**動作モード**: Audit（検知のみ、ブロックしない）

**注意**: このポリシーはKubernetes APIへのアクセスが必要なため、CI環境（`kyverno apply`）ではスキップされます。実際のKubernetesクラスターにKyvernoをデプロイした際に機能します。

## 🔍 新しいポリシーの追加方法

1. `policies/` ディレクトリに新しいYAMLファイルを作成
2. `resources/test-resources/` にテスト用リソースを追加
3. `kyverno-test.yaml` にテストケースを追加（オプション）
4. ローカルでテストを実行
5. コミットしてプッシュすると、自動的にCIが実行されます

## 📚 参考リンク

- [Kyverno公式ドキュメント](https://kyverno.io/)
- [Kyverno CLI](https://kyverno.io/docs/kyverno-cli/)
- [Kyvernoポリシーライブラリ](https://kyverno.io/policies/)