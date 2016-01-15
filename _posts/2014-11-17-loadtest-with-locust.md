---
layout: post
title: Locust - 負荷試験
description: "Let it crash!"
modified: 2014-11-18
tags: [python,locust,load test]
image:
  feature: abstract-3.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---

REF

[1] - [Locust document](http://docs.locust.io/en/latest/)

***

はじめで日本語で書きます。よろしくお願いします！

***


# 負荷試験の難しさ

- 一つのThreadで、Requestを送信するのは遅い。
- 複数のThreadでRequestを送信しても、一つのサーバー（パソコン）の性能は限界がある。
- 負荷の指定するのは難しい。
- ResponseTimeの統計はめんどい。（二番目を解決するため、複数サーバーが負荷をかけるともっとめんどくなる）
- Responseの内容をチェックしたいんですが、チェックするとRequestを送信するのが遅くなる。バランスが取りにくい。
- 自分でそんな負荷試験プログラム書きたくない。

Locustは以上の全ての問題を解決できます。

# Locustの基本機能

Locustでは、何個ユーザー（client）、どうの頻度でどうのrequestが送信するのを設定できます。そして、実行している時現時点の秒間リクエスト数や、各リクエストの送信回数と平均・最大・最小ResponseTimeや、失敗したリクエストを表示することができます。


# LocustのInstall


pipとeasy_installでinstallするのは一番やりやすい方法です：

{% highlight bash %}
pip install locustio
{% endhighlight %}

{% highlight bash %}
easy_install locustio
{% endhighlight %}

WindowsとMacのInstall方法は[こちら](http://docs.locust.io/en/latest/installation.html)に参考してください。

# Locustを起動する

Locustでの負荷試験は簡単です。
負荷試験を起動する必要なものは一つのpythonのTestCaseです。
LocustのTestCaseも書きやすいです。以下は典型的な例の一つ：

{% highlight python %}
from locust import HttpLocust, TaskSet, task

# 本物のTestCase前に、Test用の資料を用意します
textureDirName = "texture"
userIdFileName = "user_id.txt"
userIdFilePath = os.path.join(os.path.abspath(os.path.dirname(__file__)), textureDirName, userIdFileName)

globalUserArr = list(line.rstrip('\n') for line in open(playerIdFilePath))

class SimpleTaskSet(TaskSet):

  # テスト起動する時invokeするメソード
  def on_start(self):
    randomIndex = random.randint(0, len(globalUserArr) - 1)
    self.testingUserId = globalUserArr[randomIndex]

  # test case 1
  @task(1)
  def testRequestGet(self):
    # urlのpathの部分を作成
    requestPath = "/services/users/" + self.testingUserId + "/checkSomeThing"
    # requestを送信
    self.client.get(requestPath)

  # test case 2
  # @task(2) が指定して、このtest caseの送信頻度を上のtest case 1の二倍になる。
  @task(2)
  def testRequestPost(self):
    requestPath = "/services/users/doSomeThing/checkin"
    # Post requestの読み方
    self.client.post(requestPath, {"user": self.testingUserId })

# Locustのclass
class SimpleLocust(HttpLocust):
  task_set = ReadStateTaskSet　# 上のtest caseをimportします
  # 各ユーザーが前回のrequestを送信した後、何秒を待つかという設定
  min_wait = int(1)
  max_wait = int(2)

{% endhighlight %}


>　一つの注意点があります。 *List* 、 *Dict*のようなMemberがTaskSetに入らない方がいいと思います。


そして、

{% highlight python %}
python -H [you_url_or_ip_of_your_application] -f [path_of_your_SimpleTask]
{% endhighlight %}

を実行しで、Localhostの8089Portで負荷試験Consoleを訪問できます。そこで負荷が設定して試験を始めます。


# Distributedでの負荷試験

Locustを使ったら、分散型負荷試験がやりやすいです。

まず複数のサーバーを用意して、そしてさきのscriptをサーバーにUploadします。

一つのサーバーは **Master** を担当し、他のサーバーが **Slave** になる。 **Master** が設定した負荷を **Slave** に配分して、 **Slave** がリクエストを送信します。

masterで:

{% highlight bash %}
python -H [you_url_or_ip_of_your_application] -f [path_of_your_simple] --master
{% endhighlight %}


slaveで:

{% highlight bash %}
python -H [you_url_or_ip_of_your_application] -f [path_of_your_simple] --slave --master-port=[master_ip]
{% endhighlight %}

>　分散型負荷試験をやる時、３つの注意点がある：
>
>　1.　MasterとSlaveと通信があるので、FirewallやSecurityGroupを設定しないといけない。
>
>　2.　重い負荷をテストするとHealthCheckなどのため、同時でたくさんファイルがOpenするから、Ulimitを超えるかも。だから、MasterとSlaveの各サーバーで、Locustを起動する前に`ulimit 4096`をさきに実行します。
>
>　3.　Defaultの8089のportを変更したいなら、Masterで`--master-bind-port=5557`のように追加して、Slaveで`--master-port=5557`を追加します。
