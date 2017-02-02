---
layout: post
title:  "Django Channels (WebSocket) - Session Based Auth -> Token Based Auth"
date:   2017-02-02 12:09:12
author: Suchang Ko
categories: etc
---
Django의 WebSocket App인 Django-Channels([Link](https://github.com/django/channels))를,
채팅 구현으로 사용하기로 하였다.
나의 경우 Django에서 Django-Rest-Framework를 이용해 App과 Web을 전부 커버하려는 상황이었는데,
Auth Token기반으로 초반 설계를 하게 되었다.

Channels를 이용하려고, Example를 찾던 중 좋은 예제를 찾았다 ([Link](https://github.com/andrewgodwin/channels-examples))

MultiChat이라는 부분이 가장 도움이 되었는데, 문제가 있었다.

잘 살펴보면, MultiChat의 consumers.py에서
Websocket HandShaking할때, (ws_connect),
Chat관련된 기능을 수행할때 (chat_join, chat_leave, chat_send),
해당 함수에 Decorator가 붙어있는 것을 확인 할 수 있다.

나의 케이스는 로그인을 하고, Django에서 AuthToken을 Response하여, Web단에서 React를 이용해 내부 Cookie에 저장해두고 쓰게 해두었는데...

해당 예제의 경우 Session Based로 이루어져 있었다.

결국 Decorator를 뜯어보게 되었고,
channels.auth의 channel_session_user_from_http,
channel_session_user를 살펴보면
일반적으로 Django의 Session Based Auth에 의해 http Session 안에
user_hash, user_id, django의 backend_model이 들어가게 된다.

하지만 나의 경우 이미 인증이 완료되어 나오는 Token이 Client에게 존재하였고,
결론적으로 해당 Decorator에서 필요했던 기능은 해당 Channel_session에서 user 정보만 확인할 수 있으면 되는 것이었다.

Token Based로 처리하는 간단한 예시는 다음과 같다.
~~~ python
def channel_session_user_from_token(func):
    @channel_session
    @functools.wraps(func)
    def inner(message, *args, **kwargs):
        if not hasattr(message, "channel_session"):
            raise ValueError("Did not see a channel session to get auth from")
        if message.channel_session is None:
            from django.contrib.auth.models import AnonymousUser
            message.user = AnonymousUser()
        else:
            key = ""
            for item in message.content['headers']:
                if str(item[0], "utf-8") == "cookie":
                    for split_str in str(item[1], "utf-8").split(";"):
                        tmp_str = split_str.lstrip()
                        if "key" in tmp_str:
                            key = tmp_str[4:]

            if not key is "":
                tmp_token = Token.objects.get(key=key)
                tmp_user = UserProfile.objects.get(user=tmp_token.user)
                message.user = tmp_user.user
                message.channel_session['key'] = key
            else:
                from django.contrib.auth.models import AnonymousUser
                message.user = AnonymousUser()
        return func(message, *args, **kwargs)
    return inner
~~~

간단하게 설명하자면, Client에서 Cookie로 들어있는 Key(Auth Token)을 가져와서,
해당 Token기반으로  User를 찾아 channel_session의 message.user에 넣어주면 간단하게 완료된다.
즉 인증되지 않는 사용자에게는 AnonymousUser를 넣어주고, 인증된 사용자에게는 사용중인 User Model를 message.user에게 넣어주면 간단하게 사용할 수 있다.

이와 같이 만들어진 Decorator를 사용하는 방법은, 위에 올려진 Example을 참고하면 더 빠르다.
예를 들어, @channel_sessionn_user_from_http가 아닌,
@channel_session_user_from_token등으로 사용하면 된다.

처음엔 Auth Token을 이용하여 기존 세션인증방식으로 그대로 사용하고자 Model_Backend, user_id, user_hash를 역으로 추적하여, 강제로 channel_session을 생성해주려 했으나,
channels.auth를 살펴보니 결국 Decorator에서 만족해야 하는 기능은, User를 인증하여 message.user만 넣어주면 해결된다.

App에서도 사용이 간단해진다.
WebSocket Handshaking할때 Cookie를 넣어주면 된다.

예를 들어, Android에서 WebSocket Library인 'nv-websocket-client'를 사용할때,
~~~ python
new WebSocketFactory()
                .setConnectionTimeout(TIMEOUT)
                .createSocket(SERVER)
                .addListener(new WebSocketAdapter() {
                    // A text message arrived from the server.
                    public void onTextMessage(WebSocket websocket, String message) {
                        System.out.println(message);
                    }
                })
                .addExtension(WebSocketExtension.PERMESSAGE_DEFLATE)
                .addHeader("cookie","key=4ac18618b19f1f03dc9055a4f13bddc04c0cf8bf;")
                .connect();
~~~ python
이런 형태의 사용이 가능하다.