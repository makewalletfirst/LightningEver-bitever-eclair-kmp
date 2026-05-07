# LightningEver-bitever-eclair-kmp (Phoenix-fork client lightning engine)

BitEver(BEC) 라이트닝에버 앱(Phoenix 포크)이 **단말 내부에서 라이트닝 노드를 실제로 구동**하기 위해 사용하는 Kotlin Multiplatform 라이브러리. ACINQ 의 [lightning-kmp](https://github.com/ACINQ/lightning-kmp) v1.11.5를 기반으로 BEC 체인 호환 + close handler 진단·debug logging이 추가된 포크.

## 🧭 이 라이브러리의 위치

```
[Eclair LSP (Scala, 서버)] ←━━━ Lightning P2P ━━━→ [Phoenix Android/iOS 앱]
                                                            │
                                                            └─ depends on ─→ [eclair-kmp 라이브러리 (이 repo)]
```

- **Eclair (Scala)** = `LightningEver-bitever-eclair` — 항상 켜져있는 LSP 서버용 풀스펙 라이트닝 노드.
- **eclair-kmp (Kotlin)** = 본 repo — Android/iOS 단말에 임베드되는 라이트닝 코어. wallet-only 모드, trampoline + LSP 의존 경량.
- **Phoenix 앱** = `LightningEver-bitever-phoenix` — UI/네트워킹/DB 셸. 채널 상태머신/HTLC/Splice/Close 협상은 **전부 eclair-kmp 안에서** 동작.

## 🚀 핵심 패치 사항 (이전 작업분 누적)

- **Header Pass-through:** `ElectrumClient.kt` 수정 — 일렉트럼 서버 블록 헤더를 검증 없이 수용. 비트코인 메인넷과 다른 체크포인트의 BEC 체인에서도 지갑 동기화 정상 동작.
- **KMP Support 유지:** Kotlin Multiplatform — Android/iOS 동일 라이트닝 로직 공유.

## 🆕 260507 브랜치 — 이전 (`1.11.5`) 대비 변화

### 1. Close handler 진단 logger 추가
파일: `modules/core/src/commonMain/kotlin/fr/acinq/lightning/channel/states/Normal.kt`

다음 8개 위치에 `logger.warning { "[260507-DBG] ..." }` 삽입:
- `is ChannelCommand.Close.MutualClose` 진입
- `is ChannelCommand.Close.ForceClose` 진입
- `is Shutdown` 메시지 수신 (scriptPubKey, closeeNonce, spliceStatus, activeCount)
- 4개 거부 분기 (invalid scriptPubKey / remote unsigned HTLCs / remote unsigned UpdateFee / local unsigned HTLCs)
- clean else 분기 진입 + `Send(localShutdown)` 직전 + `startClosingNegotiation` 직전 + `ShuttingDown` 전이 직전

이 logger들은 Phoenix 앱의 **Settings → Troubleshooting → View Logs** 에서 `260507-DBG` 키워드로 검색해 close가 silent fail되는 분기를 핀포인트하기 위함.

### 2. 버전
`gradle.properties` 의 `version` 을 `1.11.5` → `1.11.5-DEBUG`. Phoenix 의 `gradle/libs.versions.toml` 도 함께 동기화 필요.

### 왜 이 패치가 필요했는가
LSP `/close` 가 정상적으로 `Shutdown(with ShutdownNonce TLV)` 를 보내도 앱이 무응답이었음. single-commitment NORMAL 채널에서도 동일 → close handler 내부 silent fail이 의심됐고, 정확한 위치를 찾기 위해 logger 추가 진단 빌드를 만든 것.

## 🛠 기술 스택

- **언어:** Kotlin Multiplatform (commonMain → JVM/Android/iOS/Linux/Native)
- **빌드:** Gradle 8.x + Kotlin 2.x
- **주요 의존성:** `bitcoin-kmp`, `secp256k1-kmp 0.22`, `kotlinx-serialization`
- **Java:** OpenJDK 21
- **퍼블리시 대상:** JVM publication + Kotlin Multiplatform metadata

## 🏗 빌드 및 배포 방법

### 사전 요구 사항
- Java 21 (`JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64`)
- Gradle wrapper(`./gradlew`) 사용 — 별도 설치 불필요

### Phoenix 가 가져갈 위치(local Maven)에 publish
```bash
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH

./gradlew :lightning-kmp-core:publishJvmPublicationToMavenLocal \
           :lightning-kmp-core:publishKotlinMultiplatformPublicationToMavenLocal \
           --console=plain
# ~2분 22초
```

결과: `~/.m2/repository/fr/acinq/lightning/lightning-kmp-core/1.11.5-DEBUG/`
- `lightning-kmp-core-1.11.5-DEBUG.jar` (~228 KB)
- `lightning-kmp-core-1.11.5-DEBUG.pom`
- `lightning-kmp-core-1.11.5-DEBUG.module`
- `lightning-kmp-core-1.11.5-DEBUG-sources.jar`
- `lightning-kmp-core-1.11.5-DEBUG-javadoc.jar`

### 전체 publish (linux/native target까지)
```bash
./gradlew publishToMavenLocal --console=plain
```
JVM 만 필요한 경우 위의 두 task 만으로 충분.

## 📡 Phoenix 앱이 이 라이브러리를 사용하게 하는 법

1. `LightningEver-bitever-phoenix` 의 `gradle/libs.versions.toml` 에서 `lightningkmp = "1.11.5-DEBUG"` 로 설정.
2. `settings.gradle.kts` 의 dependencyResolutionManagement 에 `mavenLocal()` 가 첫 번째 repository로 포함되어야 함 (현재 포함됨).
3. Phoenix 빌드 시 자동으로 `~/.m2/repository` 에서 `1.11.5-DEBUG` 를 찾아 사용.

### 전체 워크플로우 (3개 repo 공동 작업)
```
[1] eclair-kmp 패치 → publishToMavenLocal
        ↓
[2] LightningEver-bitever-phoenix:phoenix-android:assembleDebug
        ↓
[3] APK → 안드로이드 단말 sideload (adb install -r 또는 SCP)
        ↓
[4] 앱 강제 종료 → 재실행 → close 시도
        ↓
[5] 앱 내장 로그 뷰어에서 `260507-DBG` 검색
```

## 🌿 브랜치 정책
- `master` / `main` (있다면) — upstream `1.11.5` 추적용.
- **`260507`** — close handler 진단 logger + version `-DEBUG` (현 브랜치).

## 🔗 함께 보는 문서
- LSP 운영 + close 절차 통합 가이드: 운영 머신 `/root/md/260507_CLOSECOMPLETE.md`.
- LSP 측 패치 / REST API: `LightningEver-bitever-eclair` repo `260507` 브랜치 README.
- 앱 빌드 / APK 배포: `LightningEver-bitever-phoenix` repo `260507` 브랜치 README.

## ⚠️ 주의

- **테스트 체인 전용**. `1.11.5-DEBUG` 는 진단 logger 가 들어 있어 production 노드에는 적합하지 않음.
- close 핸들러 silent fail 의 진짜 원인이 핀포인트되면 logger 제거 + 정식 패치 + 새 버전(`1.11.5-fix-close` 등)으로 재배포 권장.

## ⚙️ CI/CD (GitHub Actions)
`.github/workflows/build.yml` 파일을 통해 Kotlin Multiplatform 빌드 유효성을 자동 검사합니다.
