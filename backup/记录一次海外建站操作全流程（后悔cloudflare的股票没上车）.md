步骤1：购买VPS，我使用的是BandwagonHost，可以根据个人需要使用任何云服务商的VPS

对访问速度有要求的话，最好选择CN2 GIA线路（就是价格稍高一些，肉疼）
图：官方页面
<img width="3685" height="1005" alt="Image" src="https://github.com/user-attachments/assets/f2cf0b1e-536b-4428-bbbe-a69323318bc2" />

购买成功后选择需要的操作系统版本启动，并记录下IP地址，后需要关联域名

步骤2：因为是管理系统，需要进行登录操作，为了保证安全性，使用https进行访问。这里需要申请域名及托管

购买域名使用了这个网站，可以根据需要自行选择想要的域名及付费
图：官方页面
https://www.namesilo.com/
<img width="3797" height="1791" alt="Image" src="https://github.com/user-attachments/assets/c891c31a-e69e-43ce-a928-05131d649f00" />

域名托管使用了cloudflare著名的赛博活菩萨的DNS解析服务，并通过cloudflare生成SSL证书（免费）
图：官方页面
![Image](https://github.com/user-attachments/assets/372f8526-08b0-4f1b-8812-e4fad71f9820)
具体步骤可以查询相关操作流程


步骤3：启动自己的管理应用

步骤4：使用docker启动一个nginx服务器，进行请求转发，避免暴露端口号
通过shell工具连接到vps后，先安装运行时环境等相关变量
把SSL证书文件配置到对应的路径，然后修改nginx的配置文件，指定SSL证书的路径
图：配置文件指定路径
![Image](https://github.com/user-attachments/assets/c2deb9b2-06ac-4e4d-8b27-574a1ff8373e)

图：启动成功
![Image](https://github.com/user-attachments/assets/021fa8b9-765b-409a-8dac-8c0dc701f06c)

图：访问成功！
![Image](https://github.com/user-attachments/assets/bae82dce-454b-406d-8c65-00550690d5bb)
