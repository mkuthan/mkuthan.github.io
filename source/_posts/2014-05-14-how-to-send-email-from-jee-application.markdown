---
layout: post
title: "How to send email from JEE application"
date: 2014-05-14
categories: [JMS, smtp, spring]
---

Sending email notifications from enterprise application is very common
scenario. I know several methods to solve this requirement, below you can find
short summary.  
  
Because release application binary (e.g: WAR file) should be portable across
environments (integration, QA, staging, production) configuration must be
externalized. Environment variables or JNDI entries can be used to configure
application.  
  
Sample JNDI entry for JBoss:  

``` xml
<?xml version="1.0" encoding="UTF-8"?>  
<server>  
  <mbean code="org.jboss.mail.MailService" name="jboss:service=mailSession">  
    <attribute name="JNDIName">mail/mailSession</attribute>  
    <attribute name="Configuration">  
      <configuration>  
        <property name="mail.smtp.host" value="smtp.company.com"/>  
      </configuration>  
    </attribute>  
    <depends>jboss:service=Naming</depends>  
  </mbean>  
</server>  
```

Sample JNDI entry for Tomcat:

``` xml
<?xml version="1.0" encoding="UTF-8"?>  
<Context>  
  <Resource name="mail/mailSession"   
    auth="Container"   
    type="javax.mail.Session"   
    mail.smtp.host="smtp.company.com"/>      
</Context>  
```

If you are using Spring Framework mail session could be configure as follows:

``` xml
<jee:jndi-lookup id="mailSession" jndi-name="mail/mailSession" />   
<bean id="mailSender" class="org.springframework.mail.javamail.JavaMailSenderImpl">  
  <property name="session" ref="mailSession"/>  
</bean>  
```

Now it is time for more tough part, how to use mail session correctly? There
are at least four options, choose the best one for you:  

* _Direct (Sync)_ \- use mail session directly from the application service in the web request thread.
* _Direct (Async)_ \- use mail session directly from the application service using `@Async` Spring annotation.
* _Database Queue_ \- save messages into db table and create cron job to send the emails.
* _JMS Queue_ \- put messages into JMS queue and create listener to send the emails.

|                                           |Direct (Sync)|Direct (Async)|Database Queue|JMS Queue
|-------------------------------------------|:-----------:|:------------:|:------------:|:-------:
|Application works even if the SMTP is down |no|no|yes|yes
|Web request thread is not blocked          |no|yes|yes|yes
|Mail aggregation, scheduled sending, etc.  |no|no|yes|limited
|Control over SMTP requests throttle        |no|limited|limited|yes
|Redelivery policy, do not lost messages if SMTP is down |no|no|limited|yes
|Monitoring                                 |no|no|yes|yes
  
I would start with "Database Queue" approach, if JMS is not already used in
the project. "Direct" method is not an option at all IMHO.  
  
Separate part of the subject is to how to create email body. In most situation
I have been using some template engine, like _Freemarker_ or _Thymeleaf_. The
template can be defined as internal WAR resource or can be loaded from
database if the template needs to be adjusted on runtime.
