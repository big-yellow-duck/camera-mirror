# 🎥 Self Stream Mirror

A simple camera and audio mirror application built with SvelteKit for real-time feedback. Perfect for presentations, video testing, or practicing in front of a camera.

## ✨ Features

- **📹 Real-time Camera Display** - Mirror mode camera feed with customizable aspect ratio
- **🎤 Microphone Audio Monitoring** - Hear your voice through speakers with adjustable delay
- **🎛️ Device Selection** - Choose from available microphones
- **📊 Audio Level Visualization** - Live microphone level meter with frequency spectrum
- **⏱️ Audio Delay Control** - Sync audio with video (0-1000ms delay)
- **🖥️ Fullscreen Mode** - Immersive video-only fullscreen experience
- **📱 Responsive Design** - Works on desktop and mobile devices
- **🌙 Dark Theme** - Easy on the eyes for extended use

## 🚀 Quick Start

### Prerequisites

- Node.js 18+ and npm
- Modern web browser (Chrome 53+, Firefox 36+, Safari 11+, Edge 12+)
- Camera and microphone (built-in or external)
- Speakers/headphones for audio monitoring

### Installation

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd self-stream
   ```

2. **Install dependencies**
   ```bash
   npm install
   ```

3. **Start development server**
   ```bash
   npm run dev
   ```

4. **Open your browser**
   Navigate to [http://localhost:5173](http://localhost:5173)

## 📖 How to Use

### Basic Setup

1. **Allow Permissions**
   - Click "Open Camera Mirror"
   - Allow camera and microphone access when prompted
   - Click "Start Mirror" to begin streaming

2. **Audio Mirror Setup**
   - Click "Enable Audio Mirror" to hear your voice
   - Adjust the audio delay slider if lips and sound don't sync
   - Click "Mute Audio Mirror" (red button) to stop audio monitoring

3. **Video Controls**
   - Hover over video to reveal control buttons
   - Click aspect ratio button to switch between "Fit" and "Fill" modes
   - Click fullscreen button for immersive viewing

### Advanced Features

#### 🎛️ Microphone Selection
- Choose from connected microphones in the dropdown menu
- Automatically detects available audio devices
- Switch microphones without restarting the stream

#### ⏱️ Audio Delay Tuning
- **Purpose**: Compensate for latency between video and audio
- **Range**: 0-1000ms (1 second)
- **Common Settings**:
  - USB Webcam: 100-200ms
  - Built-in Webcam: 50-150ms
  - Bluetooth Speakers: 150-300ms
  - Wired Headphones: 0-100ms

#### 📺 Video Display Modes
- **Fit Mode**: Maintains aspect ratio, black borders may appear
- **Fill Mode**: Crops video to fill entire screen, no black borders

#### 📊 Audio Monitoring
- **Level Meter**: Real-time volume indicator with color coding (green/yellow/red)
- **Frequency Display**: Visual spectrum analysis of microphone input
- **Status Panel**: Shows connection states for all components

## 🔧 Development

### Project Structure

```
self-stream/
├── src/
│   ├── routes/
│   │   ├── +page.svelte          # Landing page
│   │   ├── +layout.svelte        # App layout
│   │   └── camera/
│   │       └── +page.svelte      # Main camera mirror page
│   ├── app.html                  # HTML template
│   └── lib/                      # Library files
├── package.json
├── svelte.config.js
├── vite.config.ts
└── README.md
```

### Available Scripts

```bash
# Start development server
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview

# Type checking
npm run check

# Type checking with watch mode
npm run check:watch

# Code formatting
npm run format

# Code linting
npm run lint
```

### Technology Stack

- **Framework**: SvelteKit 2.48.5+
- **Language**: TypeScript 5.9.3+
- **Styling**: Tailwind CSS 4.1.17+
- **Build Tool**: Vite 7.2.2+
- **Runtime**: Browser-native APIs (WebRTC, Web Audio API)

## 🔒 Privacy & Security

- **No Data Collection**: All processing happens locally in your browser
- **No Internet Required**: Works completely offline once loaded
- **No Server Upload**: Camera and microphone streams never leave your device
- **HTTPS Required**: Production deployments need HTTPS for camera/microphone access

## 🌐 Browser Support

| Browser | Minimum Version | Notes |
|---------|-----------------|-------|
| Chrome | 53+ | Full support |
| Firefox | 36+ | Full support |
| Safari | 11+ | Full support |
| Edge | 12+ | Full support |

## 🎯 Common Use Cases

### 🎭 Presentation Practice
- Practice speeches and presentations
- Monitor body language and gestures
- Check microphone volume and clarity

### 🎵 Audio Testing
- Test microphone quality and positioning
- Check audio levels for recording
- Verify audio equipment setup

### 📹 Video Testing
- Test camera quality and positioning
- Check lighting conditions
- Verify video quality before streaming

### 🎮 Content Creation
- Practice for YouTube videos
- Test streaming setup
- Monitor audio/video sync

## 🛠️ Troubleshooting

### Camera/Microphone Not Working
- Ensure you've granted browser permissions
- Check if other applications are using the devices
- Try refreshing the page and granting permissions again
- Verify device connections (USB, Bluetooth, etc.)

### Audio Not Syncing with Video
- Increase audio delay slider gradually
- Test with different delay values (start with 100ms)
- Check if using Bluetooth audio (may need more delay)

### Audio Not Playing
- Click "Enable Audio Mirror" button
- Check speaker/headphone connections
- Ensure system volume is up
- Try different browsers if autoplay is blocked

### Fullscreen Issues
- Click the fullscreen button on the video (not browser fullscreen)
- Press ESC to exit fullscreen
- Ensure browser supports fullscreen API

## 📄 License

This project is open source and available under the [MIT License](LICENSE).

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## 📞 Support

If you encounter any issues:
1. Check the troubleshooting section above
2. Ensure you're using a supported browser
3. Verify camera/microphone permissions
4. Open an issue on GitHub with details about your setup

---

**Built with ❤️ using SvelteKit**
