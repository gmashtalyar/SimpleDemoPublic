# SimpleDemo

**Transform screen recordings into interactive, step-by-step product demonstrations.**

SimpleDemo is a SaaS platform that captures user interactions—clicks, scrolling, typing—and converts them into guided, interactive walkthroughs. Perfect for onboarding, product demos, and documentation.

---

## Business Logic

### Core Value Proposition

Traditional video tutorials are passive. Users watch, forget, and struggle to replicate steps. SimpleDemo solves this by creating **interactive demonstrations** where users click through actual interface elements, reinforcing learning through action.

### Key Capabilities

| Feature | Description |
|---------|-------------|
| **Browser Extension Recording** | Capture WebM video + JSON action logs directly from user activity |
| **Automatic Step Extraction** | AI-powered video processing identifies click events and extracts precise screenshots |
| **Interactive Hotspots** | Define clickable regions with descriptions, guiding users through each step |
| **Embeddable Player** | Share demos via iframe or direct link with full interactivity |
| **Team Collaboration** | Organization-based accounts with role-based access control |

### User Workflow

```
Record → Upload → Process → Edit → Share
   │         │         │        │       │
   ▼         ▼         ▼        ▼       ▼
Browser   Extension  Celery  Hotspot  Embed/
Extension   API      Worker  Editor   Link
```

---

## Technical Architecture


### Tech Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Task Queue** | Celery 5.4 | Distributed async task processing |
| **Message Broker** | Redis 5.0 | Task queue backend + caching |
| **Video Processing** | FFmpeg + MoviePy | WebM→MP4 conversion, frame extraction |
| **Monitoring** | Prometheus + Promtail | Metrics and log aggregation |
| **Containerization** | Docker Compose | Multi-service orchestration |

### Data Model

```
┌──────────────────┐       ┌──────────────────┐
│   Organization   │       │      User        │
│  ────────────────│       │  ────────────────│
│  corporate_email │◄──────│  organization_id │
│  payment_status  │       │  role (groups)   │
└──────────────────┘       └────────┬─────────┘
                                    │
                                    ▼
┌──────────────────┐       ┌──────────────────┐
│      Demo        │       │      Step        │
│  ────────────────│       │  ───────────────│
│  video_file      │◄──────│  step_type       │
│  actions_json    │       │  order           │
└──────────────────┘       └────────┬─────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│   Screenshot     │   │  VideoSegment    │   │     Hotspot      │
│  ────────────────│   │  ────────────────│   │  ───────────────│
│  image_file      │   │  video_file      │   │  x, y coords     │
│  timestamp       │   │  start/end       │   │  description     │
└──────────────────┘   └──────────────────┘   └──────────────────┘
```

### Video Processing Pipeline

The asynchronous processing pipeline transforms raw browser recordings into interactive demos:

```python
# 1. Extension uploads WebM + JSON action log
├── video.webm          # Screen recording
└── actions.json        # Click/scroll/type events with timestamps

# 2. Celery task triggered
@shared_task
def run_process_demo(demo_id):
    call_command('process_demo', demo_id=demo_id)

# 3. FFmpeg conversion (WebM → MP4)
ffmpeg -i input.webm -c:v libx264 -preset fast -crf 22 output.mp4

# 4. Frame extraction at click events
for action in actions:
    if action.type == 'click':
        frame = video.get_frame(action.timestamp)
        Screenshot.objects.create(image=frame, timestamp=action.timestamp)

# 5. Video segmentation for scroll/type actions
for action in actions:
    if action.type in ['scroll', 'type']:
        segment = video.subclip(action.start, action.end)
        VideoSegment.objects.create(video=segment)
```


