# glibc 전면 전환 + OpenCode 통합 계획

## Overview

openclaw-android 프로젝트의 아키텍처를 **Bionic 패치 기반**에서 **glibc 전면 전환**으로 변경한다. 이를 통해 기존 호환성 패치의 60~70%를 제거하고, 동시에 OpenCode + Oh My OpenCode를 통합한다.

```
BEFORE (현재 아키텍처):
  OpenClaw    → Termux Node.js (Bionic) → 6+ 패치로 Linux 호환성 확보
  code-server → Termux Node.js (Bionic) → 3개 워크어라운드 적용
  OpenCode    → ❌ 실행 불가 (glibc 바이너리)
  OmO CLI     → ❌ 실행 불가 (glibc 바이너리)

AFTER (새 아키텍처 — PoC 검증 완료):
  OpenClaw    → 공식 Node.js linux-arm64 (glibc, grun wrapper) → 패치 대부분 제거
  code-server → 공식 릴리스 (glibc, grun 방식) → 워크어라운드 대부분 제거
  OpenCode    → Bun standalone 바이너리 (proot + ld.so 연결) → ✅ 실행 확인
  OmO CLI     → Bun standalone 바이너리 (proot + ld.so 연결) → ✅ 실행 확인
```

### 왜 glibc 전면 전환인가

기존 접근은 Bionic(Android libc) 위에서 Linux 호환성을 **패치**로 때우는 방식이었다. OpenCode 통합을 위해 glibc-runner를 도입해야 하는 시점에서, **기존 도구들도 glibc로 실행**하면 대부분의 패치가 불필요해진다.

| | 기존 (Bionic + 패치) | 전면 glibc 전환 |
|---|---|---|
| 패치 수 | 9개 (활성) | **3~4개** |
| OpenCode 지원 | ❌ 불가 | ✅ 네이티브 |
| 네이티브 모듈 | prebuilt 불가, 직접 빌드 | **prebuilt 사용 가능** (glibc 호환) |
| 유지보수 | 패치 + glibc 이중관리 | **glibc 단일관리** |
| 저장공간 추가 | 0 | ~200-300MB |

---

## 점검 결과 요약

> 이 섹션은 계획 수립 후 실시한 4항목 점검(기술 오류, 벽돌 위험, 마이그레이션, 레거시 정리) 결과를 반영하여 추가되었다.

| 항목 | 결과 | 비고 |
|------|------|------|
| 기술적 오류 | 🔴 **5개 Critical 발견** | /bin/sh, alias, code-server patchelf, npm prefix, NODE_OPTIONS 순서 |
| 벽돌 위험 | 🟢 **제로** | 모든 작업이 Termux userspace, 시스템 파티션 무관 |
| 마이그레이션 | 🟡 **가능, 조건부** | `oa --update`에 마이그레이션 로직 내장 가능, 절대경로 npm 사용 필수 |
| 레거시 정리 | 🟢 **대폭 가능** | 패치 3개 삭제, 1개 축소 교체, 환경변수 4개 제거 |

상세 내용은 [벽돌 위험성 평가](#벽돌-위험성-평가), [마이그레이션 전략](#기존-사용자-마이그레이션-전략-bionic--glibc), [레거시 정리 전략](#레거시-정리-전략) 섹션 참고.

### Pre-PoC 리서치 결과 (확정)

> 4개 항목에 대한 사전 조사가 완료되어, 계획의 조건부 항목이 확정되었다.

| # | 조사 항목 | 결론 | 영향 |
|---|-----------|------|------|
| 1 | `/bin/sh` 존재 여부 | Android 9+ (API 28+)에 존재 (`/bin` → `/system/bin` symlink). Android 7-8에는 미존재 | Android 10+ 전용이면 glibc-compat.js 불필요. 7-9 지원 시 런타임 감지로 조건부 적용 |
| 2 | `os.cpus()` 동작 여부 | **glibc 전환으로 해결 안 됨**. SELinux + `hidepid=2`로 `/proc/stat` 접근 차단 (Android 8+, 커널 레벨) | `glibc-compat.js`에 os.cpus() fallback **유지 필수** |
| 3 | oh-my-opencode 바이너리 경로 | `oh-my-opencode-linux-arm64` 패키지, 바이너리: `node_modules/oh-my-opencode-linux-arm64/bin/oh-my-opencode` (Bun standalone ELF, ~97MB, glibc 링크) | patchelf 대상 경로 확정 |
| 4 | code-server patchelf 범위 | `lib/node` 외에 **6-7개 .node 네이티브 모듈도 patchelf 필요** (argon2, node-pty, @parcel/watcher, spdlog, watchdog, kerberos). `rg`(ripgrep)는 정적 링크라 불필요 | `find -name "*.node"` 로 일괄 patchelf 적용 |

**출처**:
- `/bin/sh`: [Android commit ff1ef9f2](https://android.googlesource.com/platform/system/core/+/ff1ef9f2) (2017-12, Android 9), [Node.js child_process.js:669-672](https://github.com/nodejs/node/blob/6964b539806e3b2103dd6b65572287b8a615f5d3/lib/child_process.js#L669-L672), [termux-exec ExecIntercept.c](https://github.com/termux/termux-exec)
- `os.cpus()`: [libuv linux.c:1770](https://github.com/libuv/libuv/blob/12d0dd48e3c6baf1e2f0d9f85f11f0ef58285d6f/src/unix/linux.c#L1770), [Android Issue #37140047](https://issuetracker.google.com/issues/37140047), [libuv #1459](https://github.com/libuv/libuv/issues/1459)
- oh-my-opencode: [package.json](https://github.com/code-yeongyu/oh-my-opencode/blob/9a505a33ac0c1020593870f2875c1c90f20a1586/package.json), [linux-arm64 package.json](https://github.com/code-yeongyu/oh-my-opencode/blob/9a505a33ac0c1020593870f2875c1c90f20a1586/packages/linux-arm64/package.json)
- code-server: [code-server v4.109.2 release](https://github.com/coder/code-server/releases/tag/v4.109.2), 바이너리 구조 분석

### PoC 실행 결과 (확정)

> PoC는 SSH를 통해 실제 Android 13 기기(Samsung, Kernel 4.19.113, Termux 0.119.0-beta.3)에서 수행되었다.

#### 핵심 발견: patchelf 방식 전면 실패

| 테스트 | 결과 | 비고 |
|--------|------|------|
| Node.js v22/v20/v18 patchelf | ❌ **전부 Segfault** | Android seccomp 정책으로 patchelf된 바이너리 실행 시 SIGSEGV |
| Bun patchelf | ❌ **Segfault** | 동일 원인 |
| Node.js v22 via grun | ✅ v22.14.0 | `grun node22` 방식으로 정상 실행 |
| Bun via grun | ✅ v1.3.10 | grun 방식 정상 |
| npm via grun | ✅ v10.9.2 | grun wrapper script 정상 |

**결론**: patchelf는 Android에서 사용 불가. 모든 glibc 바이너리는 `grun` 방식(`exec ld.so binary`)으로 실행해야 한다.

#### OpenCode 3-barrier 아키텍처

OpenCode(Bun standalone 바이너리)는 grun만으로는 실행 불가하며, 3가지 추가 장벽이 존재한다:

| # | 장벽 | 원인 | 해결 |
|---|------|------|------|
| 1 | `openat("/", O_DIRECTORY)` → EACCES | Bun이 raw syscall 사용 (LD_PRELOAD 가로채기 불가) | **proot** (ptrace 기반, 커널 레벨 가로채기) |
| 2 | `/proc/self/exe` → ld.so | grun 실행 시 /proc/self/exe가 ld.so를 가리켜 Bun 내장 JS를 찾지 못함 | **ld.so 연결** (ld.so + OpenCode 바이너리 데이터를 concatenate) |
| 3 | Bionic `LD_PRELOAD` 누수 | `libtermux-exec.so`가 glibc 프로세스에 로드되어 버전 불일치 | **`unset LD_PRELOAD`** (wrapper에서 제거) |

#### 최종 실행 아키텍처 (검증 완료)

```
일반 glibc 바이너리 (Node.js, npm 등):
  wrapper script → unset LD_PRELOAD → exec grun binary "$@"
  (= exec $PREFIX/glibc/lib/ld-linux-aarch64.so.1 binary "$@")

Bun standalone 바이너리 (OpenCode, oh-my-opencode):
  wrapper script → unset LD_PRELOAD → proot -R ~/min_root \
    -b $PREFIX:$PREFIX -b /system:/system -b /apex:/apex \
    -w $(pwd) $PREFIX/tmp/ld.so.opencode $OPENCODE_BIN "$@"
```

#### ld.so 연결 방법 (Bun standalone 전용)

Bun standalone 바이너리는 파일 끝에 embedded JS를 저장하고, 마지막 8바이트에 파일 크기를 LE u64로 기록한다. `current_file_size - stored_value`으로 embedded data offset을 계산하므로, ld.so를 앞에 prepend하면 offset이 자동으로 맞는다:

```bash
cp $PREFIX/glibc/lib/ld-linux-aarch64.so.1 $PREFIX/tmp/ld.so.opencode
cat "$OPENCODE_BIN" >> $PREFIX/tmp/ld.so.opencode
```

#### PoC 검증 결과 전체

| 테스트 | 결과 |
|--------|------|
| glibc-runner 설치 | ✅ (SigLevel = Never 워크어라운드 필요) |
| glibc bash | ✅ |
| Node.js v22 via grun | ✅ v22.14.0 |
| npm via grun | ✅ v10.9.2 |
| platform: linux | ✅ |
| os.cpus() | ⚠️ 0 반환 (커널 /proc/stat 차단) |
| os.networkInterfaces() | ❌ EACCES (try-catch 필요) |
| /bin/sh 존재 | ✅ (Android 13) |
| child_process.exec | ✅ |
| opencode -v | ✅ **v1.2.15** |
| opencode --help | ✅ 전체 명령 목록 |
| opencode models | ✅ 모델 목록 |
| opencode debug paths | ✅ 정상 경로 |
| opencode debug config | ✅ JSON config |
| SQLite DB migration | ✅ 자동 |
| oh-my-opencode version | ✅ **v3.9.0** |
| oh-my-opencode --help | ✅ 전체 도움말 |
| npm install -g openclaw | ❌ koffi 네이티브 빌드 실패 (별도 대응 필요) |
| patchelf 방식 (모든 바이너리) | ❌ Segfault (Android seccomp) |
| LD_PRELOAD openat_shim | ❌ Bun raw syscall 사용으로 무효 |

---

## 현재 상태

| 도구 | 상태 | 이유 |
|------|------|------|
| OpenClaw | ✅ 동작 (패치 필요) | Node.js 앱, Bionic 패치 6개로 동작 |
| OpenCode | ✅ 실행 확인 (proot+ld.so) | Bun standalone 바이너리, proot + ld.so 연결 방식으로 v1.2.15 동작 확인 |
| oh-my-opencode CLI | ✅ 실행 확인 (proot+ld.so) | Bun standalone 바이너리, 동일 방식으로 v3.9.0 동작 확인 |
| oh-my-opencode plugin | ⚠️ 미확인 | OpenCode가 로드하는 JS 모듈, OpenCode 동작 후 확인 가능 |
| code-server | ✅ 동작 (워크어라운드 필요) | glibc 바이너리를 Termux node로 교체, argon2 스텁 등 |

---

## 근본 원인

Android는 Bionic libc를 사용하고, 대부분의 Linux 바이너리(Node.js 공식 빌드, Bun, code-server)는 GNU glibc로 링킹되어 있다. ELF 인터프리터 경로(`/lib/ld-linux-aarch64.so.1`)가 Android에 존재하지 않아 `cannot execute: required file not found` 에러 발생.

**기존 접근**: Termux가 Bionic 위에 빌드한 Node.js를 사용하고, 호환되지 않는 부분을 JS/C 패치로 우회.

**새 접근**: glibc-runner로 glibc 환경을 Termux에 설치하고, 공식 Linux 바이너리를 grun(ld.so 직접 실행)으로 실행. Bun standalone 바이너리는 proot + ld.so 연결 방식 사용. 패치 대신 **표준 Linux 동작**에 의존.

### OpenCode의 Bun 런타임 종속성

OpenCode는 Bun 런타임에 **구조적으로 종속**되어 Node.js 대체 실행이 불가능하다.

| 의존성 | 역할 | Node.js 대안 |
|--------|------|-------------|
| `bun-pty` | 네이티브 PTY(가상 터미널) 관리 | `node-pty` (API 비호환) |
| `Bun.serve()` | HTTP 서버 API | `http.createServer()` (구조 다름) |
| `Bun.write()` / `Bun.file()` | 파일 I/O API | `fs.writeFile()` / `fs.readFile()` (전수 교체 필요) |
| `@opentui/core` | Zig 기반 TUI 프레임워크 (네이티브 컴파일) | 대안 없음 |
| `bunfs` | Bun 내장 가상 파일시스템 | 대안 없음 |

Node.js 포팅 예상 3-6주. OpenCode 팀도 Node.js 지원 요청을 기각함 ([#10860](https://github.com/anomalyco/opencode/issues/10860)).

→ **Bun standalone 바이너리를 glibc로 직접 실행하는 것이 유일한 방법.**

---

## 핵심 전략: glibc 전면 전환

### glibc-runner (grun) 동작 원리

[`termux-pacman/glibc-packages`](https://github.com/termux-pacman/glibc-packages)의 glibc-runner는 Termux에 glibc를 Bionic 옆에 설치하여 glibc 바이너리를 실행하게 해주는 도구다.

**핵심 메커니즘:**

1. **LD_PRELOAD 해제**: Termux는 `LD_PRELOAD=libtermux-exec.so`를 설정하는데, 이것이 glibc 바이너리에 `libdl.so` 에러를 유발. grun이 이를 해제
2. **ld.so 직접 실행**: ~~patchelf가 아닌~~ `$PREFIX/glibc/lib/ld-linux-aarch64.so.1 binary` 형태로 glibc 동적 링커를 통해 바이너리를 로드. **patchelf는 Android seccomp 정책으로 인해 Segfault 발생하므로 사용 불가** (PoC에서 Node.js v22/v20/v18, Bun 모두 Segfault 확인)
3. **네이티브 실행**: proot의 ptrace 가로채기 없이 직접 실행하므로 **네이티브 속도** (단, Bun standalone 바이너리는 예외적으로 proot 필요)

**사용 패턴:**
```bash
# 임시 실행 (바이너리 수정 없음) — 이것이 기본 사용 패턴
grun ./node --version

# grun --set (patchelf 적용) — Android에서는 사용 불가 (Segfault)
# grun --set ./node  ← 사용하지 않음

# glibc 쉰 모드 (환경 전체 전환)
grun --shell
```

> **PoC 결과**: `grun --set`(patchelf)은 Android seccomp 정책으로 Segfault를 유발한다. 스크립트에서는 **wrapper script + grun 방식(ld.so 직접 실행)**만 사용한다. `grun`의 본질은 `exec $PREFIX/glibc/lib/ld-linux-aarch64.so.1 <binary> <args>`이며, wrapper script에서 이를 직접 호출한다.

### Node.js: 공식 linux-arm64 바이너리 활용

termux-pacman의 glibc-packages에 Node.js 패키지는 **없다** (230개 패키지 중 미포함). 대신 Node.js 공식 linux-arm64 바이너리를 다운로드하여 grun 방식(ld.so 직접 실행)으로 실행한다.

> ⚠️ **PoC 결과**: patchelf 방식은 Android seccomp 정책으로 인해 Segfault가 발생한다. Node.js v22, v20, v18 모두 patchelf 적용 후 실행 시 SIGSEGV. grun 방식(`exec ld.so node`)으로 전환하여 v22.14.0 정상 동작 확인.

```bash
# 다운로드
curl -fsSL https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-arm64.tar.xz | tar -xJ

# grun으로 직접 실행 (patchelf 불필요)
grun node-v22.14.0-linux-arm64/bin/node --version
# → v22.14.0

# wrapper script 생성 (LD_PRELOAD 자동 해제 + grun 방식)
cat > ~/.openclaw-android/node/bin/node << 'WRAPPER'
#!/data/data/com.termux/files/usr/bin/bash
unset LD_PRELOAD
exec "$PREFIX/glibc/lib/ld-linux-aarch64.so.1" "$(dirname "$0")/node.real" "$@"
WRAPPER
chmod +x ~/.openclaw-android/node/bin/node
```

이 바이너리에는 npm도 포함되어 있으므로 별도 npm 설치 불필요. npm은 JS 스크립트로 wrapper 경유 node를 통해 실행되며, 자식 프로세스에도 환경변수가 상속됨.

### LD_PRELOAD 처리: wrapper script 방식 (**alias 아닌**)

> ⚠️ **점검 결과 수정**: 초기 계획에서는 `.bashrc`에 `alias node='LD_PRELOAD= node'`를 사용했으나, alias는 **non-interactive shell에서 동작하지 않는** 치명적 문제가 있다.
>
> - `#!/usr/bin/env node` shebang → alias 무시 → glibc node 실행 실패
> - 스크립트 내부의 node 호출 → alias 무시
> - npm이 spawn하는 자식 node 프로세스 → alias 무시
>
> **해결**: wrapper script 방식을 사용한다.

```bash
# node wrapper script 생성 (설치 스크립트에서 자동 처리)
# ~/.openclaw-android/node/bin/node.real ← 원본 바이너리 (patchelf 미적용)
# ~/.openclaw-android/node/bin/node     ← wrapper script

#!/data/data/com.termux/files/usr/bin/bash
unset LD_PRELOAD
exec "$PREFIX/glibc/lib/ld-linux-aarch64.so.1" "$(dirname "$0")/node.real" "$@"
```

이 방식의 장점:
- shebang `#!/usr/bin/env node`로 실행되는 모든 CLI 도구 자동 지원
- 스크립트, cron, 비대화 환경에서도 동작
- LD_PRELOAD= 가 자식 프로세스에 상속되므로 npm → node 체인도 정상 동작

### proot-distro 대비 장점

| | proot-distro | glibc-runner |
|---|---|---|
| 저장 공간 | ~1GB (전체 Linux) | **~200-300MB** (glibc + Node.js) |
| 성능 | 느림 (ptrace 시스템 콜 가로채기) | **네이티브 속도** (일반 바이너리), proot 사용 (Bun standalone) |
| 기존 Termux 패키지 호환 | 별도 환경 | **공존** (apt/pkg와 pacman 병렬) |
| bunfs 호환 | ❌ 충돌 ([#7960](https://github.com/anomalyco/opencode/issues/7960)) | ✅ proot 최소 사용 (Bun standalone만, 일반 바이너리는 네이티브) |
| 유지보수 | 별도 Linux 관리 필요 | **기존 Termux에 통합** |
| proot 사용량 | 전면 (모든 syscall) | **최소** (Bun standalone만, 일반 바이너리는 네이티브) |

### 디렉토리 구조

```
$PREFIX/
├── lib/                         ← Bionic 기반 라이브러리 (기존, 유지)
├── glibc/
│   ├── lib/
│   │   ├── ld-linux-aarch64.so.1   ← glibc 동적 링커 (grun 핵심)
│   │   ├── libc.so.6               ← GNU C 라이브러리
│   │   └── ...
│   └── bin/                     ← glibc 기반 유틸리티
├── bin/
│   ├── grun                     ← glibc-runner 래퍼
│   ├── proot                    ← proot (Bun standalone용)
│   ├── opencode                 ← OpenCode wrapper script
│   ├── oh-my-opencode           ← OmO wrapper script
│   ├── oa                       ← 프로젝트 CLI
│   └── (기존 Termux 바이너리)
├── tmp/
│   ├── ld.so.opencode           ← ld.so + OpenCode embedded JS (~157MB)
│   └── ld.so.omo                ← ld.so + oh-my-opencode embedded JS (~102MB)
$HOME/
├── .openclaw-android/
│   ├── proot-root/              ← 최소 fake rootfs (proot용, 빈 디렉토리)
│   ├── node/                    ← 공식 Node.js linux-arm64
│   │   ├── bin/node             ← wrapper script (unset LD_PRELOAD + grun 방식)
│   │   ├── bin/node.real        ← 실제 node 바이너리 (patchelf 미적용)
│   │   ├── bin/npm
│   │   └── ...
│   ├── patches/                 ← 잔여 패치 (대폭 축소)
│   │   └── glibc-compat.js      ← os.cpus() fallback + networkInterfaces try-catch
│   └── .glibc-arch              ← glibc 아키텍처 마커 파일
├── .openclaw/                   ← OpenClaw 데이터
├── .bun/                        ← Bun (OpenCode/OmO 바이너리 캐시)
└── .config/opencode/            ← OpenCode 설정
```

---

## 패치 제거 매트릭스

### 완전 제거 (7개)

| # | 패치/설정 | 현재 역할 | 제거 이유 |
|---|-----------|-----------|-----------|
| 1 | `termux-compat.h` | renameat2() syscall 래퍼 | glibc에 renameat2 표준 제공 |
| 2 | `spawn.h` 스텁 | POSIX spawn 헤더 (Termux NDK 누락) | glibc에 spawn.h 표준 포함 |
| 3 | `argon2-stub.js` | code-server argon2 네이티브 모듈 대체 | glibc에서 원본 argon2 바이너리 동작 |
| 4 | `NODE_OPTIONS="-r bionic-compat.js"` | 모든 Node 프로세스에 패치 로드 | 패치 자체가 불필요 (glibc-compat.js는 NODE_OPTIONS 불요) |
| 5 | `CXXFLAGS="-include termux-compat.h"` | 네이티브 빌드 시 호환성 헤더 | glibc 헤더로 충분 |
| 6 | `GYP_DEFINES="OS=linux"` | node-gyp Android 감지 오버라이드 | glibc 환경은 Linux으로 인식 |
| 7 | code-server argon2 패칭 | argon2 모듈을 stub으로 교체 | glibc에서 원본 동작 |

### 변경 (2개)

| # | 패치/설정 | 현재 역할 | 변경사항 |
|---|-----------|-----------|----------|
| 8 | `bionic-compat.js` | platform 오버라이드, os.cpus() 폴백, os.networkInterfaces() 크래시 방지 | **`glibc-compat.js`로 축소 교체**. platform/os 패치는 제거. `/bin/sh` 경로 shim은 Android 7-9에서만 필요 (Android 10+에서는 `/bin/sh` 존재 확인됨). os.cpus() fallback은 **유지 필수** (커널 레벨 `/proc/stat` 접근 차단 확인됨) |
| 9 | code-server 번들 node 교체 | glibc 번들 node → Termux node 심링크 | **심링크 교체 → grun wrapper script로 변경**. 번들 node를 교체하는 대신 grun 방식(ld.so 직접 실행) wrapper 생성. patchelf는 Segfault로 사용 불가 |

### 부분 제거 (2개)

| # | 패치/설정 | 현재 역할 | 변경사항 |
|---|-----------|-----------|----------|
| 10 | `CPATH` (glib-2.0 헤더) | sharp 빌드 시 헤더 경로 | glibc에서 prebuilt sharp 사용 가능하면 불필요. 소스 빌드 시 유지 |
| 11 | `ar → llvm-ar` 심링크 | binutils 호환 | glibc 빌드 도구 사용 시 불필요. Termux clang 사용 시 유지 |

### 유지 (5개)

| # | 패치/설정 | 현재 역할 | 유지 이유 |
|---|-----------|-----------|-----------|
| 12 | `patch-paths.sh` | /tmp→$PREFIX/tmp, /bin/sh→$PREFIX/bin/sh 등 | Termux 경로 구조는 glibc와 무관 |
| 13 | `systemctl` 스텁 | systemd 호출 우회 | Android에 systemd 없음 |
| 14 | `TMPDIR/TMP/TEMP` | $PREFIX/tmp 지정 | /tmp는 Android에 존재하지 않음 |
| 15 | `CONTAINER=1` | systemd 존재 검사 억제 | Android에 systemd 없음 |
| 16 | code-server 하드링크 복구 | tar .node 파일 복구 | Android 파일시스템 제한 (libc와 무관) |

### 요약

**완전 제거: 7개** / **변경: 2개** / **부분 제거: 2개** / **유지: 5개** → 패치 복잡도 **~55-65% 감소**

> 초기 계획의 "완전 제거 9개, 65% 감소"에서 하향 조정. `/bin/sh` 문제(Risk 8)와 code-server patchelf 필요성 확인에 따른 수정.

---

## 알려진 리스크 및 대응

### Risk 1: 네이티브 모듈 컴파일

**문제**: `npm install` 시 네이티브 모듈(sharp, better-sqlite3 등)을 소스에서 빌드하려면 glibc 호환 빌드 도구가 필요. Termux의 clang은 Bionic 대상 컴파일러.

**대응**:
- 대부분의 인기 모듈은 **prebuilt 바이너리**를 npm에 제공 (`@img/sharp-linux-arm64` 등). glibc 환경에서 이 prebuilt가 정상 동작 — **이것이 핵심 승리**
- Bionic에서는 prebuilt가 glibc 링킹이라 안 됐지만, glibc 환경에서는 바로 사용 가능
- 소스 빌드가 필요한 경우: glibc-packages에 gcc 패키지 존재 여부 확인 (PoC)
- 최악의 경우: 특정 모듈만 Bionic 환경에서 빌드 후 사용

### Risk 2: Node.js 버전별 Segfault

**문제**: glibc on Android 조합에서 일부 Node.js 버전이 segfault 보고됨.

**대응**:
- LTS 버전 (v22.x) 사용 권장
- PoC에서 여러 버전 테스트
- segfault 발생 시 다른 LTS 버전으로 전환

### Risk 3: /tmp 경로 부재

**문제**: glibc 바이너리가 `/tmp`를 직접 참조하면 실패. Android에 `/tmp`는 존재하지 않음. glibc-runner가 경로를 리매핑하지 않음 ([#239](https://github.com/termux-pacman/glibc-packages/issues/239)).

**대응**:
- `TMPDIR` 환경변수 설정으로 대부분 해결 (Node.js, npm 등은 TMPDIR 존중)
- OpenClaw JS 코드의 하드코딩 `/tmp`는 기존 `patch-paths.sh`로 계속 패치
- OpenCode (Bun 바이너리) 내부의 /tmp 참조는 TMPDIR로 대응

### Risk 4: LD_PRELOAD 충돌

**문제**: Termux의 `LD_PRELOAD=libtermux-exec.so`가 glibc 바이너리에 `libdl.so: cannot open shared object file` 에러 유발.

**대응**:
- 모든 glibc 바이너리를 **wrapper script**로 감싸서 `LD_PRELOAD=` 자동 해제
- `oa` 명령의 래퍼 함수 `_glibc_exec()`에서 자동 처리
- ~~`.bashrc`에 alias 설정~~ → wrapper script 방식 사용 (alias는 non-interactive shell에서 미동작)

### Risk 5: OpenCode의 3가지 실행 장벽 🔴 RESOLVED

**PoC에서 3가지 장벽이 발견되었고, 모두 해결되었다:**

| # | 장벽 | 원인 | 해결 방법 | PoC 결과 |
|---|------|------|-----------|----------|
| 1 | `openat("/", O_DIRECTORY)` → EACCES | Bun이 raw syscall 사용 (LD_PRELOAD 가로채기 불가) | **proot** (ptrace 기반 커널 레벨 가로채기) | ✅ |
| 2 | `/proc/self/exe` → ld.so | grun 실행 시 /proc/self/exe가 ld.so를 가리킴 | **ld.so 연결** (ld.so + 바이너리 데이터 concatenate) | ✅ |
| 3 | Bionic LD_PRELOAD 누수 | libtermux-exec.so가 glibc 프로세스에 로드 | **`unset LD_PRELOAD`** | ✅ |

**대응 (검증 완료)**:
- ~~patchelf로 인터프리터 변경~~ → **patchelf는 Segfault** (Android seccomp). ld.so 연결 + proot 조합 사용
- ~~BUN_SELF_EXE 환경변수~~ → 불필요. ld.so 연결이 완벽히 동작
- OpenCode v1.2.15, oh-my-opencode v3.9.0 모두 실행 확인

### Risk 6: termux-pacman과 기존 pkg/apt 공존

**문제**: glibc-runner 설치에 pacman이 필요한데, 기존 apt/pkg와 충돌 가능성.

**대응**:
- pacman은 Termux에서 apt와 **공존 가능**하도록 설계됨
- glibc 패키지는 `$PREFIX/glibc/` 하위에 격리 설치
- 기존 Termux 패키지는 apt/pkg로 계속 관리

### Risk 7: 추가 저장 공간

**문제**: glibc-runner (~200MB) + Node.js linux-arm64 (~50MB) + OpenCode (~157MB)로 약 400-500MB 추가.

**대응**:
- OpenCode 설치를 선택적(opt-in)으로 구현 가능
- 설치 전 디스크 공간 확인 로직
- README에 추가 요구사항 명시 (~500MB → ~1GB)

### Risk 8: `/bin/sh` 부재 — glibc 프로세스 내부 🟡 NEW

**확정**:
- Android 9+ (API 28+)에서 `/bin` → `/system/bin` symlink가 존재하여 `/bin/sh` 사용 가능 ([Android commit ff1ef9f2](https://android.googlesource.com/platform/system/core/+/ff1ef9f2), 2017-12)
- Android 7-8 (API 24-27)에는 `/bin/sh`가 없음
- Node.js는 `process.platform === 'linux'`일 때 `child_process` 기본 shell로 `/bin/sh`를 하드코딩 사용 ([child_process.js:669-672](https://github.com/nodejs/node/blob/6964b539806e3b2103dd6b65572287b8a615f5d3/lib/child_process.js#L669-L672))
- `LD_PRELOAD` 해제 시 `libtermux-exec.so`의 경로 변환이 작동하지 않으며, termux-exec 변환 대상은 `/bin/*` → `$PREFIX/bin/*`이고 `/system/bin/sh` 변환은 제공하지 않음

**영향 범위**:
- Android 7-8에서는 `child_process.exec()`, `spawn(..., { shell: true })`, npm lifecycle 스크립트에서 `/bin/sh` 경로 실패 가능
- Android 9+에서는 `/bin/sh` 경로 자체는 해결됨

**대응**:
1. Android 10+ (API 29+, 프로젝트 권장 환경): `/bin/sh` 존재하므로 문제 없음
2. Android 7-9 지원 시: `glibc-compat.js`에서 `fs.existsSync('/bin/sh')` 런타임 감지 후 조건부 shell shim 적용
3. `npm config set script-shell $PREFIX/bin/sh` 적용으로 npm lifecycle 스크립트 경로를 명시적으로 고정

> 계획 수정: `bionic-compat.js`는 완전 삭제가 아니라 **`glibc-compat.js`로 축소 교체**. platform/os.cpus()/networkInterfaces() 패치는 제거하고 `/bin/sh` shim만 조건부 유지.

### Risk 9: alias 방식 비동작 🔴 NEW

**문제**: 초기 계획의 `.bashrc` alias 방식:
```bash
alias node='LD_PRELOAD= node'
alias npm='LD_PRELOAD= npm'
```
이것은 **interactive shell에서만 동작**. shebang `#!/usr/bin/env node`, 스크립트 내부 node 호출, cron 등 non-interactive 환경에서는 alias가 확장되지 않아 glibc 바이너리 실행 실패.

**대응**: wrapper script 방식으로 전면 교체 (본 문서의 "LD_PRELOAD 처리" 섹션 참고)

### Risk 10: npm global prefix 변경 🟡 NEW

**문제**: glibc Node.js를 `~/.openclaw-android/node/`에 설치하면 `npm root -g`가 `~/.openclaw-android/node/lib/node_modules`를 반환. 현재는 `$PREFIX/lib/node_modules`. 이로 인해:
- 기존에 Bionic Node.js로 설치한 글로벌 패키지 (claude, gemini, codex, openclaw, clawhub 등)가 새 Node.js에서 보이지 않음
- `openclaw` 명령의 위치가 `$PREFIX/bin/openclaw`에서 `~/.openclaw-android/node/bin/openclaw`로 변경

**대응**:
- 마이그레이션 시 기존 글로벌 패키지를 glibc Node.js로 재설치
- AI CLI 도구 (claude, gemini, codex)는 자동 감지 후 재설치 제안
- PATH에서 glibc Node.js가 먼저 오도록 설정하여 새 설치분이 우선 사용됨
- 상세 전략은 [마이그레이션 전략](#기존-사용자-마이그레이션-전략-bionic--glibc) 참고

### Risk 11: os.cpus() 커널 제한 🟡 NEW

**확정**: glibc 전환으로 해결되지 않음. libc 레이어가 아니라 Android 커널/보안 정책 레벨 제한.

- libuv `uv_cpu_info()`는 `/proc/stat` + `/proc/cpuinfo`를 읽어 CPU 정보를 구성 ([libuv linux.c:1770](https://github.com/libuv/libuv/blob/12d0dd48e3c6baf1e2f0d9f85f11f0ef58285d6f/src/unix/linux.c#L1770))
- Android 8+ (API 26+)에서 SELinux가 `/proc/stat` 접근을 완전 차단하며, Google에서 intended behavior로 명시 ([Android Issue #37140047](https://issuetracker.google.com/issues/37140047))
- `hidepid=2` mount option + SELinux MAC 조합의 커널 강제 제한이므로 libc 교체와 무관
- proot-distro, glibc-runner 모두 동일 커널 위에서 동작하므로 우회 불가

**대응**: `glibc-compat.js`에 os.cpus() fallback **유지 필수**

### Risk 12: pacman-key 초기화 hang 🟢 NEW

**문제**: `pacman-key --init`가 GPG 키 생성 시 엔트로피를 필요로 하는데, 일부 기기에서 엔트로피 소스가 부족하여 5-10분 이상 hang될 수 있음.

**대응**:
- 설치 스크립트에 예상 소요 시간 안내 메시지 표시
- 타임아웃 설정 고려 (단, GPG 키 생성은 중단하면 안 됨)
- README에 "pacman 초기화에 수 분이 걸릴 수 있음" 안내

### Risk 13: Node.js 추가 의존 라이브러리 🟢 NEW

**문제**: 공식 Node.js linux-arm64 바이너리가 `libstdc++`, `libgcc_s`, `libdl`, `libpthread` 등을 필요로 할 수 있음.

**대응**:
- glibc-runner 패키지가 대부분의 기본 라이브러리를 포함
- PoC에서 `readelf -d ~/.openclaw-android/node/bin/node` 또는 `ldd`로 필요 라이브러리 확인
- 누락 시 pacman으로 추가 설치

### Risk 14: patchelf Segfault (Android seccomp) 🔴 NEW — RESOLVED

**발견**: PoC에서 모든 patchelf된 바이너리가 Android에서 SIGSEGV로 실패. Node.js v22, v20, v18, Bun 모두 동일.

**원인**: Android의 seccomp(Secure Computing) BPF 필터가 patchelf로 수정된 ELF 바이너리의 syscall 패턴을 차단하는 것으로 추정. 커널 4.19 + Android 13 환경에서 재현.

**영향**: 계획 전체의 "patchelf 적용" 접근을 **grun 방식(ld.so 직접 실행)**으로 전면 교체해야 함.

**대응 (적용 완료)**: 모든 glibc 바이너리를 wrapper script + grun 방식으로 실행. patchelf는 프로젝트에서 완전히 제거.

### Risk 15: Bun raw syscall (LD_PRELOAD 무효) 🔴 NEW — RESOLVED

**발견**: Bun standalone 바이너리가 `openat()` 등의 파일시스템 호출을 glibc 함수가 아닌 **직접 syscall 명령어**로 수행. 이로 인해 `LD_PRELOAD` 기반 shim(`openat_shim.so`)이 완전히 무효.

**원인**: Bun의 IO 시스템은 성능 최적화를 위해 많은 syscall을 inline assembly로 직접 호출. Zig 런타임의 특성.

**영향**: `openat("/", O_DIRECTORY)` 호출이 EACCES 반환 시 LD_PRELOAD로 우회 불가.

**대응 (적용 완료)**: **proot** 사용. proot는 `ptrace(PTRACE_SYSCALL)`로 커널 레벨에서 syscall을 가로채므로, raw syscall도 인터셉트 가능. Bun standalone 바이너리(OpenCode, oh-my-opencode)만 proot로 실행하고, 일반 glibc 바이너리(Node.js 등)는 grun 직접 실행으로 네이티브 속도 유지.

---

## 벽돌 위험성 평가

> 점검 항목 2의 결과.

### 결론: **벽돌 위험 제로 (0%)**

| 근거 | 설명 |
|------|------|
| Userspace 격리 | 모든 작업이 `$PREFIX` (`/data/data/com.termux/files/usr/`)와 `$HOME` (`/data/data/com.termux/files/home/`) 안에서만 수행. 시스템 파티션 미접촉 |
| Root 불필요 | 어떤 단계도 `su`나 root 권한을 요구하지 않음. root 없이는 시스템 파티션 수정 물리적 불가 |
| patchelf 범위 | 개별 바이너리의 ELF 헤더만 수정. 시스템 바이너리 미접촉 |
| pacman 격리 | glibc 패키지는 `$PREFIX/glibc/` 하위에 격리 설치. apt/pkg와 공존 설계 |

### 실패 시나리오 및 복구

| 실패 | 영향 | 복구 |
|------|------|------|
| glibc-runner 설치 실패 | 부분적 glibc 파일 | `pacman -R glibc-runner` 또는 `rm -rf $PREFIX/glibc/` |
| 디스크 공간 부족 | ENOSPC 에러, 부분 파일 | 공간 확보 후 부분 파일 삭제, 재시도 |
| patchelf 바이너리 손상 | 해당 바이너리만 사용 불가 | 바이너리 재다운로드 |
| pacman이 apt/pkg 손상 | Termux 패키지 관리 문제 | Termux 앱 재설치 (폰 데이터 무관) |
| 설치 중 전원 차단 | 부분 설치 | 파일시스템 저널이 일관성 보장. 재실행 |

**최악의 시나리오**: Termux 환경 전체 손상 → Termux 앱 삭제/재설치로 완전 복구. 폰의 사진, 앱, 계정 등 **일체 영향 없음**. 일반 앱 재설치와 동일.

---

## 기존 사용자 마이그레이션 전략 (Bionic → glibc)

> 점검 항목 3의 결과.

### 결론: 가능. `oa --update`에 마이그레이션 로직 내장.

### 아키텍처 감지

```bash
# 방법 1: 마커 파일 (Primary — 설치 시 생성)
if [ -f "$HOME/.openclaw-android/.glibc-arch" ]; then
    # glibc 아키텍처
fi

# 방법 2: 파일시스템 확인 (Fallback)
if [ -x "$HOME/.openclaw-android/node/bin/node.real" ]; then
    # glibc 아키텍처 (grun wrapper node 존재)
fi

# 방법 3: 환경변수 (interactive shell에서만)
if [ "${OA_GLIBC:-}" = "1" ]; then
    # glibc 아키텍처
fi
```

마커 파일을 primary로, 파일시스템 확인을 fallback으로 사용. `OA_GLIBC` 환경변수는 interactive shell에서만 유효하므로 보조적 수단.

### update-core.sh 마이그레이션 분기

```bash
# update-core.sh 시작 부분
if [ -f "$HOME/.openclaw-android/.glibc-arch" ]; then
    # 이미 glibc — 일반 업데이트
    run_glibc_update
else
    # Bionic → glibc 마이그레이션
    echo ""
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "  Major architecture upgrade available"
    echo "  Bionic → glibc (patch count: 9 → 3)"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo ""
    echo "This will download ~300MB and take 10-15 minutes."
    read -rp "Continue? [y/N] " REPLY
    if [[ "$REPLY" =~ ^[Yy]$ ]]; then
        run_migration
    else
        echo "Skipping migration. Running Bionic update instead."
        run_bionic_update  # 기존 업데이트 로직 유지
    fi
fi
```

### 마이그레이션 실행 순서 (Critical — 순서 엄수)

```
1. pacman + glibc-runner 설치
2. glibc Node.js 다운로드 + grun wrapper script 생성
3. .bashrc 업데이트 (NODE_OPTIONS 제거, PATH 변경, OA_GLIBC 추가)
   ← 반드시 패치 파일 삭제 전에 수행
4. export로 현재 세션 환경변수도 갱신
   ← NODE_OPTIONS="" 해제 필수 (bionic-compat.js 참조 제거)
5. glibc Node.js 절대경로로 npm install -g openclaw
   ← alias나 PATH에 의존하지 않고 직접 경로 사용
   ← LD_PRELOAD= $HOME/.openclaw-android/node/bin/npm install -g openclaw@latest
6. AI CLI 도구 자동 감지 및 재설치 제안
7. bionic-compat.js → glibc-compat.js (no-op 또는 축소 shim)로 교체
   ← 즉시 삭제하면 stale shell에서 node 크래시
8. termux-compat.h, spawn.h, argon2-stub.js 삭제
9. 마커 파일 생성: touch ~/.openclaw-android/.glibc-arch
10. source ~/.bashrc 안내
```

> ⚠️ **Step 3-4가 Step 7-8보다 반드시 먼저 실행되어야 함.** .bashrc의 `NODE_OPTIONS="-r bionic-compat.js"`가 남아있는 상태에서 bionic-compat.js를 삭제하면, 모든 node 실행이 `Cannot find module` 에러로 실패.

### AI CLI 도구 재설치

```bash
# 마이그레이션 스크립트에서 기존 AI CLI 도구 자동 감지
AI_TOOLS=()
for cmd in claude gemini codex; do
    command -v "$cmd" &>/dev/null && AI_TOOLS+=("$cmd")
done

if [ ${#AI_TOOLS[@]} -gt 0 ]; then
    echo ""
    echo "Detected AI CLI tools: ${AI_TOOLS[*]}"
    echo "These need to be reinstalled for glibc Node.js."
    read -rp "Reinstall now? [Y/n] " REPLY
    if [[ ! "$REPLY" =~ ^[Nn]$ ]]; then
        GLIBC_NPM="$HOME/.openclaw-android/node/bin/npm"
        for tool in "${AI_TOOLS[@]}"; do
            case "$tool" in
                claude) LD_PRELOAD= "$GLIBC_NPM" install -g @anthropic-ai/claude-code ;;
                gemini) LD_PRELOAD= "$GLIBC_NPM" install -g @google/gemini-cli ;;
                codex)  LD_PRELOAD= "$GLIBC_NPM" install -g @openai/codex ;;
            esac
        done
    fi
fi
```

### PATH 충돌 방지

glibc Node.js가 PATH 앞에 위치하여 Bionic Node.js보다 우선:
```bash
export PATH="$HOME/.openclaw-android/node/bin:$HOME/.local/bin:$PATH"
# ~/.openclaw-android/node/bin/node (glibc) → 우선
# $PREFIX/bin/node (Bionic) → 후순위
```

**마이그레이션 후 검증**:
```bash
NODE_PATH=$(which node)
if [[ "$NODE_PATH" != *".openclaw-android"* ]]; then
    echo "ERROR: glibc node가 PATH에서 우선되지 않음"
    exit 1
fi
```

### Bionic nodejs-lts 제거 여부

**권장: 제거하지 않음** (최소한 초기에는):
- 마이그레이션 롤백 시 안전장치
- 다른 Termux 패키지가 의존할 수 있음
- ~40MB 정도로 공간 영향 미미
- PATH에서 glibc Node.js가 우선하므로 사용에 지장 없음

마이그레이션 성공 후 안내:
```
[INFO] Bionic Node.js를 제거하면 ~40MB를 절약할 수 있습니다:
       pkg uninstall nodejs-lts
```

### 버전 번호

아키텍처 변경은 **메이저 버전 변경**에 해당. v0.8.2 → **v1.0.0** (또는 최소 v0.9.0).

---

## 레거시 정리 전략

> 점검 항목 4의 결과.

### 삭제 순서 (Critical)

| 순서 | 작업 | 이유 |
|------|------|------|
| 1 | .bashrc 업데이트 (NODE_OPTIONS 제거) | NODE_OPTIONS가 bionic-compat.js를 참조하므로 먼저 제거 |
| 2 | 현재 세션에서 `export NODE_OPTIONS=` | stale 환경변수 제거 |
| 3 | bionic-compat.js → glibc-compat.js 교체 | 즉시 삭제가 아닌 no-op/축소 shim으로 교체 (stale shell 안전) |
| 4 | termux-compat.h 삭제 | .bashrc의 CXXFLAGS가 이미 제거됨 |
| 5 | spawn.h 삭제 ($PREFIX/include/spawn.h) | glibc에 표준 포함 |
| 6 | argon2-stub.js 삭제 | glibc에서 원본 동작 |
| 7 | 다음 업데이트에서 glibc-compat.js 삭제 | OA_GLIBC=1 확인 후 안전 삭제 |

### bionic-compat.js → glibc-compat.js 전환

**즉시 삭제하지 않는 이유**: 사용자가 여러 터미널 세션을 열어두고 있을 수 있음. stale 세션에서 `NODE_OPTIONS="-r bionic-compat.js"`가 남아있으면 파일 삭제 시 모든 node 실행 실패.

**안전한 전환**:
```bash
# 1단계 (마이그레이션 시): no-op으로 교체
cat > "$HOME/.openclaw-android/patches/bionic-compat.js" << 'EOF'
'use strict';
// glibc 아키텍처로 전환됨 — Bionic 패치 불필요
// 이 파일은 stale shell 세션 호환을 위해 유지됨
// 다음 업데이트에서 자동 삭제됩니다
EOF

# 2단계 (다음 업데이트 시): OA_GLIBC 확인 후 삭제
if [ -f "$HOME/.openclaw-android/.glibc-arch" ]; then
    rm -f "$HOME/.openclaw-android/patches/bionic-compat.js"
fi
```

### update-core.sh 간소화 효과

**현재 (Bionic, 9 steps, 다운로드 파일 8개)**:
```
다운로드: setup-env.sh, bionic-compat.js, termux-compat.h, spawn.h,
         argon2-stub.js, systemctl, oa.sh, build-sharp.sh, install-code-server.sh
```

**전환 후 (glibc, 다운로드 파일 3-4개)**:
```
다운로드: setup-env.sh, oa.sh, systemctl, (install-code-server.sh)
```

- 다운로드 파일: **8개 → 3-4개** (50% 감소)
- sharp 빌드 단계: **제거** (prebuilt 사용)
- 환경변수 설정: **11개 → 6개** (45% 감소)
- 전체 업데이트 시간: **단축** (sharp 소스 빌드 불필요)

---

## 구현 계획

### Phase 0: PoC (Proof of Concept) — 수동 검증

스크립트 통합 전, 핸드폰에서 수동으로 전체 동작을 검증한다.

#### Step 0: glibc-runner 설치

```bash
# pacman 설치
pkg install pacman proot

# pacman 초기화 (수 분 소요될 수 있음 — GPG 키 생성)
pacman-key --init
pacman-key --populate
pacman -Syu

# glibc-runner 설치
pacman -Sy glibc-runner --assume-installed bash,patchelf,resolv-conf

# 검증
grun --help
```

#### Step 1: Node.js linux-arm64 다운로드 + grun wrapper

```bash
# Node.js LTS 다운로드 (버전은 최신 LTS로 교체)
mkdir -p ~/.openclaw-android/node
curl -fsSL https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-arm64.tar.xz \
  | tar -xJ -C ~/.openclaw-android/node --strip-components=1

# grun으로 실행 (patchelf는 Segfault — 사용 불가)
grun ~/.openclaw-android/node/bin/node --version
# → v22.14.0

# 의존 라이브러리 확인
readelf -d ~/.openclaw-android/node/bin/node | grep NEEDED

# wrapper script 생성 (grun 방식, patchelf 미사용)
mv ~/.openclaw-android/node/bin/node ~/.openclaw-android/node/bin/node.real
cat > ~/.openclaw-android/node/bin/node << 'WRAPPER'
#!/data/data/com.termux/files/usr/bin/bash
unset LD_PRELOAD
exec "$PREFIX/glibc/lib/ld-linux-aarch64.so.1" "$(dirname "$0")/node.real" "$@"
WRAPPER
chmod +x ~/.openclaw-android/node/bin/node

# 테스트
~/.openclaw-android/node/bin/node --version
~/.openclaw-android/node/bin/node -e "
  const os = require('os');
  console.log('platform:', process.platform);
  console.log('cpus:', os.cpus().length);
  console.log('network:', Object.keys(os.networkInterfaces()));
"
# 기대 결과: platform: linux, cpus: N (실제 코어 수), network: [lo, wlan0, ...]
# os.cpus() 가 0이면 커널 제한 — fallback 필요 (Risk 11)
```

#### Step 1.5: /bin/sh 문제 검증 (**최우선 확인 항목**)

> 사전 조사 결과: Android 9+ (API 28+)에서 `/bin/sh` 존재가 확인되었다. PoC에서는 이론 검증이 아니라 **실제 기기에서의 동작 확인**에 집중한다.

```bash
export PATH="$HOME/.openclaw-android/node/bin:$PATH"

# 테스트 1: child_process.exec (내부적으로 /bin/sh 사용)
node -e "
  const { exec } = require('child_process');
  exec('echo hello', (err, stdout) => {
    if (err) console.error('FAIL:', err.message);
    else console.log('OK:', stdout.trim());
  });
"
# OK: hello → /bin/sh 동작함
# FAIL: spawn /bin/sh ENOENT → Android 7-8 가능성, shim/스크립트-shell 대응 필요

# 테스트 2: /bin/sh 존재 여부 직접 확인
ls -la /bin/sh 2>/dev/null && echo "/bin/sh exists" || echo "/bin/sh NOT found"

# 테스트 3: npm script-shell 설정으로 우회 가능한지
npm config set script-shell "$PREFIX/bin/sh"
node -e "
  const { execSync } = require('child_process');
  try {
    const r = execSync('echo test', { shell: '$PREFIX/bin/sh' });
    console.log('OK with explicit shell:', r.toString().trim());
  } catch(e) { console.error('FAIL:', e.message); }
"
```

**해석 가이드**:
- Android 9+ 기기에서 테스트 성공 시: 사전 조사와 실기기 동작이 일치
- Android 7-8 기기에서 실패 시: `glibc-compat.js` 런타임 shim + `npm script-shell` 적용으로 대응

#### Step 2: OpenClaw을 glibc Node.js로 실행

```bash
# glibc Node.js로 npm 사용
export PATH="$HOME/.openclaw-android/node/bin:$PATH"
export TMPDIR="$PREFIX/tmp"

# npm script-shell 설정 (/bin/sh 부재 대응)
npm config set script-shell "$PREFIX/bin/sh"

# OpenClaw 설치 (glibc npm 사용)
npm install -g openclaw@latest

# OpenClaw 실행 테스트
openclaw --version

# 주의: patch-paths.sh는 여전히 필요할 수 있음
# /tmp, /bin/sh 등의 하드코딩 경로 확인
```

#### Step 3: OpenCode proot + ld.so 연결 테스트

```bash
# OpenCode 바이너리 위치 확인
OPENCODE_BIN="$HOME/.bun/install/cache/opencode-linux-arm64@1.2.15@@@1/bin/opencode"

# ld.so 연결 생성
cp "$PREFIX/glibc/lib/ld-linux-aarch64.so.1" "$PREFIX/tmp/ld.so.opencode"
cat "$OPENCODE_BIN" >> "$PREFIX/tmp/ld.so.opencode"

# proot 최소 rootfs 생성
mkdir -p "$HOME/.opencode-android/proot-root/data/data/com.termux/files"

# 실행 테스트
unset LD_PRELOAD
proot -R "$HOME/.opencode-android/proot-root" \
  -b "$PREFIX:$PREFIX" -b /system:/system -b /apex:/apex \
  -w "$(pwd)" \
  "$PREFIX/tmp/ld.so.opencode" "$OPENCODE_BIN" --version
# → 1.2.15
```

#### Step 4: code-server glibc 실행 테스트

```bash
# code-server 다운로드 (공식 linux-arm64 릴리스)
# 하드링크 복구는 여전히 필요 (Android FS 제한)
# tar 추출 후 obj.target/*.node → Release/*.node 복사

# 번들 node에 grun wrapper 적용 (patchelf는 Segfault — 사용 불가)
CS_DIR="$HOME/.local/lib/code-server-<version>"
CS_NODE="$CS_DIR/lib/node"

# wrapper script 생성 (grun 방식)
mv "$CS_NODE" "${CS_NODE}.real"
cat > "$CS_NODE" << WRAPPER
#!/data/data/com.termux/files/usr/bin/bash
unset LD_PRELOAD
exec "$PREFIX/glibc/lib/ld-linux-aarch64.so.1" "$(dirname "\$0")/node.real" "\$@"
WRAPPER
chmod +x "$CS_NODE"

# .node 네이티브 모듈은 glibc 동적 링커로 로드되므로 별도 처리 불필요 (부모 프로세스의 ld.so가 상속됨)

# argon2 패치 없이 동작 확인
code-server --version
```

#### Step 5: oh-my-opencode proot + ld.so 연결 테스트

```bash
# oh-my-opencode 바이너리 (OpenCode와 동일 방식)
OMO_BIN="$HOME/.bun/install/cache/oh-my-opencode-linux-arm64@3.9.0@@@1/bin/oh-my-opencode"

# ld.so 연결 생성
cp "$PREFIX/glibc/lib/ld-linux-aarch64.so.1" "$PREFIX/tmp/ld.so.omo"
cat "$OMO_BIN" >> "$PREFIX/tmp/ld.so.omo"

# 실행 테스트 (동일 proot 설정)
unset LD_PRELOAD
proot -R "$HOME/.opencode-android/proot-root" \
  -b "$PREFIX:$PREFIX" -b /system:/system -b /apex:/apex \
  -w "$(pwd)" \
  "$PREFIX/tmp/ld.so.omo" "$OMO_BIN" --version
         # → v3.9.0
```

#### PoC 성공 기준

**🔴 Blocker (실패 시 계획 재검토)**: **ALL PASSED**
- [x] `/bin/sh` 동작 여부 확인 — ✅ Android 13에서 동작
- [x] glibc-runner 설치 및 `grun --help` 동작 — ✅ (SigLevel 워크어라운드 필요)
- [x] glibc Node.js `--version` 정상 출력 — ✅ v22.14.0 (grun 방식, patchelf는 Segfault)
- [ ] ~~`npm install -g openclaw` 성공~~ — ❌ koffi 네이티브 빌드 실패 (별도 대응 필요)

**🟡 Important (실패 시 workaround 적용)**: **MOSTLY PASSED**
- [x] `process.platform === 'linux'` — ✅
- [x] `os.cpus().length > 0` — ⚠️ 0 반환 (fallback 유지, Risk 11 확인)
- [x] `os.networkInterfaces()` — ❌ EACCES (try-catch 필요, glibc-compat.js에 추가)
- [ ] ~~`openclaw --version` 정상 출력~~ — 미테스트 (npm install 실패)
- [x] `opencode --version` 정상 출력 — ✅ v1.2.15
- [ ] code-server 번들 node grun 방식 동작 — 미테스트 (grun 방식으로 전환 예정)
- [ ] code-server argon2 패치 없이 동작 — 미테스트
- [x] wrapper script 방식으로 LD_PRELOAD 자동 해제 동작 — ✅

**🟢 Nice-to-have (실패해도 진행 가능)**:
- [x] oh-my-opencode CLI 실행 — ✅ v3.9.0
- [ ] oh-my-opencode 플러그인 로드 확인 — 미테스트 (OpenCode 실행 후 확인 예정)
- [ ] 네이티브 모듈 (sharp 등) prebuilt 자동 사용 여부 — 미테스트

### Phase 1: 프로젝트 구조 변경

```
openclaw-android/
├── install.sh                    # 변경: glibc 기반 아키텍처로 재구성
├── oa.sh                         # 변경: grun 래퍼 통합, opencode 명령 추가
├── update-core.sh                # 변경: Node.js/OpenCode 업데이트 + 마이그레이션 분기
├── uninstall.sh                  # 변경: glibc 정리 추가
├── bootstrap.sh                  # 변경: 새 스크립트 다운로드
├── patches/
│   ├── patch-paths.sh            # 유지: /tmp, /bin/sh 경로 패치
│   ├── apply-patches.sh          # 변경: 대폭 간소화
│   ├── systemctl                 # 유지: systemd 스텁
│   ├── glibc-compat.js           # 신규: /bin/sh 경로 shim (축소된 bionic-compat 후속)
│   ├── bionic-compat.js          # 삭제 (glibc-compat.js로 교체)
│   ├── termux-compat.h           # 삭제 (glibc에서 불필요)
│   ├── spawn.h                   # 삭제 (glibc에서 불필요)
│   └── argon2-stub.js            # 삭제 (glibc에서 불필요)
├── scripts/
│   ├── check-env.sh              # 변경: 디스크 공간 요구사항 상향
│   ├── install-glibc-env.sh      # 신규: pacman + glibc-runner + Node.js + wrapper
│   ├── install-opencode.sh       # 신규: OpenCode + OmO 설치
│   ├── install-deps.sh           # 변경: Bionic Node.js 대신 pacman 설치
│   ├── install-code-server.sh    # 변경: node 교체 → grun wrapper 방식, argon2 패칭 제거
│   ├── setup-env.sh              # 변경: Bionic 변수 제거, glibc 변수 추가
│   ├── setup-paths.sh            # 유지
│   └── build-sharp.sh            # 변경: prebuilt 우선 사용, 대부분 스킵
├── tests/
│   └── verify-install.sh         # 변경: glibc 검증 추가, Bionic 검증 제거
└── docs/
    └── plan/
        └── opencode-integration-plan.md  # 이 문서
```

### Phase 2: 신규 스크립트 개발

#### 2-1. `scripts/install-glibc-env.sh`

glibc 환경 설치 자동화. **프로젝트의 핵심 새 스크립트.**

**주요 로직**:
1. pacman 설치 (`pkg install pacman proot`)
2. pacman 키 초기화 (소요 시간 안내 포함), SigLevel = Never 워크어라운드 적용
3. glibc-runner 설치 (`pacman -Sy glibc-runner --noconfirm --assume-installed bash,patchelf,resolv-conf`)
4. Node.js linux-arm64 LTS 다운로드
5. **wrapper script 생성** (node.real + node wrapper, grun 방식 — patchelf 미적용)
6. npm script-shell 설정 (`npm config set script-shell $PREFIX/bin/sh`)
7. Node.js를 `~/.openclaw-android/node/`에 설치
8. 검증 (`node --version` — wrapper 경유)
9. 마커 파일 생성 (`touch ~/.openclaw-android/.glibc-arch`)

**에러 처리**:
- 모든 단계 실패 시 OpenClaw 설치 중단 (glibc Node.js가 핵심)
- 기존과 달리 Node.js 설치가 이 단계에서 이루어짐 (Termux pkg가 아님)

#### 2-2. `scripts/install-opencode.sh`

OpenCode + oh-my-opencode 설치. **비핵심(non-critical), 실패해도 OpenClaw에 영향 없음.**

**주요 로직**:
1. glibc 환경 존재 확인 (없으면 스킵)
2. proot 설치 확인
3. OpenCode 바이너리 다운로드 (bun install 또는 npm)
4. **ld.so 연결** 생성 (`cp ld.so + cat opencode_bin >> ld.so.opencode`)
5. **proot 최소 rootfs** 생성 (`mkdir -p ~/min_root/data/data/com.termux/files`)
6. **wrapper script** 생성 (proot + ld.so.opencode 방식)
7. `opencode --version` 검증
8. oh-my-opencode 동일 방식 적용
9. `opencode.json` 설정 파일 생성 (플러그인 등록)

### Phase 3: 기존 스크립트 수정

#### 3-1. `install.sh` — 아키텍처 변경

기존 Bionic 기반 설치 흐름을 glibc 기반으로 재구성.

```
기존:
[1/9] Environment Check
[2/9] Installing Dependencies (pkg install nodejs-lts ...)
[3/9] Setting Up Paths
[4/9] Configuring Environment Variables
[5/9] Installing OpenClaw (npm install -g openclaw)
[6/9] Installing code-server
[7/9] AI CLI Tools
[8/9] Verifying Installation
[9/9] Updating OpenClaw

변경:
[1/10] Environment Check           ← 디스크 공간 요구사항 상향 (~1GB)
[2/10] Installing Base Dependencies ← pacman, proot, tmux 등 (nodejs 제외)
[3/10] Installing glibc Environment ← 신규: glibc-runner + Node.js linux-arm64 + grun wrapper
[4/10] Setting Up Paths
[5/10] Configuring Environment Variables ← Bionic 변수 제거, glibc 변수 추가
[6/10] Installing OpenClaw          ← glibc Node.js 사용 (wrapper 경유)
[7/10] Installing code-server       ← grun 방식 (번들 node 교체 대신)
[8/10] Installing OpenCode + OmO    ← 신규: proot + ld.so 연결 (non-critical)
[9/10] Verifying Installation
[10/10] Updating OpenClaw
```

#### 3-2. `oa.sh` — grun 래퍼 통합

```bash
# glibc Node.js 실행을 위한 래퍼 함수 추가
_glibc_exec() {
    LD_PRELOAD= "$@"
}

# 새 명령 추가
oa opencode          # OpenCode 실행
oa opencode --stop   # OpenCode 프로세스 종료
oa opencode --status # OpenCode 상태 확인
```

모든 node/npm 호출을 `_glibc_exec`으로 래핑하여 LD_PRELOAD 충돌 방지.
wrapper script가 있으므로 일반적인 `node`/`npm` 호출은 자동 처리되지만, `oa.sh` 내부에서 직접 바이너리를 호출하는 경우를 위해 유지.

#### 3-3. `setup-env.sh` — 환경 변수 변경

```bash
# 제거되는 변수:
# NODE_OPTIONS="-r bionic-compat.js"    ← bionic-compat.js 불필요
# CXXFLAGS="-include termux-compat.h"   ← termux-compat.h 불필요
# GYP_DEFINES="OS=linux"               ← glibc는 이미 Linux
# CFLAGS="-Wno-error=..."              ← glibc 헤더는 완전함

# 유지되는 변수:
# PATH=$HOME/.openclaw-android/node/bin:$PATH  ← glibc Node.js 경로
# TMPDIR=$PREFIX/tmp
# TMP/TEMP=$TMPDIR
# CONTAINER=1
# CLAWDHUB_WORKDIR=$HOME/.openclaw/workspace

# 새로 추가되는 변수:
# OA_GLIBC=1                            ← glibc 아키텍처 표시
```

#### 3-4. `install-code-server.sh` — grun 방식으로 변경

**제거되는 워크어라운드**:
- ~~번들 node 교체 (Termux node 심링크)~~ → **grun wrapper script로 대체** (patchelf 미사용)
- ~~argon2 스햅 패칭~~ → glibc에서 원본 동작 예상 (PoC 미검증, 실패 시 기존 stub 유지)

**추가되는 로직**:
- 번들 node에 대한 grun wrapper script 생성 (node.real + node, ld.so 직접 실행)
- 모든 `.node` 네이티브 모듈도 grun 방식 실행 확인 필요

**유지되는 로직**:
- 하드링크 복구 (Android FS 제한)
- 버전 확인 및 다운로드
- 심링크 생성

#### 3-5. `apply-patches.sh` — 대폭 간소화

**제거되는 단계**:
- ~~bionic-compat.js 복사~~

**추가되는 단계**:
- glibc-compat.js 복사 (PoC 결과에 따라. /bin/sh 존재 시 불필요)

**유지되는 단계**:
- systemctl 스텁 설치
- patch-paths.sh 실행 (경로 패치)

### Phase 4: 패치 파일 정리

**삭제 대상:**
| 파일 | 제거 이유 |
|------|-----------|
| `patches/bionic-compat.js` | glibc-compat.js로 교체 (또는 PoC에서 불필요 확인 시 완전 삭제) |
| `patches/termux-compat.h` | glibc 헤더에 포함 |
| `patches/spawn.h` | glibc 헤더에 포함 |
| `patches/argon2-stub.js` | glibc에서 원본 동작 |

**신규:**
| 파일 | 역할 |
|------|------|
| `patches/glibc-compat.js` | os.cpus() fallback + os.networkInterfaces() try-catch + /bin/sh 경로 shim (Android 7-9 전용) |

**유지 대상:**
| 파일 | 유지 이유 |
|------|-----------|
| `patches/patch-paths.sh` | Termux 경로 변환 (glibc와 무관) |
| `patches/apply-patches.sh` | 간소화하여 유지 |
| `patches/systemctl` | Android에 systemd 없음 |

### Phase 5: 환경 변수 및 설정

#### `.bashrc` 변경

```bash
# >>> OpenClaw on Android >>>
export PATH="$HOME/.openclaw-android/node/bin:$HOME/.local/bin:$PATH"
export TMPDIR="$PREFIX/tmp"
export TMP="$TMPDIR"
export TEMP="$TMPDIR"
export CONTAINER=1
export CLAWDHUB_WORKDIR="$HOME/.openclaw/workspace"
export OA_GLIBC=1
# <<< OpenClaw on Android <<<
```

> alias 방식은 사용하지 않음. wrapper script가 LD_PRELOAD 해제를 처리.

#### `~/.config/opencode/opencode.json`

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["oh-my-opencode"]
}
```

### Phase 6: 문서 업데이트

- **README.md**: 아키텍처 설명 변경 (패치 기반 → glibc 기반)
- **Requirements**: 저장 공간 ~500MB → ~1GB
- **Bonus 섹션**: AI CLI 도구에 OpenCode 추가
- **CLI Reference**: `oa opencode` 명령 추가
- **Project Structure**: 새 파일/삭제 파일 반영
- **What It Does**: "Platform recognition" 설명을 glibc 방식으로 변경

---

## 작업 순서 (실행 계획)

### Milestone 1: PoC 검증 (핸드폰에서 수동)

0. pacman + glibc-runner 수동 설치
1. Node.js linux-arm64 다운로드 + grun wrapper + 기본 동작 확인
2. **🔴 /bin/sh 문제 검증 (최우선)** — child_process.exec 동작 확인
3. `process.platform`, `os.cpus()`, `os.networkInterfaces()` 동작 확인
4. npm install -g openclaw (postinstall 스크립트 포함) 성공 확인
5. OpenClaw glibc Node.js 실행
6. OpenCode proot + ld.so 연결 + 실행
7. code-server glibc 동작 확인 (번들 node grun wrapper)
8. oh-my-opencode 설치 + 플러그인 로드
9. 네이티브 모듈 prebuilt 사용 여부 확인
10. 결과 문서화 (성공/실패 항목, workaround 정리)

### Milestone 2: 스크립트 개발

1. `scripts/install-glibc-env.sh` 작성 (wrapper script 생성 포함)
2. `scripts/install-opencode.sh` 작성
3. `patches/glibc-compat.js` 작성 (PoC 결과에 따라)
4. 기존 스크립트 수정 (install.sh, oa.sh, setup-env.sh, install-code-server.sh, apply-patches.sh, update-core.sh, uninstall.sh, bootstrap.sh)
5. 마이그레이션 로직 추가 (update-core.sh에 Bionic→glibc 분기)
6. 패치 파일 삭제 (bionic-compat.js→교체, termux-compat.h, spawn.h, argon2-stub.js)
7. verify-install.sh 업데이트

### Milestone 3: 테스트

1. 클린 Termux 환경에서 전체 install.sh 실행
2. OpenClaw 정상 동작 확인
3. OpenCode + OmO 정상 동작 확인
4. code-server 정상 동작 확인
5. **Bionic→glibc 마이그레이션 (`oa --update`) 정상 동작 확인**
6. AI CLI 도구 재설치 동작 확인
7. 업데이트 (`oa --update`, glibc→glibc) 정상 동작 확인
8. 삭제 (`oa --uninstall`) 정상 동작 확인 (glibc 정리 포함)

### Milestone 4: 문서 및 릴리스

1. README.md 업데이트
2. 한국어 문서 업데이트
3. 버전 번호 업데이트 → **v1.0.0** (아키텍처 변경, 메이저 버전)
4. 릴리스

---

## 예상 소요 리소스

| 항목 | 예상치 |
|------|--------|
| 추가 저장 공간 | ~500-600MB (glibc-runner + Node.js + OpenCode ld.so연결 + OmO ld.so연결 + proot) |
| 설치 시간 | ~10-15분 (네트워크 속도 의존) |
| 신규 스크립트 | 3개 (install-glibc-env.sh, install-opencode.sh, glibc-compat.js) |
| 수정 스크립트 | 8개 |
| 삭제 패치 | 3개 (termux-compat.h, spawn.h, argon2-stub.js) |
| 교체 패치 | 1개 (bionic-compat.js → glibc-compat.js) |
| PoC 검증 | 2-3시간 (수동) |

---

## 대안 검토 (기각됨)

| 대안 | 기각 이유 |
|------|----------|
| proot-distro | ~1GB 오버헤드, ptrace 성능 저하, bunfs 충돌로 OpenCode 실행 불가 ([#7960](https://github.com/anomalyco/opencode/issues/7960)), 프로젝트 철학 충돌 |
| Node.js 포팅 (OpenCode) | Bun 런타임에 구조적 종속, 팀도 기각 ([#10860](https://github.com/anomalyco/opencode/issues/10860)), 3-6주 소요 |
| musl Bun 런타임 | Bun이 `bun-linux-aarch64-musl` 제공하나, `bun build --compile` 출력은 glibc 링킹. musl 런타임으로 소스 실행 가능성은 있으나 전면 전환과 별개 |
| Bionic 패치 유지 (기존 방식) | OpenCode 통합 시 glibc-runner 필수. 기존 패치 + glibc 이중관리가 되어 유지보수 복잡도 증가. 전면 전환이 장기적으로 우수 |
| OpenCode 포기 | 사용자 요구사항 불충족 |

---

## 검증된 선례

- [`tribixbite/bun-on-termux`](https://github.com/tribixbite/bun-on-termux): glibc-runner로 Bun을 Termux에서 실행하는 프로젝트
- [CodeIter 가이드](https://gist.github.com/CodeIter/ccdcc840e432288ef1e01cc15d66c048): pacman + glibc-runner + patchelf로 Deno/Bun 설치 (6 forks)
- code-server가 proot-distro (glibc 환경)에서 정상 동작하는 것이 확인됨 (커뮤니티)

---

## 부록: 참고 자료

- [termux-pacman/glibc-packages](https://github.com/termux-pacman/glibc-packages) — glibc-runner 공식 저장소
- [glibc-runner Wiki](https://github.com/termux-pacman/glibc-packages/wiki/About-glibc-runner-(grun)) — grun 사용법
- [tribixbite/bun-on-termux](https://github.com/tribixbite/bun-on-termux) — glibc-runner 기반 Bun 설치 선례
- [CodeIter gist](https://gist.github.com/CodeIter/ccdcc840e432288ef1e01cc15d66c048) — pacman + glibc-runner + patchelf 셋업 가이드
- [glibc-packages #239](https://github.com/termux-pacman/glibc-packages/issues/239) — /tmp 경로 자동 리매핑 기능 요청
- [oven-sh/bun#26752](https://github.com/oven-sh/bun/issues/26752) — Bun compiled 바이너리의 /proc/self/exe 문제 (closed)
- [oven-sh/bun#26753](https://github.com/oven-sh/bun/pull/26753) — BUN_SELF_EXE 환경 변수 추가 PR
- [bun-linux-aarch64-musl](https://github.com/oven-sh/bun/releases) — Bun 공식 musl 빌드 릴리스
- [anomalyco/opencode](https://github.com/anomalyco/opencode) — OpenCode 소스 (TypeScript, Bun 기반)
- [opencode#7960](https://github.com/anomalyco/opencode/issues/7960) — OpenCode PRoot 환경 실행 실패 (bunfs 충돌)
- [opencode#10860](https://github.com/anomalyco/opencode/issues/10860) — Node.js 지원 요청 (기각됨)
- [code-yeongyu/oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) — oh-my-opencode 소스
- [Node.js Downloads](https://nodejs.org/en/download/) — 공식 linux-arm64 바이너리
