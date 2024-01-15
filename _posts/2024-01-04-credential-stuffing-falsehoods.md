---
layout: post
title:  "About password security and credential stuffing"
---

# Common Falsehoods About Password Security and Credential Stuffing
TechCrunch published an [article](https://techcrunch.com/2024/01/03/23andme-tells-victims-its-their-fault-that-their-data-was-breached) that gives class-action lawyers suing 23andMe a mouthpiece to editorialize about password security practices, masquerading as a news article. The upshot of the ~article~ editorial is this:

1. A large number of 23andMe's customers were subject to a credential stuffing attack.
2. Design choices in 23andMe's site allowed other customer data to be exposed via accounts compromised in the credential stuffing attack.
3. 23andMe's mitigations for credential stuffing attacks were inadequate.
4. 23andMe should be held liable for this.

I want to focus here on the third point: credential stuffing attack mitigations. I've worked quite a bit on analyzing large credential stuffing attacks and recommending mitigations for them. I also served as a technical escalation point for customers who had a wide variety of strongly held false beliefs about password security credential stuffing mitigation. In reading various social media responses the 23andMe case, I see all these false beliefs turning up again. Let's have a look at some.

## False: Rate limiting login attempts is a great mitigation
The first question we need to answer here is: what do you mean by rate limiting? Usually there are two main rate limits that people consider: per-user and per-IP address. For example, you could allow a maximum of N failed logins per username during time window T1, after which point you allow no logins for that username until time window T2 expires. The intention here is to prevent an attacker who has a valid username from trying a large list of passwords for that username. The advantage to this is that for a given single target user, it makes large scale credential stuffing slow. The problems are substantial, however:

For an attacker with a large number of valid usernames, they can spread their attempts across the username list at a rate just below the limit for any given single user. For example, if the lockout trigger is 10 failures in 1 minute with a lockout window of 10 minutes and the attacker has 10,000 valid usernames, they can try 9 passwords per user per minute without ever triggering a lockout, or 90,000 tries per minute across their target list. If they do trigger a lockout, they can just come back in 10 minutes.

The more serious problem with per-user rate limiting is its utility as a denial-of-service attack against the legitimate account owner: a harrassment-oriented attacker need only trigger the required number of intentional failures in order to temporarily lock out the legitimate owner. This can continue indefinitely.

The next response to these problems is usually to propose per-IP-address rate limiting: a given address is only allowed a certain number of failures or a certain number of logins on different accounts in a given time window before being blocked. Ten years ago, this was pretty effective. Today, it's much less effective due to the widespread availability of high-quality proxy services to attackers. It's now very common for an attacker to be able to cycle through tens of thousands of different IP addresses in a short period of time, with some of those addresses also carrying legitimate traffic. The widespread use of carrier-grade NAT complicates this even further: with widespread NAT, you often end up blocking legitmate user traffic that just happens to share an IP address with the attacker at a point in time.

You can also combine the two: allow only N failures per address A before imposing restrictions. This might work, but requires that the application track a lot of state, and also requires a predictable distribution of normal behavior. As a service grows, the compliexity of managing this increases quickly. Blocking legitimate traffic leads to angry customers and increased support tickets.

The truth is that you can probably come up with a rate-limiting policy that will help to mitigate unsophisticated credential stuffing attacks, which helps a bit. But is it a killer app? No.

## False: Most credential stuffing attacks use "brute force" techniques
Again, it depends on what you mean by "brute force". It's true that sites see combinatoric attacks against the most common usernames, such as:

- Administrative usernames ("admin", "root", etc)
- Very popular names in different languages ("tom", "antonio", etc)
- "OG" usernames ("s3x", "l33t", etc)

A few hundred usernames like these will see constant low volume dictionary-style attacks. But it's uncommon to see a large dictionary-style attack against a username chosen at random. Instead, the vast majority of credential-stuffing attacks use so-called "combolists", which are lists of username/email and password pairs that are taken from password database leaks or malware dumps, and traded in the criminal underground. This technique relies on users' proclivity to reuse passwords across sites, or to change their password back to a previously compromised one after a forced reset. This is likely the technique used in the 23andMe attack.

## False: A stolen unique password means a compromised site
One of the most common accusations I saw form customers who had compromised accounts went something like this:

> I use unique, random passwords for EVERY site. Yet an attacker logged into my account interactively. This means with CERTAINTY that your site is compromised and my password was stolen from YOU, and that you OBVIOUSLY store user passwords in clear text.

Sigh. No, what this means is that almost certainly, a system you use was compromised by malware and your cleartext password stolen from your browser or another application cache. This kind of malware is incredibly common on Windows systems and is becoming more common on other operating systems over time. It's almost certainly the largest source of net-new credential data as of this writing. While malware-sourced credential feeds usually require criminals to pay for them in cryptocurrency, some of these credentials will become incorporated into more widely distributed combolists over time as well.

Worse, stealer malware also captures browser session cookies, which enable an attacker to hijack an authenticated web session, bypassing multifactor authentication.

## What is to be done?
The attorney in the TechCrunch editorial seems to be arguing that multifactor authentication (MFA) should be mandatory. This is not an insane argument, but it comes with a large set of additional questions that need answers:

- Should all sites, no matter how small, be required to use MFA? If not, what is the bar?
- What types of MFA should be mandated? It's beyond the scope of this blog post, but each MFA method has advantages and disadvantages.
- If a company mandates MFA, should they be allowed to outsource it? Should they be _required_ to outsource it? If so, how do we ensure the security of the outsourced provider, since we've now concentrated risks there?
- What kind of account recovery should be mandated?
- What penalties and reporting requirements should be enforced?

I'm not opposed to mandatory MFA in certain contexts, but I also don't have perfect -- or even good -- answers to these questions.

Another option is to try to make password-only security better. The best version of this is email verification, sometimes called "MFA Lite":

- After signup or a successful login, the site sets a long-lived "device cookie" in the user's browser. When the user logs in from a new device, they are required to click a link in an email in order to acknowledge the new device, at which point a new cookie is saved on that device. This doesn't help prevent targeted account takeovers leveraging stealer malware (because the attacker will typically just steal the device cookie and the user's email credentials anyway), but it helps a lot with standard combolist credential stuffing; it's common for account takeovers to drop by 90% or more when this technique is implemented.

- Another option is to subscribe to a service that prevents users from setting passwords that show up in combolists. There are any number of companies that collect combolists from the criminal underground and sell this sort of service. No company will ever have complete coverage, but this can work fairly well. The drawbacks are a) it's expensive, b) it requires a fair amount of additional authentication code, and c) it leads to more support tickets when users get confused by it.
