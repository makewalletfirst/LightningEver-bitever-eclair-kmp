# LightningEver-bitever-eclair-kmp (Lightning Engine)

LightningEver 앱의 핵심 라이트닝 엔진 라이브러리입니다. 
ACINQ eclair-kmp를 기반으로 비트에버(BEC) 체인과의 SPV 동기화를 위해 개조되었습니다.

## 🚀 핵심 패치 사항
- **Header Pass-through (헤더 검증 패스):** `ElectrumClient.kt`를 수정하여 일렉트럼 서버가 전달하는 블록 헤더를 검증 과정 없이 수용하도록 패치. 이를 통해 비트코인 메인넷과 다른 체크포인트를 가진 비트에버 체인에서도 지갑이 정상 작동함.
- **KMP Support:** Kotlin Multiplatform을 지원하여 Android 및 iOS에서 동일한 라이트닝 로직 공유.

## 🛠 기술 스택
- **언어:** Kotlin (KMP)
- **빌드 도구:** Gradle (Kotlin DSL)
- **주요 의존성:** `bitcoin-kmp`
- **타겟:** JVM (Android), Linux, iOS (Optional)

## 🏗 빌드 및 배포
로컬 환경에 라이브러리를 설치하려면 다음 명령을 실행합니다.
```bash
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
./gradlew publishToMavenLocal -x allTests
```

## ⚙️ CI/CD (GitHub Actions)
`.github/workflows/build.yml` 파일을 통해 Kotlin Multiplatform 빌드 유효성을 검사합니다.
