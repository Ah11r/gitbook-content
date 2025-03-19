# Mobile Applications Security

![image](../.gitbook/assets/unsplash.jpg)

We are in the digital era where the usage of mobile devices has skyrocketted. This has caused mobile applications to be an integral part of our lives as they help us perform our daily tasks efficiently while staying connected. Thus, there has been a surge of mobile attacks.

According to a [survey by DataSparkle](https://www.datasparkle.net/home/pc/en/researchReport/index.html) that was published on [Africa Business Communities](https://africabusinesscommunities.com/africadata/africa-had-more-than-270-million-mobile-apps-users-as-of-december-2022-report/), 'the total number of monthly active users of mobile apps in Africa exceeded 270 million in December 2022, a 14% increase compared to the beginning of the year' where 'The number of active users of finance apps in Africa exceeded 73 million, an increase of 26%.\
The number of active users of shopping apps surpassed 36 million, up 6% in the same given period'

In this article, we will delve into the following

> What Mobile Application Security is\
> The importance of Mobile application security and case studies\
> Top mobile applications threats\
> Risk Factors and the attack vectors\
> Mobile application security measures to employ\
> OWASP Mobile Top 10

## What is Mobile Application Security?

Mobile Application Security is a process that employs technologies and well defined security procedures to safeguard mobile applications from cyber threats and data theft. Ideally, it encompassess identifying, fixing and enhancing the security of mobile applications. In this context, we will focus on the two competing giants - Android OS and IOS.\
The Mobile Application Security Methodology by OWASP provides a standard approach in achieving this goal

## Why is Mobile Application Security Important?

With the uprise uptake, and integration of the core operations such as banking, shopping among others into mobile applications, vast amount of sensistive and personal data is being processed by the apps. If not properly secured, this data may end up in the wrong hands hence compromising the CIA triad.\
The following are some of the specific reasons why mobile application security is important and why it should be enforced.

1. **To protect user data** - information such as financial information, personal identifiable information and health info is a gold mine and should be protected at all costs.
2. **To comply with regulations** - With the implementation of Data Protection Act 2019, compliance is at the fore-front of every organization
3. **To maintain business continuity** - Business operations are greatly affected incase of a successful attack. To avoid this, Mobile application security should be enhanced.
4. **To improve customer trust** - Gaining customers' loyalty and trust goes beyond offering them services. In this context, security is a major contributing factor to customer satisfaction and retention.

## Recent Mobile Application Threat Trends

Bearing the fact that our mobile phones are just an arm-length away means that interactions with various mobile applications happen within a very short period but the amount of data transaction that happen is massive. Attackers however don't slumber while curating new tactics. Some of the top threat trends seen in the recent past include

* Phishing attacks - this remains to be one of the persistent threats across the years. Attackers having been using Emails, text messages and even voice to achieve this goal. Mobile device users are very susceptible to these attacks since the email apps display less information on the screen to accomodate the screen size
* Data leaks - This happen due to unregulated permissions granted to mobile applications when installing them. The attackers leverage these permissions collect
* Open(unsecured) WIFI - attackers set up unprotected WIFI networks luring unsuspecting users to connect to them. This however leads to interception of traffic stealing sensitive data. While an unprotected WIFI maybe tempting to connect to, it is safe and important to avoid them.
* Spyware - a spyware is a type of malware that collects information about a user's activity without the knowledge of the user. To mitigate this, users should avoid clicking on suspicious links and regularly scan their devices
* Apps with Malicious code - malicious code is embedded in the apps donloaded mainly from third-party sources. This code is meant to steal data, destroy and even take over the device. Users should only install apps from trusted sources.

## What are the risk factors and the underlying attack vectors?

Often, we as the users facilitate some of the attacks knowingly or unknowingly. Some of the underlying risk factors that are related to mobile appliation security include

* Mobile apps vulnerabilities - mobile applications can be prone to a host of weaknesses which attackers leverage to perform malicious activities. It is vital to ensure that the application and the integrations are upto date and the appropriate security patches are applied.
* Third Party libraries - while third party libraries are often used in mobile apps to add functionality or reduce development time. However, they also introduce security vulnerabilities into apps. It is important to carefully choose and vet third party libraries, and to keep them up to date with the latest security patches
* Authentication and authorization - use strong authentication mechanisms, such as strong passwords and multi-factor authentication. Mobile apps should also implement fine-grained authorization controls to ensure that users only have access to the data and resources they need.
* Data encryption - to protect sensitive data stored on mobile devices or transmitted over networks. Mobile apps should encrypt all sensitive data, including passwords, credit card numbers, and personal identification information.
* User behaviour - some of user behaviour that contribute in compromising mobile application includes downloading and installing malicious apps, clicking on malicious links, or entering sensitive information into suspicious websites

## Mobile application security measures to employ

Protecting mobile applications is a shared responsibility between the users and developers. Users have the great responsibility since they interact with the applications more. Some of the protective measures include

* Using the official app store
* Read and regularly review the App permissions
* Update apps regularly
* Employ strong authentication measures
* Use Security apps
* Back up data
* Use VPNs when connecting to public networks
* Report Suspicios activity
* Stay informed

## Mobile Application Testing

Employing protective measures on the user side is just not enough. A mobile application assessment is crucial and essential in enhancing the security of the application. The process of mobile application testing involves

* Testing vulnerabilities through simulated attacks to assess the security strengths and weaknesses of your app.
* Analyzing internal controls and examine the code to investigate potential malware and danger.
* Monitoring the application interface and infrastructure to locate any security flaws.
* Improving security posture and crafting an actionable security plan with expert guidance.

This whole process is guided by the [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/) and the [Owasp MAS - Mobile Application Security](https://owasp.org/www-project-mobile-top-10/)\
The OWASP Mobile Top 10 details the following common android application vulnerabilities that should be tested

* Improper Platform Usage: Apps might misuse Android platform features, leading to security issues. For example, not using secure network communication.
* Insecure Data Storage: Sensitive data, like passwords, may be stored improperly, making it accessible to attackers.
* Insecure Communication: Poorly secured data transmission can expose data to eavesdropping. Apps should use SSL/TLS for secure communication
* Insecure Authentication: Weak authentication mechanisms or poor session management can lead to unauthorized access.
* Insufficient Cryptography: Weak encryption or misconfigured cryptography can expose sensitive data to attackers.
* Insecure Authorization: Improperly enforced authorization rules can allow unauthorized access to app functionality or data.
* Client Code Quality: Inadequately tested or poorly coded client-side security measures can be exploited by attackers.
* Code Tampering: Attackers may reverse engineer the app, modify it, and redistribute a malicious version.
* Reverse Engineering: Reverse engineering and decompilation can reveal sensitive information or expose vulnerabilities.
* Extraneous Functionality: Unused or debug code left in the app can be exploited, leading to security breaches

I hope this article was insightful. Read more on the rising mobile banking fraud [here](https://yelbridges.co.ke/mobile-banking-fraud-rising-in-kenya/)
