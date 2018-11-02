---
layout: page
title: About
permalink: /about/
---

Welcome to `ph4r05` dev blog. 

## Dušan Klinec (Ph4r05)
- {% include icon-github.html username="ph4r05" %} GitHub profile
- {% include icon-twitter.html username="ph4r05" %} Twitter profile
- [LinkedIn](https://www.linkedin.com/in/dklinec) profile


## Projects

### [deadcode.me](https://deadcode.me)
Our security related blog with topics on router firmware reverse engineering, deserialization vulnerabilities and more...

### [Monero Trezor integration](https://github.com/ph4r05/monero-agent)
Monero transaction signing implemented to the [Trezor](https://trezor.io) [hardware wallet](https://github.com/trezor/trezor-core). 

I've designed transaction signature [protocol](https://github.com/ph4r05/monero-trezor-doc) suitable for use with Trezor hardware wallet which is 
simple and easy to analyze.

I've later implemented the protocol in C ([trezor-crypto extensions](https://github.com/ph4r05/trezor-crypto)) and Micropython to the [hardware wallet](https://github.com/trezor/trezor-core) codebase. I've implemented native Monero C++ binding, currently as a [PR](https://github.com/monero-project/monero/pull/4241).

As a part of this project I've implemented the python version of the both wallet and device versions in the [monero-agent](https://github.com/ph4r05/monero-agent)
which may serve for educational purposes and further research prototyping. For this I need to implement:
 - Several serialization schemes used in the Monero (blockchain format, Boost, RPC/key-value format) in the [monero-serialize](https://github.com/ph4r05/monero-serialize) python library. 
 - [py-trezor-crypto](https://github.com/ph4r05/py-trezor-crypto) python library which provides python binding to the [trezor-crypto](https://github.com/ph4r05/trezor-crypto) cryptographic library. 
 - [py-cryptonight](https://github.com/ph4r05/py-cryptonight) Python binding for cryptonight PoW function

 - Port Bulletproofs, Borromean and MLSAG algorithms from C++ to Python, optimize it for memory constrained environment. 

### [ROCA - Return of the Coppersmith attack](https://crocs.fi.muni.cz/public/papers/rsa_ccs17)
I was part of the team working on the ROCA attack (known for affecting eID in Estonia and Slovakia). 
  - Performed data collection, scanning and analysis. 
  - Discovered that Estonia was still vulnerable in August 2017 by scanning and analysing public keys database. Our notification helped them to address problem prior the public disclosure. 
  - Authored the [roca detector](https://github.com/crocs-muni/roca) - versatile detection tool. 

### [keychest.net](https://keychest.net)
Certificate expiry, certificate monitoring for TLS, HTTPS, Let's Encrypt, with free cloud service. Automatic monitoring of subdomain servers as they are set up.

I've implemented the first KeyChest version based on Python backend daemon scanning, crawling and processing X509 certificates, performing analysis and storing results to the database.

Frontend was based on PHP Laravel and Vue.js with responsive elements. I've authored several NPM packages used in the UI. Backend and frontend communicated via Redis queues and database. Responsive UI was implemented using advanced Javascript and websockets.  

Technologies: python, redis, mysql/pgsql, alembic, sqlalchemy, flask, gevent, websockets, roca-detector, php, laravel, vue.js, vuex, webpack, npm, promises, acacha, admin-lte  

### [phone-x.net](https://www.phone-x.net/)
Secure mobile communication system.

* End-to-end encrypted voice calls, text messages, file transfer.
* Perfect forward secrecy.
* ZRTP, AES-256.
* [Android application](https://play.google.com/store/apps/details?id=net.phonex)
* [iOS application](https://itunes.apple.com/us/app/phonex-secure-communication/id957487057?mt=8)
* Used backend technologies: 
  * Java/Spring based servers, PHP/Laravel license server, ActiveMQ messaging, 
  * XMPP Server Openfire + our plugin for signalling over XMPP and push messages integration (GCM, iOS). 
  * OpenSips server + custom [msilo](https://github.com/ph4r05/msilo) plugin for reliable message delivery over unstable mobile links. I've controbuted to OpenSips by fixing several vulnerabilities found by Coverity. 

### [EnigmaLink.io](https://enigmalink.io)

* Very simple to use end-to-end encrypted file sharing web service.
* Works from the browser, JavaScript only, no special apps needed.
* File is encrypted in the browser, uploaded to your GoogleDrive.
* Enables to attach text message.
* AES-256-GCM, per-file encryption keys.
* Enables to set per-file password (optional).
* Uses hardware powered cloud encryption service [EnigmaBridge](https://enigmabridge.com/) to increase protection level and protect from bruteforce attacks.
* After upload you get the link to download the file. For download only link is needed.
* Open Source, {% include icon-github.html username="EnigmaBridge/EnigmaLink" %}
* Download example: <https://enigmalink.io/d#u=fw&c=FWhyyHNOMuF9TkhVe8BIbA&f=0B8RUMrk78PeINnJTcVB2WEFYVUU&n=YC4WWaE5NNRHgLtV2P4krA>

### [AES Whitebox implementation](https://github.com/ph4r05/Whitebox-crypto-AES)
My master thesis focused on analysis and implementation of the selected Whitebox schemes for AES. I've implemented the basic Chow and Karroumi scheme.
The Karroumi scheme was discovered to be vulnerable in the thesis. Implementations are released under permissive licenses. For more info please refer to my
[master thesis](https://github.com/ph4r05/masterThesis). Implementations in [C++](https://github.com/ph4r05/Whitebox-crypto-AES) and [Java](https://github.com/ph4r05/Whitebox-crypto-AES-java) are available.

### Publications

[Google Scholar](https://scholar.google.cz/scholar?hl=en&as_sdt=0%2C5&q=%22Dusan+Klinec%22%7C%22D+Klinec%22&btnG=)

### Other projects

More of my projects you can find on my [GitHub account](https://github.com/ph4r05)
Here is a small selection:

- [Booltest](https://crocs.fi.muni.cz/public/papers/secrypt2017), the randomness distinguisher tool implemented in [Python](https://github.com/ph4r05/polynomial-distinguishers).
- [php-aho-corasick](https://github.com/ph4r05/php_aho_corasick) PHP module for Aho-Corasick pattern matching
- [laravel-queue-database-ph4](https://github.com/ph4r05/laravel-queue-database-ph4) Optimistic queueing for Laravel
- [javacard-gradle-template](https://github.com/ph4r05/javacard-gradle-template) Project template for easy JavaCard development, based on Gradle.

#### Donations:

Monero: 47BEukN83whUdvuXbaWmDDQLYNUpLsvFR2jioQtpP5vD8b3o74b9oFgQ3KFa3ibjbwBsaJEehogjiUCfGtugUGAuJAfbh1Z 


<!--You can find the source code for Jekyll at-->
<!--{% include icon-github.html username="jekyll" %} /-->
<!--[jekyll](https://github.com/jekyll/jekyll)-->
