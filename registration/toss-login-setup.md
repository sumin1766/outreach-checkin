# 토스 로그인과 미니앱 접속 설정

현재 버전은 미니앱 주소 허용, 쿠키 없는 앱 세션, 서버 mTLS 인증 코드 교환, 사용자 확인, 조별 권한, 기능별 진입 경로를 구현했습니다. 실제 파트너 인증서가 이 작업 환경이나 서버에 제공된 것을 확인하지 못했으므로 실제 토스 로그인 성공·실기기 출석 저장·최종 심사 준비 완료를 주장하지 않습니다.

## 근거와 범위

- [토스 공식 로그인 예제](https://github.com/toss/apps-in-toss-examples/tree/main/toss-login)의 서버 `generate-token` → `login-me` 흐름과 AES-256-GCM 이름 복호화를 대조했습니다.
- [사용자가 지정한 인증 안내](https://developers-apps-in-toss.toss.im/guide/authentication/intro)는 이 환경의 현재 네트워크 제한으로 본문을 직접 읽지 못했습니다. 해당 호스트와 운영 상태 확인 호스트를 허용하는 환경 초안을 저장했지만 적용된 상태는 아닙니다.
- 로컬 테스트는 직접 생성한 테스트 인증서로 **실제 상호 TLS**를 수행합니다. 이는 실제 토스 인증서·API에 대한 검증과 다릅니다.

## Vercel Production 설정

기존 Supabase 변수는 유지합니다. 앱인토스 콘솔에서 실제 앱용으로 발급한 인증 자료를 Vercel의 Secret 값에 직접 넣습니다. 파일 경로가 아니라 PEM 파일의 **내용 전체**입니다. 코드·채팅·브라우저 번들에 넣지 않습니다.

| 변수 | 내용 |
| --- | --- |
| `TOSS_CLIENT_CERT` | 토스 서버 통신용 mTLS 인증서 PEM 내용 |
| `TOSS_CLIENT_KEY` | 위 인증서와 짝인 비밀 키 PEM 내용 |
| `TOSS_DECRYPTION_KEY_BASE64` | 선택: 계정 이름을 복호화하는 키 |
| `TOSS_AAD` | 선택: 위 복호화 키에 대응하는 AAD |
| `ADMIN_TOSS_USER_KEY` | 본인의 서버 검증된 토스 사용자 키 |

인증서·비밀 키가 없거나 서로 맞지 않으면 로그인은 차단합니다. 모든 TLS 인증서 검증은 활성화되어 있습니다. 파트너 API 목적지는 공식 API 호스트에 고정되어 있으며 브라우저가 변경할 수 없습니다. 응답의 AccessToken·RefreshToken·전화번호 등은 브라우저나 DB에 저장하지 않습니다. 동의된 계정 이름만 선택적으로 서버에서 복호화하고, 이름 복호화 설정이 없으면 기본 계정 이름을 사용합니다.

실제 mTLS 요청은 Vercel의 Node 서버에서 수행합니다. 관리형 클라우드에서 원격 mTLS 테스트가 필요하다면 네트워크 정책과 TLS 통과 경로를 별도로 확인해야 합니다. 프록시 우회나 TLS 검증 해제는 사용하지 않습니다.

변수 저장 후 최신 Production 배포를 재배포합니다. 상태 확인 응답의 `serverVersion: miniapp-login-v1`은 수정된 서버 버전입니다. `login: configured`는 두 인증 값이 입력되었다는 뜻이며 실제 인증 성공을 의미하지 않습니다.

## 관리자 첫 설정

관리자 사용자 키가 없을 때 **첫 로그인 사용자를 자동 관리자로 지정하지 않습니다.** 검증된 신규 계정은 승인 대기이며 명단을 볼 수 없습니다. 본인이 로그인한 뒤 대기 화면의 ‘내 계정 ID’를 확인해 Vercel의 `ADMIN_TOSS_USER_KEY`에 넣습니다. Production 재배포 후 로그아웃·다시 로그인하면 그 검증된 계정만 관리자입니다. 다른 신규 사용자는 이후 관리자가 1~10조 중 배정합니다.

## 접속 오류 수정

기본 허용 주소는 이 앱의 두 주소만입니다.

```text
https://outreach-checkin.apps.tossmini.com
https://outreach-checkin.private-apps.tossmini.com
```

기존 운영 웹 origin도 유지합니다. 추가 주소가 실제로 확인된 경우에만 `MINIAPP_ORIGINS`에 쉼표로 구분해 등록합니다. 다른 앱·전체 tossmini 하위 도메인·`null` origin은 허용하지 않습니다.

일반 웹은 HttpOnly 쿠키를 사용합니다. 미니앱은 Safari의 외부 사이트 쿠키에 의존하지 않고 서버가 발급한 앱 세션을 Bearer 헤더로 보냅니다. 이는 토스 AccessToken이 아닌 무작위 앱 세션이며 DB에는 해시로 저장합니다. 해당 앱의 sessionStorage와 메모리에서만 사용하고, 만료·로그아웃 시 제거합니다. Origin 검사와 조별 DB 권한은 그대로 적용됩니다.

네트워크 오류 화면은 실행 origin을 표시합니다. 이 정보는 계정·키·인가 코드가 아니며, 실제 기기의 접속 주소가 예상과 다른지 확인하는 용도입니다.

## 콘솔의 주요 기능

모두 ‘특정 화면’을 선택합니다. 콘솔에 `intoss://outreach-checkin/` 접두사가 이미 보이면 마지막 열만 입력합니다.

| 한국어 | 영어 | 화면 주소 입력값 |
| --- | --- | --- |
| 출석체크 | Check In | `attendance` |
| 출석 집계 | Attendance Stats | `stats` |
| 조원 관리 | Manage Members | `members` |

전체 주소는 각각 `intoss://outreach-checkin/attendance`, `intoss://outreach-checkin/stats`, `intoss://outreach-checkin/members`입니다. 앱은 HTTPS pathname, `?page=...`, `#/...` 진입도 처리합니다. 미로그인 때는 로그인 화면을 먼저 열고, 동일 앱 실행 흐름에서 로그인 완료 후 목적 화면을 유지합니다. 권한이 승인 대기라면 목적 화면보다 승인 대기 화면을 우선합니다.

## 검증

```sh
npm test
npm run build:vercel
npm run build:miniapp
node test/miniapp-connect-browser.mjs
```

브라우저 테스트는 실제 브라우저의 CORS 처리를 유지하고, 두 앱 origin에서 실제 로컬 HTTP API로 전달합니다. 다른 앱 origin 차단, 쿠키 없는 세션, 출석·집계·조원 경로를 검사합니다. API 상태 저장소와 외부 호스팅은 테스트 대체이며 실제 토스 기기·실제 파트너 API 검증이 아닙니다.

최종 심사 전 실제 인증서 설정, 실제 앱 진입·로그인·관리자 지정·조장 배정·출석 저장·다른 기기 반영·CSV 검증이 필요합니다. 샘플 데이터로 대체하거나 인증되지 않은 관리자 진입을 만들지 않습니다.
