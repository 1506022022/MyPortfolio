<p align="right">  
  <a href="https://youtu.be/sdgQF_41lS4">
    <img src="https://encrypted-tbn2.gstatic.com/images?q=tbn:ANd9GcTcOq7iACKolwWWrCgWPU0DBBCeK1l94kghVAAMEblsJxgXS8E3" width="40%" height="40%">
  </a>
</p>

># 목차
- **[파이프 라인](#파이프-라인)**
  - **[플로우](#플로우)**
  - **[히트 박스 파이프라인](#히트-박스-파이프라인)**
  - **[어빌리티 파이프라인](#어빌리티-파이프라인)**
- **[포메이션](#포메이션)**
  - **[Role](#Role)**
- **[타이머](#타이머)**
- **[애니메이션](#애니메이션)**
- **[어빌리티](#어빌리티)**
  - **[리버스 어빌리티](#리버스-어빌리티)**   
  - **[Burn](#Burn)**   
  - **[파괴](#파괴)**   
  - **[밀어내는 힘](#밀어내는-힘)**   
  - **[위성](#위성)**   
- **[히트 박스](#히트-박스)**   
  - **[히트 박스 어빌리티](#히트-박스-어빌리티)**   
- **[버튼 이벤트 액션](#버튼-이벤트-액션)**   
  - **[블록 트리거](#블록-트리거)**   
  - **[액션 버튼](#액션-버튼)**   
  - **[배치](#배치)**
- **[특성](#특성)**
  - **[Non static](#Non-Static)**
  - **[Distructibility](#Distructibility)**
  - **[Premeability](#Premeability)**
- **[크래프팅](#크래프팅)**
  - **[Item](#Item)**

># 프로젝트 구성
|개요|내용|
|---|---|
|장르|3D 퍼즐 액션|
|개발기간|2024.05.01 - (진행중)|
|참여인원|3인|
|개발환경|유니티|
|프로젝트 링크|<a href="https://github.com/CREDOCsGames/cute_minecraft">프로젝트 링크</a>|



># 파이프 라인
```
프로젝트를 시작하며 가장 먼저 프레임워크를 어떻게 설계할지 고민했습니다.

처음에는 모듈화에 집중해서 트리 구조의 프레임워크를 설계했습니다.
하위 노드에 해당하는 클래스가 부모 노드의 정보를 모르게 하여 커플링을
줄이도록 한 의도는 달성했지만 트리 구조에서의 어려움을 겪었습니다.

클래스의 상속처럼 파고 들어가는 구조이다 보니 개발을 진행할 수록
부모 자식의 관리가 어려워 질 수도 있겠다는 느낌을 받았습니다.

해당 문제를 개선하기 위해 파이프라인들의 협력 구조로 프레임워크를 설계했습니다.

- 캐릭터 간의 충돌을 관리하는 히트 박스 파이프라인
- 캐릭터의 능력 발동을 관리하는 어빌리티 파이프라인
- 컨트롤러의 입력을 관리하는 컨트롤러 파이프라인
이 세개의 파이프라인이 서로 협력하며 퍼즐 액션 시스템을 구현합니다.
```
  ## 플로우
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/20250abe-9da2-4b5e-9266-21884dd67679" width="50%" height="50%"/>

```
히트 박스가 충돌하면 히트 박스 파이프라인이 실행됩니다.
히트 박스 파이프라인은 어빌리티 파이프라인, 타격 이펙트를 호출합니다.

어빌리티 파이프라인이 실행됩니다.
어빌리티 파이프라인은 어빌리티, 어택 로그를 호출합니다.

파이프라인은 확장 가능한 시스템입니다.
예를 들어 어빌리티 파이프라인에 데미지를 추가할 수도 있습니다.

# 어택 로그 : "A가 B를 C기술로 공격하였다."와 같이 정보를 단기간 저장합니다. 
```
  ## 히트 박스 파이프라인
  <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/73c83c0d-4d9c-4702-b40f-51ec1700a394" width="30%" height="30%"/>
  
```
히트 박스 파이프라인은 행위의 주체에 대한 정보를 전달받습니다.

예를 들어 피격당한 캐릭터에게 피격 이펙트를 출력하는 기능을 추가하고 싶다면,
전달받은 행위의 주체 정보 중 피격의 주체 정보를 통해서 해당 기능을 구현할 수 있습니다.  
```
  ## 코드
``` C#
using PlatformGame.Pipeline;
using System.Linq;
using UnityEngine;
using UnityEngine.Events;

namespace PlatformGame.Character.Collision
{
    public delegate void HitEvent(HitBoxCollision collision);

    public struct HitBoxCollision
    {
        public Character Victim;
        public Character Attacker;
        public HitBoxCollider Subject;
    }

    public interface IHitBox
    {
        public Character Actor { get; set; }
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

        [SerializeField] Character mActor;
        public Character Actor
        {
            get => mActor;
            set => mActor = value;
        }
        [HideInInspector] public UnityEvent<HitBoxCollision> HitCallback;
        public bool IsDelay => Time.time < mLastHitTime + HitDelay;
        float mLastHitTime;
        HitEvent mAbilityEvent;
        Pipeline<HitBoxCollision> mHitPipeline;
        [SerializeField] UnityEvent<HitBoxCollision> mEffectEvent;

        public void StartDelay()
        {
            mLastHitTime = Time.time;
        }

        public void SetAbilityEvent(HitEvent hitEvent)
        {
            mAbilityEvent = hitEvent;
        }

        void InvokeAbilityEvent(HitBoxCollision collision)
        {
            mAbilityEvent?.Invoke(collision);
        }

        void InvokeEffectEvent(HitBoxCollision collision)
        {
            mEffectEvent.Invoke(collision);
        }

        void InvokeHitCallback(HitBoxCollision collision)
        {
            HitCallback?.Invoke(collision);
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
            mHitPipeline.InsertPipe(InvokeEffectEvent);
            mHitPipeline.InsertPipe(InvokeAbilityEvent);
        }
        
    }
}
``` 
  ## 어빌리티 파이프라인
   <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/92bf9e92-fed7-4328-90f4-3e3f12c7da6c" width="40%" height="40%"/>

```
어빌리티 파이프라인은 행위의 주체와 어빌리티에 대한 정보를 전달받습니다.

예를 들어 어택 로그는 어빌리티를 발동한 캐스터와 피격의 주체,
그리고 발동된 어빌리티에 대한 정보를 전달받아 기록합니다.
```
  ## 코드
``` C#
using PlatformGame.Character.Collision;
using PlatformGame.Pipeline;
using UnityEngine;

namespace PlatformGame.Character.Combat
{
    public abstract class Ability : ScriptableObject
    {
        Pipeline<AbilityCollision> mPipeline;

        public void DoActivation(HitBoxCollision collision)
        {
            CreatePipeline();
            var caster = collision.Subject.Actor;
            var victim = collision.Victim == caster ? collision.Attacker : collision.Victim;
            var abilityCollision = new AbilityCollision(caster, victim, this);
            mPipeline.Invoke(abilityCollision);
        }

        public abstract void UseAbility(AbilityCollision collision);

        void CreatePipeline()
        {
            mPipeline = Pipelines.Instance.AbilityPipeline;
            mPipeline.InsertPipe((collision) => UseAbility(collision));
        }
    }
}
```
># **포메이션**
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/94d0e3cf-cdc5-4b5c-b951-a686f0e658b9" width="30%" height="30%"/>

```
 포메이션을 설계하면서 축구에 빗대어 고민했습니다. 감독의 지시에 따라 포메이션 자체가 바뀔 수도 있고
수비수, 공격수 등 역할에 따라 다르게 행동하는 것이 자연스러웠습니다. 이런 구조를 유연하게 구현하기 위해
감독의 의지와 플레이어의 행동으로 구분 지어 생각해 봤습니다.

 포메이션에서는 의지만을 나타내기 위해 행동을 구현하지 않도록 고민했고, 이벤트를 활용해서 Role 클래스에
행동을 위임했습니다. 플레이어가 감독이 지시한 위치에 도달했는지, 서 있는 상태인지 판단하는 부분이
IsReached, IsStoped인데 판단도 하나의 행동이라고 생각해 Role 클래스에 위임할 수 있도록 대리자를
사용했습니다.

 대부분의 행동을 Role 클래스에 위임했지만, 플레이어가 달리기 시작한다, 멈춘다, 도착했다, 포지션이
바뀌었다는 것과 같은 상황에 대해서는 작업의 편리성을 위해 UnityEvent를 통해 인스펙터에서 조작할 수 있도록
Serializable 특성을 통해 직렬화했습니다.
```
  ## 코드
``` C#
    [Serializable]
    public class Formation
    {
        Transform mPosition;
        public Transform Position
        {
            get => mPosition;
            set
            {
                mPosition = value;
                OnChangeFormation();
            }
        }
        public UnityAction Trace;
        public Func<bool> IsStoped;
        public Func<bool> IsReached;
        public UnityEvent OnReachFormationEvent;
        public UnityEvent OnChangeFormationEvent;
        public UnityEvent OnStopMoveEvent;
        public UnityEvent OnStartMoveEvent;

        public void UpdateBehaviour()
        {
            if (IsReached() && !IsStoped())
            {
                OnReachFormation();
                return;
            }

            if (IsStoped() && !IsReached())
            {
                OnMoveToFormation();
            }
        }

        void OnReachFormation()
        {
            OnStopMove();
            OnReachFormationEvent.Invoke();
        }

        void OnChangeFormation()
        {
            OnStopMove();
            OnChangeFormationEvent.Invoke();
        }

        void OnStopMove()
        {
            OnStopMoveEvent.Invoke();
        }

        void OnStartMove()
        {
            OnStartMoveEvent.Invoke();
        }

        void OnMoveToFormation()
        {
            Trace.Invoke();
            OnStartMove();
        }

    }
```
  ## Role
```
 Role 클래스는 축구의 플레이어에 빗댈 수 있습니다. 축구에서 플레이어가 이동하는 방식은 사람마다 다를 수
있습니다. 뛰어서 이동하거나 슬라이딩해서 이동할 수도 있습니다. 이렇듯 다양한 이동을 처리하기 위해 움직임을
담당하는 MovementTransform 클래스에 위임했습니다.
```
  ## 코드
``` C#
 public class Role : MonoBehaviour
 {
     bool mbStop;
     [SerializeField] Formation mFormation;
     [SerializeField] MovementTransform mTrace;
     [SerializeField] Transform mPosition;

     void Awake()
     {
         mFormation.Trace = TraceFormation;
         mFormation.Position = mPosition;
         mFormation.IsStoped = () => mbStop;
         mFormation.IsReached = () => IsNearByDistance(transform.position, mPosition.position);
         mFormation.OnStopMoveEvent.AddListener(StopTrace);
         mFormation.OnStartMoveEvent.AddListener(StartTrace);
     }

     void TraceFormation()
     {
         StopAllCoroutines();
         StartCoroutine(mTrace.Move(transform, mPosition, true));
     }

     void StopTrace()
     {
         mbStop = true;
     }

     void StartTrace()
     {
         mbStop = false;
     }

     void Update()
     {
         mFormation.UpdateBehaviour();
     }

     void Start()
     {
         StopTrace();
     }

 }
```

  ## 타이머
<img src="https://github.com/user-attachments/assets/1e6e2390-2ede-4452-86e4-01784c0a71ee" width="30%" height="30%"/>

```
타이머 클래스는 MonoBehaviour를 상속받은 TimerComponent와, Timer class로 구현하였습니다.
처음에는 TimerComponent 하나만 구현하려고 했었는데 아무래도 Timer는 여러 클래스에서 사용하는
기능이다 보니 분리하게 되었습니다.

타이머는 시간을 재는 단순한 기능만을 가지고 있지만, 이벤트의 활용도가 매우 높다고 생각합니다.
때문에 외부에서 필요한 이벤트에 기능을 연결할 수 있도록 설계해 보았습니다.
```
  ## 코드 (TimerComponent)
``` C#
using UnityEngine;
using UnityEngine.Events;

namespace PlatformGame
{

    public class TimerComponent : MonoBehaviour
    {
        public UnityEvent<TimerComponent> OnStartTimer;
        public UnityEvent<TimerComponent> OnStopTimer;
        public UnityEvent<TimerComponent> OnPauseTimer;
        public UnityEvent<TimerComponent> OnResumeTimer;
        public UnityEvent<TimerComponent> OnTick;
        public UnityEvent<TimerComponent> OnTimeout;

        public bool IsStart => mTimer.IsStart;
        public bool IsPause => mTimer.IsPause;
        public float Timeout => mTimer.Timeout;
        public float ElapsedTime => mTimer.ElapsedTime;
        public float LastPauseTime => mTimer.LastPauseTime;
        public float LastTickTime => mTimer.LastTickTime;

        Timer mTimer = new();
        float mElapsedTime;

        [Header("Options")]
        [SerializeField] float mTimeout;
        [SerializeField] bool mbPlayOnAwake = true;

#if DEVELOPMENT
        [Header("Debug")]
        [SerializeField] bool mUseDebug;
        [SerializeField] string mTimeoutKey;
#endif

        public void Initialize(float maxTimerTim)
        {
            mTimer.SetTimeout(maxTimerTim);
            mTimer.Stop();
        }

        public void StartTimer()
        {
            mTimer.Start();
        }

        public void PauseTimer()
        {
            mTimer.Pause();
        }

        public void ResumeTimer()
        {
            mTimer.Resume();
        }

        public void StopTimer()
        {
            mTimer.Stop();
        }

        void Update()
        {
            mTimer.Tick();
#if DEVELOPMENT
            if (UnityEngine.Input.GetKeyDown(mTimeoutKey))
            {
                DebugTimeout();
            }
#endif
        }

        void Awake()
        {
            mTimer.OnPauseEvent += (t) => OnPauseTimer.Invoke(this);
            mTimer.OnResumeEvent += (t) => OnResumeTimer.Invoke(this);
            mTimer.OnStartEvent += (t) => OnStartTimer.Invoke(this);
            mTimer.OnStopEvent += (t) => OnStopTimer.Invoke(this);
            mTimer.OnTickEvent += (t) => OnTick.Invoke(this);
            mTimer.OnTimeoutEvent += (t) => OnTimeout.Invoke(this);
        }

        void Start()
        {
            if (mbPlayOnAwake)
            {
                StartTimer();
            }
        }

#if DEVELOPMENT
        void DebugTimeout()
        {
            mElapsedTime = mTimeout < 5 ? mElapsedTime
                                                : mTimeout - 5f;
        }
#endif

    }
}

``` 
  ## 코드 (Timer)
``` C#
using System;
using UnityEngine;

namespace PlatformGame
{
    public class Timer
    {
        public event Action<Timer> OnStartEvent;
        public event Action<Timer> OnStopEvent;
        public event Action<Timer> OnPauseEvent;
        public event Action<Timer> OnResumeEvent;
        public event Action<Timer> OnTimeoutEvent;
        public event Action<Timer> OnTickEvent;

        bool mbPause;
        public bool IsPause => mbPause;

        bool mbStart;
        public bool IsStart => mbStart;

        float mTimeout;
        public float Timeout => mTimeout;

        float mLastTickTime;
        public float LastTickTime => mLastTickTime;

        float mLastPauseTime;
        public float LastPauseTime => mLastPauseTime;

        float mElapsedTime;
        public float ElapsedTime => mElapsedTime;

        public void Start()
        {
            if (mbStart)
            {
                return;
            }

            mbStart = true;
            mbPause = false;
            mElapsedTime = 0f;
            mLastPauseTime = 0f;
            mLastTickTime = Time.time;
            OnStartEvent.Invoke(this);
        }

        public void Stop()
        {
            if (mbStart == false)
            {
                return;
            }

            mbStart = false;
            OnStopEvent.Invoke(this);
        }

        public void Pause()
        {
            if (mbPause)
            {
                return;
            }

            mbPause = true;
            mLastPauseTime = Time.time;
            OnPauseEvent.Invoke(this);
        }

        public void Resume()
        {
            if (!mbPause)
            {
                return;
            }

            mbPause = false;
            OnResumeEvent.Invoke(this);
        }

        public void SetTimeout(float timeout)
        {
            Debug.Assert(0 < timeout, $"The timeout({timeout}) must be greater than 0 seconds.");
            mTimeout = timeout;
        }

        public void RemoveTimeout()
        {
            mTimeout = 0f;
        }

        public void Tick()
        {
            if (!mbStart || mbPause)
            {
                return;
            }

            mElapsedTime += Time.time - mLastTickTime;
            mLastTickTime = Time.time;
            OnTickEvent.Invoke(this);

            if (Timeout == 0)
            {
                return;
            }

            if (mTimeout < mElapsedTime)
            {
                DoTimeout();
            }
        }

        void DoTimeout()
        {
            mbStart = false;
            OnTimeoutEvent.Invoke(this);
        }

    }
}
```

  ## 애니메이션
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/b5ddf6cf-6f2b-4741-86fb-8d01ab2bc7c4" width="30%" height="30%"/>
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/928537d2-581a-4ab7-9321-846e1a31bc1c" width="30%" height="30%"/>

```
캐릭터 애니메이션을 애니메이터가 관리할 수 있도록 트리거를 핸들링하는 기능을 만들었습니다.

 처음에는 어떤 애니메이션을 가진 캐릭터더라도 사용 가능한 기능을 만들기 위해 고민했습니다.
캐릭터마다 동작의 개수와 종류가 다르다는 점에 어려움을 겪었습니다. 동작들을 딕셔너리로 관리해서
애니메이션을 실행하는 방식에 대해 고민했지만, 기획자와 의논 끝에 동작을 정형화하여 한정된 범위
내에서 관리하도록 결정했습니다.

결과적으로 캐릭터의 상태에 따라 실행될 트리거를 지정하도록 구현했습니다.
```
  ## 코드
``` C#
using System;
using System.Collections.Generic;
using UnityEngine;

namespace PlatformGame.Character.Animation
{
    [Serializable]
    public class StateTriggerPair
    {
        public CharacterState State;
        public string Trigger;
    }

    [RequireComponent(typeof(Character))]
    public class CharacterAnimation : MonoBehaviour
    {
        public Animator EditorAnimator => mAnimator == null ? null : mAnimator;
        Character mCharacter;
        Dictionary<CharacterState, string> mStateMap;
        [SerializeField] Animator mAnimator;
        [HideInInspector] public RuntimeAnimatorController BeforeController;
        [HideInInspector] public List<StateTriggerPair> EditorStateTriggers = new();

        void UpdateAnimation(CharacterState state)
        {
            if (!mStateMap.ContainsKey(state))
            {
                return;
            }

            foreach (var t in mStateMap.Values)
            {
                mAnimator.ResetTrigger(t);
            }

            var trigger = mStateMap[state];
            mAnimator.SetTrigger(trigger);
        }

        void Awake()
        {
            Debug.Assert(mAnimator);

            mCharacter = GetComponent<Character>();
            mCharacter.OnChangedState.AddListener(UpdateAnimation);

            mStateMap = new Dictionary<CharacterState, string>();
            foreach (var pair in EditorStateTriggers)
            {
                Debug.Assert(!mStateMap.ContainsKey(pair.State), $"{pair.State}");
                mStateMap.Add(pair.State, pair.Trigger);
            }
        }
    }
}
``` 
># 어빌리티

```
어빌리티는 캐릭터가 사용할 수 있는 능력입니다.
어빌리티는 히트 박스 파이프라인에 의해서만 실행됩니다.

히트 박스에 어빌리티를 부여하고, 히트 박스 이벤트가 발동하면 어빌리티 파이프라인이 실행되어
최종적으로 어빌리티가 발동됩니다.

어빌리티는 행위의 주체와 발동한 어빌리티에 대한 정보를 가지고 구현됩니다.
예를 들어 [밀어내는 힘]의 경우 피격의 주체를 발동된 어빌리티가 가진 힘과 방향으로 날려보냅니다.
```

  ## 리버스 어빌리티
  <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/b17592f0-b5ee-4430-ac87-9531a5bda068" width="365" height= "240"/>
  
  ```
  캐스터와 피격의 주체를 뒤바꿉니다.
  예를 들어 파괴 어빌리티의 경우에는 피격의 주체에게 적용되는 능력이지만,
  리버스 어빌리티에서는 캐스터에게 적용되는 능력으로 뒤바뀝니다.
  ```
  ## 코드
``` C#
using UnityEngine;

namespace PlatformGame.Character.Combat
{
    [CreateAssetMenu(menuName = "Custom/Ability/ReverseAbility")]
    public class ReverseAbility : Ability
    {
        public Ability AbilityAction;

        public override void UseAbility(AbilityCollision collision)
        {
            var abilityCollision = new AbilityCollision(collision.Victim, collision.Caster, collision.Ability);
            AbilityAction.UseAbility(abilityCollision);
        }

    }
}
``` 
  ## Burn
  <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/89e30422-310b-41a1-9f7f-60024a9ee68e" width="40%" height="40%"/>
  <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/b3af29c6-ec4f-42a9-80d1-b058518cb06c" width="365" height= "240"/>
  
```
불의 번지는 성질을 구현한 능력입니다.
공격의 주체의 태그를 확인합니다. 어빌리티가 가진 값과 동일하다면 피격의 주체를 태우고, 공격의 주체를 복사합니다.
```
  ## 파괴
  <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/6ed05e6f-7888-4e7d-8a84-83a1aaac605d" width="40%" height="40%"/>
  <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/ec325d9a-e662-48b3-9ffa-85871ea6dc12" width="365" height= "240"/>

```
피격의 주체를 파괴합니다. 파괴된 오브젝트를 삭제됩니다.
대상의 특성이 파괴가능인 상태에서만 적용되는 능력입니다.
```
  ## 밀어내는 힘
  <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/056c9429-cd2d-4908-a940-05c9527fd342" width="40%" height="40%"/>
  <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/4c4fa3f2-5b75-490d-a1de-fa180cf35f8d" width="365" height= "240"/>

```
대상을 캐릭터가 바라보는 방향과 어빌리티가 가진 힘과 방향을 곱한 만큼 밀어내는 능력입니다.
대상의 특성이 Non Static일 때만 동작합니다.
```
  ## 위성
  <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/02a2404d-8482-41fc-8dd9-749b32d164de" width="40%" height="40%"/>
  <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/106066c7-a7d2-4650-8d0e-320c5b0f0154" width="365" height= "240"/>
  
```
캐릭터 주위를 선회하는 오브젝트를 소환합니다.
소환된 오브젝트는 Non Static일 때만 캐릭터 주위를 선회합니다.
```
># 히트 박스
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/450b0b4b-0c64-4d56-9893-71385f4b3531" width="30%" height="30%"/>

```
히트 박스는 캐릭터 간의 충돌 이벤트를 처리하는 기능을 담당합니다.
충돌은 피격과 공격으로 구분됩니다.

피격의 주체인 경우에는 충돌 딜레이를 부여하는 것을 권장합니다.
히트 박스는 자신이 어느 캐릭터에게 소유되어 있는지를 알고 있어야 합니다. 자기 자신과는 충돌하지 않습니다.
```
  ## 히트 박스 어빌리티
  <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/87095c5e-0087-4b8f-a23a-57af8e5cb39b" width="30%" height="30%"/>

```
히트 박스는 어빌리티를 부여 받기도 합니다.

플레이어 캐릭터가 액션을 취하면서 부여하는 경우도 있지만,
미리 어빌리티를 지정할 수도 있습니다.

예를 들어 위의 사진의 경우에는 미리 Burn 어빌리티를 부여해서 불에 타는 상자를 만드는 예시입니다.

이런 식으로 다양한 종류의 블록을 프리펩으로 만들어 두면 레벨 작업을 좀 더 편하게 할 수 있습니다.
```
># 버튼 이벤트 액션
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/235aa8d2-5467-40cd-adfa-c2b969bae559" width="30%" height="30%"/>

```
캐릭터는 기본적으로 방향키를 통해 이동할 수 있습니다.
그 외로 액션 버튼 눌러 상황에 따른 액션을 취할 수 있습니다.

기본적으로 캐릭터가 밟고 있는 블록에 따라서 버튼 이벤트가 변경됩니다.
예를 들어 녹색 블록을 밝으면 버튼 이벤트가 상자를 여는 것으로 바뀝니다.
이 상태에서 액션 버튼을 누르면 상자를 열게 됩니다.

이런 기능을 활용하면 특정한 상황에서 실행되어야 하는 액션을 구현하는데 용이합니다.
```
  ## 블록 트리거
  <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/a60322eb-3e64-4633-8969-610cdd02982d" width="30%" height="30%"/>

```
캐릭터의 발 밑에는 트리거가 있습니다.
이 트리거는 밟고 있는 블록과 충돌해서 캐릭터의 상태와 버튼 이벤트를 변화시킵니다.

예를 들어 밝고 있는 블록이 지면 블록이라면 캐릭터는 이동이 가능하고 버튼 이벤트로는 점프가 할당될 수 있습니다.
하지만 공기 블록이라면 캐릭터는 이동할 수 없고 버튼 이벤트는 텅 빈 상태가 되어서 아무런 액션도 취할 수 없습니다.
```
  ## 액션 버튼
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/1f5e0a8a-9445-4c0a-b7d8-5a12ccc44b76" width="30%" height="30%"/>

```
액션 버튼은 하나만 존재합니다.
하지만 밟고 있는 블록에 따라서 다른 이벤트가 발동하기 때문에 여러 버튼을 누르는 것처럼 동작할 수 있습니다.

예를 들어 아무런 동작도 할 수 없는 상태일 수도 있고, 점프, 공격, 상자 열기, 포탑 건설 등 매우 다양한 이벤트가 발동할 수 있습니다.
```
  ## 배치
  <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/9ad27990-4c52-46d8-99e1-dfb89ec5fe24" width="15%" height="15%"/>

```
블록의 배치는 매우 중요한 사항입니다.
블록을 어떻게 배치하냐에 따라 게임의 장르와 밸런스가 달라집니다.

블록을 겹치게 배치하는 행위는 지양하는 것을 권장합니다.
블록의 크기를 정규화하고 일정한 간격으로 빠짐없이 배치하는 것이 바람직합니다.

예를 들어 마인크래프트에서 블록을 배치하는 것과 동일합니다.
마인크래프트는 보기에는 지형 블록만 배치되어 있다고 생각할 수도 있지만
보이지 않는 블록(공기, 빛 등)들로 꽉 차 있는 상태입니다.
```
># **특성**
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/7ae120b6-8a87-473f-bc9f-c881fed824c0" width="30%" height="30%"/>

```
캐릭터는 여러가지 특성을 가지고 있습니다.
특성은 플래그 형식으로 구현되어 있어서 조합이 가능합니다.

예를 들어 Non static 특성과 Premeability 특성을 활성화하면
움직일 수 있고 통과할 수 있는 NPC 같은 특성의 캐릭터가 됩니다.
```
  ## Non static
  ```
  활성화하면 캐릭터가 움직일 수 있는 상태가 됩니다.
  비활성화하면 정적인 캐릭터가 되어서 어떤 짓을 해도 움직이지 않습니다.

 예를 들어 비활성화 시에는 밀어내는 힘 어빌리티를 사용해도 캐릭터가 움직이지 않습니다.
  ```
  ## Distructible
  ```
  활성화하면 캐릭터가 파괴될 수 있는 상태가 됩니다.
  비활성화하면 정적인 캐릭터가 되어서 어떤 짓을 해도 파괴되지 않습니다.

 예를 들어 비활성화 시에는 파괴 어빌리티를 사용해도 캐릭터가 파괴되지 않습니다.
  ```
  ## Premeability
  ```
  활성화하면 다른 캐릭터가 이 캐릭터를 통과할 수 있습니다.

  예를 들어 물 블록의 경우에 이 옵션을 적용하면 블록을 통과해 이동할 수 있습니다.
  ```
># **크래프팅**
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/83ba5ff9-5fab-430c-b88c-4ced9ef2909f" width="30%" height="30%"/>

```
 이 기능을 만들 당시 세부적인 기획 내용을 전달받지 못한 상태에서 구현하다 보니
OCP를 준수하는 것에 어려움이 있었습니다. 때문에 기획에서 확실한 내용과 모호한 내용을
구분는 것에 집중했습니다.

- 확실한 내용 : 레시피에 있는 재료를 모두 입력받으면 결과물을 반환한다.
- 모호한 내용 : 재료를 입력받았을 때의 리액션, 결과물이 반환됐을 때의 리액션.

 전체적인 틀은 변하지 않아야 하므로 '확실한 내용'을 기반으로 설계하고,
'모호한 내용'을 구현하는 것은 유연해야 하므로 이벤트를 사용해서 구현했습니다.


 또한 크래프팅은 다양한 게임에 포함되는 내용이다 보니 모듈화하여 구현하고 싶었습니다.
이 부분에서 문제가 되었던 부분이, 이 게임에서는 결과물을 반환하는 과정에서 크래프팅을
행한 '주체'를 알아야 한단 부분이었습니다. 하지만 '주체'는 이 게임에서만 존재하는
Character 클래스였기에 모듈화에 문제가 발생했습니다.

 이러한 문제를 해결하기 위해 크래프팅의 '주체'에게 결과물을 직접 반환하는 것이 아닌
간접적으로 반환하여 '주체'로 하여금 습득하게 하는 방식으로 대체했습니다.

```
  ## 코드
``` C#
using PlatformGame.Character.Collision;
using System.Collections.Generic;
using System.Linq;
using UnityEngine;
using UnityEngine.Events;

namespace PlatformGame
{
    public class Crafting : MonoBehaviour
    {
        [Header("References")]
        [SerializeField] List<QuestItem> mRecipe;
        public List<QuestItem> Recipe => mRecipe.ToList();
        [SerializeField] GameObject mResultItem;

        [Header("Option")]
        public UnityEvent<Item> OnFailEvent;
        public UnityEvent<Item> OnInputItem;
        public UnityEvent<GameObject> OnOutputItem;
        [Tooltip("요구 개수 초과")]
        [SerializeField] bool mbOvercount;
        [Tooltip("다른 재료 허가")]
        [SerializeField] bool mbOtherInputItem;

        public void OnHit(HitBoxCollision collision)
        {
            var item = collision.Attacker.GetComponent<Item>();
            if (!item)
            {
                return;
            }

            InputItem(item);
        }

        public void ChangeRecipe(List<QuestItem> recipe)
        {
            mRecipe = recipe;
            Init();
        }

        void Init()
        {
            mRecipe.ForEach(x => x.Count = 0);
        }

        void InputItem(Item input)
        {
            var item = mRecipe.Find(x => x.Item.ID == input.ID);
            if (!mbOtherInputItem && item == null)
            {
                OnFailEvent.Invoke(input);
                return;
            }

            if (!mbOvercount && item.IsFull)
            {
                return;
            }

            item.Count++;
            OnInputItem.Invoke(input);

            if (mRecipe.Any(x => !x.IsFull))
            {
                return;
            }
            OutputItem();
            Init();
        }

        void OutputItem()
        {
            var obj = Instantiate(mResultItem);
            OnOutputItem.Invoke(obj);
        }

        void Awake()
        {
            Init();
        }

    }
}
```
# Item
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/55429f91-e8e9-4827-bd2c-3013b44bcc33" width="30%" height="30%"/>

```


 처음에는 하나의 재료당 한개, 재료끼리는 겹치지 않는다는 상황을 가정하고 작성하다 보니
Dictionary 자료형을 사용해 아이템 ID와 소유 여부를 확인했지만 같은 재료를 여러개 요구하는
레시피에 대해서도 고민하게 되었습니다.

 Dictionary로는 복수의 재료를 관리하기 어렵다고 판단했지만, List를 사용하기에도 ID와 개수
두 가지 정보를 관리하기에는 어려웠습니다. 때문에 해당 정보를 관리하는 새로운 클래스인 QueseItem
을 추가하게 되었습니다.

 Item은 최소한의 정보만을 담기 위해 고민했습니다. 크래프팅을 구현할 당시에는 아이템 끼리의 구분만
가능하면 되었기에 ID값 하나만을 가지도록 했습니다.
```
  ## 코드
``` C#
using System;
using UnityEngine;

namespace PlatformGame
{
    [Serializable]
    public class QuestItem
    {
        [SerializeField] Item mItem;
        public Item Item => mItem;
        [SerializeField] byte mRequiredCount;
        public byte RequiredCount => mRequiredCount;
        byte mCount;
        public byte Count
        {
            get => mCount;
            set => mCount = (byte)Mathf.Clamp(value, 0, 255);
        }
        public bool IsFull => mRequiredCount <= Count;
    }

    public class Item : MonoBehaviour
    {
        [SerializeField] int mID;
        public int ID => mID;
    }

}
```
