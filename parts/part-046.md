# Part 046: WebRTC Integration
## วิดีโอคอลและ Screen Sharing ใน Electron

---

## 🎯 เป้าหมายของบทเรียนนี้

- Peer-to-peer video calls ใน Electron
- getUserMedia สำหรับ camera/microphone
- Screen sharing ด้วย desktopCapturer
- Signaling server
- STUN/TURN servers
- WebRTC security

---

## 1. WebRTC Architecture

```
Peer A (Electron)          Signaling Server          Peer B (Electron/Browser)
      |                           |                           |
      |------- offer ---------->  |                           |
      |                           |------- offer -----------> |
      |                           |                           |
      |                           | <------ answer ---------- |
      | <------ answer ---------  |                           |
      |                           |                           |
      | <----------- ICE candidates (STUN/TURN) -----------> |
      |                           |                           |
      | <============= P2P Connection ======================> |
      |          (video/audio/data direct)                    |
```

---

## 2. Main Process - desktopCapturer

### src/main/screenshare.ts

```typescript
import { ipcMain, desktopCapturer, BrowserWindow } from 'electron';

// Handler สำหรับดึงรายการหน้าจอ/หน้าต่าง
ipcMain.handle('screenshare:getSources', async (_, options: {
  types: Array<'screen' | 'window'>;
  thumbnailSize?: { width: number; height: number };
}) => {
  const sources = await desktopCapturer.getSources({
    types: options.types || ['screen', 'window'],
    thumbnailSize: options.thumbnailSize || { width: 320, height: 180 },
    fetchWindowIcons: true,
  });

  // แปลงให้ serialize ได้ (thumbnail เป็น dataURL)
  return sources.map(source => ({
    id: source.id,
    name: source.name,
    thumbnail: source.thumbnail.toDataURL(),
    appIcon: source.appIcon?.toDataURL() || null,
    display_id: source.display_id,
  }));
});

// Handler สำหรับเลือก source
ipcMain.handle('screenshare:getStream', async (event, sourceId: string) => {
  // ส่ง sourceId กลับไปให้ renderer จัดการ getUserMedia เอง
  // เพราะ getUserMedia ต้องทำงานใน renderer context
  return { sourceId };
});
```

---

## 3. Preload Script

### src/preload/preload.ts

```typescript
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('electronAPI', {
  // Screen sharing
  getScreenSources: (options?: { types?: Array<'screen' | 'window'> }) =>
    ipcRenderer.invoke('screenshare:getSources', options || { types: ['screen', 'window'] }),
  
  // WebRTC signaling (ผ่าน IPC สำหรับ desktop-to-desktop call)
  sendSignal: (targetId: string, signal: any) =>
    ipcRenderer.invoke('webrtc:sendSignal', { targetId, signal }),
  
  onSignal: (callback: (data: { fromId: string; signal: any }) => void) => {
    ipcRenderer.on('webrtc:signal', (_, data) => callback(data));
    return () => ipcRenderer.removeAllListeners('webrtc:signal');
  },
  
  // Get peer ID
  getPeerId: () => ipcRenderer.invoke('webrtc:getPeerId'),
});
```

---

## 4. Renderer - WebRTC Manager

### src/renderer/webrtc/WebRTCManager.ts

```typescript
// STUN/TURN configuration
const ICE_SERVERS = [
  // Google's public STUN servers
  { urls: 'stun:stun.l.google.com:19302' },
  { urls: 'stun:stun1.l.google.com:19302' },
  { urls: 'stun:stun2.l.google.com:19302' },
  
  // TURN servers (ต้องใช้ของตัวเองในการ production)
  // {
  //   urls: 'turn:your-turn-server.com:3478',
  //   username: 'your-username',
  //   credential: 'your-password',
  // },
];

export class WebRTCManager {
  private peerConnection: RTCPeerConnection | null = null;
  private localStream: MediaStream | null = null;
  private remoteStream: MediaStream = new MediaStream();
  
  private onRemoteStream?: (stream: MediaStream) => void;
  private onConnectionState?: (state: RTCPeerConnectionState) => void;
  private onIceCandidate?: (candidate: RTCIceCandidate) => void;
  
  constructor(callbacks: {
    onRemoteStream?: (stream: MediaStream) => void;
    onConnectionState?: (state: RTCPeerConnectionState) => void;
    onIceCandidate?: (candidate: RTCIceCandidate) => void;
  }) {
    this.onRemoteStream = callbacks.onRemoteStream;
    this.onConnectionState = callbacks.onConnectionState;
    this.onIceCandidate = callbacks.onIceCandidate;
  }

  // ==========================================
  // Media Access
  // ==========================================

  // ขอ camera + microphone
  async getUserMedia(constraints: MediaStreamConstraints = {
    video: true,
    audio: true,
  }): Promise<MediaStream> {
    this.localStream = await navigator.mediaDevices.getUserMedia(constraints);
    return this.localStream;
  }

  // ขอ screen share
  async getDisplayMedia(sourceId?: string): Promise<MediaStream> {
    if (sourceId) {
      // ใช้ Electron's desktopCapturer
      this.localStream = await navigator.mediaDevices.getUserMedia({
        audio: false,
        video: {
          // @ts-ignore - Electron-specific constraint
          mandatory: {
            chromeMediaSource: 'desktop',
            chromeMediaSourceId: sourceId,
            minWidth: 1280,
            maxWidth: 1920,
            minHeight: 720,
            maxHeight: 1080,
            minFrameRate: 15,
            maxFrameRate: 30,
          },
        },
      });
    } else {
      // Standard Web API
      this.localStream = await navigator.mediaDevices.getDisplayMedia({
        video: {
          frameRate: { max: 30 },
          width: { max: 1920 },
          height: { max: 1080 },
        },
        audio: true,
      });
    }
    
    return this.localStream;
  }

  // ==========================================
  // PeerConnection
  // ==========================================

  createPeerConnection(): RTCPeerConnection {
    this.peerConnection = new RTCPeerConnection({
      iceServers: ICE_SERVERS,
      iceTransportPolicy: 'all',
      bundlePolicy: 'balanced',
    });

    // Add local tracks
    if (this.localStream) {
      this.localStream.getTracks().forEach(track => {
        this.peerConnection!.addTrack(track, this.localStream!);
      });
    }

    // Handle remote tracks
    this.peerConnection.ontrack = (event) => {
      event.streams[0].getTracks().forEach(track => {
        this.remoteStream.addTrack(track);
      });
      
      this.onRemoteStream?.(this.remoteStream);
    };

    // Handle ICE candidates
    this.peerConnection.onicecandidate = (event) => {
      if (event.candidate) {
        this.onIceCandidate?.(event.candidate);
      }
    };

    // Handle connection state
    this.peerConnection.onconnectionstatechange = () => {
      const state = this.peerConnection!.connectionState;
      console.log('Connection state:', state);
      this.onConnectionState?.(state);
    };

    // Handle ICE connection state
    this.peerConnection.oniceconnectionstatechange = () => {
      console.log('ICE state:', this.peerConnection!.iceConnectionState);
    };

    return this.peerConnection;
  }

  // สร้าง offer (caller)
  async createOffer(): Promise<RTCSessionDescriptionInit> {
    if (!this.peerConnection) this.createPeerConnection();
    
    const offer = await this.peerConnection!.createOffer({
      offerToReceiveVideo: true,
      offerToReceiveAudio: true,
    });
    
    await this.peerConnection!.setLocalDescription(offer);
    
    return offer;
  }

  // รับ offer และสร้าง answer (callee)
  async createAnswer(offer: RTCSessionDescriptionInit): Promise<RTCSessionDescriptionInit> {
    if (!this.peerConnection) this.createPeerConnection();
    
    await this.peerConnection!.setRemoteDescription(new RTCSessionDescription(offer));
    
    const answer = await this.peerConnection!.createAnswer();
    await this.peerConnection!.setLocalDescription(answer);
    
    return answer;
  }

  // รับ answer (caller)
  async setAnswer(answer: RTCSessionDescriptionInit): Promise<void> {
    await this.peerConnection!.setRemoteDescription(new RTCSessionDescription(answer));
  }

  // เพิ่ม ICE candidate
  async addIceCandidate(candidate: RTCIceCandidateInit): Promise<void> {
    await this.peerConnection!.addIceCandidate(new RTCIceCandidate(candidate));
  }

  // ==========================================
  // Data Channel
  // ==========================================

  createDataChannel(label: string = 'chat'): RTCDataChannel {
    const channel = this.peerConnection!.createDataChannel(label, {
      ordered: true,
    });
    
    channel.onopen = () => console.log('Data channel opened');
    channel.onclose = () => console.log('Data channel closed');
    
    return channel;
  }

  // ==========================================
  // Controls
  // ==========================================

  // Mute/unmute audio
  toggleAudio(enabled: boolean): void {
    this.localStream?.getAudioTracks().forEach(track => {
      track.enabled = enabled;
    });
  }

  // Enable/disable video
  toggleVideo(enabled: boolean): void {
    this.localStream?.getVideoTracks().forEach(track => {
      track.enabled = enabled;
    });
  }

  // เปลี่ยน video track (screen share)
  async replaceVideoTrack(newStream: MediaStream): Promise<void> {
    const newVideoTrack = newStream.getVideoTracks()[0];
    
    const sender = this.peerConnection!
      .getSenders()
      .find(s => s.track?.kind === 'video');
    
    if (sender && newVideoTrack) {
      await sender.replaceTrack(newVideoTrack);
    }
  }

  // Hangup
  hangup(): void {
    // หยุด local stream
    this.localStream?.getTracks().forEach(track => track.stop());
    
    // ปิด peer connection
    this.peerConnection?.close();
    this.peerConnection = null;
    this.localStream = null;
    this.remoteStream = new MediaStream();
  }

  // Getters
  getLocalStream(): MediaStream | null {
    return this.localStream;
  }

  getRemoteStream(): MediaStream {
    return this.remoteStream;
  }

  getStats(): Promise<RTCStatsReport> | null {
    return this.peerConnection?.getStats() || null;
  }
}
```

---

## 5. React Component - Video Call

### src/renderer/components/VideoCall.tsx

```tsx
import React, { useState, useEffect, useRef, useCallback } from 'react';
import { WebRTCManager } from '../webrtc/WebRTCManager';

interface Source {
  id: string;
  name: string;
  thumbnail: string;
}

function VideoCall() {
  const [webrtc] = useState(() => new WebRTCManager({
    onRemoteStream: (stream) => {
      if (remoteVideoRef.current) {
        remoteVideoRef.current.srcObject = stream;
      }
    },
    onConnectionState: (state) => {
      setConnectionState(state);
    },
    onIceCandidate: (candidate) => {
      // ส่งผ่าน signaling server
      signalingRef.current?.send(JSON.stringify({
        type: 'ice-candidate',
        candidate,
      }));
    },
  }));
  
  const localVideoRef = useRef<HTMLVideoElement>(null);
  const remoteVideoRef = useRef<HTMLVideoElement>(null);
  const signalingRef = useRef<WebSocket | null>(null);
  
  const [connectionState, setConnectionState] = useState<string>('disconnected');
  const [isVideoEnabled, setIsVideoEnabled] = useState(true);
  const [isAudioEnabled, setIsAudioEnabled] = useState(true);
  const [isScreenSharing, setIsScreenSharing] = useState(false);
  const [sources, setSources] = useState<Source[]>([]);
  const [showSourcePicker, setShowSourcePicker] = useState(false);

  // เชื่อมต่อ signaling server
  const connectSignaling = useCallback(() => {
    const ws = new WebSocket('wss://your-signaling-server.com');
    
    ws.onopen = () => {
      console.log('Connected to signaling server');
    };
    
    ws.onmessage = async (event) => {
      const message = JSON.parse(event.data);
      
      switch (message.type) {
        case 'offer':
          const answer = await webrtc.createAnswer(message.offer);
          ws.send(JSON.stringify({ type: 'answer', answer }));
          break;
          
        case 'answer':
          await webrtc.setAnswer(message.answer);
          break;
          
        case 'ice-candidate':
          await webrtc.addIceCandidate(message.candidate);
          break;
      }
    };
    
    signalingRef.current = ws;
  }, [webrtc]);

  // เริ่มวิดีโอ
  const startVideo = async () => {
    const stream = await webrtc.getUserMedia({ video: true, audio: true });
    
    if (localVideoRef.current) {
      localVideoRef.current.srcObject = stream;
    }
    
    webrtc.createPeerConnection();
  };

  // Call
  const startCall = async () => {
    const offer = await webrtc.createOffer();
    signalingRef.current?.send(JSON.stringify({ type: 'offer', offer }));
  };

  // Screen share
  const startScreenShare = async (sourceId?: string) => {
    const stream = await webrtc.getDisplayMedia(sourceId);
    
    if (localVideoRef.current) {
      localVideoRef.current.srcObject = stream;
    }
    
    await webrtc.replaceVideoTrack(stream);
    setIsScreenSharing(true);
    setShowSourcePicker(false);
  };

  // Show source picker (Electron only)
  const showScreenPicker = async () => {
    if (window.electronAPI) {
      const sources = await window.electronAPI.getScreenSources();
      setSources(sources);
      setShowSourcePicker(true);
    } else {
      // Browser fallback
      await startScreenShare();
    }
  };

  // Stop screen share
  const stopScreenShare = async () => {
    const stream = await webrtc.getUserMedia({ video: true, audio: true });
    await webrtc.replaceVideoTrack(stream);
    setIsScreenSharing(false);
  };

  // Hangup
  const hangup = () => {
    webrtc.hangup();
    setConnectionState('disconnected');
    setIsScreenSharing(false);
  };

  return (
    <div style={{ padding: '20px' }}>
      <h2>Video Call</h2>
      
      <div style={{ display: 'grid', gridTemplateColumns: '1fr 1fr', gap: '16px' }}>
        {/* Local video */}
        <div>
          <p>Local</p>
          <video
            ref={localVideoRef}
            autoPlay
            muted
            playsInline
            style={{ width: '100%', background: '#000', borderRadius: '8px' }}
          />
        </div>
        
        {/* Remote video */}
        <div>
          <p>Remote ({connectionState})</p>
          <video
            ref={remoteVideoRef}
            autoPlay
            playsInline
            style={{ width: '100%', background: '#111', borderRadius: '8px' }}
          />
        </div>
      </div>
      
      {/* Controls */}
      <div style={{ display: 'flex', gap: '8px', marginTop: '16px', justifyContent: 'center' }}>
        <button onClick={startVideo}>Start Video</button>
        <button onClick={startCall}>Call</button>
        
        <button onClick={() => {
          webrtc.toggleAudio(!isAudioEnabled);
          setIsAudioEnabled(!isAudioEnabled);
        }}>
          {isAudioEnabled ? 'Mute' : 'Unmute'}
        </button>
        
        <button onClick={() => {
          webrtc.toggleVideo(!isVideoEnabled);
          setIsVideoEnabled(!isVideoEnabled);
        }}>
          {isVideoEnabled ? 'Stop Video' : 'Start Video'}
        </button>
        
        <button onClick={isScreenSharing ? stopScreenShare : showScreenPicker}>
          {isScreenSharing ? 'Stop Sharing' : 'Share Screen'}
        </button>
        
        <button onClick={hangup} style={{ background: 'red', color: 'white' }}>
          End Call
        </button>
      </div>
      
      {/* Source Picker Modal */}
      {showSourcePicker && (
        <div style={{
          position: 'fixed', inset: 0, background: 'rgba(0,0,0,0.7)',
          display: 'flex', alignItems: 'center', justifyContent: 'center',
        }}>
          <div style={{ background: 'white', padding: '20px', borderRadius: '8px', maxWidth: '600px' }}>
            <h3>Select Screen to Share</h3>
            <div style={{ display: 'grid', gridTemplateColumns: 'repeat(3, 1fr)', gap: '8px' }}>
              {sources.map(source => (
                <div
                  key={source.id}
                  onClick={() => startScreenShare(source.id)}
                  style={{ cursor: 'pointer', border: '2px solid transparent', borderRadius: '4px' }}
                >
                  <img src={source.thumbnail} alt={source.name} style={{ width: '100%' }} />
                  <p style={{ fontSize: '12px', textAlign: 'center' }}>{source.name}</p>
                </div>
              ))}
            </div>
            <button onClick={() => setShowSourcePicker(false)}>Cancel</button>
          </div>
        </div>
      )}
    </div>
  );
}

export default VideoCall;
```

---

## 6. Signaling Server (Node.js)

### signaling-server/index.js

```javascript
const WebSocket = require('ws');
const { v4: uuidv4 } = require('uuid');

const wss = new WebSocket.Server({ port: 8080 });
const rooms = new Map(); // roomId -> Set of clients
const clients = new Map(); // clientId -> ws

wss.on('connection', (ws) => {
  const clientId = uuidv4();
  clients.set(clientId, ws);
  
  ws.send(JSON.stringify({ type: 'connected', clientId }));
  
  ws.on('message', (data) => {
    const message = JSON.parse(data.toString());
    
    switch (message.type) {
      case 'join-room':
        const room = rooms.get(message.roomId) || new Set();
        room.add(clientId);
        rooms.set(message.roomId, room);
        
        // แจ้ง clients อื่นในห้อง
        room.forEach(id => {
          if (id !== clientId) {
            clients.get(id)?.send(JSON.stringify({
              type: 'peer-joined',
              peerId: clientId,
            }));
          }
        });
        break;
        
      case 'signal':
        // ส่ง signal ไปยัง target
        const target = clients.get(message.targetId);
        if (target) {
          target.send(JSON.stringify({
            type: 'signal',
            fromId: clientId,
            signal: message.signal,
          }));
        }
        break;
    }
  });
  
  ws.on('close', () => {
    clients.delete(clientId);
    rooms.forEach((room, roomId) => {
      room.delete(clientId);
      if (room.size === 0) {
        rooms.delete(roomId);
      }
    });
  });
});

console.log('Signaling server running on ws://localhost:8080');
```

---

## 7. สรุป

### WebRTC ใน Electron vs Browser

| Feature | Browser | Electron |
|---------|---------|---------|
| getUserMedia | Standard | Standard |
| getDisplayMedia | Standard (limited) | desktopCapturer (full control) |
| Screen selection | Browser UI | Custom UI ด้วย desktopCapturer |
| Audio capture | Screen audio (Chrome only) | แยกกัน |
| CORS | มีข้อจำกัด | น้อยกว่า |

### Security

- ใช้ WSS (WebSocket Secure) สำหรับ signaling
- ใช้ TURN server เพื่อ relay เมื่อ P2P ไม่ได้
- ตรวจสอบ user identity ก่อน join room
- End-to-end encryption ด้วย DTLS/SRTP (built-in ใน WebRTC)

---

*จบ Part 046 - ต่อไป Part 047: Electron with Python*
