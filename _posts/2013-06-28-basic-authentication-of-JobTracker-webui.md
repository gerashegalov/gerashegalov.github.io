---
layout: post
title: Password protection of JobTracker webUI
---

All hadoop daemons use an embedded Jetty web container to host JSP for webUI,
 e.g., currently v6.1.26 in branch-1. So the question is [how to configure
security with embedded jetty][1].

JobTracker's UI is located under something like
`${hadoop.home.dir}/webapps/job`. For example, in `webapps/job/WEB-INF/web.xml`
descriptor we can add the following to make sure that all urls are accessible
only by the role "admin" and that [basic access authentication][2] for simpe 
password password protection is used.

{% highlight xml %}
  <security-constraint>
    <web-resource-collection>
      <web-resource-name>Protected</web-resource-name>
      <url-pattern>/\*</url-pattern>
    </web-resource-collection>
    <auth-constraint>
      <role-name>admin</role-name>
    </auth-constraint>
  </security-constraint>
  <login-config>
    <auth-method>BASIC</auth-method>
    <realm-name>jtRealm</realm-name>
  </login-config>
{% endhighlight %}

Now we need to define the realm jtRealm that is referenced in web.xml. For
this, we create a new file `webapps/job/WEB-INF/jetty-web.xml`

{% highlight xml %}
<Configure class="org.mortbay.jetty.webapp.WebAppContext">
  <Get name="securityHandler">
    <Set name="userRealm">
      <New class="org.mortbay.jetty.security.HashUserRealm">
        <Set name="name">jtRealm</Set>
        <Set name="config">
          <SystemProperty name="hadoop.home.dir"/>/jetty/etc/realm.properties
        </Set>
      </New>
    </Set>
  </Get>
</Configure>
{% endhighlight %}

Here we have specified jtRealm as HashUserRealm based on the
[realm.properties file][3]: `${hadoop.home.dir}/jetty/etc/realm.properties`
Now we can create this file with the following content to define user1 as an
admin:
{% highlight properties %}
user1: pass1,admin
{% endhighlight %}

After restarting JobTracker, we will have to log in as `user0` authenticated
by password `pass1` to see the webUI.  

[1]: http://docs.codehaus.org/display/JETTY/How+to+Configure+Security+with+Embedded+Jetty
[2]: http://en.wikipedia.org/wiki/Basic_access_authentication
[3]: http://docs.codehaus.org/display/JETTY/Realms
