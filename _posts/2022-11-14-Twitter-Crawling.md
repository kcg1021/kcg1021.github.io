---
title: Twitter Crawling
author: Kim Chae Gyu
date: 2022-11-14 09:16:00 +0800
categories: [전공심화]
tags: [Python, Twitter, Elastic, logstash]
---

아래의 결과를 확인 하기 위해서는 로그스태시와 파이선을 동시에 실행을 시켜주어야 한다.

## 트위터 API를 이용한 크롤링 소스 [ Python ]
---

```python
import tweepy
import logging
import logstash

BEARER_TOKEN = "토큰 값 입력"

class IDPrinter(tweepy.StreamingClient):
    # on Connect 이벤트에서 트위터 API에 등록된 규칙들을 불러와 규칙이 존재할 경우
    # 등록된 규칙들을 삭제하고, 검색하고자 하는 규칙을 추가함
    def on_connect(self):
        if self.get_rules().data != None:
            for rule in self.get_rules().data:
                self.delete_rules(rule.id)
        self.add_rules(tweepy.StreamRule("수능"))

    # on Tweet 이벤트에서 logger를 생성하거나 불러와서 로거에 로그스태시 핸들러가 존재하지 않는다면
    # addHandler를 통해 로그스태시 핸들러를 등록해주고, tweet Dictionary의 text를 info Level의 로거를 출력함
    def on_tweet(self, tweet):
        tlogger = logging.getLogger('Twitter 크롤링')
        if len(tlogger.handlers) == 0:
            tlogger.setLevel(logging.INFO)
            tlogger.addHandler(logstash.LogstashHandler('localhost', 12123, version=1))
        try:
            print(tweet)
            tlogger.info(tweet.text)
        except:
            None

    # on Errors 이벤트에서 오류가 발생할 경우 에러 로그를 출력하고 트위터 API의 연결을 종료함
    def on_errors(self, errors):
        print(f"Received error code {errors}")
        self.disconnect()
        return False

# Class를 불러와 filter() 를 통해 Stream 받아옴
printer = IDPrinter(BEARER_TOKEN)
printer.filter()
```

## Logstash config 값
---

```conf
input {
  # 데이터를 받아오는 경로를 udp port의 12123번과 연결한다.
  # codec은 json으로 결과값을 json으로 받는다.
  udp {
    port => 12123
    codec => json
  }
}

output {
  # elasticsearch의 hosts 서버에서 "twitter_ko" Index에 결과 값을 넣는다.
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "twitter_ko"
  }
  # stdout을 이용해 로그스태시 콘솔에도 표시를 시켜준다.
  stdout { }
}
```

## Python 로그 결과
---

![Python Log](https://media.discordapp.net/attachments/1045516878161903666/1045516930984984687/logstash_python.PNG)

## Logstash 로그 결과
---

![Python Log](https://media.discordapp.net/attachments/1045516878161903666/1045516931312136222/logstash_log.PNG)