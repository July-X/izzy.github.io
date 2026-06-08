---
title: "tts-tool 的 WAV 编码与格式转换：手写 RIFF Header 的实践"
date: 2026-06-06 15:00:00
tags:
  - Electron
  - TTS
  - WAV
  - 音频
categories:
  - Electron实战
top_img: false
---

<!-- more -->

TTS 工具需要支持 WAV 和 MP3 两种导出格式。API 返回的音频数据有两种形态：一种是 JSON 里包着 Base64 字符串，另一种是裸二进制流。`tts-tool` 的做法是：能用二进制就用二进制，Base64 只作为 fallback。同时手写了一个 WAV 生成器，用于 API 不可用时的合成降级。

## RIFF/WAVE 格式结构

WAV 文件由 RIFF 容器包裹，结构很简单：

```
偏移   大小   内容
0      4     "RIFF" 标识
4      4     文件总大小 - 8
8      4     "WAVE" 格式标识
12     4     "fmt " 子块标识
16     4     fmt 子块大小（固定 16）
20     2     音频格式（1 = PCM）
22     2     声道数（1 = 单声道）
24     4     采样率（44100）
28     4     字节率（采样率 × 声道 × 位深/8）
32     2     对齐帧大小（声道 × 位深/8）
34     2     位深（16）
36     4     "data" 子块标识
40     4     数据大小
44     ...   PCM 音频数据
```

总共 44 字节的 header，之后是 PCM 数据。手写这 44 字节比引入一个音频库轻量得多。

## 手写 RIFF Header

`tts.js` 的 `_generateWav` 方法：

```typescript
_generateWav(text: string, speed: number, pitch: number): Buffer {
  const sampleRate = 44100;
  const baseFreq = 220 + ((pitch || 0) * 10);
  const duration = Math.min(Math.max(text.length / 12, 0.5), 30);
  const numSamples = Math.floor(sampleRate * duration);
  const buffer = Buffer.alloc(44 + numSamples * 2);

  // RIFF header
  buffer.write('RIFF', 0);
  buffer.writeUInt32LE(36 + numSamples * 2, 4);
  buffer.write('WAVE', 8);
  buffer.write('fmt ', 12);
  buffer.writeUInt32LE(16, 16);
  buffer.writeUInt16LE(1, 20);      // PCM 格式
  buffer.writeUInt16LE(1, 22);      // 单声道
  buffer.writeUInt32LE(sampleRate, 24);
  buffer.writeUInt32LE(sampleRate * 2, 28);  // 字节率
  buffer.writeUInt16LE(2, 32);      // 对齐帧
  buffer.writeUInt16LE(16, 34);     // 16-bit
  buffer.write('data', 36);
  buffer.writeUInt32LE(numSamples * 2, 40);

  // ... 填充 PCM 数据
}
```

几个关键参数的含义：

- **采样率 44100**：CD 音质标准，对 TTS 足够
- **16-bit PCM**：每个采样点 2 字节，动态范围约 96dB
- **单声道**：TTS 输出不需要立体声
- **duration**：根据文本长度估算，每秒约 12 个字符，最短 0.5 秒，最长 30 秒

`buffer.writeUInt32LE` 的参数顺序是（值, 偏移），写入小端序的 4 字节无符号整数。

## PCM 数据生成

合成音频不是真正的语音，而是基于文本哈希生成的谐波信号：

```typescript
let hash = 0;
for (let i = 0; i < text.length; i++) {
  hash = ((hash << 5) - hash + text.charCodeAt(i)) | 0;
}

const speedFactor = speed || 1.0;
const harmonics = 3 + Math.abs(hash % 4);  // 3-6 次谐波

for (let i = 0; i < numSamples; i++) {
  const t = (i / sampleRate) * speedFactor;
  const fadeLen = Math.min(0.05, duration * 0.1);
  const fade = Math.min(1, t / fadeLen, (duration - t / speedFactor) / fadeLen);

  let sample = 0;
  for (let h = 1; h <= harmonics; h++) {
    const amp = 0.3 / h;
    const phase = ((hash * h) % 1000) / 1000 * Math.PI * 2;
    sample += Math.sin(2 * Math.PI * baseFreq * h * t + phase) * amp;
  }
  sample += Math.sin(2 * Math.PI * 5 * t) * 0.05;  // 低频嗡鸣
  sample *= fade * 0.6;

  buffer.writeInt16LE(Math.round(sample * 32767), 44 + i * 2);
}
```

这段代码做的事：

1. 从文本生成哈希，决定谐波数和相位
2. 叠加多个正弦波（基频 + 谐波），制造"有人在说话"的假象
3. 加淡入淡出包络，避免爆音
4. 写入 16-bit PCM 数据（乘 32767 映射到 Int16 范围）

这不是真正的语音合成，是降级方案。API 正常时用真实 TTS 音频，API 挂了才走这条路。

## 双响应处理

TTS API 的响应格式不统一。`_callApiOnce` 方法处理了两种情况：

```typescript
if (contentType.includes('application/json')) {
  // 格式 1：JSON 包裹的 Base64
  const jsonResponse = JSON.parse(rawResponse.toString('utf-8'));
  if (jsonResponse.choices?.[0]?.message?.audio?.data) {
    return Buffer.from(jsonResponse.choices[0].message.audio.data, 'base64');
  } else if (jsonResponse.choices?.[0]?.message?.content) {
    const content = jsonResponse.choices[0].message.content;
    if (content.startsWith('data:audio')) {
      // data:audio/wav;base64,xxxx 格式
      return Buffer.from(content.split(',')[1], 'base64');
    } else {
      return Buffer.from(content, 'base64');
    }
  }
} else {
  // 格式 2：裸二进制
  if (rawResponse.length > 100 && rawResponse[0] === 0x52) {
    // 0x52 = 'R'，WAV 文件的第一个字节
    return rawResponse;
  }
}
```

为什么要检查 `rawResponse[0] === 0x52`？裸二进制响应可能不是音频，先验证文件头是 "RIFF"（0x52 0x49 0x46 0x46）再当 WAV 用。

Base64 解码很简单：`Buffer.from(base64String, 'base64')`。Node.js 原生支持，不需要额外库。

## Base64 vs 二进制的选择

API 返回 Base64 的原因：JSON 不能直接包含二进制数据，所以把音频编码成 Base64 字符串嵌入 JSON。

直接返回二进制更高效，省掉 Base64 编码/解码的 33% 体积膨胀。但 API 不是自己能控制的，两种都要支持。

处理优先级：先尝试解析 JSON，找到音频数据就解码；找不到就当裸二进制处理。这样不管 API 返回哪种格式都能接住。

## 音频缓存

生成的音频文件会被缓存：

```typescript
_buildCacheKey(text, voiceId, speed, styleDesc, format, ...) {
  const key = `${text}|${voiceId}|${speed}|${styleDesc || ''}|${format}|...`;
  let hash = 0;
  for (let i = 0; i < key.length; i++) {
    hash = ((hash << 5) - hash + key.charCodeAt(i)) | 0;
  }
  return `tts_${Math.abs(hash).toString(36)}`;
}
```

缓存键由文本内容、音色、语速等参数哈希生成。相同参数的重复请求直接返回缓存文件，不重新调 API。缓存上限 100 条，超出时淘汰最老的。

## 导出格式：WAV 和 MP3

WAV 是无损格式，体积大但质量高。MP3 是有损压缩，体积小但音质有损。对于 TTS 场景，WAV 更合适：

- 游戏引擎（Godot、Unity）直接读 WAV，不需要额外解码
- WAV 是 PCM 裸数据，处理简单
- TTS 生成的音频通常很短（几秒到几十秒），WAV 体积可以接受

MP3 导出留给需要小文件的场景（比如分享到社交平台）。

> WAV 格式的简洁性是它被广泛支持的原因。44 字节的 header，之后全是 PCM 数据，任何语言都能解析。手写 RIFF header 比引入一个音频处理库更可控。
