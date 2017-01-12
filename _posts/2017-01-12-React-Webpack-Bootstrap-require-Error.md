---
layout: post
title:  "React Wepack Bootstrap Require Error"
date:   2017-01-12 11:26:01
author: Suchang Ko
categories: etc
---
Webstorm에서 React App을 개발하려고 세팅중, 문제가 발생하였다.
`require('jquery')`를 했을 경우, Uncaught Reference Error: jQuery is not defined
라는 에러와 함께 해당 페이지가 제대로 렌더링 되지 않는다.

해결법은, [링크](https://github.com/fronteed/icheck/issues/322)에서, 친절하게 코멘트가 달려있다.

	var $ = require('jquery');
    window.jQuery = $;
를 해주면, 말끔하게 에러가 해결된다.