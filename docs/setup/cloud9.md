# Cloud9による環境の準備

## Cloud9とは

AWSが提供しているクラウド上で動く､アプリケーション開発環境です｡  
コードエディタと開発環境がWeb上から使えます｡  
AWS CLIとAWS SAM CLIなどAWSを使った開発に必要なツールや基本的な言語環境が
インストール済みです｡  

<a href="https://aws.amazon.com/jp/cloud9/" target="_blank">AWS Cloud9（Cloud IDE でコードを記述、実行、デバッグ）\| AWS </a>	

## なぜ今回Cloud9を使うか

- 最初からAWSを使った開発に必要なツールがインストールされている
- 環境の準備という本質的でない所でつまづいて欲しくない
- ハンズオンの性質上ローカルマシンを統一できない

弊社とは無関係ですが､実際の開発現場で使用されている例もあります｡  

<a href="https://speakerdeck.com/marumoto/jaws-days2018?slide=75" target="_blank">ユーザー企業におけるサーバレスシステムへの移行/JAWS DAYS2018 </a>	

## Cloud9 Environmentの作成

!!! Warning
    マネジメントコンソールの右上で､
    リージョンが**東京(ap-northeast-1)**になっている事を確認してください｡

!!! Info
    SafariでCloud9を使用する場合には､**サイト越えトラッキングを防ぐ**と**すべてのCookieをブロック**を無効にする必要があります｡  
    <a href="https://support.apple.com/ja-jp/guide/safari/sfri11471/mac" target="_blank">Mac の Safari で Cookie と Web サイトのデータを管理する 
</a>

1. <a href="https://ap-northeast-1.console.aws.amazon.com/cloud9/home/product" target="_blank">AWS Cloud9 </a>を開く	
1. **Create environment**を選択します｡  
1. 環境を設定し､**Next step**を選択します｡  
    - Name: hands-on
    - Description: 何も入力しない
1. デフォルトの設定のまま､**Next step**を選択します｡ 
    - Environment type: Create a new instance for environment (EC2)
    - Instance type: t2.micro (1 GiB RAM + 1 vCPU)
    - Platform: Amazon Linux
    - Cost-saving setting: Afrer 30minutes(default)
    - IAM role: AWSServiceRoleForAWSCloud9
1. 設定を確認し､**Create environment**を選択します｡
1. Cloud9が起動します｡  

!!! Info
    Cloud9を開いているウィンドウを閉じてしまった場合は､再度<a href="https://ap-northeast-1.console.aws.amazon.com/cloud9/home/product" target="_blank">AWS Cloud9 </a>を開き､
    **hands-on**の**Open IDE**を選択します｡
    

## nodenv(anyenv)のインストール

多言語に対応している開発環境のバージョン管理ツール<a href="https://github.com/anyenv/anyenv" target="_blank">anyenv</a> を使って、Node.jsのバージョン管理ツール<a href="https://github.com/nodenv/nodenv" target="_blank">nodenv</a>をインストールします。

本ハンズオンでは、現時点でのNode.js LTS版の`12.16.3`を使用します。

```
git clone https://github.com/anyenv/anyenv ~/.anyenv
echo 'export PATH="$HOME/.anyenv/bin:$PATH"' >> ~/.bash_profile
echo 'eval "$(anyenv init -)"' >> ~/.bash_profile
~/.anyenv/bin/anyenv init
exec $SHELL -l
anyenv install --init
anyenv install nodenv
exec $SHELL -l
NODE_VERSION=12.16.3
nodenv install ${NODE_VERSION}
nodenv global ${NODE_VERSION}
exec $SHELL -l
```

途中で何回か実行している`exec $SHELL -l`はシェルを再読み込みするコマンドです。

## AWSリージョンの設定

ハンズオンではAWS CLIを使い､コマンドラインでAWSを操作します｡  
利用するAWSリージョンを指定する為に､環境変数を設定します｡  

```bash
echo 'AWS_DEFAULT_REGION="ap-northeast-1"' >> ~/.bashrc
source ~/.bashrc
```

設定を確認します｡  
```bash
echo ${AWS_DEFAULT_REGION}
aws ec2 describe-instances --query \
    'sort_by(Reservations[].Instances[].{Tags:Tags[?Key==`Name`].Value|[0],InstanceType:InstanceType,State:State.Name},&Tags)' --output table
```
(例)
```text
ap-northeast-1
-----------------------------------------------------------------------------------------
|                                   DescribeInstances                                   |
+--------------+----------+-------------------------------------------------------------+
| InstanceType |  State   |                            Tags                             |
+--------------+----------+-------------------------------------------------------------+
|  t2.micro    |  running |  aws-cloud9-hands-on-[Random alphanumeric]              |
+--------------+----------+-------------------------------------------------------------+
```
