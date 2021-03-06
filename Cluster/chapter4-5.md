# 选项配置

在该页面需要录入Server用于自动化的访问集群中其他目标主机的相关配置信息。

* 输入目标主机名列表，注意每一行代表一个目标主机。

![](/assets/4.5-hosts.png)

* 之前我们配置了server主机针对其他目标主机的SSH免密登陆，这里我们先获取其SSH私钥。

```
[root@server .ssh]# cat /root/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEpgIBAAKCAQEAzXYScP6GHLivxzsytl4eeFss6oi2nsHHLsjsi2cHlnm+R6Vu
fOMkOXw0rtxoWp7I7op7fv+gmv+Jv3sY+ja0RdhbbYspDS1jHxfMERRwYhVpexPa
WQ3nlcy0YDAONqKGzQ86+9zCQ1lOMvdzjd53NI48j/u5bDCbhwucMPVGw3z988Ot
QoOZbEVhJ5FlSbz/Ap3UGIbILlqfVBgVnvEYhEwsmY9joUXXOjSLGiNejr/wH616
KvOLmhSDqh6w/hxq+uFcga7HbEmL+82Gwp8Z8z+hCh19tq8arpswhb2T7TyrkSwi
m7vZBgP3t1sdGePw9BwPT8uVqhQY43J0tFBmhwIDAQABAoIBAQCA0NM1Fs78uOo0
LjBYWGAgM4HQtdBRbsqz0XNE317JgCDFiLniAQMYK4BYVYXzsvPlYtuUvy5xn188
ty/syFl0JPcFkic1xMwNlXzzBG6FgEk2yjaueOJGcCZy3A49QN7lN/RSLpF5akd1
+uDvBJiWUcs0tq0FYOBR5fySUWWBb/+OK2VkV8ijnhEZcNvZBqPpQHiJKw9fLawl
25gUDqqcHu+tuyijfHGJH26dNx66l9SYVK/9P29fRw06AKflPqIEfL1z/m8RCUQo
Fdl8O2738Al1pPnz+nTz0vZDvVqk/2k80a+3fDqPC30G1/IAJ7IyfdQk7OonxLpJ
0Gg1Ng4BAoGBAOxZ5etKtaItgDFcBO9ElQK+NDLyaT8T4YB/BvN+87TvAsvmJrmt
c9gNd2ltcghYBK6zIlVXVEn0Gr9Xc+izfbYxAJdPme2LSo+sWWU/5uJmD82jGeYj
eRGQmRn0+RtoE2+Z0Rw6ghTJx8uTCodIrS4F4QO5CUADH5u1KmOo3G0BAoGBAN6K
w5yMyl1glZg4JPj+8QPjPcNBmAcUhdjZcTf1WC4diQ2u38e5L4ofAdqwFwXrIOpc
FMPB/Muj+rLD76W7cUwuBtA4BQtTk6wfir/2CcV6KlphGUOL1O3rfI7h1uqiMby7
99Pmcz3TXXLMarKdCVCbs/EZT2Izvl2G73oJA+uHAoGBAJkuTohnnD6m9L2I4R3d
uiHT+mrGl5WtIeqw6WVo8zRh79MMsC6JD1qIp8rphw2HVkmPigH7noJrteYrHNFF
e4VYTwTCL4Y4T7O8RRgNCWvUMAvb2I5CkVXj/IZJMiYkFuyuqUt9VA97E4WKIDm7
zZnVb5eFFkypeZPmH7oFmA8BAoGBANCB+11GnKR4xjDlCd8yHuehllDHuIWJuQ7A
TNA9U+2BRtRHMOyUmfIzsy0PJ8Mn1qM+u0XfD9hNP6sW4gbKZREXXtLgafl+yTHQ
K9RH1kfseppLt7wN2+c/aGkHOLKGXUuUYlNr7DXVQA07cg0ADaY0/Je9Ox+rk4VV
1DLnF4EpAoGBAI5uqN894BYxhzXrmK7cwdGzg3IBg1GR48dFK0ilApacxyew4UWF
FPxZTXxXTpfJaNNnWOQWaugAGt7W6NBtbqMte9TTG+trf/7S2MUvbI75WxcMUhLw
A6Lg2p05bvTCDIUTaL4sEG9Slk9GO3zxrW/VmlfFxREqW1+CL8NfzTAq
-----END RSA PRIVATE KEY-----
```

* 拷贝server主机的SSH私钥至输入框。

![](/assets/4.5-sshkey.png)

* 默认使用root用户和22端口登陆SSH。

![](/assets/4.5-user.png)

