---
layout: post
title:  "Such a great second post!"
date:   2019-02-05 21:13:37 +0300
author: AetherEternity
---
Hello there!

No flag even here, but another opportunity to advertise [our shop][shop].

```c
#include <stdio.h>
#include <stdlib.h>
int main(void)
{
	char al[36]="ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
	char no[33]={0};
	srand(0xF00DCAFE);
	for (int i = 0; i < 36; ++i)
	{
		no[i]=al[rand()%36];
	}
	puts(no);
	return 0;
}
```

[shop]: https://kappactf.ru/shop/

