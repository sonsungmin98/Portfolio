
## Contents
### 업적

**Main**

<img width="48%" src = "https://github.com/user-attachments/assets/2845caca-7e00-4596-91b3-9663e58180d2">
<img width="48%" src = "https://github.com/user-attachments/assets/46abf7dd-7502-4c3e-a3bf-e24ca2efac82">

플레이어가 특정 조건을 달성하면 해당 업적이 클리어 되고 업적에 따라 보상을 획득합니다.  
  
업적의 경우 기본 구현은 Observer pattern을 이용하여 구현하였습니다. Observer pattern으로 구현한 이유는 업적 달성 확인에 대한 세부 구현 사항을 분리하고 싶었기 때문입니다. 
먼저 singleton class를 만들고 subject를 추가합니다. 그 후 Observer class를 구성해 subject에 추가합니다. 
각 업적의 업데이트가 필요한 상황에서 singleton class의 외부 호출부를 통해 관련 데이터들을 넘겨주면 그 후 Subject에 등록된 Observer들을 통해 업적 데이터들을 업데이트 해줬습니다. 

**Gallery & Figure**  

<img width="33%" src = "https://github.com/user-attachments/assets/476e15a7-644f-4a76-8866-a879fc33dcce">
<img width="33%" src = "https://github.com/user-attachments/assets/7380258b-b7f4-427f-ae12-4dc192d2bdf4">
<img width="33%" src = "https://github.com/user-attachments/assets/5fbf4d80-d9b5-42a0-ae44-30aa03f4f3f8">

- Gallery는 업적을 클리어한 개수에 따라 해금되는 컨텐츠 입니다.
- Figure의 경우에는 특정 업적을 달성하면 지급하는 리워드입니다.

피규어의 경우에는 미리 생성해놓는 것이 아닌 클리어한 업적에 따라 동적으로 생성하고 생성된 피규어 대해서만 캐싱해뒀습니다.

### 뱃지(장비) - 연출

<img width="48%" src = "https://github.com/user-attachments/assets/545232cb-9b3b-424d-91dd-c56d915fef45">
<img width="48%" src = "https://github.com/user-attachments/assets/932d4745-0856-431f-8652-6ef28ae5aa69">

보상에 따라 뱃지를 동적으로 생성해 배치를 하여 연출을 진행했습니다. 뱃지의 등급에 따라 glow를 주는 연출도 하였습니다.  
등급에 따른 glow를 그래픽 파트에서 편하게 변경하기 위한 툴을 제작했습니다.

### UI
이외에도 전투 결과창, 보상창, 업적 클리어 노티 창등의 UI를 제작했습니다.

<img width="33%" src = "https://github.com/user-attachments/assets/21decce2-51be-4c27-8850-890e945fae4c">
<img width="33%" src = "https://github.com/user-attachments/assets/f6e228d5-af83-4eee-ac09-3e8881bb2f3d">
<img width="33%" src = "https://github.com/user-attachments/assets/2e0642e7-95ad-44c7-bc67-c817b6a6f020">

## 이외
### FCM, IOS 빌드

Firebase Clould Message, Adjust등의 sdk를 부착했고 Ios 빌드의 빌드 자동화 부분을 추가하는 등의 작업을 했습니다.
