---
layout: post
title:  "Playing with Google's protocol QUIC pt.1"
date:   2015-07-19 08:01:21
categories: general
---

I recently heard a bit about Google's protocol to improve web connection's called QUIC. I thought maybe I could learn something cool by messing with it.

At the moment there doesn't seem to be much adoption of QUIC out side of Google's servers and documentation is light compared to other tech things. After reading what I could and wanting to get started I opened up tcpdump and took a loot at what was actually using the protocol. It was pretty easy to narrow down and get a filter, not much talks using UDP on port 443. I'm not sure what it is actually being used for, google.com and searches don't bring anything up but browsing docs and gmail showed some network activity.
Tcpdump has packets with the destinations like `sfo07s13-in-f6.1e100.net.https`. I found a server to test some packets on. Chromium includes a test server but it was a bit of work to get it set up. My macbook air doesn't have a ton of extra free space for the source either.

I figured a good place to start would be to see if I can get the server to respond. I captured some more packets, read some documents and opened wireshark and started to try to make sense of what I was seeing.

~~~
	> sfo03s01-in-x01.1e100.net.https: [udp sum ok] UDP, length 1350
	0x0000:  383b c83a f511 60f8 1dd0 3012 86dd 6002  8;.:..`...0...`.
	0x0010:  6e32 054e 1140 2602 030a c045 aba0 807b  n2.N.@&....E...{
	0x0020:  ac02 4d15 e04c 2607 f8b0 4005 0805 0000  ..M..L&...@.....
	0x0030:  0000 0000 2001 dbf9 01bb 054e cca8 0de7  ...........N....
	0x0040:  98a0 a8e7 2491 e251 3032 3501 83c9 1ada  ....$..Q025.....
	0x0050:  8646 0095 023d b400 01a0 0114 0543 484c  .F...=.......CHL
	0x0060:  4f0f 0000 0050 4144 002b 0400 0053 4e49  O....PAD.+...SNI
~~~
According the document I read the first thing is probably a public flag. 8 bytes that hold some information about what's coming up. Wireshark helped to make send of these bytes.

I wrote a small program to send some UDP packets. The first thing I tried was sending a small packet that started with `0x08` and a 64 bit connection id of `{0x02, 0x03, 0x04, 0x05, 0x06, 0x08, 0x99, 0xff}` and looked for this in the packets.

~~~
	0x0000:  383b c83a f511 60f8 1dd0 3012 0800 4500  8;.:..`...0...E.
	0x0010:  002b 83e4 0000 4011 98d3 c0a8 0146 d83a  .+....@......F.:
	0x0020:  c3e1 c472 01bb 0017 0b5d 0802 0304 0506  ...r.....]......
	0x0030:  0899 ff51 3032 3501 53                   ...Q025.S
~~~

~~~
	0x0000:  60f8 1dd0 3012 383b c83a f511 0800 4500  `...0.8;.:....E.
	0x0010:  0069 0612 0000 3711 1f68 d83a c3e1 c0a8  .i....7..h.:....
	0x0020:  0146 01bb c472 0055 5829 0a02 0304 0506  .F...r.UX)......
	0x0030:  0899 ff50 5253 5403 0000 0052 4e4f 4e08  ...PRST....RNON.
	0x0040:  0000 0052 5345 5110 0000 0043 4144 5224  ...RSEQ....CADR$
	0x0050:  0000 00b5 690f 0000 0000 0051 0000 0000  ....i......Q....
	0x0060:  0000 000a 0000 0000 0000 0000 0000 00ff  ................
	0x0070:  ffac 045a ba72 c4                        ...Z.r.
~~~

I got something. My public header and following bytes for my connection id. Q025 is also in this request, i took the version from the packets I was sending from chrome in my initial captures. The S at the end is some extra text I added on. The response also includes my connection id on the 0x0020 line. I checked the page I found and realized the public flag part is wrong.

~~~ go
conn, err := net.Dial("udp", "216.58.195.225:443")
		var m byte = 0x00
		m ^= 0x01
		m ^= 0x0C //sending an 8 byte cid
		message := string(m) //public byte
		id := []byte{0x02, 0x03, 0x04, 0x05, 0x06, 0x08, 0x99, 0xff} //cid
		version := "Q025" //version
		seq := []byte{0x01} //packet number
~~~

Set up a new packet to send and it looks a bit better in wireshark

~~~
	0x0000:  383b c83a f511 60f8 1dd0 3012 0800 4500  8;.:..`...0...E.
	0x0010:  002b 9fc6 0000 4011 7cf1 c0a8 0146 d83a  .+....@.|....F.:
	0x0020:  c3e1 e8b7 01bb 0017 e217 0d02 0304 0506  ................
	0x0030:  0899 ff51 3032 3501 53                   ...Q025.S
~~~

and got a new response after a short wait....

~~~
	0x0000:  60f8 1dd0 3012 383b c83a f511 0800 4500  `...0.8;.:....E.
	0x0010:  005e e1c8 0000 3611 44bc d83a c3e1 c0a8  .^....6.D..:....
	0x0020:  0146 01bb e8b7 004a 0208 0c02 0304 0506  .F.....J........
	0x0030:  0899 ff01 698d 6854 218b 5e83 447d 0cec  ....i.hT!.^.D}..
	0x0040:  0040 0000 ffff 0006 0000 0219 0000 001b  .@..............
	0x0050:  004e 6f20 7265 6365 6e74 206e 6574 776f  .No.recent.netwo
	0x0060:  726b 2061 6374 6976 6974 792e            rk.activity.
~~~

Thats where I'm at for the moment.
I need to do a bit more reading. :)
