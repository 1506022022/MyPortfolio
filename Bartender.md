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
    const string SubjectTargetTag = "Glass";
    public UnityEvent subjectEvent;

    void OnTriggerEnter(Collider other)
    {
        EnterSpace(other);
    }

    void EnterSpace(Collider other)
    {
        if (other.tag != SubjectTargetTag) return;

        var target = other.gameObject;
        var targetRigid = target.GetComponent<Rigidbody>();
        var targetInteractable = target.GetComponent<Interactable>();
        var attachedToHand = targetInteractable?.attachedToHand;

        if (Checker.Exist(targetInteractable, attachedToHand)) DetacchToHand();
        if (targetRigid) NonKinematicTarget();

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
```
 제출 공간은 게임의 목표에 해당하는 기능입니다. 제출된 칵테일을 확인한 후
목표 달성에 따른 이벤트들이 실행되어야 했습니다.

이 기능을 만들면서 실행될 이벤트가 확장되거나, 변경될 가능성이 높다고 생각했습니다.   
그래서 수정되지 않을 부분과, 기능이 확장될 부분을 분리하는 것에 집중했습니다.

 우선 제출을 인식하는 기능과, 제출된 칵테일을 플레이어로부터 분리하는 기능은   
반드시 있어야 하는 부분으로 생각해 고정된 코드로 작성했습니다.

 반면 제출 이벤트라는 확장 가능성 높은 기능은 유니티 이벤트를 사용해서 구현했습니다.   
유니티 이벤트를 사용한 이유는 이벤트를 통해 제출 이펙트를 실행하고, 제출했다는 신호를
보내어 추후에 채점 기능을 추가하게 되었을 때에 사용할 수 있을 거라고 판단했기 때문입니다.
```
##
- **아이템**   
```C#
public static class ID
{
    static int _groupRange = 10000;
    public enum ItemGroup
    {
        None = 0,
        Bottle = 1,
        Garnish = 2,
        Glass = 3,
        Tool = 4
    }
    public static ItemGroup GetGroup(int ID)
    {
        ItemGroup group = ItemGroup.None;
        int gid = ID / _groupRange;

        if (0 < gid && gid <= GetItemGroupEnumCount())
            group = (ItemGroup)gid;

        return group;
    }
    static int GetItemGroupEnumCount()
    {
        return Enum.GetNames(typeof(ItemGroup)).Length;
    }
}
```
```
 칵테일을 제조하는 과정에서 아이템을 구분해서 인식해야 하는 상황들이 있었습니다.
가니쉬를 장식할 때가 대표적인 상황입니다. 이때 사용하는 도구인 픽은 가니쉬만 집을 수 있어야 했습니다.
 이러한 기능을 구현하기 위해 아이템에 ID값을 부여하고, GUI로 구분했습니다.
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
