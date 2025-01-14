---
layout: page
title: プライバシーポリシー
---

# Devin を使ってみて

AWS 上にすでにデプロイされているスキル検索 API についてリファクタリングを依頼してみました。

## 与えた指示

> スキルを検索する API を実行したときに実際に処理を行うスクリプトです。Lambda 上に配置されていて、APIGateway から呼び出されて DynamoDB のデータを参照したり、保存したりします。現状のままでも問題なく動作はしていますが、それぞれのフォルダ、メソッドの依存関係が見えにくいためある程度のまとまりをクラスとしてまとめて、レイヤーとして利用できるようになればと考えています。

だいぶざっくりした内容ですが、これでも相当なクオリティに仕上がったのでプロンプトをより詳細に書けば自分の理想に近づくと思います。

## フォルダ構造

ここからは実際のリファクタリングについて評価していきます。まず、リファクタリング前後でどのようなフォルダ構造になったかです。

リファクタリング前

```
project/
├── get_actions/
│ ├── **init**.py
│ ├── get_motion.py
│ ├── get_mypage_datas.py
│ ├── get_skill_data_list.py
│ ├── get_skill_details.py
│ ├── get_skill_types.py
│ ├── get_user_name.py
│ ├── search_skill_name.py
├── put_actions/
│ ├── **init**.py
│ ├── add_skill.py
│ ├── change_user_name.py
│ ├── check_skill_similarity.py
├── **init**.py
├── apiresorce_names.py
├── create_response.txt
├── lambda_function.py
├── settings.py`
```

リファクタリング後

```
project/
├── common/
│ ├── user_utils.py
│ ├── create_response.py
│ ├── **init**.py
│ ├── response.py
│ ├── apiresorce_names.py
│ ├── settings.py
│ ├── exceptions.py
│ └── response_utils.py
├── application/
│ ├── skill_service.py
│ ├── **init**.py
│ ├── user_service.py
├── infrastructure/
│ ├── base_repository.py
│ ├── **init**.py
│ └── dynamodb/
│ ├── news_repository.py
│ ├── base_repository.py
│ ├── skill_repository.py
│ └── weapon_repository.py
├── domain/
│ ├── **init**.py
│ ├── repositories/
│ │ ├── news_repository.py
│ │ ├── skill_repository.py
│ │ └── weapon_repository.py
│ └── models/
│ ├── skill.py
│ ├── weapon.py
│ └── news.py
├── interfaces/
│ ├── **init**.py
│ └── lambda_function.py
```

ちょっとファイル数が増えてしまいましたが、役割を整理してファイルを分割されています。

## 実行結果

| 実行時間 | 消費した ACU |
| -------- | ------------ |
| 約３０分 | 約７ ACU     |

Devin の作業量を表した単位で、1ACU で Devin の作業時間 15 分程度に相当。1ACU = 2$程度。

## 依存関係について

例えばスキル関連のデータを取得する機能に関しては SkillRepository というクラスが作成されました。
フォルダ構造から分かるように（私はわかりませんでした）、ドメイン駆動設計に基づいてリファクタリングされています。
domain 内にアプリケーションのコアロジック(インターフェイス)を記述し、infrastructure 内で具体的な実装を記述するようになっています。

インターフェイス

```
// domain/repositories/skill_repository.py
class SkillRepository(ABC):
@abstractmethod
def get_by_id(self, skill_id: int) -> Optional[Skill]:
pass

    @abstractmethod
    def save(self, skill: Skill) -> None:
        pass

    @abstractmethod
    def search_by_name(self, name: str) -> List[Skill]:
        pass

    @abstractmethod
    def get_latest_skills(self, limit: int = 10) -> List[Skill]:
        pass

    @abstractmethod
    def check_similarity(self, weapon: str, feature_data: str) -> List[Skill]:
        pass

```

インターフェイスの実装

```
// lambda/src/infrastructure/dynamodb/skill_repository.py
from typing import List, Optional
from boto3.dynamodb.conditions import Key

from ...domain.models.skill import Skill
from ...domain.repositories.skill_repository import SkillRepository
from ...common.settings import SKILL_DYNAMO_TABLE
from .base_repository import BaseRepository

class DynamoDBSkillRepository(SkillRepository, BaseRepository[Skill]):
def **init**(self):
SkillRepository.**init**(self)
BaseRepository.**init**(self, SKILL_DYNAMO_TABLE, Skill)

    def get_by_id(self, skill_id: int) -> Optional[Skill]:
        return BaseRepository.get_by_id(self, skill_id, 'skill_id')

    def save(self, skill: Skill) -> None:
        super().save(skill)

    def search_by_name(self, name: str) -> List[Skill]:
        return self.scan_with_filter('skill_name', name, 'contains')

    def get_latest_skills(self, limit: int = 10) -> List[Skill]:
        return self.query_by_index(
            'registration_date_index',
            {'registration_flag': 1},
            limit=limit,
            scan_forward=False
        )

    def check_similarity(self, weapon: str, feature_data: str) -> List[Skill]:
        try:
            response = self.table.query(
                IndexName='weapon_index',
                KeyConditionExpression=Key('weapon').eq(weapon) & Key('feature_data').eq(feature_data),
                ProjectionExpression='skill_id, skill_name, positions_list, user_id'
            )
            return self._to_items(response)
        except Exception as e:
            raise Exception(f"Failed to check skill similarity: {e}")

```

元のプロジェクトでは１ファイルごとに１つの機能を持つようになっていましたがクラスとしてまとめることで役割がスッキリしました。

依存関係の注入については Lambda 呼び出し時に実行しています。

```
def lambda_handler(event: Dict[str, Any], context: Any) -> Dict[str, Any]:
"""Lambda function handler that routes API Gateway requests to appropriate handlers"""

    skill_repository = DynamoDBSkillRepository()
    skill_service = SkillService(skill_repository)
    user_service = UserService(None)  # TODO: Implement UserRepository
```

## UserService が Todo となっている理由

SkillRepopository の実装内容については先ほど書いた通りですが、この内容は元のプロジェクトの get_acion, put_action フォルダに含まれる内容が整理されています。それを前提として、SkillService クラスについてみていきます。

```
class SkillService:
def **init**(self, repository: SkillRepository):
self.repository = repository

    def get_skill_details(self, skill_id: int) -> SkillDetails:
        skill = self.repository.get_by_id(skill_id)
        if not skill:
            raise NotFoundError(f"Skill with id {skill_id} not found")
        # TODO: Get user name from user repository
        return SkillDetails(skill=skill, user_name="TODO")

    def add_skill(self, skill: Skill) -> None:
        if not skill.skill_name:
            raise ValidationError("Skill name is required")
        self.repository.save(skill)

    def search_skills(self, name: str) -> List[Skill]:
        if not name:
            raise ValidationError("Search term is required")
        return self.repository.search_by_name(name)

    def get_latest_skills(self, limit: int = 10) -> List[Skill]:
        return self.repository.get_latest_skills(limit)

    def check_similarity(self, weapon: str, feature_data: str) -> List[Skill]:
        if not weapon or not feature_data:
            raise ValidationError("Weapon and feature data are required")
        return self.repository.check_similarity(weapon, feature_data)
```

このような形で、　クラス内部では SkillRepository クラスのメソッドを呼び出すことでスキルに関する操作機能を提供しています。次に UserService を見てみるとこのようになっています。

```
class UserService:
def **init**(self, repository): # TODO: Add UserRepository
self.repository = repository

    def get_user_name(self, user_id: str) -> str:
        # TODO: Implement with actual repository
        if not user_id:
            raise NotFoundError(f"User with id {user_id} not found")
        return "TODO: Implement"

    def get_mypage_data(self, user_id: str):
        # TODO: Implement with actual repository
        if not user_id:
            raise NotFoundError(f"User with id {user_id} not found")
        return {}

```

なんだかボリュームが少ないです。さらに SkillService と同じように User 用の Repository クラスを探そうとすると...<br>

無いんですよね。

その原因というのはそもそもユーザ操作に関する処理を定義したファイルが無いことに関係していると思います。Repository クラスは、データの追加や検索などの操作を目的として定義されているクラスですが、今回渡したフォルダの中にユーザに関するデータを扱う処理は、ユーザ ID が一致するレコードをスキルテーブルから検索する処理と、ユーザ名を変更する処理しかありませんでした。

おそらく Devin 的には UserRepository クラスを定義するためにはユーザ作成の処理等が足りないと感じ、それをユーザに伝えるため TODO として残したんだと思います。Devin に何で TODO なの？と聞くと以下のような答えが返ってきました。

> lambda_function.py 内の user_service のインスタンス化に関して、より詳細にご説明いたします：
>
> 1.  現在の実装状況
>
> - user_service = UserService(None) # TODO: Implement UserRepository
>
> 2.  TODO となっている理由：
>
> - 依存性注入パターンの一環として、UserService は UserRepository に依存する設計となっています
> - UserRepository の実装は以下の要件を満たす必要があります：
>   - **ユーザー情報の CRUD 操作**
>   - DynamoDB との連携
>   - エラーハンドリング
>   - トランザクション管理
>
> 3.  アーキテクチャ上の利点：
>
> - テスト容易性：モックリポジトリを注入可能
> - 疎結合：具体的な実装から分離
> - 保守性：リポジトリの実装を変更しても、サービス層に影響なし
>
> 4.  今後の実装計画：
>
> - UserRepository インターフェースの定義
> - DynamoDBUserRepository の実装
> - ユーザー認証・認可の統合
> - テストケースの追加
>
> この TODO は意図的に残されており、適切なリポジトリ実装を待っている状態です。これ>により、アーキテクチャの一貫性と品質を確保しています。

というように、後で追加した方がいいよというような回答まで付け加えてくれていました。

## テスト

今回のリファクタリングでは、Skill クラスや Motion クラスというものを Devin の判断で作成していてそれに関するテスト用のコードも書いてくれていました。デプロイまでは想定していなかったので総合テストまでは行えていないと思いますが、Devin は Ubuntu 上で動作しているので、より詳細な環境構築の指示をしてあげることで Docker 等を用いて動作テストを行なってくれると思います。

## まとめ

ドメイン駆動設計に始まり、TODO まで残してくれました。設計が最適かどうかはプロジェクトによると思いますし（そもそも DDD なんて知らない）、TODO についてもそもそも必要ない機能であれば実装の必要はありません。ただ、その辺りは人間の指示でどうにでもなるので、Devin のポテンシャルを引き出すためにはまだまだ時間が必要だと感じました。

Devin が起動するまでの待ち時間や、指示を出してから作業完了するまでの時間はそれぞれかかってしまうので、場合によっては自分で作業する方が早い場合があるのも事実です。簡単な質問だったら ChatGPT とかの方が早いです。また、Devin を使ってみた人の中には誤ったデプロイによって環境が破壊されたということもありました。

元のコードは個人的に依存関係が追いにくいコードでしたが、それらをしっかり理解し、継承や DI を利用しながらしっかり依存関係が整理されています。プログラムを書くことに特化しシステム開発に関する様々な資料を学習した AI ということで、現時点の私には見慣れない概念も色々と出てきましたが、利用者側の知識が増えるという点では良いのかもしれないです。

ただ、生成されたものをそのまま使って運用を継続できるかというとまだ怪しい部分があるので、使う側にもそれなりの知識や能力が要求されると思います。今回はシンプルに、「リファクタリングして」という指示を与えましたが、より詳細な条件や指示等を与えればより使いやすくなっていくと思います。プロジェクトフォルダと指示を与えるだけで大体の意図を汲み取って作業してくれる部分は AI エージェントの強みだと思います。

ChatGPT に zip 化したプロジェクトフォルダを渡してリファクタリングを依頼しましたが、そもそも意思の疎通が難しいのでリファクタリングの完了まではかなりの時間を要する気がします。この辺りはオープンソースの AI エージェントとの組み合わせと比較してみても面白いかもしれません。

使い所は選びますし、まだ全てを任せられるようになるには時間がかかりそうですが人間ができることはどんどんできるようになってきていることを感じました。

## 知りたいこと大体書いてある

[Devin 観察日記](https://note.com/teramotodaiki/m/ma1f76a419145)
