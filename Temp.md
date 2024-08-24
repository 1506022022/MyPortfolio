<p align="right">  
  <a href="https://youtu.be/IA1g5FpPwRc">
    <img src="https://github.com/user-attachments/assets/e4766513-99c9-4bac-a200-d9cae57518e8" width="70%" height="70%">
  </a>
</p>

># 목차
- **[이벤트 핸들러](#이벤트-핸들러)**
  - **[IF](#IF)**
  - **[Condition](#Condition)**
  - **[Portal](#Portal)**
  - **[Player Controller](#Player-Controller)**
  - **[Hitbox](#Hitbox)**
  - **[Timer](#Timer)**
- **[이벤트 체인](#이벤트-체인)**
  - **[HitEventChain](#HitEventChain)**
- **[Scriptable Obejct](Scriptable-Obejct)**
  - **[Debug Log](Debug-Log)**
  - **[Player Character Manager](Player-Character-Manager)**
- **[정형화](#정형화)**
  - **[Character](Character)**
  - **[Load Manager](Load-Manager)**

># 프로젝트 구성
|개요|내용|
|---|---|
|장르|3D 퍼즐 액션|
|개발기간|2024.05.01 - (진행중)|
|참여인원|5인|
|개발환경|유니티|
|깃허브 링크|<a href="https://github.com/CREDOCsGames/cute_minecraft">프로젝트 링크</a>|

>## 이벤트 핸들러
```
가장 기본적인 이벤트 핸들러로는 오브젝트끼리 부딪혔을 때 이벤트를 발동시키는 트리거 이벤트 핸들러가 있습니다.
이벤트 핸들러는 다음과 같은 상황에 유용하게 사용할 수 있었습니다.
- 프로토타입의 제작
- 단순한 기능 구현
- 협업

이벤트 핸들러는 프로그래밍에 대한 수준이 높지 않은 사람도 에디터에서 작업이 가능합니다.
또한 조건에 따라 행동이 달라지는 경우에 복잡한 switch문을 대체할 수 있다는 점에서 코드를 간결하게 만드는 이점이 있습니다.

"이러한 이점을 살리면 코드를 에디터에서 작성할 수 있지 않을까?"하는 생각에서 다음의 유형을 컴포넌트로 만들어 봤습니다.
```

## IF
  <img src="https://github.com/user-attachments/assets/fc35f875-77b4-47e7-9ab6-5aea4b042805" width="40%" height="40%"/>
  <img src="https://github.com/user-attachments/assets/a012ff72-e283-4cf7-b11f-7c5e43e2150a" width="40%" height="40%"/>

```
퍼즐 중에 화로에 불이 붙으면 해결되는 퍼즐을 만들었습니다.
이렇듯 조건(불을 붙이면)을 만족하면 이벤트(퍼즐이 해결된다)를 실행하도록 IF 컴포넌트를 통해서 구현할 수 있습니다.

반면에 IF처럼 참일 때 이벤트를 실행하지만 거짓일 때도 이벤트를 실행하는 Branch가 있습니다.
불이 꺼져있을때 공격받으면 불이 켜지고, 불이 켜져있을땐 꺼지는 화로는 Branch 컴포넌트를 통해서 구현합니다.

조건문을 이벤트 핸들러로 만든덕에 시간이 남는 팀원이 간단한 부분을 대신 만들어 주는 등 능률이 좋아졌습니다.
혹은 기획이 바뀌어 화로에 불을 붙이면 클리어되는 퍼즐의 조건이 바뀌어도 유연하게 대처할 수 있었습니다.
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
  <img src="https://github.com/user-attachments/assets/dc70fa60-1a40-47ca-a118-d6cf013eae03" width="40%" height="40%"/>

```
하나의 조건만 확인하는 bool 이외에도 여러 조건을 확인하는 경우도 있습니다.
- 모든 조건을 만족해야 참이다. (AND)
- 하나의 조건만 만족해도 참이다. (OR)

또는 조건끼리 서로 연결되어있는 경우도 있습니다.
- A가 참이면 B도 참이다.
- A가 참이고 B가 거짓이어야 C가 참이다.

이러한 경우를 BOOL, AND, OR 컴포넌트로 구현하였습니다.

조건을 컴포넌트로 만들어 복잡한 규칙도 구현할 수 있게 되었습니다.
특히 이 게임에서 구현한 모든 퍼즐의 프로토타입은 조건과 조건문의 컴포넌트를 사용해서 만들었습니다.

하나의 퍼즐을 구현하는데 1 ~ 2시간 정도로 적게 걸립니다. 이렇게 구현의 속도를 높여주는게
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
  <img src="https://github.com/user-attachments/assets/cdcac5af-2308-426e-a84f-40fd94e9b3ac" width="34%" height="34%"/>
  <img src="https://github.com/user-attachments/assets/3f5f9504-3667-4953-b74e-1eca1d496e6d" width="40%" height="40%"/>

```
공격을 받거나 공격을 하면 이벤트를 발동하는 히트 박스는 굉장히 자주 사용되는 기능인 것 같습니다.
활용도도 높아서 파이프게임을 만들 때 히트박스만으로도 구현할 수 있었습니다.

파이프는 물을 전달받아서 반대 방향으로 보낸다는 특징을 가지고 있는데 이를 히트 박스로 구현해봤습니다.

히트 박스에는 Attacker 옵션이 켜진 공격 히트 박스와 Attacker 옵션이 꺼진 피격 히트 박스가 있는데
이 두가지로 파이프들을 연결시켰습니다.

피격 히트 박스에 충돌이 발생하면 공격 받지 않은 모든 방향의 공격 히트 박스를 잠시 활성화합니다.
이를 반복하면 아래와 같은 흐름이 됩니다.

1번 파이프 왼쪽 피격 히트 박스 충돌 -> 1번 파이프 오른쪽 공격 히트박스 잠깐 활성
-> 2번 파이프 왼쪽 피격 히트 박스 충돌 -> 2번 파이프 오른쪽 공격 히트 박스 잠깐 활성
-> ....

결국 공격을 흘려 보내서 골 지점의 히트 박스를 히트하면 파이프 게임이 클리어됩니다.
```
  ## 코드
``` C#
    public delegate void HitEvent(HitBoxCollision collision);

    public struct HitBoxCollision
    {
        public Transform Victim;
        public Transform Attacker;
        public HitBoxCollider Subject;
    }

    public interface IHitBox
    {
        public Transform Actor { get; set; }
        public bool IsDelay { get; }
        public bool IsAttacker { get; set; }
        public void DoHit(HitBoxCollision collision);
    }

    [RequireComponent(typeof(Collider))]
    [RequireComponent(typeof(Rigidbody))]
    public class HitBoxCollider : MonoBehaviour, IHitBox
    {
        public float HitDelay;
        [SerializeField] bool mbAttacker;
        public bool IsAttacker
        {
            get => mbAttacker;
            set => mbAttacker = value;
        }

        [SerializeField] Transform mActor;
        public Transform Actor
        {
            get => mActor;
            set => mActor = value;
        }
        public bool IsDelay => Time.time < mLastHitTime + HitDelay;
        float mLastHitTime;
        Pipeline<HitBoxCollision> mHitPipeline;
        [SerializeField] UnityEvent<HitBoxCollision> mHitEvent;

        public void StartDelay()
        {
            mLastHitTime = Time.time;
        }

        void InvokeHitEvent(HitBoxCollision collision)
        {
            mHitEvent.Invoke(collision);
        }

        void SendCollisionData(IHitBox victim)
        {
            var attacker = this;
            var collsion = new HitBoxCollision()
            {
                Attacker = attacker.Actor,
                Victim = victim.Actor,
            };
            victim.DoHit(collsion);
            attacker.DoHit(collsion);
        }

        public void DoHit(HitBoxCollision collision)
        {
            StartDelay();
            collision.Subject = this;
            mHitPipeline.Invoke(collision);
        }

        bool CanAttack(IHitBox targetHitBox)
        {
            return IsAttacker &&
                   !targetHitBox.IsDelay &&
                   !targetHitBox.IsAttacker &&
                   !Actor.Equals(targetHitBox.Actor);
        }

        void OnTriggerStay(Collider other)
        {
            if (!IsAttacker)
            {
                return;
            }

            var victim = other.GetComponent<IHitBox>();
            if (victim == null)
            {
                return;
            }

            if (!CanAttack(victim))
            {
                return;
            }

            SendCollisionData(victim);
        }


        void Start()
        {
            Debug.Assert(Actor, $"Actor not found : {gameObject.name}");
            Debug.Assert(GetComponents<Collider>().Any(x => x.isTrigger), $"Trigger not found : {gameObject.name}");
            Debug.Assert(GetComponent<Rigidbody>().isKinematic, $"Not set Kinematic : {gameObject.name}");
        }

        void Awake()
        {
            mLastHitTime = Time.time - HitDelay + 0.1f;
            mHitPipeline = Pipelines.Instance.HitBoxColliderPipeline;
            mHitPipeline.InsertPipe(InvokeHitEvent);
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

>## 제목
  <img src="" width="40%" height="40%"/>

```

```
  ## 코드
``` C#

```
