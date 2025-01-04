# PlayerCharacter
## Player FSM

<p align="center" width="100%">
    <img width="40%" src="https://github.com/user-attachments/assets/b9607b5a-243a-4c14-a3b1-24e5be5f9349">
</p>

<p align="center" width="100%">
    <img width="40%" src="https://github.com/user-attachments/assets/e4a6b9b5-76ef-4318-ab62-46de391be4e6">
</p>

플레이어 FSM을 설계하였고 State 클래스는 enter, update, exit 인터페이스가 추상화되어 있습니다. 여기서 State의 Key값을 enum class에서 int형으로 바꿨습니다. 이렇게 만든 이유는 2가지가 있습니다.  
1. FSM을 사용하는 일부 시스템들이 Concrete State의 세부 사항에 의존하게 된다.
2. enum class와 Concrete State가 1:1 대응이 되어 같은 Type이지만 바인딩 된 데이터에 따라 행동 양식이 바뀌는 State를 표현하지 못하였습니다.

## Skill Tree
<p align="center" width="100%">
    <img width="40%" src="https://github.com/user-attachments/assets/aa20c7e1-dee8-4640-8ed1-d9c59c3962ac">
</p>
<p align="center" width="100%">
    <img width="70%" src="https://github.com/user-attachments/assets/1b83acec-1211-42cc-8a15-a8ba2b29a957">
</p>

플레이어의 콤보공격은 위와 같이 구성 되어있습니다. 입력마다 여러가지 다른 공격들을 조합하여 사용하는 게임입니다. 이러한 구조는 트리 구조로 만들면 좋겠다고 생각 했습니다(각 스킬들이 부모에 바인딩 되어있는 형식이기 때문에). 
그리고 이런 트리구조를 다른 곳에서도 사용 가능하게 재사용성 측면을 고려하여 설계하였습니다.  
또한 기획자가 편하게 콤보 공격을 디자인 할 수 있도록 블루프린트에서 헬퍼함수를 지원하여 직접 조립할 수 있게 제작했습니다.

## 퀄리티 및 조작감 업 R&D
3인칭 백뷰 게임을 만들다보니 조작감을 올리기 위한 디테일이 중요하다고 생각했습니다. 조그마한 디테일의 차이가 조작감에 많은 영향을 미친다고 생각하였고 정말 조작감은 잘 잡았다고 생각합니다.
여러 게임들을 분석하며 만든 내용들을 적으려고 합니다. 이 밑의 대부분의 작업의 경우 제가 기획자에게 건의한 내용입니다. (니어오토마타를 많이 참고하였습니다.)

**IK + 이동**  
플레이어 다리에 IK를 적용하였습니다. 그리고 플레이어는 급하게 꺾을 때에 좌, 우로 기울어지게 하였습니다.
<p align="center" width="100%">
    <img width="70%" src="https://github.com/user-attachments/assets/d2fab338-7275-4172-9250-ab7a924a2636">
</p>

**Look At**
캐릭터가 플레이어가 바라보는 방향으로 고개를 돌리도록 하였습니다.
<p align="center" width="100%">
    <img width="70%" src="https://github.com/user-attachments/assets/8da522f6-af2b-48a8-977c-10743ab93f30">
</p>

**상하체 분리**
방어 시에 상체와 하체의 분리를 하였고, 점프 착지 시에는 각 본에 가해지는 블렌딩을 차이나게 하여 착지가 자연스럽게 되도록 만들었습니다.
<p align="center" width="100%">
    <img width="70%" src="https://github.com/user-attachments/assets/f011355d-2829-47b1-a366-dd162f5a81a4">
</p>

**카메라 블렌딩 & 쉐이킹**
카메라의 암이 오브젝트에 막혀 짧아질 때에는 순간적으로 줄이지만 멀어질 때에는 천천히 멀어지게 제작했습니다. 다시 돌아오는 거리 x에 대해서는 거리에 상관없이 돌아올 때에는 똑같은 시간이 걸리도록 하였습니다.  
카메라의 쉐이킹의 경우에는 Curve의 형태로 제작하여 각 스킬마다 고유의 쉐이킹을 만들어 자연스럽게 만들었습니다.
<p align="center" width="100%">
    <img width="70%" src="https://github.com/user-attachments/assets/c4bc4140-0178-4ac0-90d7-44a7aa695fae">
</p>

# Util
## UI
<p align="center" width="100%">
    <img width="40%" src="https://github.com/user-attachments/assets/09481b6f-ce05-4ef2-8fcf-db6a547a61df">
</p>

모든 UI들은 위와 같이 Enter과 Exit을 가진 기본 UI블루프린트를 상속받습니다.


