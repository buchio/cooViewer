# Apple Silicon (arm64) ビルド手順

cooViewer を Apple Silicon Mac 向けにビルドする手順です。

## 前提条件

- macOS (Apple Silicon Mac)
- Xcode (フルバージョン、App Storeからインストール)
- Homebrew (オプション、The Unarchiverのインストールに使用)

## 1. Xcode のセットアップ

```bash
# Xcode のパスを設定
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer

# ライセンスに同意
sudo xcodebuild -license accept

# 初回セットアップ
sudo xcodebuild -runFirstLaunch
```

## 2. フレームワークの更新

プロジェクトに含まれる `XADMaster.framework` と `UniversalDetector.framework` は x86_64 のみのため、ユニバーサルバイナリ版に更新する必要があります。

### The Unarchiver からフレームワークを取得

```bash
# The Unarchiver をインストール
brew install --cask the-unarchiver

# 既存のフレームワークをバックアップ
mv XADMaster.framework XADMaster.framework.bak
mv UniversalDetector.framework UniversalDetector.framework.bak

# 古いフレームワーク構造を復元（ヘッダーファイルを保持するため）
cp -R XADMaster.framework.bak XADMaster.framework
cp -R UniversalDetector.framework.bak UniversalDetector.framework

# バイナリのみユニバーサル版に置き換え
cp "/Applications/The Unarchiver.app/Contents/Frameworks/XADMaster.framework/Versions/A/XADMaster" \
   XADMaster.framework/XADMaster
cp "/Applications/The Unarchiver.app/Contents/Frameworks/UniversalDetector.framework/Versions/A/UniversalDetector" \
   UniversalDetector.framework/UniversalDetector
```

### アーキテクチャの確認

```bash
file XADMaster.framework/XADMaster
# 出力例: Mach-O universal binary with 2 architectures: [x86_64] [arm64]
```

## 3. ビルド

```bash
xcodebuild -project cooViewer.xcodeproj \
  -scheme cooViewer_deploy \
  -configuration Deployment \
  -arch arm64 \
  -derivedDataPath ./build \
  clean build
```

ビルド成功後、アプリは `build/Deployment/cooViewer.app` に生成されます。

## 4. フレームワークの埋め込みと署名

```bash
# フレームワークをアプリバンドルにコピー
mkdir -p build/Deployment/cooViewer.app/Contents/Frameworks
cp -R XADMaster.framework build/Deployment/cooViewer.app/Contents/Frameworks/
cp -R UniversalDetector.framework build/Deployment/cooViewer.app/Contents/Frameworks/

# rpath を追加
install_name_tool -add_rpath @executable_path/../Frameworks \
  build/Deployment/cooViewer.app/Contents/MacOS/cooViewer

# アドホック署名
codesign --force --deep --sign - build/Deployment/cooViewer.app
```

## 5. 動作確認

```bash
open build/Deployment/cooViewer.app
```

## 配布用パッケージの作成

```bash
cd build/Deployment
zip -r cooViewer-VERSION-arm64.zip cooViewer.app
```

## トラブルシューティング

### `Library not loaded: @rpath/XADMaster.framework/...`

フレームワークがアプリバンドルに埋め込まれていないか、rpath が設定されていません。手順4を実行してください。

### `code signature invalid`

コード署名が無効です。`codesign --force --deep --sign -` でアドホック署名を行ってください。

### `MACOSX_DEPLOYMENT_TARGET` の警告

プロジェクト設定で古い macOS バージョン (10.8) がターゲットになっています。動作には影響しませんが、必要に応じて Xcode でデプロイメントターゲットを 10.13 以上に変更してください。
