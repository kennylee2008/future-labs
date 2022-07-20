---
title: How to use ACME4J to get SSL certificate automatically
date: 2021-09-21 14:46:40
tags:
  - acme4j
  - letsencrypt
  - ssl certificate
---

# Why

After you have created a website, an SSL certificate has become a mandatory configuration in order to publish it to the Internet. [Let's Encrypt](https://letsencrypt.org/getting-started/) can issue a certificate for your website’s domain. When you want to get an SSL certificate from Let's Encrypt, you have a number of tools to choose from. The most common tool is Certbot. If you are applying for an SSL certificate for the first time, you may have some trouble using this tool. This is because it is very powerful and there are so many options available for configuration. Especially when I want to generate a wildcard type certificate, there are so many manual steps that I may have to reapply every three months. I like automated tools. My DNS provider is alibaba cloud, so I started searching the web for an automated tool that could automatically request wildcard type letsencrypt certificates for alibaba cloud. But I was disappointed that I didn't find the tools I wanted. Therefore, I decided to look for a relevant library and prepare to write one by myself.

<!-- more -->

# The idea

If you want to implement a process to automatically obtain certificates, then you first need to understand its process. That's the idea:
![How to get certificate from Letsencrypt](2021-09-21T155356.png)

# The implementation 

I found the acme4j library. It is very clear about how to get a certificate from Letsencrypt. 

## Start a session

```java
// Create a session for Let's Encrypt.
// Use "acme://letsencrypt.org" for production server
// Use "acme://letsencrypt.org/staging" for test server
Session session = null;
if(stage == STAGE.PROD) {
    session = new Session("acme://letsencrypt.org");
}else {
    session = new Session("acme://letsencrypt.org/staging");
}
```

## Create an account

```java

//Create user account key
KeyPair userKeyPair = KeyPairUtils.createKeyPair(KEY_SIZE);

//Create an account
Account account = new AccountBuilder().agreeToTermsOfService().useKeyPair(userKeyPair).create(session);

```

## Challenge

![Challenge Steps](2021-09-21T162922.png)

* The **Challenage.TYPE** in **step 3** are [DNS or HTTP](https://letsencrypt.org/docs/challenge-types/).
* The **step 4** is exactly the part I want to improve. It required you to manually log into your domain service provider to add TXT type domain values for the Let's Encrypt server to initiate a challenge. The problem with this is that you still have to repeat the process of manually adding challenge values when the certificate is about to expire after three months.

### Turn step 4 into automation

There are two keys to turning step 4 into automation.
* The first is to automatically write a text value, via the API provided by the DNS Provider, to the appropriate domain name. This step varies depending on the API provided by each DNS Provider. If you want to know how to call alibaba's API, you can refer to the source code link at the end of this article. The sample code is here:

```java

@Override
public void addTxtValueToDomain(String domainName, String txtValue, String rR) {
    IClientProfile profile =  DefaultProfile.getProfile(
            "cn-hangzhou",                     //Region ID
            this.accessKey,             // AccessKey ID
            this.accessSecret);        // AccessKey Secret
    
    AddDomainRecordRequest request = new AddDomainRecordRequest();
    request.setActionName("AddDomainRecord");
    request.setDomainName(domainName);
    request.setRR(rR);
    request.setType("TXT");
    request.setValue(txtValue);
    
    IAcsClient client = new DefaultAcsClient(profile);
    try {
        AddDomainRecordResponse response = client.getAcsResponse(request);
        this.recordId = response.getRecordId();
        LOG.info("Set TXT value to " + rR + "." + domainName + " has been completed.");
    } catch (Exception e) {
        throw new RuntimeException("Failed to set TXT value to " + rR + "." + domainName,e);
    }
}

```

* Second, automatically determine if the text value you set is already in effect. Then, after it has taken effect, go ahead and trigger the Letsencrypt challenge process. I used a library called _dnsjava_ to implement this step.

```java

package com.futlabs.letsencrypt;

import org.xbill.DNS.DClass;
import org.xbill.DNS.Lookup;
import org.xbill.DNS.SimpleResolver;
import org.xbill.DNS.Type;

public class DnsLookupHelper {
	
	public static boolean isValid(String domain,String digest) {
		
		try {
			SimpleResolver res = new SimpleResolver();
			res.setTCP(true);
			Lookup l = new Lookup("_acme-challenge."+domain, Type.TXT, DClass.IN);
			
			l.run();
			if (l.getResult() == Lookup.SUCCESSFUL) {
			    String r = l.getAnswers()[0].rdataToString();
			    if(r != null) {
			    	if(r.contains("\"")) {
			    		r = r.replaceAll("\"", "");
			    	}
			    }
			    if(digest.equals(r)) {
			    	return true;
			    }
			}
		} catch (Exception ignore) {
		}
		return false;
	}
}

```

## Create a certificated signing request(CSR)

```java

//Create domain key
KeyPair domainKeyPair = KeyPairUtils.createKeyPair(KEY_SIZE);

//Build the CSR
CSRBuilder csrb = new CSRBuilder();
csrb.addDomains(domains);
csrb.sign(domainKeyPair);
byte[] csr = csrb.getEncoded();

```

[What is a Certificate Signing Request (CSR)?](https://www.globalsign.com/en/blog/what-is-a-certificate-signing-request-csr)

## Get certificate

```java

//Request for a certificate
order.execute(csr);

// Wait for the order to complete
try {
    int attempts = 50;
    while (order.getStatus() != Status.VALID && attempts-- > 0) {
        // Did the order fail?
        if (order.getStatus() == Status.INVALID) {
            LOG.error("Order has failed, reason: {}", order.getError());
            throw new AcmeException("Order failed... Giving up.");
        }

        // Wait for a few seconds
        Thread.sleep(3000L);

        // Then update the status
        order.update();
    }
} catch (InterruptedException ex) {
    LOG.error("interrupted", ex);
    Thread.currentThread().interrupt();
}

// Get the certificate
Certificate certificate = order.getCertificate();

```

You just need to write the certificate and the domain key to the files.



# Practices

Yes, I have written a complete working application. I only support using Alibaba Cloud as DNS provider. And only used DNS as the challenge type. If you want to support more DNS providers or other challenge types, it would be a very easy thing to do as long as you are familiar with the Java language.

The following are the related projects in Github:
* [kennylee2008/letsencrypt-alibaba](https://github.com/kennylee2008/letsencrypt-alibaba) The source code of the project mentioned in this article.
* You can run with this [kennylee2008/letsencrypt-alibaba](https://hub.docker.com/r/kennylee2008/letsencrypt-alibaba) docker image right now.
* If you prefer to use docker-compose, you can clone this project [kennylee2008/letsencrypt-alibaba-docker](https://github.com/kennylee2008/letsencrypt-alibaba-docker), modify the environment variables, and run it right away.
