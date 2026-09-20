---
layout: post
title: "Why Cyber Criminals Can Feel Safe Committing Crime?"
date: 2026-09-20 00:00:00 +0100
categories: [Investigations, Underground, Cybercrime]
tags: [cybercrime, adversary]
---

Cybercrime is often discussed in terms of vulnerabilities, malware, stolen credentials, and the techniques attackers use to compromise their targets. But the technical side is only part of the picture. Behind many successful cybercrime operations is a much broader ecosystem shaped by money, anonymity, infrastructure, and, perhaps most importantly, jurisdiction.

For this investigation, I wanted to look beyond the technical details and examine the factors that allow cybercriminals to operate with relatively little fear of consequences. I explored underground forums, Malware-as-a-Service platforms, leaked discussions, and other sources to understand how cybercriminals perceive risk, what motivates them, and how the legal and geopolitical environment can influence their behavior.

One of the most interesting factors is the gap between where a cybercrime is committed and where the person responsible is physically located. A hacker can target victims thousands of kilometers away while remaining in a country where the legal consequences may be very different from those faced by the victim. This creates a complicated intersection between cybercrime, national jurisdiction, extradition, and international law.

## The Jurisdictional Safe Haven
Certain legislation can indirectly create incentives for cybercriminals to carry out attacks internationally without facing prosecution in their home countries. Two of the most well-known examples are Russia and the DPRK, but there are additional complexities to consider.

Many countries do not extradite their citizens, even when those individuals commit crimes against entities in other countries. When the alleged offenses are not committed domestically, this can create a situation where the perpetrators face neither local charges nor extradition to the countries where the crimes were committed.

One example often discussed in this context involves the individuals behind the notorious RATs njRAT and hWorm. U.S. authorities brought charges against individuals associated with these malware families, but the cases also illustrate the difficulties of pursuing cybercrime across jurisdictions. The specific legal circumstances and outcomes of these cases should be distinguished from the broader issue of extradition and domestic prosecution.

From this, we can see how cybercriminals operating from certain jurisdictions may believe they can target foreign organizations, steal money, or sell stolen data while remaining outside the reach of the victim country's law enforcement. However, this does not necessarily mean they are immune from prosecution: they may still face domestic charges, international arrest warrants, or arrest and extradition if they travel to a country willing and legally able to cooperate with the requesting state.

A recent example is the [arrest](https://www.justice.gov/usao-wdny/pr/algerian-man-arrested-extradited-united-states-his-role-black-market-fraud-conspiracy) of the Algerian threat actor SPOX while he was in Spain. According to the U.S. Department of Justice, Abdellah Belmili, also known as SPOX, was arrested in Spain and extradited to the United States in June 2026 to face charges related to an alleged black-market fraud conspiracy.

## When Malware Developers Relocate

I have observed that some threat actors who develop and distribute malware while presenting their tools as legitimate security products appear to relocate to jurisdictions where the legal framework may be more permissive or where enforcement is more difficult. These actors often claim that their software is intended for educational purposes, authorized penetration testing, red-team operations, or legitimate remote administration, even when the same tools are marketed or distributed within criminal ecosystems.

This creates an interesting distinction between the stated purpose of a tool and the way it is actually developed, distributed, and used. A developer operating in a jurisdiction with strict laws against malware development or distribution may face significant legal consequences, while relocating to another jurisdiction can potentially make prosecution more difficult.

This phenomenon extends beyond RATs to other forms of malware, including information stealers, botnets, loaders, ransomware, and other offensive tooling. In some cases, the location of the developer or operator can become almost as important as the technical infrastructure they operate, particularly when international cooperation, extradition, and differences in national legislation come into play.

The Remcos RAT case provides a contemporary example of these legal challenges. Public reporting and research have associated the people behind Remcos with Europe, including Italy and Germany, while more recent observations have pointed toward Hong Kong as a possible base of operations. The official Remcos website continues to market the software as a legitimate remote-administration and security tool, while also offering paid licenses and accepting conventional payment methods alongside cryptocurrency.

The case is particularly interesting because Remcos has long occupied a controversial space between legitimate remote-administration software and malware used by threat actors. Its developers explicitly promote uses such as remote administration, security auditing, and red-team operations, while security researchers have documented Remcos being used in malicious campaigns.

## Cryptocurrency and the Fragmentation of Payments

Cryptocurrency has also changed how cybercriminals receive and move money. One important feature is the ability to generate large numbers of wallet addresses without opening a traditional bank account for each one. A criminal operation can generate a unique address for different victims, campaigns, affiliates, or transactions, allowing incoming payments to be separated at the wallet level.

This becomes particularly useful for Malware-as-a-Service and other criminal business models. For example, an operator can assign a different cryptocurrency address to each customer or affiliate. Payments can then be associated with a particular client without requiring the operator to expose a conventional identity or maintain a separate bank account for every relationship. Depending on the cryptocurrency and wallet architecture, generating new addresses can be inexpensive and largely automated.

This does not make cryptocurrency transactions inherently anonymous. Most major blockchains maintain a publicly accessible transaction history, meaning investigators can often trace the movement of funds between addresses. The challenge is connecting those addresses to real-world individuals and following funds as they move through multiple wallets, exchanges, bridges, and other services.

For cybercriminals, the value therefore comes less from complete anonymity and more from pseudonymity, automation, and fragmentation. A single operation can involve hundreds or thousands of addresses, creating additional investigative work and allowing financial relationships between victims, affiliates, developers, and operators to be separated across the blockchain. Combined with international jurisdictions and cryptocurrency services operating across borders, this can make attribution and prosecution considerably more complicated.
