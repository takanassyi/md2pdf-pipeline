[Qiita記事へのリンク](https://qiita.com/takanassyi/items/f933cdadebae7ddf88bc)


AWS が提供するサービスを組み合わせて、Git で管理された Markdown を PDF に一括変換する CI/CD パイプラインを構築した。

# パイプライン構成図

![md2pdf-pipeline.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/e08b5a13-358c-7bb6-f244-20c994e4c840.png)


# 処理フロー

1. Markdown を含むファイルを Git で CodeCommit のリポジトリにプッシュ
2. CodeBuild で CodeCommit から Markdown を含むソース一式を取得
3. ECR にある Pandoc の Docker イメージを利用して Markdown から PDF に変換
4. PDFをまとめて zip に圧縮し S3 へ転送

# パイプライン構築手順

**【注意】各種サービスのリージョンは同一リージョンに揃える必要がある**

## S3

生成したPDFを格納するS3バケットを構築する。特に注意点はなし。

## ECR

Pandoc の Docker イメージをプッシュするためのレジストリを予め作成しておき、Docker イメージをプッシュする。

### Pandoc Docker イメージ 

k1low さん作成の Docker イメージを利用させていただきました。[^1]

[^1]: 参考資料 k1low/alpine-pandoc-ja | Docker Hub を参照


```dockerfile
FROM k1low/alpine-pandoc-ja
```

上記 Dockerfile をビルドしたイメージを ECR にプッシュする

### アクセス許可

CodeBuild から ECR の Docker イメージをプルできるようにアクセス許可を与える必要がある。
Pandoc が格納されているレジストリの Permissions を選択し、ポリシー JSON を下記のように編集する

```json
{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Sid": "CodeBuildAccess",
      "Effect": "Allow",
      "Principal": {
        "Service": "codebuild.amazonaws.com"
      },
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ]
    }
  ]
}
```


![Screen Shot 2020-04-16 at 20.46.35.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/0c3f0bd3-757a-8ee3-3b0a-cf1432e650d1.png)



## CodeCommit

Git リポジトリを作成する。

### ディレクトリ構成

```
.
├── buildspec.yml
└── src
    ├── foo.md
    └── bar.md
```

- CodeCommit 上のリポジトリに必要なファイル及びディレクトリ
  - buildspec.yml : Codebuild で必要な YAMLファイル。次節で解説。
  - src : PDF に変換したい Markdown 一式を格納するディレクトリ。



## CodeBuild

CodeCommit からソース一式と ECR からPandoc の Docker イメージを持ってきて、Markdown を PDF に変換する。

### ビルドプロジェクト作成

- 任意のプロジェクト名を入力
- ソースプロバイダは AWS CodeCommit を選択し、リポジトリは Markdownを含むリポジトリを選択

![Screen Shot 2020-04-16 at 20.17.44.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/57ba90ac-3081-0e17-a44d-7179ddd1a18c.png)

- 環境イメージはECRにプッシュしたPandocのDocker イメージを利用するため、カスタムイメージを選択
- 本稿のDockerfileを利用する場合、下記のように設定
    - 環境タイプは Linux
    - イメージレジストリは Amazon ECR
    - ECR は自分の ECR アカウント
    - レポジトリは Pandoc の Docker イメージをプッシュしたリポジトリ
    - 特権付与にチ![Screen Shot 2020-04-16 at 20.26.12.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/2e2b70c5-c859-75ce-685e-ee31c692a6b6.png)
ェック
    - サービスロールは既存のものがあれば選択、ない場合は新規で作成

![Screen Shot 2020-04-16 at 20.19.42.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/ef982536-4d90-dc1a-254d-4adb233ff1c3.png)

- BuildSpec は buildspec.ymlを利用
- アーティファクト（生成した PDF）を任意のS3のバケットにアップロードするように設定

![Screen Shot 2020-04-16 at 20.23.03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/ce0ac4d8-87d5-e3c4-0f6c-6550d784fcf2.png)


### サービスロールに割り当てるポリシー

今回のCI/CDパイプラインを動作させるために、CodeBuild のサービスロールに下記のポリシーを IAM から割り当てる。


```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "ecr:DescribeImageScanFindings",
                "ecr:GetLifecyclePolicyPreview",
                "ecr:GetDownloadUrlForLayer",
                "ecr:GetAuthorizationToken",
                "ecr:ListTagsForResource",
                "ecr:UploadLayerPart",
                "ecr:PutImage",
                "ecr:BatchGetImage",
                "ecr:CompleteLayerUpload",
                "ecr:DescribeImages",
                "ecr:InitiateLayerUpload",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetRepositoryPolicy",
                "ecr:GetLifecyclePolicy"
            ],
            "Resource": "*"
        }
    ]
}
```

![Screen Shot 2020-04-16 at 20.26.12.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/f5f79a36-1bf6-e4ab-2ca4-0be1f2992e21.png)
![Screen Shot 2020-04-16 at 20.35.58.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/999fdfa9-7647-0d93-c9c2-23be696465d1.png)
![Screen Shot 2020-04-16 at 20.35.58.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/f87b6ada-cabe-f3b3-3520-45e88792129a.png)



### buildspec.yml

実際に Markdown を PDF に変換する処理を buildspec.yml に記述する。

```yaml
version: 0.2

phases:
  install:
    commands:
      - echo Nothing to do in the install phase...
  build:
    commands:
      - echo Build started on `date`
      - ls #for execution test on docker environment
      - mkdir -p _build
      - for file in `find src -type f -name "*.md"`;
          do
            out=`echo $file | (sed -e s/src/_build/ | sed -e s/\.md//)`;
            echo $out.pdf;
            pandoc $file -f markdown -o $out.pdf -V documentclass=ltjarticle -V classoption=a4j -V geometry:margin=1in --pdf-engine=lualatex;
          done
      - ls _build
  post_build:
    commands:
      - echo Build completed on `date`
artifacts:
  files:
    - '_build/*'
```

上記 YAML の build フェーズの commands が Pandoc の Docker イメージで実行されるコマンド。 
`src/` にある Markdown を `_build/` に PDF に変換したファイルを格納する。
artifacts フェーズで、 `_build/` にある全ての PDF をアップロードすることを記述している。


## CodePipeline

CodeCommit, CodeBuild をつなげる。デプロイステージは今回は省略した。


![Screen Shot 2020-04-16 at 20.35.58.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/6cbc4a79-1b13-b18f-2b00-fec94d4dbc78.png)

![Screen Shot 2020-04-16 at 20.37.05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/598f51ad-d5ad-b240-b93e-8d3591bcb8f3.png)

![Screen Shot 2020-04-16 at 20.37.43.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/3521d99d-99ff-fba1-512a-b8ce3c913e0e.png)

![Screen Shot 2020-04-16 at 20.38.48.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/54bb1e8c-56f8-7e13-f978-a210c292e842.png)

これまで構築したサービスが正しく設定されていれば、CodePipelineが成功するはず。
![Screen Shot 2020-04-16 at 20.40.51.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/272faaf0-9dc7-f962-3dcf-799678550d03.png)

Markdown を編集して CodeCommit に push すると、CodePipeline が動作し、編集した内容を反映させてPDFを作成する。
作成した PDF は ログから保存先の S3 バケットにアクセスできる。zip で保存されているため、ダウンロードして展開すると生成した PDF が参照できる。

![Screen Shot 2020-04-18 at 0.29.13.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/278110/a3ca4363-43a7-6329-cd72-aa41773d55dc.png)


# サンプルソース

- GitHubに公開
  - https://github.com/takanassyi/md2pdf-pipeline

# 参考資料

- [k1low/alpine-pandoc-ja | Docker Hub](https://hub.docker.com/r/k1low/alpine-pandoc-ja/)
- [GitHub - k1LoW/docker-alpine-pandoc-ja: Pandoc for Japanese based on Alpine Linux](https://github.com/k1LoW/docker-alpine-pandoc-ja)
- [MarkdownからWordやPDF生成ができるようにする (またはPandoc環境の構築方法) (2017/12版) - Copy/Cut/Paste/Hatena](https://k1low.hatenablog.com/entry/2017/12/22/083000)
- [CodeBuild の Docker サンプル - AWS CodeBuild](https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/sample-docker.html#sample-docker-files)
- [CodeBuild の Amazon ECR サンプル - AWS CodeBuild](https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/sample-ecr.html)
- [MarkdownをCircleCI上でPDFに変換してGoogleドライブにデプロイする \| QUARTETCOM TECH BLOG](https://tech.quartetcom.co.jp/2018/05/14/markdown-pdf-circleci-googledrive-deployment/)


