# WebRTC Video Calling Application

## Project Structure

```

WebRTC-video-calling-site/
├── client/                          # React frontend application
│   ├── src/
│   │   ├── pages/
│   │   │   ├── Home.jsx             # Landing page with room join form
│   │   │   └── Room.jsx             # Video calling room component
│   │   ├── providers/
│   │   │   └── SocketProvider.jsx   # Socket.IO context provider
│   │   ├── App.jsx                  # Main app component with routing
│   │   └── main.jsx                 # React app entry point
│   ├── package.json                 # Frontend dependencies
│   └── vite.config.js               # Vite build configuration
└── signalling-server/              # Node.js signaling server
├── index.js                     # Socket.IO server implementation
├── package.json                 # Server dependencies
└── .env                         # Environment variables

````

---

## Code Architecture

### Frontend (React + Vite)

#### 1. **App Component** - `src/App.jsx`
Main application component that sets up routing and socket context.

```jsx
<Router>
  <SocketProvider>
    <Routes>
      <Route path="/" element={<Home />} />
      <Route path="/room/:roomId" element={<Room />} />
    </Routes>
  </SocketProvider>
</Router>
````

**Features:**

* React Router for navigation
* Global socket context provider
* Two main routes: Home and Room

---

#### 2. **SocketProvider** - `src/providers/SocketProvider.jsx`

Context provider for Socket.IO client connection.

```jsx
const SocketProvider = ({ children }) => {
  const socket = io("http://localhost:5000");
  return (
    <SocketContext.Provider value={socket}>{children}</SocketContext.Provider>
  );
};
```

**Features:**

* Creates Socket.IO connection to signaling server
* Provides socket instance to all child components
* Custom hook `useSocket()` for easy access

---

#### 3. **Home Component** - `src/pages/Home.jsx`

Landing page where users enter room ID to join a video call.

```jsx
const handleJoin = (e) => {
  e.preventDefault();
  socket.emit("join-room", roomId);
  navigate(`/room/${roomId}`);
};
```

**Features:**

* Form to enter room ID
* Emits `"join-room"` event to server
* Navigates to room page

---

#### 4. **Room Component** - `src/pages/Room.jsx`

Main video calling interface with WebRTC peer-to-peer connection.

##### Media Stream Setup

```js
const getMediaStream = async () => {
  const stream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true,
  });
  localVideo.current.srcObject = stream;
  return stream;
};
```

##### Peer Connection Setup

```js
const setupPeerConnection = (stream) => {
  peerConnection.current = new RTCPeerConnection(configuration);

  stream.getTracks().forEach((track) => {
    peerConnection.current.addTrack(track, stream);
  });

  peerConnection.current.onicecandidate = (event) => {
    if (event.candidate) {
      socket.emit("ice-candidate", roomId, event.candidate);
    }
  };

  peerConnection.current.ontrack = (event) => {
    remoteVideo.current.srcObject = event.streams[0];
  };
};
```

##### WebRTC Signaling

```js
// Create and send offer
const createOffer = async () => {
  const offer = await peerConnection.current.createOffer();
  await peerConnection.current.setLocalDescription(offer);
  socket.emit("offer", roomId, offer);
};

// Handle incoming offer
const handleOffer = async (offer) => {
  await peerConnection.current.setRemoteDescription(offer);
  const answer = await peerConnection.current.createAnswer();
  await peerConnection.current.setLocalDescription(answer);
  socket.emit("answer", roomId, answer);
};
```

**Features:**

* WebRTC peer-to-peer connection
* Video/audio stream capture
* ICE candidate exchange
* Offer/Answer signaling pattern
* Real-time video display

---

### Backend (Node.js + Socket.IO)

#### Signaling Server - `index.js`

Socket.IO server that facilitates WebRTC signaling between peers.

```js
io.on("connection", (socket) => {
  // Room management
  socket.on("join-room", (roomId) => {
    socket.join(roomId);
    io.to(roomId).emit("joined-room", roomId);
  });

  // WebRTC signaling
  socket.on("offer", (roomId, offer) => {
    socket.to(roomId).emit("offer", offer);
  });

  socket.on("answer", (roomId, answer) => {
    socket.to(roomId).emit("answer", answer);
  });

  socket.on("ice-candidate", (roomId, candidate) => {
    socket.to(roomId).emit("ice-candidate", candidate);
  });
});
```

**Socket Events:**

* `join-room`: User joins a specific room
* `offer`: Initiates WebRTC connection
* `answer`: Responds to WebRTC offer
* `ice-candidate`: Exchanges ICE candidates for NAT traversal

---

## WebRTC Connection Flow

1. **User A** joins room → Emits `join-room` event
2. **User B** joins same room → Both users are in the room
3. **User A** creates offer → Calls `createOffer()`
4. **Offer** sent via signaling → `socket.emit("offer")`
5. **User B** receives offer → Handles with `handleOffer()`
6. **User B** creates answer → Sends back via `answer` event
7. **ICE candidates** exchanged → Network info shared
8. **Direct P2P connection** established → Media streams flow directly

---

## Key Technologies

* **Frontend**: React 18, Vite, React Router, Socket.IO Client
* **Backend**: Node.js, Socket.IO Server, Express
* **WebRTC**: RTCPeerConnection, `getUserMedia` API
* **Styling**: Tailwind CSS
* **Build Tool**: Vite with HMR (Hot Module Replacement)

---

## Configuration

### STUN Server

```js
const configuration = {
  iceServers: [{ urls: "stun:stun.l.google.com:19302" }],
};
```

Uses Google’s public STUN server for NAT traversal.

### Port Configuration

* **Frontend**: Vite dev server (default port **5173**)
* **Backend**: Socket.IO server on port **5000**

---

## Usage

1. **Start signaling server**:

   ```bash
   cd signalling-server && npm start
   ```

2. **Start React app**:

   ```bash
   cd client && npm run dev
   ```

3. **Open browser**:
   Go to `http://localhost:5173`

4. **Join room**:
   Enter room ID and click **Submit**

5. **Start call**:
   Click **"Create Offer"** button to initiate the connection

---

## Summary

This architecture provides a complete WebRTC video calling solution with real-time signaling and peer-to-peer media streaming using modern frontend and backend technologies.
