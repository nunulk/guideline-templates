# PHP_CodeSniffer 利用ガイド

## このドキュメントについて

バックエンドにおけるコード整形の方針ややり方について記します。

## このドキュメントが必要な理由

本プロジェクトではコーディング標準を定めています。チームでコンセンサスを取りつつ最大限活用するために、ルール除外と実行に関するガイドラインが必要です。

## ルールの除外

ルールについては CakePHP 標準のルールで運用していますが、すでに決定している除外ルールは以下のとおりです。今後増やす場合は以下の場所に除外したいルールを追記してください。

```xml
<rule ref="CakePHP">
    <!-- 除外するルールを以下に記述（除外理由を記載してください） -->
    <!-- use しているクラスに関しては FQCN でなくてもいいため -->
    <exclude name="SlevomatCodingStandard.Namespaces.FullyQualifiedClassNameInAnnotation" />
    <!-- 既存のコードにスネークケースの変数が残っているため -->
    <exclude name="Zend.NamingConventions.ValidVariableName.StringVarNotCamelCaps" />
</rule>
```

1. 原則的にルールは除外しないようにしてください
2. チームで協議の上、コード整形のコストがベネフィットを上回ると判断した場合は除外しても構いませんが、除外する理由を記載するようにしてください

## CI での実行

プルリクエスト作成時に `composer run cs-check` を実行しています。

composer.json での定義は以下のようになっています。

```bash
phpcs --standard=phpcs.xml --colors -p src tests/Factory tests/TestCase
```

1. チェックでエラーが出た場合、マージできませんので、エラーを修正してください

## ローカルでの実行

### lefthook の利用

git の pre-commit フックを利用して、コミット前に自動で実行することができます。

https://github.com/evilmartians/lefthook

1. ルール違反があれば CI でエラーになるようになってはいますが、プッシュしてから気づくと修正が面倒なので、ぜひこちらをご利用ください
