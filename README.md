# JUL and MDC example on EAP 8

Java Util Logging (JUL) でも Message Diagnostic Context (MDC) を使えることを示すサンプルプロジェクト。

EAPのクイックスタートの "helloworld" をベースにしている。

- https://github.com/jboss-developer/jboss-eap-quickstarts/tree/8.1.x/helloworld

バージョン8.1のブランチから取得しているが8.0でも動作する。
一旦サンプルに不必要な部分を削除し、その上で変更箇所には "CHANGED" のコメントを入れている。
以下のコマンドでどんな変更を加えたかを一覧できる。

    grep -r -A5 CHANGED

## 使い方

```
# ビルド
$ mvn clean package

# デプロイ（localhostでEAPをあらかじめ起動しておく）
$ mvn wildfly:deploy

# 動作確認
$ curl localhost:8080/helloworld/HelloWorld
<html><head><title>helloworld</title></head><body>
<h1>Hello World!</h1>
</body></html>
```

呼び出しのたびに以下のようなログが出力される。

```
025-11-14 14:46:36,889 INFO  [org.jboss.as.quickstarts.helloworld.HelloWorldServlet] (default task-1) ### JUL Logger called ###
```

参考までに、JSONフォーマッターの場合は以下のようになる。

```
{"timestamp":"2025-11-14T16:25:11.691+09:00","sequence":52,"loggerClassName":"org.jboss.logmanager.Logger","loggerName":"org.jboss.as.quickstarts.helloworld.HelloWorldServlet","level":"INFO","message":"### JUL Logger called ###","threadName":"default task-1","threadId":171,"mdc":{"test":"1763105111691"},"ndc":"","hostName":"p1gen4i","processName":"jboss-modules.jar","processId":3577515}
```

## ログ設定のカスタマイズ

以下のような仕様でログの出力方法を変更する。

- カテゴリ "org.jboss.as.quickstarts.helloworld" を対象にする
- デフォルトの出力先（server.logとコンソールの標準出力）には出さない
- 二つファイルにログ出力する
  - 両者で出力内容は異なる（公開・非公開の使い分けを想定）
- "test" というキーで設定したMDCの値 (`%X{test}`) をログに含める

standalone.xmlのロギングサブシステムに以下の内容をコピーしてEAPを再起動する。

```
        <subsystem xmlns="urn:jboss:domain:logging:8.0">
            (...)

            <formatter name="MY-PATTERN-1">
                <pattern-formatter pattern="%d{yyyy-MM-dd HH:mm:ss,SSS} %-5p [%c] (%t) %s%e%n"/>
            </formatter>
            <formatter name="MY-PATTERN-2">
                <pattern-formatter pattern="%d{yyyy-MM-dd HH:mm:ss,SSS} %-5p [%c] (%t) %X{test}%n"/>
            </formatter>
            <formatter name="my-json-formatter">
                <json-formatter/>
            </formatter>
            <periodic-rotating-file-handler name="MY-FILE-1" autoflush="true">
                <formatter>
                    <named-formatter name="MY-PATTERN-1"/>
                </formatter>
                <file relative-to="jboss.server.log.dir" path="my-file-1.log"/>
                <suffix value=".yyyy-MM-dd"/>
                <append value="true"/>
            </periodic-rotating-file-handler>
            <periodic-rotating-file-handler name="MY-FILE-2" autoflush="true">
                <formatter>
                    <named-formatter name="MY-PATTERN-2"/>
                    <!-- <named-formatter name="my-json-formatter"/> -->
                </formatter>
                <file relative-to="jboss.server.log.dir" path="my-file-2.log"/>
                <suffix value=".yyyy-MM-dd"/>
                <append value="true"/>
            </periodic-rotating-file-handler>
            <logger category="org.jboss.as.quickstarts.helloworld" use-parent-handlers="false">
                <level name="INFO"/>
                <handlers>
                    <handler name="MY-FILE-1"/>
                    <handler name="MY-FILE-2"/>
                </handlers>
            </logger>
        </subsystem>
```

上記の設定ではserver.logと同じ $JBOSS_HOME/standalone/log/ に my-file-1.log と my-file-2.log で出力される。
（絶対パスで指定したい場合は relative-path を削除し path に絶対パスを書けばよい。）

フォーマッターを変えているので内容は異なっており、片方にはMDCで設定した値が取れていることを確認できる。

フォーマッターに使える書式は以下で確認できる。

- https://docs.redhat.com/ja/documentation/red_hat_jboss_enterprise_application_platform/8.0/html/configuration_guide/log_formatter_attributes

## ログハンドラーやフォーマッターをカスタマイズする場合

デフォルトで用意されている <periodic-rotating-file-handler> などで十分なはずで、
フォーマッターについてもMDCで任意の項目を出力可能になるし、JSONやXMLのフォーマッターであればデフォルトで用意されている。
しかし例えばリクエストやジョブごとにファイルを変えたいといった特殊な要件には対処できない。

そうした特殊な要件が必要な場合にはじめてカスタム実装を検討する。
流れとしては `java.util.logging.Handler` や `java.util.logging.Formatter` を継承したクラスを実装し、モジュールとしてパッケージして $JBOSS_HOME/modules/ 以下に配置する。
そしてそのクラス名とモジュール名を指定して <custom-handler> や <custome-formatter> をstandalone.xmlに設定する。

- デフォルトで用意されているハンドラー
  - https://docs.redhat.com/ja/documentation/red_hat_jboss_enterprise_application_platform/8.0/html-single/configuration_guide/index#configuring-log-handlers_logging-with-jboss-eap
  - カスタムログハンドラーの設定方法
    - https://docs.redhat.com/ja/documentation/red_hat_jboss_enterprise_application_platform/8.0/html-single/configuration_guide/index#configuring-a-custom-log-handler_configuring-log-handlers
- デフォルトで用意されているフォーマッター
  - https://docs.redhat.com/ja/documentation/red_hat_jboss_enterprise_application_platform/8.0/html-single/configuration_guide/index#configuring-log-formatters_logging-with-jboss-eap
  - カスタムログフォーマッターの設定方法
    - https://docs.redhat.com/ja/documentation/red_hat_jboss_enterprise_application_platform/8.0/html-single/configuration_guide/index#configuring-a-custom-log-formatter_configuring-log-formatters

