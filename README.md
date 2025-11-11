# Kyverno Config Repository

このリポジトリはKyvernoポリシーを管理し、CI/CDパイプラインで自動検証を行うためのものです。

## 📁 ディレクトリ構造

```
.
├── policies/                    # Kyvernoポリシーファイル
│   ├── require-labels.yaml
│   ├── disallow-latest-tag.yaml
│   ├── require-semver-tag.yaml
│   ├── require-resource-limits.yaml
│   ├── require-security-context.yaml
│   ├── require-probes.yaml
│   ├── require-image-pull-policy.yaml
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

### 3. require-semver-tag.yaml

コンテナイメージのタグがセマンティックバージョニングまたは日付形式に準拠していることを検証します。

**ポリシー内容**:
- `validate-image-tag-format`: イメージタグのフォーマットを検証
  - セマンティックバージョニング（例：1.2.3, v1.2.3）の使用を推奨
  - 日付ベースのタグ（例：20240101, 2024-01-01）も許可
  - 曖昧なタグ（latest, dev, stable, production, staging等）を禁止

**対象リソース**: Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob

**動作モード**: Audit（検知のみ、ブロックしない）

**禁止されるタグ例**: `latest`, `dev`, `develop`, `master`, `main`, `stable`, `production`, `prod`, `staging`, `test`

### 4. require-resource-limits.yaml

コンテナにCPUとメモリのrequestsとlimitsが設定されていることを検証します。

**ポリシー内容**:
- `validate-resources`: 全てのコンテナにリソース制限が定義されているか確認
  - CPU requests/limits
  - Memory requests/limits

**対象リソース**: Pod, Deployment, StatefulSet, DaemonSet

**動作モード**: Audit（検知のみ、ブロックしない）

### 5. require-security-context.yaml

コンテナが適切なセキュリティコンテキストで実行されることを検証します。

**ポリシー内容**:
- `check-run-as-non-root`: コンテナがroot以外のユーザーで実行されることを確認
- `check-privilege-escalation`: 権限昇格が無効化されていることを確認
- `drop-all-capabilities`: 全てのLinux capabilitiesがドロップされていることを確認

**対象リソース**: Pod, Deployment, StatefulSet, DaemonSet

**動作モード**: Audit（検知のみ、ブロックしない）

**セキュリティ**: High - セキュリティベストプラクティスの遵守

### 6. require-probes.yaml

コンテナにliveness probeとreadiness probeが設定されていることを検証します。

**ポリシー内容**:
- `validate-readiness-probe`: readiness probeが定義されているか確認
- `validate-liveness-probe`: liveness probeが定義されているか確認

**対象リソース**: Pod, Deployment, StatefulSet, DaemonSet

**動作モード**: Audit（検知のみ、ブロックしない）

### 7. require-image-pull-policy.yaml

コンテナイメージのpullポリシーが適切に設定されていることを検証します。

**ポリシー内容**:
- `validate-image-pull-policy`: imagePullPolicyが`Always`または`IfNotPresent`であることを確認
  - `Never`の使用を禁止

**対象リソース**: Pod, Deployment, StatefulSet, DaemonSet

**動作モード**: Audit（検知のみ、ブロックしない）

### 8. require-hpa.yaml

HPA（HorizontalPodAutoscaler）リソースが適切に設定されていることを検証します。

**ポリシー内容**:
- `validate-hpa-configuration`: HPAリソースが適切に設定されていることを確認
  - `scaleTargetRef`: Deploymentへの参照が正しく設定されているか
  - `minReplicas`: 1以上であること
  - `maxReplicas`: 2以上であり、minReplicasより大きいこと

**対象リソース**: HorizontalPodAutoscaler

**動作モード**: Audit（検知のみ、ブロックしない）

### 9. check-hpa-exists.yaml

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