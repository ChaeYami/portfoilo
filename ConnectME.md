# ConnectME
>맞춤 친구 추천, 1:1 채팅까지! 맞춤 추천 친구 만들기 커뮤니티 (팀 프로젝트)   
>https://connectme.co.kr/

</br>

## 1. 제작 기간 & 참여 인원
- 개발기간 : 2023.06.05 ~ 2023.07.10
- 팀 프로젝트
  <details>
  <summary> 역할 </summary>
  <div markdown='1'></div>
    
  - 팀장
  - User 앱 전반
      - 회원가입, 로그인, 계정 비활성화 / 소셜로그인 / 친구신청, 수락, 삭제 / 유저 신고, 차단, 자동해제 기능
      - SMS 인증(아이디 찾기) / 이메일 인증(비밀번호 재설정)
  - counsel 앱
    - 게시글, 댓글 serializer
    - 게시글 작성/조회/수정/삭제, 댓글 작성/조회/수정/삭제
  - counsel app 테스트코드
  - Validator 생성 및 적용
  - 팀원 코드 피드백 및 리팩토링
  - Amazon S3 static 파일 업로드 + cloudfront 배포
  - 팀 문서(노션,README) 작성 및 관리
  - User 앱 관련 프론트엔드 (JavaScript ajax로 백엔드-프론트엔드 연결)
  
  </details>

</br>

## 2. 사용 기술
### **Back-end**
  - Python 3.10.6
  - Django 4.2.2
  - DRF 3.14.0
  - Django simple JWT 5.2.2
  - Selenium
  - Redis 4.1.0
  - Django Channels 4.0.0
### **Infra**
  - AWS EC2
  - AWS Route 53
  - AWS CloudFront
  - Gunicorn
  - Nginx 1.18.0
  - Docker 20.10.21
  - AWS S3
  - Daphne
### **Database**
  - PostgreSQL
### **Front-end**
  - HTML5
  - JavaScript
  - CSS3

</br>

## 3. ERD 설계
![Connect ME (5)](https://github.com/ChaeYami/portfoilo/assets/120750451/ff7fec81-dccb-4009-8498-a354a1642ad4)

</br>

## 4. 핵심 기능

### 4-1. 사용자 기반 서비스
- 회원가입, 로그인, 개인정보/프로필 관리, 소셜로그인
  - 구글 : Google OAuth API
  - 카카오 : KaKao OAuth 2.0 API
- SMS인증/이메일 인증을 통한 본인인증, 아이디/비밀번호 찾기
  - SMS 인증 : NAVER Cloud Simple & Easy Notification Service - SMS API
  - 이메일 인증 : SMTP 사용
- 신고기능, 친구맺기, 회원탈퇴(계정 비활성화, 30일 경과후 계정 삭제)

### 4-2. 1:1 채팅 기능
- `django-channels`
- 채팅하기 버튼으로 채팅방 자동 생성 / 입장
- 채팅방 참가 권한 인증 (jwt token)
- 입장/퇴장 메시지 출력, 참가중인 채팅방 목록, 이전 채팅 메시지 불러오기 (50개까지)


### 4-3. 맛집 추천
- 데이터 크롤링 (`selenium`) , 글 검색
- 관리자/일반회원 권한 분리
  - 관리자 : 추천 글 작성/수정/삭제
  - 일반 회원 : only 조회
- 위치 기반 추천
  - 지도 : Kakao Maps API
  - 위치 : Geolocation API

### 4-4. 모임생성기능
- 모임 모집 글 작성/조회/수정/삭제, 댓글CRUD , **모임 참가하기**, 글 검색
- 모임 상태 표시 : 모집중, 자리없음, 모집종료(`Django-apscheduler`)



</br>

## 5. 핵심 트러블 슈팅
### 5-1. 채팅방 권한 인증
#### 배경
- 1:1 채팅방을 구현할 때, 각 채팅방의 URL은 참가자 두 명의 고유한 pk 값을 활용하여 생성했다.  
  ex) pk가 14,23인 사용자의 채팅방일 때(정렬을 통해 양쪽 모두 같은 url을 사용하도록 함)  
  : `connectme.co.kr/chat_room.html?room=14n23`
#### 문제
- 이 방식은 식별하긴 쉽지만, 비교적 제3자가 URL에 접근하기 쉬울 수 있다. 유저들의 pk만 알면 URL을 입력해 접근할 수 있기 때문  
- 따라서 이를 방지하기 위해 `jwt token` 을 사용하여 참가 권한을 확인하기로 했다.  
#### 해결
- 웹소켓 URL의 쿼리 파라미터로 사용자의 JWT 토큰을 전달하여 서버에서 디코딩하여 정보를 얻는다.
  <details>
  <summary>기존 채팅방 연결 함수</summary>
  <div markdown = '1'></div>
  
  ```python
  async def connect(self):
      self.room_name = self.scope["url_route"]["kwargs"]["room_name"]
      self.room_group_name = "chat_%s" % self.room_name
      await self.channel_layer.group_add(self.room_group_name, self.channel_name)
      await self.accept()
  ```
  
  </details>
  
  <details>
  <summary>해당 코드</summary>
  <div markdown = '1'></div>
  
  ```python
  async def connect(self):
      self.room_name = self.scope["url_route"]["kwargs"]["room_name"]
      self.room_group_name = "chat_%s" % self.room_name
      await self.channel_layer.group_add(self.room_group_name, self.channel_name)
      await self.accept()
  
      # 파라미터에서 token 추출
      query_string = self.scope["query_string"].decode()
      query_params = parse_qs(query_string)
      token = query_params.get("token", [None])[0]
  
      # 토큰이 없을 경우 WebSocket 연결 끊기
      if not token:
          await self.send_alert("로그인 해주세요.")
          await self.close()
          return
  ```
  
  </details>

- 이후, 토큰에 저장된 사용자 pk값, 즉 id 와 채팅방 URL에 사용된 참가자들의 id를 비교하여 사용자의 채팅방 참가 권한을 확인한다.

  <details>
  <summary>개선된 전체 코드</summary>
  <div markdown = '1'></div>
  
  ```python
  async def connect(self):
      self.room_name = self.scope["url_route"]["kwargs"]["room_name"]
      self.room_group_name = "chat_%s" % self.room_name
      await self.channel_layer.group_add(self.room_group_name, self.channel_name)
      await self.accept()
  
      # 파라미터에서 token 추출
      query_string = self.scope["query_string"].decode()
      query_params = parse_qs(query_string)
      token = query_params.get("token", [None])[0]
  
      # 토큰이 없을 경우 WebSocket 연결 끊기
      if not token:
          await self.send_alert("로그인 해주세요.")
          await self.close()
          return
  
      # jwt 토큰 decoding
      try:
          decoded_token = jwt.decode(
              token, key=settings.SECRET_KEY, algorithms=["HS256"], verify=True
          )
  
      # 토큰이 유효하지 않은 경우 WebSocket 연결 끊기
      except jwt.InvalidTokenError:
          await self.send_alert("로그인이 만료되었습니다.")
          await self.close()
          return
  
      room_name = self.room_name
      participants = room_name.split("n")
      self.user_id = str(decoded_token["user_id"])
      user_id = self.user_id
  
      # 채팅방 참가자가 아닐 경우 WebSocket 연결 끊기
      if user_id not in participants:
          await self.send_alert("채팅 참가 권한이 없습니다.")
          await self.close()
          return
  
      # 채팅방 참가자일 경우 참가 메시지 보내기
      user = await database_sync_to_async(User.objects.filter(id=user_id).first)()
      nickname = user.nickname
  
      join_message = f"{nickname}님이 채팅방에 참가했습니다."
  
      message = {
          "author": "system",
          "content": join_message,
          "timestamp": str(datetime.now()),
      }
      content = {
          "command": "new_message",
          "message": message,
      }
      await self.send_chat_message(content)
      return
  
  ```
  </details>

</br>

### 5-2. 소셜로그인 회원가입 오류
#### 문제
- OAuth API로 SNS로그인을 구현할 때, 인증시에 가져오는 정보는 이메일/닉네임 등으로 **회원 아이디는 따로 가져올 수 없다**. (카카오의 경우 서비스 자체에서 회원 아이디를 사용하지 않음.)
- 해당 SNS 로그인이 처음일 때는 해당 정보로 회원가입이 자동으로 진행되어 `User` 테이블에 정보가 저장되도록 했다.
- 회원 아이디를 가져올 수 없으니 공란(` `)으로 저장되는 현상이 발생했고, 아이디(`User` 테이블의 `account` 필드)는 `unique=True`이므로 
- `account`가 공란(` `)으로 중복이 발생해 최초 소셜로그인 가입자 이후에는 가입이 안 되는 현상이 발생했다.
- <img src="https://github.com/ChaeYami/portfolio/assets/120750451/03ff1394-df13-41f7-addc-57e0d85b4dda" width="450px"></img><br/>



#### 해결방안 논의
- 방법1. 아이디에 무작위 값을 부여한다.
- 방법2. 가져온 이메일 정보에서 아이디를 잘라 저장한다.
- 이메일을 사용하는 경우에도 다른 SNS에서 이미 동일한 아이디를 사용 중일 때 중복이 발생할 가능성이 있기 때문에 **방법1 선택**

#### 해결
- uuid를 사용해서 랜덤값을 아이디에 넣는 방법을 선택했다.
- `uuid1` vs `uuid4`
  - `uuid1`은 현재 시각을 기반으로 하기 때문에 중복 가능성이 더 낮지만, MAC 주소 등 정보 유출 보안 문제가 있어 안전성을 고려하여 `uuid4`를 사용했다.
1. `uuid.uuid4()` 사용해 랜덤값 생성 ex) `93ded08d-d014-4516-b46f eb3078c8f351`
2. `-` 제거 후 슬라이싱, 해당 소셜 로그인 이니셜을 넣어 어떤 SNS 로그인인지 판별 후, 회원가입 시 account에 저장하도록 함
   
  <details>
  <summary>해당 코드</summary>
  <div markdown = '1'></div>
  
  ```python

  """ 카카오로그인 """
    # 생략
  new_account = "K" + str(uuid.uuid4()).replace("-", "")[:10]
  data = {
      "account": new_account,
      "email": user_data.get("kakao_account").get("email"),
      "nickname": user_data.get("properties").get("nickname"),
      "signup_type": "카카오",
      "is_active": True,
  }
    # 생략
  
  """ 구글로그인 """
    # 생략
  new_account = "G" + str(uuid.uuid4()).replace("-", "")[:10]
  data = {
      "account": new_account,
      "email": user_data.get("email"),
      "nickname": user_data.get("name"),
      "signup_type": "구글",
      "is_active": True,
  }
  return SocialLogin(**data)
    # 생략
  ```
  </details>
  
</br>

### 5-3. 글 작성시 HTML 태그가 반영되는 문제
#### 문제
- 모임생성 글/댓글, 맛집추천 댓글, 고민상담 글/댓글/대댓글 등 사용자가 글을 작성하고 업로드할 때에 html태그를 작성하면 그대로 반영되는 문제 발생
- 이는 미관상으로도 매우 좋지 않을 뿐 아니라, XSS(Cross Site Script)취약점 문제 등 보안상으로도 치명적이다.
#### 해결방안 논의
- bleach 라이브러리 사용 vs escape() 함수 사용
- escape 함수는 html를 삭제하는 방식이 아니라서 원본 텍스트를 유지할 수 있다는 장점이 있지만, bleach 라이브러리는 허용할 html 태그 속성을 선택해서 제한할 수 있다는 장점이 있다.
- 유연성을 위해 bleach 라이브러리를 사용하기로 했다.
#### 해결
- 모든 글/댓글 작성 serializer에 html 태그를 삭제하는 validator 생성
   
  <details>
  <summary>ex) 기존 고민상담 글작성 Serializer</summary>
  <div markdown = '1'></div>
  
  ```python
  # counsel/serializer.py
  
  """글 작성, 수정"""
  
  class CounselCreateSerializer(serializers.ModelSerializer):
      class Meta:
          model = Counsel
          exclude = ["user", "like", "created_at", "updated_at"]
          extra_kwargs = {
              "title": {
                  "error_messages": {
                      "blank": "제목을 입력해주세요",
                  }
              },
              "content": {
                  "error_messages": {
                      "blank": "내용을 입력해주세요",
                  },
              },
          }
   
  ```
  </details>

  <details>
  <summary>ex) 개선한 고민상담 글작성 Serializer</summary>
  <div markdown = '1'></div>
  
  ```python
  """글 작성, 수정"""
  
  class CounselCreateSerializer(serializers.ModelSerializer):
      class Meta:
          model = Counsel
          exclude = ["user", "like", "created_at", "updated_at"]
          extra_kwargs = {
              "title": {
                  "error_messages": {
                      "blank": "제목을 입력해주세요",
                  }
              },
              "content": {
                  "error_messages": {
                      "blank": "내용을 입력해주세요",
                  },
              },
          }
  
      def validate(self, attrs):
          title = attrs.get("title")
          content = attrs.get("content")
  
          # title 필드에서 HTML 태그 제거
          cleaned_title = bleach.clean(title, tags=[], strip=True)
  
          # content 필드에서 HTML 태그 제거
          cleaned_content = bleach.clean(content, tags=[], strip=True)
  
          attrs["title"] = cleaned_title
          attrs["content"] = cleaned_content
  
          return attrs
  
  ```
  </details>

</br>

## 6. 그 외 트러블 슈팅
>[프로젝트 노션 - 트러블슈팅](https://rhetorical-cilantro-7e4.notion.site/2ee5f4b3a95544e1a9bb35df82eafaed?v=fd8e920e16ca4c2787ab503c3ea6e3b2&pvs=4)

</br>

## 7. 회고
>

</br>

## 8. 프로젝트 노션(현황/회의록)
> [ConnectME 프로젝트 노션](https://rhetorical-cilantro-7e4.notion.site/538c12449cf94e28b0c20a9f4ac0a3fc?v=96c787ffabfa458586546ec93833852b&pvs=4)
