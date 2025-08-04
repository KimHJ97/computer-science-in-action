# RTMP와 HLS

## 1. HLS

HLS(HTTP Live Streaming)은 가장 널리 사용되는 비디오 스트리밍 프로토콜입니다. 이 프로토콜은 동영상을 여러 개의 작은 세그먼트로 분할하고, 이러한 세그먼트를 HTTP 기반의 웹 서버를 통해 전송합니다. 클라이언트는 이 세그먼트를 다운로드하여 연속적으로 재생함으로써 스트리밍 영상을 시청할 수 있습니다.

__HLS는 영상을 `짧은 조각들(.ts 파일)`로 잘라서, 그 목록을 담은 `재생목록 파일(.m3u8)`을 통해 HTTP로 전송하는 방식입니다.__

### 1-1. HLS 구조 설명

 - `🎥 입력 영상 → 🔀 변환 → 📂 출력 파일`
    - 전체 영상을 일정 시간(보통 2~6초) 단위로 나눔 → .ts (MPEG-TS segment)
    - 조각들의 목록을 .m3u8 형식으로 정리 → 재생 리스트
    - 시청자는 이 .m3u8 파일을 다운받고, 거기에 적힌 .ts 파일을 순서대로 받아서 재생
```bash
# 예시 디렉토리
/hls/
├── stream.m3u8             # 재생 목록 (index)
├── stream1.ts              # 첫 번째 조각 (2~6초 분량)
├── stream2.ts              # 두 번째 조각
├── ...
```
<br/>

 - `작동 흐름(브라우저 입장)`
    - 브라우저가 .m3u8 파일 요청 (GET /stream.m3u8)
    - 내부에서 segment100.ts ~ segment102.ts 항목 확인
    - 첫 ts 파일부터 HTTP로 다운로드 시작 → segment100.ts
    - 재생하면서 다음 segment들도 순차적으로 미리 요청
    - .m3u8 파일을 주기적으로 다시 요청하여 최신 세그먼트 목록 갱신 (라이브 방송일 경우)
```
1. 브라우저가 stream.m3u8 파일 요청
2. 서버가 재생 목록 (.m3u8) 제공
3. 브라우저가 목록에 적힌 .ts 파일을 하나씩 요청
4. 실시간 재생
```
<br/>

 - `m3u8 예시`
    - .m3u8 파일은 HLS(HTTP Live Streaming)의 재생목록(playlist) 역할을 하는 파일이며, 브라우저(또는 플레이어)가 이 파일을 파싱해서 .ts 영상 조각(segment)들을 순서대로 요청한다.
    - .m3u8 = M3U UTF-8 인코딩 형식 파일
    - HLS에서 영상의 세그먼트(ts 파일)들을 순서대로 나열하는 목록
```m3u8
#EXTM3U
#EXT-X-TARGETDURATION:5
#EXT-X-VERSION:3
#EXT-X-MEDIA-SEQUENCE:100

#EXTINF:5.0,
stream100.ts
#EXTINF:5.0,
stream101.ts
#EXTINF:5.0,
stream102.ts
```

 - `m3u8 파일 흐름 예시`
   - HLS의 stream.m3u8 파일은 라이브 스트리밍 중에 계속해서 갱신되며, 그 안에 있는 #EXT-X-MEDIA-SEQUENCE 값도 계속 증가한다.
   - 세그먼트가 한 개씩 밀려나며 계속 추가됨
   - 시퀀스 번호는 항상 오름차순 증가
   - 플레이어가 중복 없이 새로운 ts 조각을 받기 위해. 중간에 끊겨도 재접속 시 최신 위치부터 재생 가능하게 하기 위해.
```m3u8
###### 초기 m3u8 파일
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:5
#EXT-X-MEDIA-SEQUENCE:600

#EXTINF:5.0,
segment600.ts
#EXTINF:5.0,
segment601.ts
#EXTINF:5.0,
segment602.ts

###### 5초 후 업데이트
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:5
#EXT-X-MEDIA-SEQUENCE:601

#EXTINF:5.0,
segment601.ts
#EXTINF:5.0,
segment602.ts
#EXTINF:5.0,
segment603.ts

```

### 1-2. HLS 장단점

 - `HLS 장점`
    - ✅ HTTP 기반 → CDN, 브라우저 캐시, HTTPS 쉽게 적용
    - ✅ 거의 모든 디바이스에서 작동
    - ✅ 실시간 + VOD 모두 가능
    - ✅ 네트워크 상태에 따라 자동 화질 조절
 - `HLS 단점`
    - ❗ 지연 시간이 있음 (기본 HLS는 6~30초 지연)
    - ❗ 초저지연이 필요한 서비스(예: 화상회의, 게임)는 WebRTC가 더 적합
    - 이 문제를 해결하기 위해 Apple은 Low-Latency HLS (LL-HLS)도 제안하고 있습니다.

<br/>

## 2. RTMP → HLS 구조

 - `📤 송출 경로 (Input)`
    - 송출자 (OBS 등) → RTMP 프로토콜 사용
    - 주소 예시: rtmp://your-server/live/streamKey
    - 서버는 이를 실시간으로 수신
 - `⚙️ 서버 처리`
    - RTMP 수신 후, 미디어 스트림을 HLS용으로 인코딩 및 세그먼트 분할
    - .m3u8 (HLS playlist) + .ts (MPEG-TS chunks) 생성
 - `📥 시청 경로 (Output)`
    - 클라이언트(브라우저, 모바일 등)는 HLS로 재생
    - 주소 예시: http://your-server/hls/streamKey.m3u8
    - HTML5 video 태그 또는 hls.js 로 재생

```
[송출자]
 (RTMP)
   ↓
[RTMP 수신 서버 (nginx-rtmp)]
   ↓
[HLS 변환 (.m3u8 + .ts)]
   ↓
(HTTP)
[시청자 (웹, 앱)]
```
<br/>

## 3. RTMP와 HLS 분리

 - ✅ __RTMP는 송출에 적합__: 낮은 지연, 설정이 단순하며 OBS 등에서 기본 지원
 - ✅ __HLS는 시청에 적합__: HTML5 브라우저가 기본 지원, 안정적이고 확장성 높음
 - ❌ __RTMP는 클라이언트에서 재생이 어려움__: 플래시가 사라지면서 브라우저에서 직접 재생 불가

| 구성 요소              | 역할           | 프로토콜                 |
| ------------------ | ------------ | -------------------- |
| OBS                | 송출 도구        | RTMP                 |
| NGINX + RTMP 모듈    | 수신 서버        | RTMP 수신 + HLS 생성     |
| HTTP 서버 (같은 nginx) | 시청자에게 HLS 제공 | HLS (HTTP + `.m3u8`) |
| 브라우저               | 시청자 플레이어     | HTML5 (HLS 지원)       |
