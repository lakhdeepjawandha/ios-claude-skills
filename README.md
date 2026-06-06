# iOS Claude Skills — by Lakhdeep

> A public library of expert-level Claude skills for native iOS, iPadOS, macOS, visionOS, and watchOS development.

Built and maintained by **Lakhdeep** — experienced iOS developer.

---

## Available skills

| Skill | Status | Covers |
|---|---|---|
| [ios-graphics](./ios-graphics/) | ✅ Ready | Core Graphics, Core Animation, SwiftUI Canvas, Metal, SpriteKit, SceneKit, CIFilter |
| [ios-camera](./ios-camera/) | 🚧 Coming soon | AVFoundation, Vision framework, photo capture, real-time filters |
| [ios-swift-language](./ios-swift-language/) | 🚧 Coming soon | Swift concurrency, generics, protocols, macros, Swift 5.9+ |
| [ios-ar-vr](./ios-ar-vr/) | 🚧 Coming soon | ARKit, RealityKit, visionOS, SceneKit, LiDAR |
| [ios-banking-fintech](./ios-banking-fintech/) | 🚧 Coming soon | Apple Pay, Secure Enclave, Keychain, Face ID, CryptoKit |
| [ios-chat-messaging](./ios-chat-messaging/) | 🚧 Coming soon | WebSockets, Firebase, push notifications, message UI |
| [ios-networking](./ios-networking/) | 🚧 Coming soon | URLSession, async/await networking, Combine, REST, GraphQL |
| [ios-ml-coreml](./ios-ml-coreml/) | 🚧 Coming soon | CoreML, CreateML, Vision models, NLP, on-device inference |

---

## How to install a skill

1. Download the `.skill` file from the release you want
2. In Claude.ai → Settings → Skills → Install from file
3. Select the downloaded `.skill` file

Or copy the raw `SKILL.md` content from this repo into a custom Claude skill.

---

## Skill structure

Each skill follows this layout:

```
ios-{name}/
├── SKILL.md              ← main skill (loaded automatically by Claude)
└── references/
    ├── {framework}.md    ← deep-dive references (loaded on demand)
    └── ...
```

The `SKILL.md` is kept lean (< 500 lines) so it loads fast. Framework-specific reference
files are read by Claude only when the task needs them.

---

## Contributing

Found a bug or want to add a reference file? PRs welcome.
Please keep all code examples in **Swift 5.9+** and tested on **iOS 17+**.

---

*Made with ❤️ by Lakhdeep — iOS developer & Claude skill author*
