# Kyverno Config Repository

このリポジトリはKyvernoポリシーを管理し、CI/CDパイプラインで自動検証を行うためのものです。

## 📁 ディレクトリ構造

```
.
├── policies/                    # Kyvernoポリシーファイル
│   ├── require-labels.yaml
│   ├── disallow-latest-tag.yaml
│   └── require-hpa.yaml
├── resources/
│   └── test-resources/         # テスト用Kubernetesリソース
│       ├── good-pod.yaml
│       ├── good-deployment.yaml
│       └── good-hpa.yaml
├── kyverno-test.yaml           # Kyverno testコマンド用テスト定義
├── values.yaml                 # apiCallモック用のコンテキスト変数定義
└── .github/
    └── workflows/
        └── kyverno-test.yml    # GitHub Actions CI/CDワークフロー
```

## 🚀 GitHub Actions CI/CD

このリポジトリでは、GitHub Actionsを使用してKyverno CLIによる自動検証を実行しています。

### ワークフローの内容

1. **ポリシーテスト**: `kyverno apply` コマンドで全ポリシーをテストリソースに適用
   - `values.yaml`を使用してapiCallの結果をモック
   - `--values-file`オプションでコンテキスト変数を提供
   - 全てのポリシー（HPA検証を含む）をテスト可能
2. **テストケース実行**: `kyverno test` コマンドで定義されたテストケースを実行
   - 構造化されたテストケースで期待結果を検証
   - `kyverno-test.yaml`内のvaluesフィールドでもapiCallをモック

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
# ポリシーをリソースに適用してテスト（values.yamlでapiCallをモック）
kyverno apply policies/ --resource resources/test-resources/ --values-file values.yaml

# valuesファイルなしで基本ポリシーのみテスト
kyverno apply policies/require-labels.yaml policies/disallow-latest-tag.yaml \
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

Deployment/StatefulSetに対応するHPA（HorizontalPodAutoscaler）が実際に存在することを検証します。

**含まれるポリシー**:
- `check-hpa-exists`: API経由で同じnamespace内のHPAを取得し、Deployment/StatefulSetに対応するHPAが存在するかを確認
  - contextとapiCallを使用して実際のクラスター状態を参照
  - HPAのscaleTargetRef.nameにリソース名が含まれているかをチェック
  - テスト時は`values`フィールドでapiCallの結果をモック
- `validate-hpa-configuration`: HPAリソースが適切に設定されていることを確認（scaleTargetRef、minReplicas、maxReplicasの検証）

**対象リソース**: Deployment, StatefulSet, HorizontalPodAutoscaler

**動作モード**: Audit（検知のみ、ブロックしない）

**テスト時のapiCallモック**:

`values.yaml`ファイルでapiCallの結果をモック：
```yaml
policies:
  - name: check-hpa-exists
    rules:
      - name: validate-hpa-exists
        values:
          hpas:
            - myapp-deployment  # HPAのscaleTargetRef.nameリストをモック
```

`kyverno-test.yaml`内でもモック可能：
```yaml
results:
  - policy: check-hpa-exists
    rule: validate-hpa-exists
    resource: myapp-deployment
    kind: Deployment
    result: pass
    values:
      hpas:
        - myapp-deployment
```

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