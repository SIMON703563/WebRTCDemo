# WebRTCDemo
simple remote webrtc from home peer to peer

Absolutely! Let’s consolidate everything we’ve done into a **step-by-step WebRTC home testing lab setup** that covers installation, testing, HTTPS, and remote connectivity. I’ll make it clear and practical so you can follow it without missing steps.

---

# **Home WebRTC Test Lab – Complete Tutorial**

### **Prerequisites**

* Windows PC
* Admin access
* Home internet with Plusnet ISP
* Node.js installed
* OpenSSL installed (for self-signed cert)
* Modern browser (Chrome, Edge, Firefox)
* Optional: mobile device for remote testing

---

## **1. Install Node.js and Verify**

1. Install Node.js from [https://nodejs.org](https://nodejs.org).

2. Verify installation:

   ```cmd
   where node
   node -v
   ```

   * Should return the path and version.

3. Add Node.js to PATH if not recognized:

   * Right-click **This PC → Properties → Advanced system settings → Environment Variables → Path → Edit**
   * Add: `C:\Program Files\nodejs\`
   * Open a new CMD and check `node -v`.

---

## **2. Create Project Structure**

```cmd
mkdir webrtc-test
cd webrtc-test
mkdir public
```

* `public/` will hold client-side HTML and JS.

---

## **3. Install Dependencies**

```cmd
npm init -y
npm install express socket.io
```

---

## **4. Create Server (`server.js`)**

```javascript
const fs = require('fs');
const https = require('https');
const express = require('express');
const app = express();
const io = require('socket.io');

app.use(express.static('public'));

// Load certificate and key
const options = {
  key: fs.readFileSync('server.key'),
  cert: fs.readFileSync('server.cert')
};

// Create HTTPS server
const server = https.createServer(options, app);
const socket = io(server);

socket.on('connection', (s) => {
  console.log('A user connected');

  s.on('signal', (data) => {
    s.broadcast.emit('signal', data);
  });

  s.on('disconnect', () => {
    console.log('User disconnected');
  });
});

// Listen on port 3000
server.listen(3000, () => {
  console.log('HTTPS server running at https://192.168.1.65:3000');
});

```

---

## **5. Create Client (`public/index.html`)**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>WebRTC Test</title>
<style>
  video { width: 45%; margin: 5px; border: 1px solid black; }
</style>
</head>
<body>
<h1>WebRTC Test</h1>
<video id="localVideo" autoplay playsinline muted></video>
<video id="remoteVideo" autoplay playsinline></video>

<script src="/socket.io/socket.io.js"></script>
<script>
const socket = io();
const localVideo = document.getElementById('localVideo');
const remoteVideo = document.getElementById('remoteVideo');

let pc = new RTCPeerConnection({
  iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
});

// Display remote stream
pc.ontrack = event => {
  remoteVideo.srcObject = event.streams[0];
};

// Handle ICE candidates
pc.onicecandidate = event => {
  if (event.candidate) {
    socket.emit('signal', { candidate: event.candidate });
  }
};

// Get local media
navigator.mediaDevices.getUserMedia({ video: true, audio: true })
  .then(stream => {
    localVideo.srcObject = stream;
    stream.getTracks().forEach(track => pc.addTrack(track, stream));
  })
  .catch(err => console.error('getUserMedia error:', err));

// Listen for signaling data
socket.on('signal', async data => {
  if (data.sdp) {
    await pc.setRemoteDescription(new RTCSessionDescription(data.sdp));
    if (data.sdp.type === 'offer') {
      const answer = await pc.createAnswer();
      await pc.setLocalDescription(answer);
      socket.emit('signal', { sdp: pc.localDescription });
    }
  } else if (data.candidate) {
    try {
      await pc.addIceCandidate(data.candidate);
    } catch(e) { console.error(e); }
  }
});

// Create initial offer if first connected (PC)
navigator.mediaDevices.getUserMedia({ video: true, audio: true }).then(() => {
  pc.createOffer().then(offer => {
    pc.setLocalDescription(offer);
    socket.emit('signal', { sdp: offer });
  });
});
</script>
</body>
</html>

```

---

## **6. Run Server**

```cmd
node server.js
```

* Open `http://localhost:3000` → browser asks for camera/mic permission.
* Confirm your local video appears.

---

## **7. Enable HTTPS (Self-Signed Cert)**

1. Generate cert using OpenSSL:

```cmd
cd "C:\Program Files\OpenSSL-Win64\bin"
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365
```

* Enter dummy info when prompted.

2. Modify server for HTTPS:

```javascript
const fs = require('fs');
const https = require('https');
const options = {
    key: fs.readFileSync('key.pem'),
    cert: fs.readFileSync('cert.pem')
};
const server = https.createServer(options, app);
const io = require('socket.io')(server);

server.listen(3000, () => console.log('HTTPS Server running'));
```

---

## **8. Configure Port Forwarding**

1. Login to Plusnet router (`192.168.1.1`).
2. Forward **TCP port 3000** to your PC’s LAN IP (`192.168.1.65`).
3. Test externally:

```powershell
Test-NetConnection -ComputerName YOUR_PUBLIC_IP -Port 3000
```

* `TcpTestSucceeded : True` → port reachable.

---

## **9. Test Remote Connection**

1. Use mobile device on cellular network.
2. Open `https://YOUR_PUBLIC_IP:3000`.

   * Accept self-signed certificate warning.
   * Confirm both videos appear.

---

## **10. Notes**



