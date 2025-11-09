# Kyverno Config Repository

このリポジトリはKyvernoポリシーを管理し、CI/CDパイプラインで自動検証を行うためのものです。

## 📁 ディレクトリ構造

```
.
├── policies/                    # Kyvernoポリシーファイル
│   ├── require-labels.yaml
│   └── disallow-latest-tag.yaml
├── resources/
│   └── test-resources/         # テスト用Kubernetesリソース
│       ├── good-pod.yaml
│       ├── bad-pod-no-label.yaml
│       └── bad-pod-latest-tag.yaml
├── kyverno-test.yaml           # Kyverno testコマンド用テスト定義
└── .github/
    └── workflows/
        └── kyverno-test.yml    # GitHub Actions CI/CDワークフロー
```

## 🚀 GitHub Actions CI/CD

このリポジトリでは、GitHub Actionsを使用してKyverno CLIによる自動検証を実行しています。

### ワークフローの内容

1. **ポリシーテスト**: `kyverno apply` コマンドでポリシーをテストリソースに適用
2. **テストケース実行**: `kyverno test` コマンドで定義されたテストケースを実行

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
# ポリシーをリソースに適用してテスト
kyverno apply policies/ --resource resources/test-resources/

# 特定のポリシーとリソースでテスト
kyverno apply policies/require-labels.yaml --resource resources/test-resources/good-pod.yaml
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