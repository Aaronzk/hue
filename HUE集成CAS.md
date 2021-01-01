## 两种方案
HUE 集成单点登录支持LDAP、OAuth、PAM、Spnego、SAML等认证方式。
里面并没有 CAS，所以想要集成 CAS，要么使用其他协议(SAML)，要么改源码。我们选择改源码。
### 方案一：SAML2
> CAS can act as a SAML2 identity provider accepting authentication requests and producing SAML assertions.  
CAS可以充当SAML2身份提供者，接受身份验证请求并生成SAML断言。

> Prior to CAS 5, supporting both protocols together required setting up a Shibboleth server alongside the CAS server, and configuring one server to use the other as its authentication source. Although this more or less worked, it was difficult to manage, and didn’t handle all aspects of single sign-on (especially single log-out) very cleanly.  
在CAS 5之前，同时支持这两种协议需要在CAS服务器旁边设置一个Shibboleth服务器，并配置一台服务器使用另一台服务器作为其身份验证源。尽管这或多或少可以工作，但它很难管理，并且不能非常干净地处理单点登录(尤其是单点注销)的所有方面。

这种方案对于CAS Server端有要求，如果使用的是CAS 4.x 或者更低版本，需要在CAS Server端进行配置，来支持SAML2，如果是5.x或者更高版本，则已经默认支持SAML2，只需要在HUE端进行配置即可。

[CAS Adding SAML support](https://dacurry-tns.github.io/deploying-apereo-cas/building_server_saml_overview.html)

[CAS6 配置 SAML2](https://apereo.github.io/cas/6.2.x/installation/Configuring-SAML2-Authentication.html)

[HUE 配置 SAML2](https://docs.gethue.com/administrator/configuration/server/#saml)。

现在公司使用的版本是 CAS 4.1.7 版本，由于CAS不是本部门工作内容，且公司对CAS暂无升级计划，该方案暂无法继续推进，不再深入探讨。

### 方案二：django-cas-ng
在 HUE 网站中我们没有找到关于CAS的集成方案，但是把 HUE 作为一个基于 Django 框架的普通工程，我们可以查到很多通过集成 [django-cas-ng](https://docs.djangoproject.com/en/dev/topics/auth/#writing-an-authentication-backend) 来实现集成 CAS 的方案。

#### 环境介绍

CentOS 7.2.1511

#### HUE编译过程

```
yum install epel-release
yum install npm
yum install ant asciidoc cyrus-sasl-devel cyrus-sasl-gssapi cyrus-sasl-plain gcc gcc-c++ krb5-devel libffi-devel libxml2-devel libxslt-devel make mysql mysql-devel openldap-devel python-devel sqlite-devel gmp-devel
make apps

export PREFIX=/opt/cloudera/parcels/CDH-5.13.0-1.cdh5.13.0.p0.29/lib
make install

```

#### HUE编译问题解决
- 异常信息一
  > [ERROR] Failed to execute goal on project hue-plugins: Could not resolve dependencies for project com.cloudera.hue:hue-plugins:jar:3.9.0-cdh5.13.0: The following artifacts could not be resolved: org.apache.hadoop:hadoop-client:jar:2.6.0-mr1-cdh5.13.0, org.apache.hadoop:hadoop-test:jar:2.6.0-mr1-cdh5.13.0: Could not find artifact org.apache.hadoop:hadoop-client:jar:2.6.0-mr1-cdh5.13.0 in cdh.releases.repo (https://repository.cloudera.com/content/groups/cdh-releases-rcs) -> [Help 1]  
  org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal on project hue-plugins: Could not resolve dependencies for project com.cloudera.hue:hue-plugins:jar:3.9.0-cdh5.13.0: The following artifacts could not be resolved: org.apache.hadoop:hadoop-client:jar:2.6.0-mr1-cdh5.13.0, org.apache.hadoop:hadoop-test:jar:2.6.0-mr1-cdh5.13.0: Could not find artifact org.apache.hadoop:hadoop-client:jar:2.6.0-mr1-cdh5.13.0 in cdh.releases.repo (https://repository.cloudera.com/content/groups/cdh-releases-rcs)

  解决方法：需要配置下载环境CDH库，pom.xml加上这个就好了
  ```
    <repositories>
        <repository>
            <id>cloudera</id>
            <url>https://repository.cloudera.com/artifactory/cloudera-repos/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>
  ```

- 异常信息二
  > error: Could not find suitable distribution for Requirement.parse('logilab-astng>=0.24.3')
  解决方法：
  ```
  yum install python-pip
  pip install logilab-astng
  ```
#### 代码调整
[基于django-cas-ng的HUE单点登录实现](https://blog.csdn.net/zrq010511005/article/details/51069007?utm_medium=distribute.pc_relevant.none-task-blog-title-2&spm=1001.2101.3001.4242)

遇到的问题-1
```
Q: NoReverseMatch: Reverse for 'desktop.auth.views.dt_login' with arguments '()' and keyword arguments '{}' not found. 0 pattern(s) tried: []
A:  5修改文件：urls.py    
 (r'^hue/accounts/loginx/$', 'dt_login'), 这一行不了屏蔽，loginx 写成login2 login3...也行 
```

```python
dynamic_patterns = patterns('desktop.auth.views',
  (r'^hue/accounts/loginx/$', 'dt_login'),
  #(r'^accounts/login/$', 'dt_login'),
  #(r'^accounts/logout/$', 'dt_logout', {'next_page':'/'}),
  ...
...
```
遇到的问题-2  
指定 [CAS 协议版本](https://apereo.github.io/cas/4.2.x/protocol/CAS-Protocol-Specification.html)。默认版本是2，我们用的是1。
```/opt/cloudera/parcels/CDH-5.13.0-1.cdh5.13.0.p0.29/lib/hue/desktop/core/src/desktop/setting.py```
```
CAS_VERSION = '1'
```

遇到的问题-3  
<span id='config'>修改desktop和auth变量</span>  
添加变量应该在对应的域里。CDH版本HUE添加环境变量是在[hue_safety_valve_server.ini](https://docs.cloudera.com/documentation/enterprise/5-13-x/topics/hue_adm_ini_files.html#concept_jvv_t1m_51b)
```
[desktop]
redirect_whitelist==^\/.*$,^.*\/cas\/login.*$,^.*\/cas\/logout.*$
[[auth]]
backend=desktop.auth.backend.CASBackend
```

#### 再次编译代码
其实编译过程只是编译了django_cas_ng-3.4.2-py2.7.egg文件并将其安装到了本地。我们实际修改的 4 个文件并不需要编译。

代码编译时，需要设置PREFIX环境变量，因为python编译时使用全路径。
```
export PREFIX=/opt/cloudera/parcels/CDH-5.13.0-1.cdh5.13.0.p0.29/lib
```

#### 部署安装
代码编译后，不要全量替换，按照下面的步骤将django-cas-ng的egg文件发布到线上环境即可

```shell
# 该步骤未执行  /opt/cloudera/parcels/CDH-5.13.0-1.cdh5.13.0.p0.29/lib/hue/desktop/core/ext-eggs/django_cas_ng-3.4.2-py2.7.egg
```

1. 添加```django_cas_ng```，拷贝目录内容到对应目录
```
/opt/cloudera/parcels/CDH-5.13.0-1.cdh5.13.0.p0.29/lib/hue/build/env/lib/python2.7/site-packages/django_cas_ng-3.4.2-py2.7.egg
```
编辑文件，加入第23行
```
/opt/cloudera/parcels/CDH-5.13.0-1.cdh5.13.0.p0.29/lib/hue/build/env/lib/python2.7/site-packages/easy-install.pth
```
```
  1 import sys; sys.__plen = len(sys.path)
  2 ./Paste-2.0.1-py2.7.egg
  3 ./py4j-0.9-py2.7.egg
  4 ./Django-1.6.10-py2.7.egg
  5 ./configobj-4.6.0-py2.7.egg
  6 ./sqlparse-0.2.0-py2.7.egg
  7 ./pycrypto-2.6.1-py2.7-linux-x86_64.egg
  8 ./sasl-0.1.1-py2.7-linux-x86_64.egg
  9 ./pysaml2-2.4.0-py2.7.egg
 10 ./thriftpy-0.3.9-py2.7-linux-x86_64.egg
 11 ./guppy-0.1.10-py2.7-linux-x86_64.egg
 12 ./pytz-2015.2-py2.7.egg
 13 ./enum-0.4.4-py2.7.egg
 14 ./django_auth_ldap-1.2.0-py2.7.egg
 15 ./thrift-0.9.1-py2.7-linux-x86_64.egg
 16 ./argparse-1.4.0-py2.7.egg
 17 ./Mako-0.8.1-py2.7.egg
 18 ./kazoo-2.0-py2.7.egg
 19 ./djangosaml2-0.13.0-py2.7.egg
 20 ./Markdown-2.0.3-py2.7.egg
 21 ./navoptapi-0+untagged.4944.g4b9a5db.dirty-py2.7.egg
 22 ./simplejson-2.0.9-py2.7-linux-x86_64.egg
 23 ./django_cas_ng-3.4.2-py2.7.egg
 24 ./MarkupSafe-0.9.3-py2.7-linux-x86_64.egg
 25 ./django_nose-1.3-py2.7.egg
 26 ./python_dateutil-2.4.2-py2.7.egg
```
2. 拷贝以下文件到对应目录
```
desktop/core/src/desktop/auth/backend.py
desktop/core/src/desktop/middleware.py
desktop/core/src/desktop/settings.py
desktop/core/src/desktop/urls.py
```

3. [修改desktop和auth变量](#config)


代码管理在[github](https://github.com/Aaronzk/hue/tree/origin/cdh5-3.9.0_5.13.0)

```
git push -u origin dev:origin/cdh5-3.9.0_5.13.0
```