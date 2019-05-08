---
layout: post
title:  "在vscode中开发调试python"
categories: tools
tags:  python,tools
author: 刚子
---

* content
{:toc}












## 在vscode中开发调试python

1. 到[python官网](https://www.python.org/)下载安装包

2. 示例安装[windows 2.7 版本](https://www.python.org/downloads/release/python-2716/)，安装过程中选择将python.exe添加到环境变量中，或者安装之后手动添加

3. 新建目录, 右键用vscode打开，添加hello.py，内容如下

```python
msg = "Hello World"
print(msg)
```

4. 安装python插件

ctrl+p，输入ext install python，选择Python插件安装

5. 在hello.py文件中右击，选择在终端中运行python

## python 代码示例

md5参数签名

```python
#! -*- coding: utf-8 -*-
import hashlib
import datetime

if __name__ == '__main__':
    current_time = datetime.datetime.utcnow()
    # timestamp = (current_time - datetime.timedelta(minutes=1)).strftime('%Y%m%d%H%M%S')
    timestamp =current_time.strftime('%Y%m%d%H%M%S')
    private_key = 'privatekey'
    params = {
        "pk":"publickey",
        "timestamp": timestamp,
    }

    params_data = {}
    for key in params:
        if key not in ("sign",):
            params_data[key] = params[key]

    keys = sorted(params_data.keys())
    params_str = ""
    for key in keys:
        val = params_data.get(key)
        if isinstance(val, unicode):
            val = val
        else:
            val = str(val)
        params_str += '='.join((key, val)) + "&"

    sign = hashlib.md5(private_key + params_str + private_key).hexdigest()

print(private_key + params_str + private_key)

print(sign)
```

json解析、http请求示例

```python
import time
import json
import urllib
import urllib2
def login():
	loginurl='https://xxx/api/Login/ValidateUser'
	with open("/root/job/logindata.json",'r') as f:
		logindata = json.load(f)
		jlogindata=json.dumps(logindata)
		req=urllib2.Request(loginurl,jlogindata,{'Content-Type':'application/json'})
		resS=urllib2.urlopen(req)
		response=resS.read()
		resS.close()
		return response

def punch(timeStamp,sessionId):
	punchurl='https://xxx/api/Attendance/SubmitEmployeeAttendanceDataByMAP'
	with open("/root/job/punchdata.json",'r') as f:
		punchdata=json.load(f)
		punchdata['timeStamp']=timeStamp
		punchdata['SessionId']=sessionId
		jpunchdata=json.dumps(punchdata)
		req=urllib2.Request(punchurl,jpunchdata,{'Content-Type':'application/json'})
		res=urllib2.urlopen(req)
		response=res.read()
		res.close()
		return(response)

def main():
	loginInfo=login()
	print(loginInfo)
	jloginInfo=json.loads(loginInfo)
	sessionId=jloginInfo['data']['SessionId']
	print(sessionId)
	timeStamp=time.strftime('%Y%m%d%H%M%S',time.localtime(time.time()))
	print timeStamp
	punchInfo=punch(timeStamp,sessionId)
	print(punchInfo)

main()
```

```
# /root/job/logindata.json
{
	"phoneSysVersion": "10.3.3",
	"loginName": "4giZPVGlNcNAttDXnYUKcA==",
	"simId": "26215fdc9506938f10b7829dc6113f89|",
	"password": "hDqJdUlw5rkmWQIK2DtMsg==",
	"isClearGesturePwd": 0,
	"language": "ZH-CN",
	"registrationId": "141fe1da9e933aca820",
	"mobileId": "d090d623b5f69ee8dfb3fb2980f2f3ad",
	"phoneModel": "Apple iPhone 6s Plus",
	"appVersion": "3.9.2.5",
	"companyCode": "benlai",
	"resolution": "1242x2208"
}

# /root/job/punchdata.json
{
	"Remark": "",
	"Latitude": "31.193328823666427",
	"Longitude": "121.4296281200414",
	"Address": "上海市上海市徐汇区中山西路1515号-8楼",
	"timeStamp": "{{loginTime2}}}}",
	"SessionId": "{{sessionid}}",
	"MachineID": "88a92445-54c0-4d4e-92b6-9dad7c0dd3c6",
	"sign": "d06a34d1c29fc6a14729505039e957e773bf406aed9d858ddfb662334cd70151"
}
```

## 参考

[Python 入门指南](https://www.runoob.com/manual/pythontutorial/docs/html/)