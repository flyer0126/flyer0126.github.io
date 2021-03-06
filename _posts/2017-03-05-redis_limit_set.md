---
layout: post
title: "利用Redis实现单位时间内请求次数限制"
author: "flyer0126"
---

问题需求：  
用户请求发短信接口限制规则，10分钟之内请求超3次即显示图形验证码（需要先验证图形验证码通过后再发送短信）。

解决思路：
利用Redis List数据格式；
key：ImageCode_RequestLimit_Uid;
value: 请求时间戳。

验证实现：

```
$key = 'ImageCode_RequestLimit_Uid';
$listLen = lLen($key);
if($listLen < 3){
	// 直接将当前时间戳插入List尾部
	Lpush($key, now());
} else {
	$index0Time = Lindex($key);
	if((当前时间 - $index0Time) < 10min){
		// 触发10min内请求大于3次，提醒，“请求过多，请稍后再试。”
		echo "请求过多，请稍后再试。";
		exit;
	} else {
		// 将当前时间戳插入List尾部
		// 取出List头部首元素
		Lpush($key, now());
		Ltrim($key, 0, 9);
	}
}
```

<br>

_The end_