---
title: "【SQS/Lambda】SQS + Lambda での部分バッチ応答を試してみる"
emoji: "💿"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","Lambda","SQS"]
published: true
---

本記事は AWS LambdaとServerless Advent Calendar 2021 22 日目の記事です。
https://qiita.com/advent-calendar/2021/lambda

## はじめに

先日、SQSをイベントソースとしたLambdaで部分的なバッチ応答が可能になったことが発表されました。
https://aws.amazon.com/about-aws/whats-new/2021/11/aws-lambda-partial-batch-response-sqs-event-source/?nc1=h_ls
これが出来なくて悩んだ経験があった自分としては、とても嬉しいアップデートです！
部分バッチ応答が可能になったことで、複数メッセージ処理時に、失敗したメッセージのみをキューに再表示できるようになりました。(元々はLambda側で自前実装が必要でした)

これは試さずにはいられないということで、動作確認をしてみました。
リソースの作成にはCDKを用い、Lambda関数はGoで書きました。

## 環境

- macOS 12.0.1
- cdk 2.2.0
- go 1.17.3

## リソース作成

今回使用するリソースは、

- SQSキュー(イベントソース用キュー、デッドレターキュー)
- Lambda関数

となります。
ディレクトリ構成は下記の`tree`コマンドの出力の通りで、今回は一つのスタックに全てのリソースを詰め込んでいます。
`cli-input/send-message-batch.json`は後述のサンプルメッセージ送信に用います。

```sh
% tree -I "node_modules|cdk.out"
.
├── README.md
├── bin
│   └── cdk-batch-failure.ts
├── cdk.json
├── cli-input
│   ├── README.md
│   └── send-message-batch.json
├── jest.config.js
├── lambda
│   └── go
│       ├── go.mod
│       ├── go.sum
│       └── main.go
├── lib
│   └── batch-stack.ts
├── package.json
├── test
│   └── cdk-batch-failure.test.ts
├── tsconfig.json
└── yarn.lock

6 directories, 14 files
```

下記が今回作成するスタックです。

```typescript:batch-stack.ts
import { Stack, StackProps, Duration } from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as lambda from '@aws-cdk/aws-lambda-go-alpha';
import { SqsEventSource } from 'aws-cdk-lib/aws-lambda-event-sources';

export class CdkBatchFailureStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const deadLetterQueue = new sqs.Queue(this, "deadLetterQueue", {
      queueName: "deadLetterQueue",
    })

    // 一度でもLambda関数内で処理に失敗したメッセージはデッドレターキューに移動
    const queue = new sqs.Queue(this, "queue", {
      queueName: "queue",
      deadLetterQueue: {
        queue: deadLetterQueue,
        maxReceiveCount: 1,
      }
    })

    const source = new SqsEventSource(queue, {
      reportBatchItemFailures: true, // 部分バッチ応答
    })
    new lambda.GoFunction(this, 'RandomResult', {
      entry: 'lambda/go',
      events: [source],
    })
  }
}
```

イベントソース作成時のパラメータ`reportBatchItemFailures`をtrueにすることで、部分バッチ応答を有効にしています。
また、一度でもLambda関数内で処理に失敗した(キューに再表示された)メッセージはデッドレターキューに移動するようにしています。

## 関数作成

下記が今回作成するLambda関数です。

```go:lambda/go/main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"math/rand"
	"time"

	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
)

type SqsBatchResponse struct {
	BatchItemFailures []BatchItemFailure `json:"batchItemFailures"`
}

type BatchItemFailure struct {
	ItemIdentifier string `json:"itemIdentifier"`
}

func handler(ctx context.Context, sqsEvent events.SQSEvent) (res SqsBatchResponse, err error) {
	fmt.Printf("Received %d records\n", len(sqsEvent.Records))

	rand.Seed(time.Now().Unix())
	var randInt int
	if len(sqsEvent.Records) > 1 {
		randInt = rand.Intn(len(sqsEvent.Records))
	}

	for i, message := range sqsEvent.Records {
		fmt.Printf("The message %s for event source %s = %s \n", message.MessageId, message.EventSource, message.Body)

		if len(sqsEvent.Records) > 1 && i == randInt {
			fmt.Printf("Failuer message %s for event source %s = %s \n", message.MessageId, message.EventSource, message.Body)
			res.BatchItemFailures = append(res.BatchItemFailures, BatchItemFailure{message.MessageId})
		}
	}

	// デバッグ用に出力
	b, err := json.Marshal(res)
	if err != nil {
		return
	}
	fmt.Println(string(b))

	return
}

func main() {
	lambda.Start(handler)
}
```

動作確認をやりやすくする為、複数件のメッセージがバッチ処理された場合にランダムで一件のみ失敗させるようにします。
handler関数のレスポンスの型は、[ドキュメント](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)を参考に独自で宣言しています。

## 動作確認

実際にキューにサンプルメッセージを送信してみて、関数を実行させます。
メッセージ送信はCLIでさくっとやります。

```sh
% aws sqs send-message-batch --queue-url https://sqs.ap-northeast-1.amazonaws.com/${ACCOUNT_ID}/queue --entries file://cli-input/send-message-batch.json
```

```json:cli-input/send-message-batch.json
[
    {
        "Id": "1",
        "MessageBody": "1"
    },
    {
        "Id": "2",
        "MessageBody": "2"
    }
]
```

Lambdaのログを確認してみます。

![](https://storage.googleapis.com/zenn-user-upload/7160c863777a-20211218.png)

送信した二件のメッセージがバッチ処理され、そのうち一件が失敗メッセージとして返却されています。(複数メッセージがキュー内にあっても、必ずバッチ処理してくれるとは限らないので注意)
続いて同じIDのメッセージがデッドレターキューの方にエンキューされていることを確認します。

![](https://storage.googleapis.com/zenn-user-upload/b7d9d2545f85-20211218.png)

想定通り、失敗メッセージがデッドレターキューに入っていました🎊

## さいごに

SQS + Lambda は個人的に大好きな組み合わせなので、今回のアップデートはとても嬉しいです。(二回目)
AWS はかなりハイペースで各サービスの機能の追加や拡張が成されますので、キャッチアップを怠らず日々のエンジニア活動に勤しみたい所存です。

また、今回のCDKのソースコードはGitHubでも公開しています。
御参考になれば幸いです。
https://github.com/s-beats/cdk-batch-failure

## 参考

https://aws.amazon.com/about-aws/whats-new/2021/11/aws-lambda-partial-batch-response-sqs-event-source/?nc1=h_ls
https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html
https://intro-to-cdk.workshop.aws/the-workshop.html
https://dev.classmethod.jp/articles/sqs-lambda/
