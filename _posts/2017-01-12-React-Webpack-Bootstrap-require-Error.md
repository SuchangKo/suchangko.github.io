---
layout: post
title:  "Django Channels (WebSocket) - Session Based Auth -> Token Based Auth"
date:   2017-02-02 12:09:12
author: Suchang Ko
categories: etc
---
Django의 WebSocket App인 Django-Channels([Link](https://github.com/django/channels))를, 채팅 구현으로 사용하기로 하였다.
나의 경우 Django에서 Django-Rest-Framework를 이용해 App과 Web을 전부 커버하려는 상황이었는데,
Auth Token기반으로 초반 설계를 하게 되었다.

Channels를 이용하려고, Example를 찾던 중 좋은 예제를 찾았다 ([Link](https://github.com/andrewgodwin/channels-examples))

MultiChat이라는 부분이 가장 도움이 되었는데, 문제가 있었다.

잘 살펴보면, MultiChat의 consumers,py에서 Websocket HandShaking할때, (ws_connect),
Chat관련된 기능을 수행할때 (chat_join, chat_leave, chat_send), 해당 함수에 Decorator가 붙어있는 것을 확인 할 수 있다.

나의 케이스는 로그인을 하고, Django에서 AuthToken을 Response하여, Web단에서 React를 이용해 내부 Cookie에 저장해두고 쓰게 해두었는데...

해당 예제의 경우 Session Based로 이루어져 있었다.

결국 Decorator를 뜯어보게 되었고,
channels.auth의 channel_session_user_from_http channel_session_user를 살펴보면
일반적으로 Django의 Session Based Auth에 의해 http Session 안에 user_hash, user_id, django의 backend_model이 들어가게 된다.

하지만 나의 경우 이미 인증이 완료되어 나오는 Token이 Client에게 존재하였고,
결론적으로 해당 Decorator에서 필요했던 기능은 해당 Channel_session에서 user 정보만 확인할 수 있으면 되는 것이었다.

Token Based로 처리하는 간단한 예시는 다음과 같다.
	def channel_session_user_from_token(func):
	"""
	Presents a message.user attribute obtained from a user ID in the channel
	session, rather than in the http_session. Turns on channel session implicitly.
        """
        @channel_session
        @functools.wraps(func)
        def inner(message, *args, **kwargs):
            # If we didn't get a session, then we don't get a user
            if not hasattr(message, "channel_session"):
                raise ValueError("Did not see a channel session to get auth from")
            if message.channel_session is None:
                # Inner import to avoid reaching into models before load complete
                from django.contrib.auth.models import AnonymousUser
                message.user = AnonymousUser()
            # Otherwise, be a bit naughty and make a fake Request with just
            # a "session" attribute (later on, perhaps refactor contrib.auth to
            # pass around session rather than request)
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
            # Run the consumer
            return func(message, *args, **kwargs)
        return inner

를 해주면, 말끔하게 에러가 해결된다.