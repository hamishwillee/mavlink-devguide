# Images test

Note that this markdown is in /en/guide/serialization.md
And the image  is located in both this guide folder and in the folder `/assets/packets/`, which is a peer


1. 
   ![MAVLink v2 packet](../../assets/packets/packet_mavlink_v2.jpg) 

2. 
   ![MAVLink v2 packet](/assets/packets/packet_mavlink_v2.jpg)

3. 
   ![MAVLink v2 packet](packet_mavlink_v2.jpg)

4. 
   <img src="packet_mavlink_v2.jpg" title="MAVLink v2 packet">
   
My index imports the alias: 
```
'/en/(.*)':
  'https://raw.githubusercontent.com/hamishwillee/mavlink-devguide/docsify_test/en/$1',
```

