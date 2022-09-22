---
title: "How to send email from JEE application"
date: 2013-12-06
tags: [Architecture, Spring]
---

Sending email notifications from enterprise application is very common scenario. 
I know several methods to solve this puzzle, below you can find short summary.  
  
To send an email from the application at least SMTP server address must be configured. 
Because released application binary (e.g: WAR file) should be portable across environments (integration, QA, staging, 
production) configuration must be externalized.  
Below I present code snippets to configure SMTP server address as JNDI entry.  
  
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

When mail session is configured as JNDI resource, it can be easily utilized by Spring Framework mail sender:

``` xml
<jee:jndi-lookup id="mailSession" jndi-name="mail/mailSession" />

<bean id="mailSender" class="org.springframework.mail.javamail.JavaMailSenderImpl">  
  <property name="session" ref="mailSession"/>  
</bean>  
```

Now it's time for more tough part, how to use mail sender correctly? 
There are at least four options, choose the best one for you:  

* _Direct (Sync)_ Use mail session directly from the application service in the web request thread.
* _Direct (Async)_ Use mail session directly from the application service using `@Async` Spring annotation.
* _Database Queue_ Save messages into database table and create cron job to send the emails periodically.
* _JMS Queue_ Put messages into JMS queue and attach JMS listener to process and send emails.

I collected a few non-functional and functional common requirements together with short categorization for each method. 

|                                           |Direct (Sync)|Direct (Async)|Database Queue|JMS Queue
|-------------------------------------------|:-----------:|:------------:|:------------:|:-------:
|Application works even if the SMTP is down |no|no|yes|yes
|Web request thread isn't blocked          |no|yes|yes|yes
|Mail aggregation, scheduled sending, etc.  |no|no|yes|limited
|Control over SMTP requests throttle        |no|limited|limited|yes
|Redelivery policy, don't lost messages if SMTP is down |no|no|limited|yes
|Monitoring                                 |no|no|yes|yes
  
I would start with "Database Queue" approach, at least if JMS isn't already used in the project or you don't have to send thousands of emails. 
"Direct" method isn't an option at all IMHO.  
  
Separate part of the subject is to how to create email body. In most situation
I used some template engine, like _Freemarker_ or _Thymeleaf_. The
template can be defined as internal WAR resource or can be loaded from
database if the template needs to be adjusted on runtime.
