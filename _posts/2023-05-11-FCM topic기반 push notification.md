---
layout: post
title: FCM 푸시알림
subtitle: FCM topic기반 push notification 구현하기
categories: markdown
tags: [test]
---

# push

하나의 상점A을 여러 명이 즐겨찾기 한다면?

처음에는 상점A를 즐겨찾기한 회원을 모두 찾아와서 한 명씩 푸시 알림을 보내야 한다?

이는 비효율적으로 보인다. push메시지를 보낼 때 동기방식으로 동작하니 그 시간동안 IO블로킹이 걸리기 때문에 스레드가 작동을 멈춘다.

다음의 과정이 반복 된다고 보면된다.

1.  새로운 상품을 등록한 상점A를 즐겨찾기 하는 사용자가 가지고 있는 토큰을 조회한다.
2.  조회한 토큰과 푸시데이터를 FCM서버로 전송한다
3.  FCM서버에서 응답이 올 때까지 대기한다(동기적)
4.  2,3번의 과정을 즐겨찾기한 회원마다 반복

사용자가 많지 않다면 부하에 대한 문제가 없지만 사용자가 늘어난다면 상당히 비효율 적이다.

# 해결 방안

이 문제를 해결하려고 FCM공식문서를 보았다.(한글지원이 돼서 비교적 다행…)

`topic`이라는 개념을 통해 해결할 수 있다.

특정 사용자가 특정 `topic`을 구독하면, `topic`을 대상으로 메시지를 보내면 `topic`을 구독한 사용자들에게 푸시메시지를 보내주는 유용한 기능이다.

위에서 했던 과정을 `topic`을 이용하면 다음과 같다.

1.  새로운 상품을 등록한 상점의 `topic`을 조회한다
2.  `topic`과 푸시 메시지 정보를 FCM서버로 전송한다
3.  FCM서버에서 응답이 올 때까지 대기한다

한번만 보내고 끝낼 수 있기 때문에 처음에 생각한 방식보다는 훨씬 효율적인 것을 한눈에 알 수 있다.

백엔드에서 구체적인 로직을 구사하면 다음과 같다

상점을 즐겨찾기한 사용자의 FCM device token을 가져온다

이 token을 특정 message group(상점A 즐겨찾기한 사용자 모음)에 추가해 달라는 요청을 FCM에 보낸다 이 때 토픽이 Firebase 프로젝트에 아직 없다면 새 토픽이 만들어진다.

`subscribeToTopic()` 및 `unsubscribeFromTopic()` 메서드는 FCM의 응답을 포함하는 객체를 반환합니다. 반환 유형의 형식은 요청에 지정된 등록 토큰 수에 관계없이 동일합니다.

> **참고:** 단일 요청으로 기기를 최대 1,000대까지 구독하거나 구독 취소할 수 있습니다. 배열에 1,000개가 넘는 등록 토큰을 제공하면 **`messaging/invalid-argument`** 오류가 표시되면서 요청이 실패합니다. -출처: FireBase 공식문서

```java
// These registration tokens come from the client FCM SDKs.
List<String> registrationTokens = Arrays.asList(
    "YOUR_REGISTRATION_TOKEN_1",
    // ...
    "YOUR_REGISTRATION_TOKEN_n"
);

// Subscribe the devices corresponding to the registration tokens to the
// topic.
TopicManagementResponse response = FirebaseMessaging.getInstance().subscribeToTopic(
    registrationTokens, topic);
// See the TopicManagementResponse reference documentation
// for the contents of response.
System.out.println(response.getSuccessCount() + " tokens were subscribed successfully");
```

```java
// These registration tokens come from the client FCM SDKs.
List<String> registrationTokens = Arrays.asList(
    "YOUR_REGISTRATION_TOKEN_1",
    // ...
    "YOUR_REGISTRATION_TOKEN_n"
);

// Unsubscribe the devices corresponding to the registration tokens from
// the topic.
TopicManagementResponse response = FirebaseMessaging.getInstance().unsubscribeFromTopic(
    registrationTokens, topic);
// See the TopicManagementResponse reference documentation
// for the contents of response.
System.out.println(response.getSuccessCount() + " tokens were unsubscribed successfully");
```

```java
// The topic name can be optionally prefixed with "/topics/".
String topic = "highScores";

// See documentation on defining a message payload.
Message message = Message.builder()
    .putData("score", "850")
    .putData("time", "2:45")
    .setTopic(topic)
    .build();

// Send a message to the devices subscribed to the provided topic.
String response = FirebaseMessaging.getInstance().send(message);
// Response is a message ID string.
System.out.println("Successfully sent message: " + response);FirebaseMessagingSnippets.java
```

```java
// Define a condition which will send to devices which are subscribed
// to either the Google stock or the tech industry topics.
String condition = "'stock-GOOG' in topics || 'industry-tech' in topics";

// See documentation on defining a message payload.
Message message = Message.builder()
    .setNotification(Notification.builder()
        .setTitle("$GOOG up 1.43% on the day")
        .setBody("$GOOG gained 11.80 points to close at 835.67, up 1.43% on the day.")
        .build())
    .setCondition(condition)
    .build();

// Send a message to devices subscribed to the combination of topics
// specified by the provided condition.
String response = FirebaseMessaging.getInstance().send(message);
// Response is a message ID string.
System.out.println("Successfully sent message: " + response);FirebaseMessagingSnippets.java
```

[Android에서 주제 메시징  |  Firebase 클라우드 메시징](https://firebase.google.com/docs/cloud-messaging/android/topic-messaging?hl=ko#java_1)
