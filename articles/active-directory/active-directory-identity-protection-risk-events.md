---
title: Azure Active Directory risk events | Microsoft Docs
description: This topic gives you a detailed overview of what risk events are.
services: active-directory
keywords: azure active directory identity protection, security, risk, risk level, vulnerability, security policy
author: MarkusVi
manager: femila

ms.assetid: ce3b054c-7772-401f-9b06-3d24f6e7ca86
ms.service: active-directory
ms.workload: identity
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 02/22/2017
ms.author: markvi

---
# Azure Active Directory risk events

Azure Active Directory helps you to protect your identities. One part of the protection process is the detection of suspicious actions that are related to your user accounts. The detection is based on adaptive machine learning algorithms and heuristics. Each detected suspicious action is stored in a record called *risk event*. 


![Risk event](./media/active-directory-identity-protection-risk-events/91.png)


Currently, Azure Active Directory detects six types of risk events.

| Risk Event Type | Risk Level | Detection Type |
| :-- | --- | --- |
| [Users with leaked credentials](#leaked-credentials) | High | Offline |
| [Sign-ins from anonymous IP addresses](#sign-ins-from-anonymous-ip-addresses) | Medium | Real-time |
| [Impossible travel to atypical locations](#impossible-travel-to-atypical-locations) | Medium | Offline |
| [Sign-ins from unfamiliar locations](#sign-in-from-unfamiliar-locations) | Medium | Real-time |
| [Sign-ins from infected devices](#sign-ins-from-infected-devices) | Low | Offline |
| [Sign-ins from IP addresses with suspicious activity](#sign-ins-from-ip-addresses-with-suspicious-activity) | Medium | Offline|

## Risk level

The risk level property is an indicator (High, Medium, or Low) for the severity of a risk event. This property helps you to prioritize the actions you must take. 

The severity of the risk event represents the strength of the signal as a predictor of identity compromise, combined with the amount of noise that it typically introduces.

* **High**: High confidence and high severity risk event. These events are strong indicators that the user’s identity has been compromised, and any user accounts impacted should be remediated immediately.

* **Medium**: High severity, but lower confidence risk event, or vice versa. These events are potentially risky, and any user accounts impacted should be remediated.

* **Low**: Low confidence and low severity risk event. This event may not require an immediate action, but when combined with other risk events, may provide a strong indication that the identity is compromised.

![Risk Level](./media/active-directory-identity-protection-risk-events/01.png)


## Detection type

The detection type property is an indicator (Real-time, Offline) for the detection timeframe of a risk event.  
Currently, most risk events are detected offline in a post-processing operation after the risk event has occurred.

The following table lists the amount of time it takes for a detection type to show up in a related report.

| Detection Type | Reporting Latency |
| --- | --- |
| Real-time | 5 to 10 minutes |
| Offline | 2 to 4 hours |



## Risk event types

The risk event type property is an identifier for the suspicious action a risk event record has been created for.  
Currently, Azure Active Directory detects six event types.

Microsoft's continuous investments into the detection process lead to:

- Improvements to the detection accuracy of existing risk events 
- New risk event types that will be added in the future

### Leaked credentials
Leaked credentials are found posted publicly in the dark web by Microsoft security researchers. These credentials are usually found in plain text. They are checked against Azure AD credentials, and if there is a match, they are reported as “Leaked credentials” in Identity Protection.

Leaked credentials risk events are classified as a “High” severity risk event, because they provide a clear indication that the user name and password are available to an attacker.

### Sign-ins from anonymous IP addresses
This risk event type identifies users who have successfully signed in from an IP address that has been identified as an anonymous proxy IP address. These proxies are used by people who want to hide their device’s IP address, and may be used for malicious intent.

We recommend that you immediately contact the user to verify if they were using anonymous IP addresses. The risk level for this risk event type is “**Medium**” because in itself an anonymous IP is not a strong indication of an account compromise.


### Impossible travel to atypical locations
This risk event type identifies two sign-ins originating from geographically distant locations, where at least one of the locations may also be atypical for the user, given past behavior. In addition, the time between the two sign-ins is shorter than the time it would have taken the user to travel from the first location to the second, indicating that a different user is using the same credentials. 

This machine learning algorithm that ignores obvious "*false positives*" contributing to the impossible travel condition, such as VPNs and locations regularly used by other users in the organization.  The system has an initial learning period of 14 days during which it learns a new user’s sign-in behavior.

Impossible travel is usually a good indicator that a hacker was able to successfully sign-in. However, false-positives may occur when a user is traveling using a new device or using a VPN that is typically not used by other users in the organization. Another source of false-positives is applications that incorrectly pass server IPs as client IPs, which may give the appearance of sign-ins taking place from the data center where that application’s back-end is hosted (often these are Microsoft datacenters, which may give the appearance of sign-ins taking place from Microsoft owned IP addresses). As a result of these false-positives, the risk level for this risk event is “**Medium**”.

### Sign-in from unfamiliar locations
This risk event type is a real-time sign-in evaluation mechanism that considers past sign-in locations (IP, Latitude / Longitude and ASN) to determine new / unfamiliar locations. The system stores information about previous locations used by a user, and considers these “familiar” locations. The risk even is triggered when the sign-in occurs from a location that's not already in the list of familiar locations. The system has an initial learning period of 14 days, during which it does not flag any new locations as unfamiliar locations. The system also ignores sign-ins from familiar devices, and locations that are geographically close to a familiar location. 

Unfamiliar locations can provide a strong indication that an attacker is able attempting to use a stolen identity. False-positives may occur when a user is traveling, trying out a new device or uses a new VPN. As a result of these false positives, the risk level for this event type is “**Medium**”.



### Sign-ins from infected devices
This risk event type identifies sign-ins from devices infected with malware, that are known to actively communicate with a bot server. This is determined by correlating IP addresses of the user’s device against IP addresses that were in contact with a bot server. 

This risk event identifies IP addresses, not user devices. If several devices are behind a single IP address, and only some are controlled by a bot network, sign-ins from other devices my trigger this event unnecessarily, which is the reason for classifying this risk event as “**Low**”.  

We recommend that you contact the user and scan all the user's devices. It is also possible that a user's personal device is infected, or as mentioned earlier, that someone else was using an infected device from the same IP address as the user. Infected devices are often infected by malware that have not yet been identified by anti-virus software, and may also indicate as bad user habits that may have caused the device to become infected.

For more information about how to address malware infections, see the [Malware Protection Center](http://go.microsoft.com/fwlink/?linkid=335773&clcid=0x409).


### Sign-ins from IP addresses with suspicious activity
This risk event type identifies IP addresses from which a high number of failed sign-in attempts were seen, across multiple user accounts, over a short period of time. This matches traffic patterns of IP addresses used by attackers, and is a strong indicator that accounts are either already or are about to be compromised. This is a machine learning algorithm that ignores obvious "*false-positives*", such as IP addresses that are regularly used by other users in the organization.  The system has an initial learning period of 14 days where it learns the sign-in behavior of a new user and new tenant.

We recommend that you contact the user to verify if they actually signed in from an IP address that was marked as suspicious. The risk level for this event type is “**Medium**” because several devices may be behind the same IP address, while only some may be responsible for the suspicious activity. 





## Azure AD anomalous activity reports

In the Azure classic portal, some of the risk events already have been made available through the Azure AD Anomalous Activity reports. 

The table below lists the various risk event types and the corresponding **Azure AD Anomalous Activity** report. 

| Identity Protection Risk Event Type | Corresponding Azure AD Anomalous Activity Report |
|:--- |:--- |
| Leaked credentials |Users with leaked credentials |
| Impossible travel to atypical locations |Irregular sign-in activity |
| Sign-ins from infected devices |Sign-ins from possibly infected devices |
| Sign-ins from anonymous IP addresses |Sign-ins from unknown sources |
| Sign-ins from IP addresses with suspicious activity |Sign-ins from IP addresses with suspicious activity |
| Signs in from unfamiliar locations |- |


The following Azure AD Anomalous Activity reports are not included as risk events in Azure AD Identity Protection, and will therefore not be available through Identity Protection. These reports are still available in the Azure classic portal. However, they will be deprecated at some time in the future as they are being superseded by risk events in Identity Protection.

* Sign-ins after multiple failures
* Sign-ins from multiple geographies






## Next steps
* [Azure Active Directory Identity Protection](active-directory-identityprotection.md)

