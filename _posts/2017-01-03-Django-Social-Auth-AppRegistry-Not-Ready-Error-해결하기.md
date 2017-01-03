---
layout: post
title:  "Django Social Auth 관련 Error 해결 (Django 1.9)"
date:   2017-01-03 11:41:01
author: Suchang Ko
categories: etc
---
Python-Social-Auth를 통해 Django 1.9에서 Facebook Login을 구현하려던 중, 아래와 같은 에러에 봉착하였다.
`RuntimeError: Model class social_django.models.UserSocialAuth doesn't declare an explicit app_label and isn't in an application in INSTALLED_APPS.`
따라서, settings.py에 social_django.models.UserSocialAuth를 추가해 주었다.

하지만 추가해 준 뒤에도, 아래와 같은 에러가 나타나게 되었다.
`django.core.exceptions.AppRegistryNotReady: Apps aren't loaded yet.`

구글링을 해본 결과, 다음과 같은 답을 얻을 수 있었다.
https://github.com/omab/python-social-auth/blob/master/MIGRATING_TO_SOCIAL.md#django

Python-Social-Auth의 경우 2016.12.03 이후로 지원되는 Framework별로 나누어진것으로 보인다. 즉 통합으로 제공되던 패키지에서, 최신 패키지부터는 기존의 패키지와 이름이 달라진 탓인지, 패키지 명에 대한 업데이트를 해주어야 한다.
