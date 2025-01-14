<p align="right">  
  <a href="https://www.youtube.com/watch?v=FDuE9BE1mF4">
    <img src="https://github.com/user-attachments/assets/e4766513-99c9-4bac-a200-d9cae57518e8" width="70%" height="70%">
    <br><a href="https://www.youtube.com/watch?v=FDuE9BE1mF4"><b>영상</b></a>
  </a>
</p>

># 목차
- **[프레임워크](#프레임워크)**
  - **[GameManager](#GameManager)**
- **[이벤트 핸들러](#이벤트-핸들러)**
  - **[IF](#IF)**
  - **[Condition](#Condition)**
  - **[Portal](#Portal)**
  - **[Player Controller](#Player-Controller)**
  - **[Hitbox](#Hitbox)**
  - **[Timer](#Timer)**
- **[스크립터블 오브젝트](#스크립터블-오브젝트 )**
  - **[Debug Log](#Debug-Log)**
  - **[Player Character Manager](#Player-Character-Manager)**
  - **[Shy Box Manager](#Shy-Box-Manager)**
- **[정형화](#정형화)**
  - **[Shy Box](#Shy-Box)**
  - **[Flower](#Flower)**
  - **[Contents Loader](#Contents-Loader)**

># 프로젝트 구성
|개요|내용|
|---|---|
|장르|3D 퍼즐 액션|
|개발기간|2024.05.01 - (진행중)|
|참여인원|5인|
|개발환경|유니티|
|깃허브 링크|<a href="https://github.com/CREDOCsGames/cute_minecraft">프로젝트 링크</a>|

>## 프레임워크

## GameManager
  <img src="https://github.com/user-attachments/assets/a85dfd98-1184-4963-9e5c-c999a1d36f65" width="30%" height="30%"/>   

```
플레이어의 위치에 따라 이벤트를 발동시킬 수 있도록 설계했습니다.
주어진 이벤트들을 통해 게임의 규칙을 관리합니다.

GameManager 클래스에서 읽어 들어가면 전체적인 내용을 파악할 수 있어야 한다고 생각해 이런 방식으로 작성했습니다.

씬을 로드하거나 스테이지 내에서 퍼즐 영역에 진입하게 되면 플레이어가 위치해 있는 영역을 찾습니다.

프로젝트의 초기에는 마인크래프트처럼 레벨을 블록 단위로 구성하고, 유저 크리에이티브 컨텐츠를 고려하고 있었기 때문에
레벨의 최적화에 대해 고민했었습니다. 그 중 하나로 스테이지 내에서는 영역을 좌표 기준으로 세분화해서 인접한 영역만
활성화하도록 처리하고 있습니다.
```

  ## 코드
``` C#
    public static class GameManager
    {
        public static Movie MovieCutscene { get; private set; } = new();
        public static Area PuzzleArea { get; private set; } = new();
        public static Area StageArea { get; private set; } = new();
        public static Selection Title { get; private set; } = new();
        public static Selection StageSelect { get; private set; } = new();
        static Timer GameTimer { get; set; } = new();

        public static LanternComponent Lantern;

        static bool VisitTitle;

        static GameManager()
        {
            PuzzleArea.OnEnterEvent += PuzzlePieceComponent.EnablePieceInArea;
            PuzzleArea.OnClearEvent += PuzzlePieceComponent.DisablePieceInArea;
            PuzzleArea.OnExitEvent += PuzzlePieceComponent.DisablePieceInArea;

            Title.OnEnterEvent += () =>
            {
                if (VisitTitle) Title.Skip();
                VisitTitle = true;
            };

            StageArea.OnClearEvent += (area) => StageManager.Instance.ClearCurrentStage();
            StageArea.OnEnterEvent += (area) =>
            {
                SoundManagerComponent.Instance.PlayMusic($"Stage_{StageManager.Instance.StageIndex.x}");
            };


            ContentsLoader.OnStartLoad += GameTimer.Stop;
            ContentsLoader.OnLoaded += GameTimer.Start;

            MovieCutscene.OnPlayEvent += PlayerCharacterManager.Instance.ReleaseController;
            MovieCutscene.OnEndEvent += PlayerCharacterManager.Instance.ControlDefaultCharacter;
            ContentsLoader.OnStartLoad += PlayerCharacterManager.Instance.ReleaseController;
            ContentsLoader.OnLoaded += PlayerCharacterManager.Instance.ControlDefaultCharacter;

            PuzzleArea.OnEnterEvent += (x) => AreaManager.DisConnect();
            PuzzleArea.OnClearEvent += (b) => AreaManager.Connect();

            ContentsLoader.SetLoaderType(LoaderType.LevelLoader);
            ContentsLoader.LoadContents();

            Debug.Log("Run Game");
        }
    }
```

>## 이벤트 핸들러
```
가장 기본적인 이벤트 핸들러로는 오브젝트끼리 부딪혔을 때 이벤트를 발동시키는 트리거 이벤트 핸들러가 있습니다.
이벤트 핸들러는 다음과 같은 상황에 유용하게 사용할 수 있었습니다.
- 프로토타입의 제작
- 단순한 기능 구현
- 협업

이벤트 핸들러는 프로그래밍에 대한 수준이 높지 않은 사람도 에디터에서 작업이 가능하게합니다.
또한 조건에 따라 행동이 달라지는 식의 복잡한 switch문을 대체할 수 있다는 점에서 코드를 간결하게 만드는 이점이 있습니다.

"이러한 이점을 살리면 코딩과 동일한 작업을 에디터에서 할 수 있지 않을까?"하는 생각이 들어 아래의 유형을 이벤트 핸들러로
만들었습니다.
```

## IF
  <img src="https://github.com/user-attachments/assets/a1c875a6-7107-49bf-9b29-b02c2b5d4200" width="30%" height="30%"/>
  <img src="https://github.com/user-attachments/assets/c39ed599-22cd-47b5-8c5b-000833e0fb34" width="30%" height="30%"/>

```
조건문을 이벤트 핸들러로 만들었습니다. 이 이벤트 핸들러로 복잡한 퍼즐도 빠르게 구현할 수 있었습니다.

예를 들어 화로에 불이 붙으면 해결되는 퍼즐을 만들었었습니다.
조건(불을 붙이면)을 만족하면 이벤트(퍼즐이 해결된다)를 실행하도록 IF 컴포넌트를 사용했습니다.

반면에 거짓일 때도 이벤트를 실행하는 Branch가 있습니다.
불이 꺼져있을때 공격받으면 불이 켜지고, 불이 켜져있을땐 꺼지는 화로는 Branch 컴포넌트로 만들었습니다.

조건문을 이벤트 핸들러로 만든 덕에 시간이 남는 팀원이 간단한 부분을 대신 만들어 주는 등 능률이 좋아질 수 있었습니다.
또한 기획이 바뀌어 퍼즐의 조건이나 이벤트가 바뀌어도 유연하게 대처할 수 있었습니다.
```
  ## 코드
``` C#
    public class IF : MonoBehaviour
    {
        [SerializeField] List<Condition> mConditions;
        [SerializeField] UnityEvent TrueEvent;

        public void Invoke()
        {
            OnInvoke();
        }

        protected virtual bool OnInvoke()
        {
            if (IsTrue())
            {
                OnTrue();
                return true;
            }
            return false;
        }

        public void AddListenerTrueEvent(UnityAction listener)
        {
            TrueEvent.AddListener(listener);
        }

        public void RemoveListenerTrueEvent(UnityAction listener)
        {
            TrueEvent.RemoveListener(listener);
        }

        public void AddCondition(Condition condition)
        {
            if (mConditions.Contains(condition))
            {
                Debug.Log($"already exist : {condition}");
                return;
            }

            mConditions.Add(condition);
        }

        public void RemoveCondition(Condition condition)
        {
            if (!mConditions.Contains(condition))
            {
                Debug.Log($"Not found : {condition}");
                return;
            }

            mConditions.Remove(condition);
        }

        public void ResetConditions()
        {
            foreach (var condition in mConditions)
            {
                condition.SetFalse();
            }
        }

        bool IsTrue()
        {
            if (mConditions.Count == 0)
            {
                return true;
            }
            return !mConditions.Any(x => !x.IsTrue());
        }

        void OnTrue()
        {
            TrueEvent.Invoke();
        }
    }
```

## Condition
  <img src="https://github.com/user-attachments/assets/fb63b71d-49e8-44c4-835d-5d6408923924" width="30%" height="30%"/>
  <img src="https://github.com/user-attachments/assets/df990ed5-43bb-4b46-a11f-8b534a33fbbb" width="30%" height="30%"/>

```
조건문에 사용되는 조건을 컴포넌트로 만들었습니다.

하나의 조건만 확인하는 bool 이외에도 여러 조건을 확인하는 경우.
- 모든 조건을 만족해야 참이다. (AND)
- 하나의 조건만 만족해도 참이다. (OR)

조건끼리 서로 연결되어있는 경우.
- A가 참이면 B도 참이다.
- A가 참이고 B가 거짓이어야 C가 참이다.

이러한 경우를 BOOL, AND, OR 컴포넌트로 구현하였습니다.

복잡한 규칙도 에디터에서 구현할 수 있게 되었습니다.
특히 이 게임에서 구현한 모든 퍼즐의 프로토타입은 조건과 조건문의 컴포넌트를 사용해서 만들었습니다.

하나의 퍼즐을 구현하는데 1 ~ 2시간 정도로 적게 걸렸습니다. 이렇게 구현의 속도를 높여주는게
이벤트 핸들러의 최고의 장점이라고 생각합니다.
```
  ## 코드
``` C#
    public class AND : Condition
    {
        [SerializeField] List<Condition> mConditions;

        public override bool IsTrue()
        {
            if (mConditions.Count == 0)
            {
                Debug.Log("Condition count : 0");
                return true;
            }
            return !mConditions.Any(x => !x.IsTrue());
        }

        public override void SetFalse()
        {
            foreach(var condition in mConditions)
            {
                condition.SetFalse();
            }
        }
    }
```

## Portal
  <img src="https://github.com/user-attachments/assets/5b35956e-c50d-407b-ac5f-5f7b627dd7ce" width="40%" height="40%"/>
  <img src="https://github.com/user-attachments/assets/a4610769-a8f0-462d-ac78-65b77986b873" width="33%" height="33%"/>

```
캐릭터가 포털에 진입하면 어디론가 워프(이벤트 발동)됩니다. 이런 특징을 이벤트 핸들러로 만들어 봤습니다.
몇 명의 캐릭터가 진입했을때 이벤트를 발동시킬지 지정이 가능합니다.

Portal은 플레이어블 캐릭터에 대한 전용 트리거 이벤트 핸들러입니다. 캐릭터 컴포넌트를 가진 오브젝트 중 Player 태그를
가지고 있는 대상에 대해서만 발동합니다. 이러한 기능은 퍼즐에도 사용되었습니다.

퍼즐은 일정한 범위를 기준으로 진행하는 영역이 구분되어 있습니다. 이런 영역에 들어갈 때, 나올 때에 연출을 실행하거나
퍼즐을 초기화하는 기능을 발동합니다.
```
  ## 코드
``` C#
    public class Portal : MonoBehaviour
    {
        [Range(0, byte.MaxValue)]
        public byte NeedCharacters = 1;
        protected WorkState State { get; set; }
        protected readonly Dictionary<Character.Character, bool> mEntrylist = new();
        [SerializeField] UnityEvent mRunEvent;
        [SerializeField] UnityEvent mEndEvent;

        protected virtual void RunPortal()
        {
            State = WorkState.Action;
            mRunEvent.Invoke();
        }

        protected virtual bool CanRunningPortal(Character.Character other)
        {
            Debug.Assert(NeedCharacters <= mEntrylist.Count);
            return State == WorkState.Ready &&
                   NeedCharacters <= mEntrylist.Count(x => x.Value);
        }

        public void ResetEntrylist()
        {
            mEntrylist.Clear();
            var characters = PlayerCharacterManager.Instance.JoinCharacters;
            foreach (var character in characters)
            {
                mEntrylist.Add(character, false);
            }
        }

        protected virtual void OnTriggerEnter(Collider other)
        {
            var character = other.GetComponent<Character.Character>();
            if (!(character && character.CompareTag(Character.Character.TAG_PLAYER)))
            {
                return;
            }
            mEntrylist[character] = true;

            if (!CanRunningPortal(character))
            {
                return;
            }
            RunPortal();
        }

        protected virtual void OnTriggerExit(Collider other)
        {

            var character = other.GetComponent<Character.Character>();
            if (!(character && character.CompareTag(Character.Character.TAG_PLAYER)))
            {
                return;
            }

            if (mEntrylist.ContainsKey(character))
            {
                mEntrylist[character] = false;
            }

            if (!(State is WorkState.Action))
            {
                return;
            }

            if (mEntrylist.Any(x => x.Value))
            {
                return;
            }


            State = WorkState.Ready;
            mEndEvent.Invoke();
        }

        void Start()
        {
            ResetEntrylist();
        }

    }
```

## Player Controller
  <img src="https://github.com/user-attachments/assets/6789b9d6-829c-47a6-978a-9e97e739e26d" width="39%" height="39%"/>
  <img src="https://github.com/user-attachments/assets/671650a4-cf75-44dd-94e6-34504d9e8c97" width="40%" height="40%"/>

```
컨트롤러를 만들 때 "입력에 따라서 이벤트를 실행시킬 뿐인 기능 아닌가?"하는 생각이 들었습니다.
이런 기능이라면 단순하게 이벤트 핸들러로 만들면 될 것 같았습니다.

이벤트 핸들러로 만든 다음에는 입력값을 어떻게 처리할지 고민이 들었습니다. 방향키등 키값을 문자열로 처리하면 편하긴 하지만
오타가 발생할 수 있으니 디버깅에 어려움을 겪을 것 같았고, 유효성 검사를 위해 커스텀 에디터를 작성하자니 굳이 필요할까라는
생각이 들었습니다.

그러다 enum을 사용하면 제가 생각했던 커스텀 에디터의 기능과 동일한 결과를 가져올 수 있겠다는 생각이들어 아래와 같이 구현했습니다.
```
  ## 코드
``` C#
    [Serializable]
    class ButtonData
    {
        public ActionKey.Button Key;
        public InputType InputType;
        public UnityEvent Action;
    }

    enum InputType
    {
        Down, Up, Stay
    }

    public class PlayerController : MonoBehaviour
    {
        static List<PlayerController> mInstances = new();
        public static List<PlayerController> Instances => mInstances.ToList();

        [Header("Controls")]
        [SerializeField] bool mIsActive;
        [SerializeField] List<ButtonData> ButtonEvents;

        public bool IsActive
        {
            get => mIsActive;
            private set => mIsActive = value;
        }

        public void SetActive(bool able)
        {
            IsActive = able;
        }

        public void ChangeEvent(string key, UnityEvent action)
        {
            var origin = ButtonEvents.FindIndex(x => x.Key.ToString() == key);
            if (origin == -1)
            {
                Debug.Log($"Faild. Not found key : {key}");
                return;
            }
            ButtonEvents[origin].Action = action;
        }

        public UnityEvent GetEvent(string key)
        {
            var origin = ButtonEvents.FindIndex(x => x.Key.ToString() == key);
            Debug.Assert(origin != -1, $"Not found : {key}");
            return ButtonEvents[origin].Action;
        }

        public bool ExitsKey(string key)
        {
            return ButtonEvents.FindIndex(x => x.Key.ToString() == key) != -1;
        }

        void Update()
        {
            if (!IsActive)
            {
                return;
            }

            foreach (var input in ButtonEvents)
            {
                Func<string, bool> inputButton;
                switch (input.InputType)
                {
                    case InputType.Down:
                        inputButton = UnityEngine.Input.GetButtonDown;
                        break;
                    case InputType.Up:
                        inputButton = UnityEngine.Input.GetButtonUp;
                        break;
                    case InputType.Stay:
                        inputButton = UnityEngine.Input.GetButton;
                        break;
                    default:
                        Debug.Assert(false, $"Undefined : {input.InputType}");
                        return;
                }

                if (!inputButton(input.Key.ToString()))
                {
                    continue;
                }

                input.Action.Invoke();
            }
        }

        void Awake()
        {
            mInstances.Add(this);
        }

        void OnDestroy()
        {
            mInstances.Remove(this);
        }
    }
```

## Hitbox
  <img src="https://github.com/user-attachments/assets/17d7de4d-de59-4978-bf5d-991b48082fbe" width="34%" height="34%"/>
  <img src="https://github.com/user-attachments/assets/3f5f9504-3667-4953-b74e-1eca1d496e6d" width="30%" height="30%"/>

```
여러 게임들을 분석하며 공격을 어떻게 처리하고 있는지 고민했습니다. 공격이나 피격에 딜레이가 있거나, 무적, 무적 무시, 타수 제한 등
다양한 변형이 있었습니다. 이러한 확장에 대해서는 이벤트를 통해 열어주고, 히트박스 내에서는 충돌에 대해서만 역할을 제한했습니다.

히트박스 끼리 충돌할 때 공격과 피격의 구분이 필요했는데, 플래그 변수를 주는 대신 상속을 통해 해결했습니다. 충돌은 빈번하게 처리되는
만큼 if문 하나라도 줄이고자 했습니다.

충돌 시의 행동은 부모 클래스인 CollisionBox에 위임했습니다. 이벤트를 발동하는 InvokeEvent를 캡슐화하고 단일책임의 원칙에 따라
Attack Box는 충돌만을 담당하게 하고 싶었기 때문입니다.
```
  ## 코드 (AttackBox)
``` C#
    public class AttackBox : CollisionBox
    {
        Delay mDelay;
        List<Collider> mAttacked = new List<Collider>();
        bool mbNotWithinAttackWindow => !mDelay.IsDelay();
        public AttackBox(Transform actor, float attackWindow = 0f) : base(actor)
        {
            mDelay = new Delay(attackWindow);
            mDelay.StartTime = -1f;
        }

        public void CheckCollision(Collider other)
        {
            if (mbNotWithinAttackWindow ||
                !other.TryGetComponent<IHitBox>(out var victim) ||
                victim.HitBox.Actor.Equals(Actor) ||
                mAttacked.Contains(other))
            {
                return;
            }

            mAttacked.Add(other);
            CollisionBox.InvokeCollision(this, victim.HitBox);
        }

        public void OpenAttackWindow()
        {
            mDelay.SetStartTime();
            mAttacked.Clear();
        }

    }
```

  ## 코드 (CollisoinBox)
``` C#
    public class CollisionBox
    {
        public Transform Actor
        {
            get; private set;
        }
        public event Action<HitBoxCollision> OnCollision;

        public CollisionBox(Transform actor)
        {
            Actor = actor;
        }

        protected void InvokeEvent(HitBoxCollision collision)
        {
            OnCollision.Invoke(collision);
        }

        protected static void InvokeCollision(CollisionBox attack, CollisionBox hit)
        {
            var collsion = new HitBoxCollision()
            {
                Attacker = attack.Actor,
                Victim = hit.Actor,
            };
            attack.InvokeEvent(collsion);
            hit.InvokeEvent(collsion);
        }
    }
```

>## Timer
  <img src="https://github.com/user-attachments/assets/ba8994fd-3bc8-4136-81f4-bcdb12c95ead" width="50%" height="50%"/>
  <img src="https://github.com/user-attachments/assets/942c2f2d-4986-47b1-9290-b3d2b72190da" width="36%" height="36%"/>

```
스톱워치처럼 시간에 대한 이벤트들로 구성된 타이머입니다. 에디터에서 주로 연출이나 처리에 딜레이가 필요한 경우에 사용했습니다.
타이틀에서도 캐릭터가 서 있다가 큐브가 회전할 때 뛰는 연출을 타이머로 구현했습니다.

타이머를 코드에서도 사용할 수 있도록 Timer class와 Timer Componenet로 나누어 구현했습니니다.
Timer Component는 Timer class의 이벤트를 핸들링하는 기능을 합니다.

Timer의 Timeout 이벤트에 Timer를 시작시키는 Start이벤트를 주입하면 반복타이머로 작동하기 때문에 별도의 코드를 작성하지
않았습니다.
```
  ## 코드
``` C#
    public class Timer
    {
        public event Action<Timer> OnStartEvent;
        public event Action<Timer> OnStopEvent;
        public event Action<Timer> OnPauseEvent;
        public event Action<Timer> OnResumeEvent;
        public event Action<Timer> OnTimeoutEvent;
        public event Action<Timer> OnTickEvent;

        public bool IsPause { get; private set; }

        public bool IsStart { get; private set; }

        public float Timeout { get; private set; }

        public float ElapsedTime { get; private set; }

        public float LastPauseTime { get; private set; }

        private float LastTickTime { get; set; }

        static float ServerTime => Server.ServerTime.Time;

        public void Start()
        {
            if (IsStart)
            {
                return;
            }

            IsStart = true;
            IsPause = false;
            ElapsedTime = 0f;
            LastTickTime = ServerTime;
            OnStartEvent?.Invoke(this);
        }

        public void Stop()
        {
            if (IsStart == false)
            {
                return;
            }

            IsStart = false;
            OnStopEvent?.Invoke(this);
        }

        public void Pause()
        {
            if (IsPause)
            {
                return;
            }

            IsPause = true;
            LastPauseTime = ServerTime;
            OnPauseEvent?.Invoke(this);
        }

        public void Resume()
        {
            if (!IsPause)
            {
                return;
            }

            IsPause = false;
            OnResumeEvent?.Invoke(this);
        }

        public void SetTimeout(float timeout)
        {
            Timeout = timeout;
        }

        public void RemoveTimeout()
        {
            Timeout = 0f;
        }

        public void Tick()
        {
            if (!IsStart || IsPause)
            {
                return;
            }

            ElapsedTime += (ServerTime - LastTickTime) - Mathf.Max((LastPauseTime - LastTickTime), 0);
            LastTickTime = ServerTime;
            OnTickEvent?.Invoke(this);

            if (Timeout == 0)
            {
                return;
            }

            if (Timeout < ElapsedTime)
            {
                DoTimeout();
            }
        }

        void DoTimeout()
        {
            IsStart = false;
            OnTimeoutEvent?.Invoke(this);
        }

    }
```
>## 스크립터블 오브젝트
```
스크립터블 오브젝트의 장점은 메모리에 데이터 사본을 하나만 저장한다는 것입니다.
하나의 자산을 공유할 수 있다는 장점이 static 메서드, 중복되는 컴포넌트등의 코드에도 적용될 수 있지
않을까 생각해봤습니다.

다양한 시도 끝에 스크립터블 오브젝트를 유니티 이벤트와 같이 사용해 시너지를 내는 방식의 작업 스타일을 찾아낼
수 있었습니다.

이 방식은 디버깅에서도 편리함을 제공했고 불필요한 컴포넌트를 붙이지 않아도 되어 만족스러웠습니다.
특히 유니티에서도 언리얼의 블루프린트와 같은 비주얼 스크립팅의 이점을 가질 수 있어 협업 능력 향상에 도움이
되었습니다. 
```

## Debug Log
  <img src="https://github.com/user-attachments/assets/5df5bc69-15b1-4dd8-96a8-f3c916f3dc96" width="30%" height="30%"/>
  <img src="https://github.com/user-attachments/assets/db12a7f5-f279-4166-a5a7-a78b28a53e53" width="30%" height="30%"/>

```
디버깅을 하다 보면 이 코드가 제대로 작동 하고 있는지 의문이 들 때가 있습니다.
그럴 때마다 스크립트에서 Debug.Log를 넣어 확인하고, 나중에 제거하는 식의 번거로운 과정을
거쳤습니다.

이런 번거로운 과정 없이 확인할 수는 없을지 고민하다 스크립터블 오브젝트를 사용하게 되었습니다.
의심되는 부분의 이벤트에 스크립터블 오브젝트를 넣었다 빼기만 하면 되어서 컴파일 과정을 단축할 수 있었습니다.
```
  ## 코드
``` C#
    [CreateAssetMenu(menuName = "Custom/Log")]
    public class DebugLog : ScriptableObject
    {
        public static void PrintLog(string text)
        {
            Debug.Log(text);
        }
    }
```

## Player Character Manager
  <img src="https://github.com/user-attachments/assets/751dea03-6786-4682-a558-691880067f7c" width="40%" height="40%"/>

```
플레이어가 조작하고 있는 캐릭터는 다른 기능들에서 필요로 하는 경우가 많았습니다.
스크립터블 오브젝트가 가진 하나의 자산을 공유할 수 있다는 장점을 사용할 때라고 생각했습니다.

플레이어가 조종할 캐릭터를 선택할 수 있는 기능,
조종을 중지하거나 재시작 할 수 있는 기능등을 스크립터블 오브젝트로 만들어서 기능을 공유할 수 있도록 했습니다.
```
  ## 코드
``` C#
    [CreateAssetMenu(menuName = "Custom/PlayerCharacterManager")]
    public class PlayerCharacterManager : UniqueScriptableObject<PlayerCharacterManager>
    {
        public List<Character> JoinCharacters
        {
            get => Character.Instances.Where(x => x.CompareTag(TAG_PLAYER)).ToList();
        }
        public List<PlayerController> JoinCharactersController
        {
            get
            {
                var playerControllers = PlayerController.Instances.Where(x => x.CompareTag(TAG_PLAYER)).ToList();
                if (playerControllers.Count == 0)
                {

                    playerControllers = FindObjectsOfType<PlayerController>().Where(x => x.CompareTag(TAG_PLAYER)).ToList();
                }

                if (playerControllers.Count == 0)
                {
                    Debug.Log($"No controllers with the {TAG_PLAYER} tag found.");
                }
                return playerControllers;
            }
        }
        public Character ControlledCharacter
        {
            get => mCurrentController.GetComponentInParent<Character>();
        }
        PlayerController mCurrentController;
        PlayerController mDefaultController;
        public void ControlDefaultCharacter()
        {
            if (JoinCharactersController.Count == 0)
            {
                return;
            }
            if (mDefaultController == null)
            {
                JoinCharactersController.ForEach(x => x.SetActive(false));
                mDefaultController = JoinCharactersController.First();
            }

            ReplaceControlWith(mDefaultController);
        }

        public void ReplaceControlWith(PlayerController controller)
        {
            mCurrentController?.SetActive(false);
            mCurrentController = controller;
            mCurrentController.SetActive(true);
        }

        public void SetDefaultCharacter(PlayerController controller)
        {
            Debug.Assert(controller.tag == TAG_PLAYER);
            mDefaultController = controller;
        }

        public void ReleaseController()
        {
            mCurrentController?.SetActive(false);
            mCurrentController = null;
        }

        public void SetAnimator(RuntimeAnimatorController controller)
        {
            ControlledCharacter.Animator.runtimeAnimatorController = controller;
            Debug.Log(ControlledCharacter.name);
        }

    }
```

## Shy Box Manager
  <img src="https://github.com/user-attachments/assets/0a438902-ea4a-4276-91ae-58ffb85dba16" width="40%" height="40%"/>
  <img src="https://github.com/user-attachments/assets/333545d9-0ee2-4867-be8b-bc78d15f4b15" width="40%" height="40%"/>

```
간수의 시야에 n초 이상 발각되면 쫓겨나는 규칙의 퍼즐을 구현할 때에도 스크립터블 오브젝트를 사용했습니다.

이때는 팀원이 레벨 배치에 어려움을 겪는 것 같아 어떻게 하면 편하게 레벨을 배치할 수 있도록 기능을 제공할 수 있을까
고민하게 되었습니다.

가능하면 참조 작업 없을 없애 프리펩을 배치하는 것 만으로 끝날 수 있도록 해야겠다고 생각하게 되어 모듈화를 우선으로 작업을
진행했습니다. 이 과정에 스크립터블 오브젝트가 큰 도움이 되었습니다.

'Shy Box'라고 하는 부끄럼쟁이 큐브가 간수에게 발각되는 기능을 구현할 때 Box Trigger를 사용해서 구현하는 방식을 택했습니다.
때문에 트리거 이벤트 핸들러로부터 'Trigger Enter' 이벤트 때 Shy Box의 특정 기능을 발동시켜야 했습니다. Box Trigger는 간수가
가지고 있고 기능은 Shy Box가 가지고 있어 커플링이 생겼고 모듈화를 위해서 커플링을 깨야 하는 상황이었습니다.

이런 문제 상황에서 커플링을 깨기 위해 스크립터블 오브젝트를 사용해서 Trigger Enter 이벤트로부터 Collider 정보를 핸들링하여
Shy Box에 접근할 수 있게 하여 해결했습니다.
```
  ## 코드 (Shy Box Manager)
``` C#
    [CreateAssetMenu(menuName ="Custom/Hitbox/ShyBoxManager")]
    public class ShyBoxManager : UniqueScriptableObject<ShyBoxManager>
    {
        public void StartShowing(Collider collider)
        {
            var box = collider.GetComponent<ShyBox>();
            if(box == null)
            {
                return;
            }

            box.StartShowing();
        }

        public void HideBegin(Collider collider)
        {
            var box = collider.GetComponent<ShyBox>();
            if (box == null)
            {
                return;
            }

            box.HideBegin();
        }

        public void HideEnd(Collider collider)
        {
            var box = collider.GetComponent<ShyBox>();
            if (box == null)
            {
                return;
            }

            box.HideEnd();
        }

        public void ResetTrigger(Collider collider)
        {
            collider.enabled = false;
            collider.enabled = true;

        }

    }
```

>## 정형화

```
이벤트 핸들러는 프로토타입을 만들 때 편리합니다. 하지만 유지보수에 있어서는 단점이 있습니다.
참조에 문제가 생기는 경우도 빈번했고 오류가 발생했을 때 씬에 있는 수많은 오브젝트들을 확인하며
어디서 오류가 발생했는지 찾아야 하는 어려움을 겪었습니다.

이 부분을 해결하려면 에디터에서의 참조 연결을 최소화해야 했습니다. 기획자와 협의해 변하지 않을
부분을 정리하고, 이러한 내용을 코드로 정형화시켰습니다.

결과적으로 에디터에서는 컴포넌트를 붙이기만 하면 기대했던 대로 작동하는 정형화된 컴포넌트들이
만들어졌습니다.
```

## Shy Box
  <img src="https://github.com/user-attachments/assets/225e2e00-f532-4624-8a49-8d84f90c22f7" width="37%" height="37%"/>
  <img src="https://github.com/user-attachments/assets/8f89122c-837b-48d4-958e-19bb630a11c9" width="40%" height="40%"/>
  
```
Shy Box는 부끄럼쟁이라는 특징을 가지고 있습니다. 누군가한테 오랫동안 보이게되면 도망가 버립니다.

이 기획 내용을 두 가지 규칙으로 정의했습니다.
1. 긴장한 시간을 측정하는 Tense 타이머의 시간이 임계치를 넘으면 이벤트를 발동한다.
2. 진정한 시간을 측정하는 CalmDown 타이머의 시간이 충분해지면 긴장도가 0이 되어 Tense 타이머의 시간이 초기화된다.

이 기능을 만들 당시에는 아직 프로토타입에서 벗어나지 않았고 이벤트가 변할 수 있으면 좋겠다는 기획자의
의견이 있어 Shy Box가 발동시키는 이벤트를 인스펙터에서 지정할 수 있도록 만들었습니다.
```
  ## 코드
``` C#
    public class ShyBox : MonoBehaviour
    {
        Timer mTenseTimer = new();
        Timer mCalmDownTimer = new();
        [SerializeField, Range(1, 100)] int ThresholdsTime;
        [SerializeField, Range(1, 100)] int CalmDownTime;
        [SerializeField] UnityEvent ThresholdsEvent;

        public void StartShowing()
        {
            if (!mTenseTimer.IsStart)
            {
                mTenseTimer.Start();
                return;
            }

            if (CalmDownTime <= mCalmDownTimer.ElapsedTime)
            {
                mTenseTimer.Stop();
                mTenseTimer.Start();
            }
            else
            {
                mTenseTimer.Resume();
            }
        }

        public void HideBegin()
        {
            mTenseTimer.Pause();
            mCalmDownTimer.Start();
        }

        public void HideEnd()
        {
            mCalmDownTimer.Stop();
        }

        void Awake()
        {
            mTenseTimer.SetTimeout(ThresholdsTime);
            mTenseTimer.OnTimeoutEvent += (t) => { ThresholdsEvent.Invoke(); };

            mCalmDownTimer.SetTimeout(CalmDownTime);
        }

        void Update()
        {
            mTenseTimer.Tick();
            mCalmDownTimer.Tick();
        }

    }
```

## Flower
  <img src="https://github.com/user-attachments/assets/02485202-0d68-4b35-824e-351f57e9ac52" width="37%" height="37%"/>
  <img src="https://github.com/user-attachments/assets/3233c6cb-2ede-42c6-9e86-28074709d028" width="40%" height="40%"/>

```
기획자로부터 "스테이지 내의 모든 꽃이 같은 색이 되면 클리어되는 퍼즐을 만들어주세요."라는 기획 내용을 전달받았습니다.

레벨 배치가 편해지려면 꽃을 정형화하는게 좋겠다는 생각이 들었습니다. 정형화를 위해서는 꽃이 가진 규칙과 구현 내용이
명확해야 했기 때문에 기획자와 회의를 많이 거쳤습니다. 기획자가 제시한 퍼즐의 규칙끼리 모순이 있어서 같이 고민하기도 하고
구현 내용을 구체화하기 위해서 다양한 레퍼런스를 찾아가며 이야기했습니다.

최종적으로 나온 내용에대해 아래와 같이 정형화했습니다.
 - 꽃은 색이 바뀐다.
 - 퍼즐의 클리어 조건은 꽃의 색상 코드끼리 비교해서 모두 일치는지 확인한다.
 - 꽃의 색이 바뀔 때의 연출은 변경될 수 있다.
 - 하나의 스테이지 내에서 꽃의 색이 일치되는 순간 어떤 경우에도 클리어 이벤트가 진행된다.
```

  ## 코드
  ``` C#
    public class Flower : MonoBehaviour, IColorPiece
    {
        static List<Flower> mInstances = new();
        [SerializeField] Color mColor;

        [SerializeField] List<MeshRenderer> mRenderers;
        public List<MeshRenderer> Renderers => mRenderers;
        [SerializeField] UnityEvent mChangeColorEvent;

        public Color Color
        {
            get => mColor;
            set
            {
                mColor = value;
                CheckClear();
                Colorize.Instance.Invoke(Renderers, Color);
                mChangeColorEvent.Invoke();
            }
        }

        static void CheckClear()
        {
            var color = mInstances.First().Color;
            var bClear = mInstances.All(x => x.Color.Equals(color));

            if (bClear)
            {
                PuzzleClear.Instance.InvokeClearEvent();
            }
        }

        void Awake()
        {
            mInstances.Add(this);
            var material = new Material(mRenderers.First().material);
            foreach (var renderer in mRenderers)
            {
                renderer.material = material;
            }
        }

        void OnDestroy()
        {
            mInstances.Remove(this);
        }
    }
```

## Contents Loader
  <img src="https://github.com/user-attachments/assets/2762228f-2808-4751-8dfd-18146e8a5255" width="40%" height="40%"/>
  <img src="https://github.com/user-attachments/assets/90089d0a-3f04-40ca-aa7e-7e8063ffaab1" width="40%" height="40%"/>

```
특정 콘텐츠를 불러 오거나, 다음 씬을 불러오는 등의 기능을 커멘트 패턴을 사용해 정형화했습니다.
특정 대상을 로드하고 싶을 때는 Load Manager를 통해서 Load 메서드를 호출하기만 하면 됩니다.

기획에 따라서 더 많은 로드 타입을 추가할 수도 있습니다. 혹은 로드를 시작할 때와 완료되었을 때의
이벤트를 추가할 수도 있습니다.
```

  ## 코드
``` C#
    public enum LoaderType
    {
        StageLoader,
        CubeLoader,
        LevelLoader
    }

    public class ContentsLoader : Singleton<ContentsLoader>
    {
        public WorkState State => mLoader.State;

        ILevelLoader mLoader = new LevelLoader();
        List<ILevelLoader> mLoaders = new()
        {
            new StageLoader(),
            null,
            new LevelLoader(),
        };
        [SerializeField] UnityEvent OnStartLoad;
        [SerializeField] UnityEvent OnLoaded;

        public void LoadContents()
        {
            if (!(State is WorkState.Ready))
            {
                Debug.LogWarning($"The loader is not ready : {State}");
                return;
            }
            mLoader.LoadNext();
            OnStartLoad.Invoke();
            StartCoroutine(CheckLoad());
        }

        public void SetLoaderType(LoaderType type)
        {
            Debug.Assert(Enum.IsDefined(typeof(LoaderType), type),$"Out of range : {(int)type}");
            switch (type)
            {
                case LoaderType.CubeLoader:
                    mLoader = FindAnyObjectByType<CubeLoader>();
                    break;
                default:
                    mLoader = mLoaders[(int)type];
                    break;
            }
            Debug.Assert(mLoader != null);
        }

        public void AddOnStartLoadEvent(UnityAction action)
        {
            OnStartLoad.AddListener(action);
        }

        public void RemoveOnStartLoadEvent(UnityAction action)
        {
            OnStartLoad.RemoveListener(action);
        }

        public void AddOnLoadedEvent(UnityAction action)
        {
            OnLoaded.AddListener(action);
        }

        public void RemoveOnLoadedEvent(UnityAction action)
        {
            OnLoaded.RemoveListener(action);
        }

        protected override void Awake()
        {
            base.Awake();
            DontDestroyOnLoad(gameObject);
        }

        IEnumerator CheckLoad()
        {
            WaitForSeconds mWait = new WaitForSeconds(0.5f);
            while (State != WorkState.Ready)
            {
                yield return mWait;
            }

            OnLoaded.Invoke();
        }
    }
```
