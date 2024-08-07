# 바텐더 시뮬레이션

<p align="right">  
  <a href="https://youtu.be/L0K8DpX9mrk">
    <img src="https://github.com/1506022022/BartenderSimulation/assets/88864717/5f5bdec3-15f9-49c8-83fb-764ea4aee77f.png" width="80%" height="80%">
  </a>
</p>
  
<div align="right">
    <a href="https://youtu.be/L0K8DpX9mrk">동영상으로 이동</a>
</div>

># 목차
- **[개요](#프로젝트-구성)**
- **[제출 공간](#제출-공간)**
- **[아이템](#아이템)**
- **[오브젝트](#오브젝트)**


># 프로젝트 구성
|개요|내용|
|---|---|
|장르|VR 시뮬레이션|
|개발기간|2023.03.01 - 2023.09.30|
|참여인원|4인|
|개발환경|유니티, HTC VIVE Pro 2|
|프로젝트 링크|<a href="https://github.com/1506022022/BartenderSimulation">프로젝트 링크</a>|

># 제출 공간
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/e8dc2d56-7eef-42c7-973b-c1035e1194ac" width="40%" height="40%"/>
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/b1adf08e-65d3-4ab1-b0bf-9e69facdecae" width="26.8%" height="26.8%"/>

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
  ## 코드
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

># 아이템
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/2fa4ff9d-315d-4ab7-b3c5-b1b1ca367ff2" width="50%" height="50%"/>
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/15aab53a-a279-42d0-a1c7-3b3709fc4152" width="37%" height="37%"/>

```
 칵테일을 제조하는 과정에서 아이템을 구분해서 인식해야 하는 상황들이 있었습니다.
가니쉬를 장식할 때가 대표적인 상황입니다. 이때 사용하는 도구인 픽은 가니쉬만 집을 수 있어야 했습니다.
 이러한 기능을 구현하기 위해 아이템에 ID값을 부여하고, GUI로 구분했습니다.
```

  ## 코드
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

># 오브젝트
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/50ba029c-f6f1-4d48-88ac-90a026555406" width="40%" height="40%"/>
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/0dfd7304-4dff-43c7-afb2-6f0b1ac4c89e" width="42.7%" height="42.7%"/>

```
 오브젝트를 파괴하는데 발생하는 오버헤드를 줄이기 위해서 오브젝트가 지닌 값만 변경하여
전혀 다른 오브젝트로 동작할 수 있도록 의도했습니다.
 오브젝트가 가져야 하는 기능을 작은 단위로 구분짓고 데이터를 기반으로 초기화하였습니다.
```

  ## 코드
```C#
public class GameObjectManager : MonoBehaviour
{
    #region static
    static List<GameObjectManager> managers;

    public static void InitGameObject()
    {
        foreach (var manager in managers)
        {
            if (!manager._isDontInit)
                manager.Destroy();
        }
        foreach (var manager in managers)
        {
            if (!manager._isDontInit)
                manager.Create();
        }
    }
    #endregion

    public bool IsCreated { get; private set; }
    PositionIniter _posIniter;
    [SerializeField] bool _isInitPos;
    [SerializeField] bool _isDontDestoryChildrens;
    [SerializeField] bool _isDontDestoryComponents;
    [SerializeField] bool _isDontInit;
    [SerializeField] bool _isDontCreate;
    [SerializeField] Features _features;
    [SerializeField] List<Component> _dontDestroy;

    public int GetID => _features.ID;
    public string GetName => _features.Name;
    public string GetDescription => _features.Description;
    public bool isEmptyGameObject => _features != null;
    public ICustomEditorItemData GetGUI => _features.GetCustomEditor();

    public void Create()
    {
        if (_isDontCreate) return;
        if (IsCreated) return;
        if (_isInitPos) _posIniter.SetPos();
        _features.GetCustomEditor().Create(gameObject);
        gameObject.SetActive(true);
        IsCreated = true;
    }

    public void Destroy()
    {
        if (!_isDontDestoryChildrens)
            GetGUI?.Destroy(gameObject);
        IsCreated = false;
        gameObject.SetActive(false);
    }

    void Awake()
    {
        if (managers == null) managers = new List<GameObjectManager>();
        managers.Add(this);
    }

    void OnDestroy()
    {
        managers.Remove(this);
    }

    void Start()
    {
        _posIniter = new PositionIniter(transform, transform.position, transform.rotation);
        DestroyOtherComponent();
        Create();
    }

    class PositionIniter
    {
      Transform _target;
      Vector3 _firstPos;
      Quaternion _firstRot;

      public PositionIniter(Transform target, Vector3 firstPos, Quaternion firstRot)
      {
          _target = target;
          _firstPos = firstPos;
          _firstRot = firstRot;
      }
      public void SetPos()
      {
          _target.SetPositionAndRotation(_firstPos, _firstRot);
      }
    }
```
