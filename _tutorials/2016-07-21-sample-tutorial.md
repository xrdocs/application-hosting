---
published: true
date: '2016-07-21 14:55 -0700'
title: Sample Tutorial
author: Akshat Sharma
excerpt: adsa
position: hidden
---


```

telemetry model-driven  
 destination-group DGroup1  
   address family ipv4 172.30.8.4 port 5432  
   encoding self-describing-gpb  
   protocol tcp
  !
 !
 sensor-group SGroup1
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
 !  
 subscription Sub1  
  sensor-group-id SGroup1 sample-interval 30000  
  destination-id DGroup1 

```

