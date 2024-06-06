# 바텐더 시뮬레이션

<p align="right">  
  <a href="https://youtu.be/L0K8DpX9mrk">
    <img src="https://github.com/1506022022/BartenderSimulation/assets/88864717/5f5bdec3-15f9-49c8-83fb-764ea4aee77f.png" width="80%" height="80%">
  </a>
</p>
  
<div align="right">
    <a href="https://youtu.be/L0K8DpX9mrk">동영상으로 이동</a>
</div>


# 프로젝트 구성
|개요|내용|
|---|---|
|장르|VR 시뮬레이션|
|개발기간|2023.03.01 - 2023.10.31|
|참여인원|4인|
|개발환경|유니티, HTC VIVE Pro 2|

# 담당 구현요소

- **제출 공간**
``` C#
public class SubjectSpace : MonoBehaviour
{
    
    const string SubjectTargetTag = "Glass";    // 이벤트를 처리할 대상
    public UnityEvent subjectEvent;             // 발동시킬 이벤트

    void OnTriggerEnter(Collider other)
    {
        EnterSpace(other);
    }

    void EnterSpace(Collider other)
    {
        if (other.tag != SubjectTargetTag) return;

        // 대상으로 부터 정보를 받아온다.
        var target = other.gameObject;
        var targetRigid = target.GetComponent<Rigidbody>();
        var targetInteractable = target.GetComponent<Interactable>();
        var attachedToHand = targetInteractable?.attachedToHand;

        // 대상을 자유로운 상태로 만든다.
        if (Checker.Exist(targetInteractable, attachedToHand)) DetacchToHand();
        if (targetRigid) NonKinematicTarget();

        // 제출 이벤트 발동
        if(subjectEvent.GetPersistentEventCount() > 0) subjectEvent.Invoke();

        #region localFunc
        void DetacchToHand()
        {
            attachedToHand.DetachObject(target);
            targetInteractable.attachedToHand = null;
        }

        void NonKinematicTarget()
        {
            targetRigid.isKinematic = true;
        }
        #endregion
    }
}
```
##
- **타이머**   
```
상태에 따라 게이지를 제어합니다.   
타임아웃 이벤트를 발송합니다.
```
##
- **카드**   
```
문양과 숫자 값을 가지고 있습니다.   
값과 상태에 따라 애니메이션을 관리합니다.   
```
##
- **플레이어**   
```
유저의 정보를가지고 있습니다.   
손패를 가지고 있습니다.   
마우스 이벤트를 기반으로 손패를 관리합니다.   
```
##
- **게임 매니저**   
```
프로그램의 생명주기를 관리합니다.   
데이터 리소스를 읽어들여 인스턴스들을 생성합니다.   
```
##
- **게임 오브젝트**   
```
트랜스폼 값을 가진 가장 기본적인 구성요소입니다.   
생성과 파괴로 메모리를 관리합니다.   
트랜스폼 참조를 통해 다른 컴포넌트들이 동작합니다.   
```
##
- **마우스 이벤트**   
```
옵저버 패턴으로 마우스 이벤트를 관리합니다.   
```
##
- **키보드 이벤트**   
```
옵저버 패턴으로 키보드 이벤트를 관리합니다.   
```
##
- **애니메이션**   
```
이미지 배열을 가지고 있습니다.
재생속도에 따라 이미지를 순차적으로 재생합니다.   
```
##
- **무브먼트**   
```
게임 오브젝트를 이동시킵니다.   
```
##
- **사운드 매니저**   
```
오디오 소스들을 가집니다.
싱글톤 패턴으로 사운드를 재생을 관리합니다.   
```
