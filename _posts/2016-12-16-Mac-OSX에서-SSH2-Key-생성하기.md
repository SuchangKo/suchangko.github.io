---
layout: post
title: 'Mac OSX에서 SSH2 Key 생성하기'
categories: 'jekyll update'
---

`ssh-keygen -t rsa -f mykey`

입력 후, passphrase를 입력해 준다.

`ssh-keygen -e -f mykey.pub`

위의 명령어를 실행한다.

Azure VM의 SSH Public Key를 등록할 경우, 위의 명령어에 나오는 결과물중

 	---- BEGIN SSH2 PUBLIC KEY ----
	~~~~~~~~
	---- END SSH2 PUBLIC KEY ----

부분을 등록하면, Public Key 등록이 완료된다.