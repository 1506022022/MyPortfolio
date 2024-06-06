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
- **정보창**   
```C#
[RequireComponent(typeof(GameObjectManager))]
[RequireComponent(typeof(ItemDataComponent))]
public class ItemDataWindow : MonoBehaviour
{
    GameObject UIWindow;
    static GameObject focusObject;
    GameObjectManager _manager;
    Text ui;

    private void OnDestroy()
    {
        View(false);
    }

    private void OnDisable()
    {
        View(false);
    }

    void Awake()
    {
        _manager = GetComponent<GameObjectManager>();
    }

    void Update()
    {
        if (focusObject != gameObject)
        {
            View(false);
            return;
        }

        var interactable = GetComponent<Interactable>();
        if (interactable != null && interactable.attachedToHand != null)
        {
            View(false);
            return;
        }

        ui.text = "";


        if (_manager.IsCreated)
        {
            ui.text = "[" + _manager.GetName + "]";
            ui.text += "\n\n" + _manager.GetDescription;
        }

    }

    protected virtual void OnHandHoverBegin(Hand hand)
    {
        View(true);
    }

    protected virtual void OnHandHoverEnd(Hand hand)
    {
        View(false);
    }

    public void View(bool isView)
    {
        if (isView)
        {
            RemoveBeforeWindow();
            OpenWindow();
            SetWindowPos();
            Operating(true);
        }
        else
        {
            Destroy(UIWindow);
            UIWindow = null;
            Operating(false);
        }

        void RemoveBeforeWindow()
        {
            var before = focusObject?.GetComponent<ItemDataWindow>()?.UIWindow;
            if (before != null) Destroy(before);
        }

        void SetWindowPos()
        {
            Vector3 pos = transform.position;
            pos.y = +1;
            UIWindow.transform.position = pos;
        }

        void OpenWindow()
        {
            UIWindow = Instantiate(Resources.Load<GameObject>("ItemDataWindow"));
            ui = UIWindow.GetComponentInChildren<Text>();
            focusObject = gameObject;
            UIWindow.SetActive(true);
        }

        void Operating(bool _is)
        {
            enabled = _is;
        }
    }
}
```
```
 플레이를 원할하게 진행하기 위해서 아이템을 호버링하면 정보창을 출력하는 기능이 필요했습니다.
정보창은 동시에 하나만 출력해야 한다는 조건이 있었습니다. 때문에 싱글톤 패턴을 활용해서 포커스된
오브젝트의 정보창만 출력하도록 설계했습니다.  
```
##
- **오브젝트**   
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
```
 오브젝트를 파괴하는데 발생하는 오버헤드를 줄이기 위해서 오브젝트가 지닌 값만 변경하여
전혀 다른 오브젝트로 동작할 수 있도록 의도했습니다.
 오브젝트가 가져야 하는 기능을 작은 단위로 구분짓고 데이터를 기반으로 초기화하였습니다.

```
##
- **병 속의 액체**   
```C#
public class AllMeshPressure : MonoBehaviour
{
    Vector3 beforeRot, beforePos;

    Mesh _mesh;
    Vector3[] _vertices;
    Dictionary<float, List<Vector3>> _layerVertices;
    Mesh _prefabMesh;
    public float ThresholdY = 5.0f;
    public Vector2 RangeY = new Vector2();
    [SerializeField] GameObject _vertexPrefab;
    [SerializeField] Mesh _originMesh;
    private void Awake()
    {
        _mesh = GetComponent<MeshFilter>()?.mesh;
        _vertices = _mesh.vertices;

        Debug.Assert(_mesh != null, "Mesh를 추가해주세요.");
        PrefabMeshCreate();
        LoweringPrecision();
        Layering();
    }
    void Update()
    {
        PrefabMeshCreate();
        if (beforePos != transform.position || beforeRot != transform.eulerAngles)
        {
            beforePos = transform.position;
            beforeRot = transform.eulerAngles;
            for (int i = 0; i < _prefabMesh.vertices.Length; i++)
            {
                _prefabMesh.vertices[i] = transform.rotation * _prefabMesh.vertices[i] + transform.position;
                _prefabMesh.vertices[i].y = Mathf.Clamp(Floor(_prefabMesh.vertices[i].y, 1), _prefabMesh.vertices.Min(x => x.y), _prefabMesh.vertices.Max(x => x.y));
            }
            Layering();
        }
        ClampY();
        Pressure();
    }

    private void PrefabMeshCreate()
    {
        _prefabMesh = Instantiate(_originMesh);
    }


    private void ClampY()
    {
        if (RangeY == Vector2.zero)
            ThresholdY = Mathf.Clamp(ThresholdY, _layerVertices.Min(x => x.Key), _layerVertices.Max(x => x.Key));
        else
            ThresholdY = Mathf.Clamp(ThresholdY, RangeY.x, RangeY.y);
    }

    float Round(float originalNumber, float numberOfDigits)
    {
        float roundedNumber = Mathf.Round(originalNumber * Mathf.Pow(10, numberOfDigits)) / Mathf.Pow(10, numberOfDigits);
        return roundedNumber;
    }
    float Floor(float originalNumber, float numberOfDigits)
    {
        float flooredNumber = Mathf.Floor(originalNumber * Mathf.Pow(10, numberOfDigits)) / Mathf.Pow(10, numberOfDigits);
        return flooredNumber;
    }
    void Pressure()
    {
        Vector3[] copyVertices = _prefabMesh.vertices.ToArray();
        bool isFlat = _prefabMesh.vertices.Where(x => x.y <= ThresholdY).Count() == 0;

        if (_mesh == null) return;
        if (isFlat) return;

        float upSideLayer = _prefabMesh.vertices.Max(x => x.y);
        if (_vertices.Where(x => x.y > ThresholdY).Count() > 0)
            upSideLayer = _prefabMesh.vertices.Where(x => x.y > ThresholdY).Min(x => x.y);
        var targetLayer = _layerVertices.Where(x => x.Key <= ThresholdY).Max(x => x.Key);
        var targetLayerVertices = _layerVertices[targetLayer];

        for (int i = 0; i < copyVertices.Length; i++)
        {
            if (copyVertices[i].y > ThresholdY)
            {
                var vertex = copyVertices[i];

                var upLayerVertices = _prefabMesh.vertices.Where(x => x.y == upSideLayer);
                var upLayerMinDistance = upLayerVertices.Min(x => Vector3.Distance(vertex, x));
                var upLayerMinDistanceVector = upLayerVertices.Where(x => Vector3.Distance(vertex, x) == upLayerMinDistance).First();
                vertex = upLayerMinDistanceVector;

                var minDistance = targetLayerVertices.Min(x => Vector3.Distance(vertex, x));
                var minDistanceVector = targetLayerVertices.Where(x => Vector3.Distance(vertex, x) == minDistance).First();
                float lerpT = Mathf.Abs(vertex.y - ThresholdY) / Mathf.Abs(vertex.y - minDistanceVector.y);
                var lerpVector = Vector3.Lerp(vertex, minDistanceVector, lerpT);

                copyVertices[i].y = ThresholdY;
                copyVertices[i].x = lerpVector.x;
                copyVertices[i].z = lerpVector.z;
            }
        }

        _mesh.vertices = copyVertices;
        _mesh.RecalculateBounds();
    }
    [ContextMenu("d")]
    void CreateShowVerticesModel()
    {
        List<Vector3> unique = new List<Vector3>();
        Mesh newMesh = new Mesh();
        newMesh.vertices = _vertices;
        newMesh.triangles = _mesh.triangles;
        GameObject obj = new GameObject("Model");

        Debug.Assert(_vertices != null && _vertices.Length > 0, "vertices를 먼저 할당해주세요.");

        obj.AddComponent<MeshRenderer>();
        obj.AddComponent<MeshFilter>().mesh = newMesh;
        int i = 0;
        foreach (var item in _layerVertices)
        {
            foreach (var vertex in item.Value)
            {
                if (!unique.Any(x => x.Equals(vertex)))
                {
                    unique.Add(vertex);
                    var abc = Instantiate(_vertexPrefab, obj.transform);
                    abc.transform.SetLocalPositionAndRotation(vertex, Quaternion.identity);
                    abc.GetComponent<MeshRenderer>().material.color = i % 2 == 0 ? Color.black : Color.red;
                }
            }
            i++;
        }
    }

    void LoweringPrecision()
    {
        Debug.Assert(_originMesh.vertices != null && _originMesh.vertices.Length > 0, "vertices를 먼저 할당해주세요.");

        for (int i = 0; i < _originMesh.vertices.Length; i++)
        {
            _originMesh.vertices[i].y = Round(_originMesh.vertices[i].y, 2);
            _originMesh.vertices[i].y = _vertices[i].y > 1.5f ? 1.56f : _originMesh.vertices[i].y;
            _originMesh.vertices[i].x = Round(_originMesh.vertices[i].x, 2);
            _originMesh.vertices[i].z = Round(_originMesh.vertices[i].z, 2);
        }
    }

    void Layering()
    {
        _layerVertices = new Dictionary<float, List<Vector3>>();
        var list = _prefabMesh.vertices?.OrderBy(x => x.y).ToList();
        var unique = new List<Vector3>();

        Debug.Assert(_prefabMesh.vertices != null && _prefabMesh.vertices.Length > 0, "vertices를 먼저 할당해주세요.");

        foreach (var item in list)
        {
            if (unique.Any(x => x.Equals(item))) continue;

            unique.Add(item);
            if (_layerVertices.TryGetValue(item.y, out List<Vector3> e))
                e.Add(item);
            else
                _layerVertices.Add(item.y, new List<Vector3>() { item });
        }
    }
}
```
