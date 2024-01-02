# chzzk-onair-webhook
치지직 생방송 시작시 Webhook을 발송해주는 서비스

[webhook.twitchzzk.tv](https://webhook.twitchzzk.tv)

## Project Goal
- 치지직 플랫폼에서 스트리머가 방송을 켜면, 개발자가 미리 설정해둔 Webhook 주소로 정해진 규격의 웹훅을 발송합니다.
- 방송이 켜졌는지 확인하기 위해 개별 앱에서 치지직 API를 일일이 호출할 필요가 없도록 합니다.
- 방송을 자주 켜는 스트리머는 생방송 여부를 더 자주 확인하고, 방송을 가끔 켜는 스트리머는 생방송 여부를 더 가끔 확인합니다.

## User Flow
- 개발자가 Frontend 대시보드에 접속하여 소셜 로그인을 진행합니다.
- 개발자는 정해진 Quota 내에서 생방송 알림을 수신받을 스트리머와 Webhook 주소를 지정할 수 있습니다. (대시보드나 API 호출을 통해 지정 가능)
- 개발자는 대시보드에서 API Token을 발급받을 수 있고, API Token을 이용해 대시보드 설정을 변경할 수 있습니다. (Twitch EventSub API 호환 버전도 지원 예정)
- 스트리머의 방송이 켜지거나 꺼지면 지정된 Webhook 주소로 알림을 받을 수 있습니다.

## Internal Structure
### Frontend
- 백엔드 없이 Next.js만을 이용해 Frontend 대시보드를 구현합니다.
- DB는 PlanetScale을 이용합니다. (Serverless MySQL)
- API 서버도 Frontend에서 구현합니다.

### Etc. Services
쿠버네티스 기능을 적극 활용해 안정적인 서비스를 구현합니다.
- 생방송 여부 확인 Operator Job : 방송 빈도에 따라 정해진 Tier대로 1분/3분/5분/10분 주기로 Operator Job을 실행합니다. Operator Job은 실제 생방송 여부를 확인하는 하위 Job을 실행합니다. 이 때, IP Rotate가 원활히 이뤄지도록 Node Anti Affinity 설정을 걸어서 하위 Job을 실행합니다.
  - 생방송 여부 확인 하위 Job : 하위 Job은 환경변수로 넘어온 스트리머 목록에 대해 실제 생방송 여부를 확인하는 작업을 수행합니다. 이 때 DB에 저장된 마지막 생방송 여부와 비교하여 방송이 켜졌는지를 확인합니다. 만일 방송이 켜졌다면 발신할 웹훅 주소를 DB에서 조회해 개별 주소별로 SQS 메세지를 발행합니다.
- SQS Consumer 앱 : SQS로부터 웹훅 주소와 데이터를 받아서 웹훅을 발송합니다.
