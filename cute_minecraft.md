

# Cutscene
[기술 스택] Camera, Timeline, Timer, Layer, Game Manager, Scene Manager   
> "오프닝이나 엔딩, 보스 몬스터를 처치했을 때 등의 상황에서 컷신이 재생됩니다. 컷신이 출력되는 동안 플레이는 제어됩니다. 이 파트에서는 컷신의 재생과 카메라와 캐릭터를 제어한 방법을 설명합니다."

###	Camera   
컷신이 실행되면 현재 재생 중인 씬의 카메라를 모두 비활성하고, 컷신 씬을 서브 씬으로 로드하여 재생합니다.   
컷신의 재생이 끝나거나 스킵 버튼이 눌리면 서브 씬이 제거되고, 원래의 플레이로 복귀합니다.

<p align = "Right"> <image src = "https://github.com/user-attachments/assets/07c12022-13ab-4556-bf39-e3fb9f7ddbec"></p>

### Culling Mask   
팀의 작업 환경을 분리하여 충돌을 최소화하기 위해 컬링 마스크를 사용했습니다.   
컷신을 작업한 씬에서는 모든 레이어를 Movie로 통일하고, 카메라의 컬링 마스크를 Movie로 설정했습니다.   
컷신만 비추는 카메라가 렌더링 비용을 줄여줍니다.   

<p align = "Left"> <image src = "https://github.com/user-attachments/assets/454bcc9c-066d-4ed7-8267-3fad38ca65ee"></p>

###	Movie   

컷신은 모니터에 하나만 출력되며, 전투 도중처럼 다양한 상황에서 재생되어야 해서 전역 클래스로 구현했습니다.   
그리고, 상황에 따라 다른 이벤트가 발생할 것을 고려해, 행동을 수행하는 프로젝터를 교체할 수 있도록 설계했습니다.   

``` C#
namespace Cinema
{
    public static class Movie
    {
        private static Projector _projector = Projector.DEFAULT;
        public readonly static Projector INTRO = new(FilmContainer.Search("Intro"));
        public readonly static Projector ENTER_GAME = new(FilmContainer.Search("StartGame"));
        public readonly static Projector ENTER_BOSS = new(FilmContainer.Search("EnterBoss"));

        public static void ChangeCamera(Projector movie)
        {
            if (_projector.IsPlaying)
            {
                _projector.DoSkip();
            }
            _projector = movie;
        }

        public static void DoPlay()
        {
            _projector.DoPlay();
        }

        public static void DoPlay(Projector movie)
        {
            ChangeCamera(movie);
            DoPlay();
        }

        public static void DoSkip()
        {
            _projector.DoSkip();
        }
    }
}


```

### Projector(1/2)   
예외 상황을 줄이기 위해 DEFAULT 변수를 사용했습니다.   
프로젝터를 재사용하기 위해 필름을 갈아 끼워 다른 컷신을 재생할 수 있도록 설계했습니다.   
프로젝트의 동작이 타이머의 동작과 동일하다고 판단해 Timer 클래스를 프로젝터 플레이어로 활용했습니다.   

``` C#
namespace Cinema
{
    public class Projector
    {
        #region portfolio part1
        public static readonly Projector DEFAULT = new Projector();
        public bool IsStart => IsStart && !_player.IsPause;
        public bool IsPlaying => _player.IsStart && !_player.IsPause;
        public readonly byte ClosingDuration;
        private Film _film = Film.EMPTY;
        private readonly Timer _player = new();

        public Projector(Film film, byte closingDuration = 2) : this(closingDuration)
        {
            ChangeFilm(film);
        }

        public void ChangeFilm(Film film)
        {
            if (_player.IsStart)
            {
                _player.DoStop();
            }

            if (film.IsBad)
            {
                _film = Film.EMPTY;
                return;
            }

            _film = film;
            _player.SetTimeout(film.Time + ClosingDuration);
        }

        public void DoPlay() => _player.DoStart();
        public void DoSkip() => _player.DoStop();
        #endregion

        #region portfolio part2
        #endregion
    }
}

```

### Projector(2/2)   
느슨한 결합도를 유지하면서 게임의 흐름을 제어하기 이벤트를 사용했습니다.   
컷신을 플레이하기 위한 기능 중 재사용성이 높아 보이는 것들은 CameraUtil 클래스로 분리하여 사용했습니다.   
```C#
namespace Cinema
{
    public class Projector
    {
        #region portfolio part1
        #endregion

        #region portfolio part2
        public event Action OnPlay;
        public event Action OnSkip;
        public event Action OnEnd;
        private readonly CameraUtil _cameraUtil = new();
        private readonly CoroutineRunner _runner = CoroutineRunner.instance;

        public Projector(byte closingDuration = 2)
        {
            AddEventMoviePlay();
            AddEventMovieEnd();
            AddEventMovieSkip();
            ClosingDuration = closingDuration;
        }

        private void LoadFilm(Film film)
        {
            SceneManager.LoadSceneAsync(film.Name, LoadSceneMode.Additive);
        }

        private void UnloadFilm(Film film)
        {
            SceneManager.UnloadSceneAsync(film.Name);
        }

        private void AddEventMoviePlay()
        {
            _player.OnStart += (t) => _cameraUtil.DisableAllCamerasInScene();
            _player.OnStart += (t) => LoadFilm(_film);
            _player.OnStart += (t) => OnPlay?.Invoke();
            _player.OnStart += (t) => _runner.StartCoroutine(_player.Update());
        }

        private void AddEventMovieEnd()
        {
            _player.OnTimeout += (t) => UnloadFilm(_film);
            _player.OnTimeout += (t) => _cameraUtil.EnableAllCamerasInScene();
            _player.OnTimeout += (t) => _runner.StopCoroutine(_player.Update());
            _player.OnTimeout += (t) => OnEnd?.Invoke();
        }

        private void AddEventMovieSkip()
        {
            _player.OnStop += (t) => OnSkip?.Invoke();
            _player.OnStop += (t) => UnloadFilm(_film);
            _player.OnStop += (t) => _cameraUtil.EnableAllCamerasInScene();
            _player.OnStop += (t) => _runner.StopCoroutine(_player.Update());
            _player.OnStop += (t) => OnEnd?.Invoke();
        }
        #endregion
    }
}

```
###	Film   
필름에서도 예외 상황을 줄이기 위해 EMPTY 변수를 사용했습니다.   
필름은 컷신에서 사용되는 데이터 변수로, 최소한의 데이터만 담도록 했습니다.   
초기화되는 값에 따라 필름을 사용할 수 없는 경우가 있습니다. Name 변수로 컷신이 제작된 씬 에셋을 탐색하는데, 찾을 수 없는 경우, 이에 대해 어떻게 처리할지 필름을 사용하는 클래스로 책임을 떠넘기기 위해 IsBas 변수를 두었습니다.   

```C#
namespace Cinema
{
    [Serializable]
    public class Film
    {
        public static readonly Film EMPTY = new("EMPTY_FILM", 3f);
        public bool IsBad { get; private set; }
        public readonly float Time;
        public readonly string Name;

        public Film(string name, float time)
        {
            Name = name;
            Time = Mathf.Max(0, time);
            IsBad = SceneManagerUtil.IsSceneUnvalid(name);

#if DEVELOPMENT
            if (SceneManagerUtil.IsSceneUnvalid(name))
            {
                Debug.LogWarning(
                    $"Please add the scene to your build settings : {name}");
            }
            if (time < 0)
            {
                Debug.LogWarning(
                    $"Duration must be greater than 0 : {time}");
            }
#endif
        }
    }
}
```
# 메시지
[기술 스택] Mediator Pattern, Event, Set   
> “큐즐 팀엔 코딩에 대한 기초 지식이 있는 기획자가 많은 팀이었습니다. 그분들이 저를 위해 많은 양의 기획 내용을 문서로 작성하는 동안, 저는 그분들을 위해 함께 작업할 수 있는 환경을 준비했습니다. 
> 그 결과가 메시지를 통한 협업 환경입니다. 퍼즐의 규칙과 상태에 대한 데이터를 생성, 가공하여 메시지를 날려 주면서, ‘SLIME_SPAWN’ 메시지를 드릴 테니까 슬라임이 등장하는 애니메이션을 연결해 주세요. 하고 부탁드렸습니다.
> 덕분에 제 작업 속도와 무관하게, 퍼즐을 표현하는 작업이 진행될 수 있었고, 효율적으로 ‘협업’을 할 수 있었습니다. 이 파트에서는 메시지를 제어한 방법을 말씀드리겠습니다.”

<p align = "Right"> <image src = "https://github.com/user-attachments/assets/1a52a220-2cfc-4f8e-9126-da9ef21d4d3c"></p>

### Instance와 Core
기본적으로 퍼즐은 데이터를 표현해 주는 인스턴스와 데이터를 가공하는 코어로 구분하고 있습니다. 인스턴스와 코어가 데이터를 주고받으며 퍼즐을 구현하는데, 기획자들과 회의를 통해 필요한 메시지를 규격화하고 주고받고 있습니다.   

예를 들어 스테이지를 시작하면 스포너 코어에서 주기적으로 ‘SLIME_SPAWN’ 메시지를 송출합니다. 보스 인스턴스는 이 메시지를 수신하면 안개 속으로 뛰어들어 슬라임을 물고, 육면체 위로 올라와 뱉어 냅니다.   

![image](https://github.com/user-attachments/assets/aa0e2d99-10d5-439a-a756-b2fa3bb8ffbe)

### Mediator
인스턴스와 코어는 중재자를 통해 데이터를 주고받습니다. 특히 데이터 리더 타입의 변수를 가지고 있는데, 이를 통해서 수신하는 데이터를 필터 합니다.   
데이터 리더를 상속한 몬스터 리더 변수를 가지고 있다면, 몬스터에 관한 데이터만 수신할 수 있습니다. 단일 책임의 원칙을 준수하게 하고, 바이트 배열 형식의 데이터를 더 효율적으로 사용하기 위해서 이런 제한을 뒀습니다.   
```C#
public class Mediator : IMediatorCore, IMediatorInstance
{
    private readonly List<ICore> _cores = new();
    private readonly List<IInstance> _instances = new();

    public void SetCores(List<ICore> cores)
    {
        _cores.Clear();
        _cores.AddRange(cores);
    }

    public void SetInstances(List<IInstance> instances)
    {
        _instances.Clear();
        _instances.AddRange(instances);
    }

    public void InstreamDataCore<T>(byte[] data) where T : DataReader
    {
        _instances.Where(instance => instance.DataReader is T)
                  .ToList()
                  .ForEach(instance => instance.InstreamData(data));
    }

    public void InstreamDataInstance<T>(byte[] data) where T : DataReader
    {
        _cores.Where(core => core.DataReader is T)
              .ToList()
              .ForEach(core => core.InstreamData(data));
    }
}

```

### CubePuzzleComponent(1/5)
이 컴포넌트는 퍼즐을 관리합니다. 중재자를 통해 퍼즐을 구성하는 인스턴스와 코어를 컨트롤하고, 퍼즐의 진행에 맞춰 중재자를 설정해 줍니다.   
인스턴스와 인스턴스, 코어와 코어 간에는 메시지가 아닌 이벤트를 통해 통신하는데, 이러한 통신도 도와줍니다.   

<p align = "Right"> <image src = "https://github.com/user-attachments/assets/efd21ef5-66a9-4328-995f-9e0b82b59d11"> </p>
Awake에서 큐브 퍼즐 데이터로부터 퍼즐을 생성합니다. 메시지를 관측하고 중재자를 관리하기 위해 옵저버를 사용합니다. 인스턴스와 코어를 위한 리더를 생성한 후 퍼즐을 시작합니다.   

``` C#
using System;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;

namespace Puzzle
{
    public class CubePuzzleComponent : MonoBehaviour
    {
        private MessageObserverFromInstance _systemMessageFromInstance;
        private MessageObserverFromCore _systemMessageFromCore;
        private CubeMap<byte> _cubeMap;
        private CubePuzzleReader _puzzleReader;
        private CubePuzzleReaderForCore _puzzleReaderForCore;
        private readonly Mediator _mediatorCenter = new();
        private readonly UnityEvent _onReady = new();
        private readonly UnityEvent<Face> _clearLevelEvent = new();
        private readonly UnityEvent<Face> _onRotate = new();
        private readonly UnityEvent<Face> _startLevelEvent = new();
        [SerializeField] private CubePuzzleData _cubePuzzleData;

        #region portfolio part1
        private void Awake()
        {
            if (_cubePuzzleData.Faces.Length != 6)
            {
                Debug.LogWarning($"Check Cube Puzzle Data. Length : {_cubePuzzleData.Faces.Length}");
                return;
            }

            _cubeMap = new(_cubePuzzleData.Width, _cubePuzzleData.Elements);
            _onReady.AddListener(StartNextPuzzle);

            _systemMessageFromCore = new MessageObserverFromCore(new SystemReader());
            _systemMessageFromCore.OnRecieveMessage += MoveNextLevel;

            _systemMessageFromInstance = new MessageObserverFromInstance(new SystemReader());
            _systemMessageFromInstance.OnRecieveMessage += OnReady;
            _systemMessageFromInstance.OnRecieveMessage += OnRotateCube;

            _puzzleReader = new(_cubePuzzleData, _onRotate);
            _puzzleReader.CoreObservers.Add(_systemMessageFromInstance);
            _puzzleReader.InstanceObservers.Add(_systemMessageFromCore);

            _puzzleReaderForCore = new(_cubeMap,
                                       _startLevelEvent,
                                       _clearLevelEvent,
                                       _onRotate,
                                       _onReady);

            SettingAllResources();
            StartPuzzle(Face.top);
        }
        #endregion

        #region portfolio part2
        #endregion

        #region portfolio part3
        #endregion

        #region portfolio part4
        #endregion

        #region portfolio part5
        #endregion
    }
}

```
### CubePuzzleComponent(2/5)
큐즐의 퍼즐은 육면체 위에서 플레이된다는 특징이 있습니다. 따라서 6개의 면에 대해 코어와 인스턴스가 연결됩니다. 아래에서는 사용할 코어와 인스턴스를 가져와 바인더를 통해 연결한 뒤, 중재자에 등록합니다. 마지막으로 퍼즐에 대한 데이터가 있어야 하는 대상에게 퍼즐 리더를 제공합니다. 팀원들이 인스턴스나 코어에서 퍼즐 정보를 편하게 얻을 수 있도록 설계했습니다.
``` C#
using System;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;

namespace Puzzle
{
    public class CubePuzzleComponent : MonoBehaviour
    {
        private MessageObserverFromInstance _systemMessageFromInstance;
        private MessageObserverFromCore _systemMessageFromCore;
        private CubeMap<byte> _cubeMap;
        private CubePuzzleReader _puzzleReader;
        private CubePuzzleReaderForCore _puzzleReaderForCore;
        private readonly Mediator _mediatorCenter = new();
        private readonly UnityEvent _onReady = new();
        private readonly UnityEvent<Face> _clearLevelEvent = new();
        private readonly UnityEvent<Face> _onRotate = new();
        private readonly UnityEvent<Face> _startLevelEvent = new();
        [SerializeField] private CubePuzzleData _cubePuzzleData;

        #region portfolio part1
        #endregion

        #region portfolio part2
        private void SettingAllResources()
        {
            _puzzleReader.ReadAllCores(out List<ICore> inUseCores);
            MediatorBinder.BindEvent(inUseCores, _mediatorCenter);
            _mediatorCenter.SetCores(inUseCores);

            _puzzleReader.ReadAllInstances(out List<IInstance> inUseInstances);
            MediatorBinder.BindEvent(inUseInstances, _mediatorCenter);
            _mediatorCenter.SetInstances(inUseInstances);

            inUseCores.ForEach(x => (x as IPuzzleCore)?.Init(_puzzleReaderForCore));
            inUseInstances.ForEach(x => (x as IPuzzleInstance)?.Init(_puzzleReader));
        }

        private void ClearAllResources()
        {
            _puzzleReader.ReadAllCores(out var usedCores);
            MediatorBinder.UnbindEvent(usedCores, _mediatorCenter);
            usedCores.ForEach(x => (x as IDestroyable)?.Destroy());

            _puzzleReader.ReadAllInstances(out var usedInstances);
            MediatorBinder.UnbindEvent(usedInstances, _mediatorCenter);
            usedInstances.ForEach(x => (x as IDestroyable)?.Destroy());
        }
        #endregion

        #region portfolio part3
        #endregion

        #region portfolio part4
        #endregion

        #region portfolio part5
        #endregion
    }
}

```
•	CubePuzzleComponent(3/5)   
큐즐에서 퍼즐을 풀어가면 육면체가 회전하며 다른 면으로 이동하게 됩니다. 이때 면마다 사용되는 코어나 인스턴스가 다를 수 있는데, 사용되는 것들만 활성화하는 방법을 고민하게 되었습니다. 아래는 차집합을 활용해 더 이상 사용하지 않는 요소는 반환하고, 계속 사용되는 요소는 유지하며, 새롭게 사용하는 요소를 할당하는 코드입니다.   
<p align = "Right"> <image src = "https://github.com/user-attachments/assets/e71113b9-df25-4774-99ec-694b5f6cc317"></p>

``` C#
using System;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;

namespace Puzzle
{
    public class CubePuzzleComponent : MonoBehaviour
    {
        private MessageObserverFromInstance _systemMessageFromInstance;
        private MessageObserverFromCore _systemMessageFromCore;
        private CubeMap<byte> _cubeMap;
        private CubePuzzleReader _puzzleReader;
        private CubePuzzleReaderForCore _puzzleReaderForCore;
        private readonly Mediator _mediatorCenter = new();
        private readonly UnityEvent _onReady = new();
        private readonly UnityEvent<Face> _clearLevelEvent = new();
        private readonly UnityEvent<Face> _onRotate = new();
        private readonly UnityEvent<Face> _startLevelEvent = new();
        [SerializeField] private CubePuzzleData _cubePuzzleData;

        #region portfolio part1
        #endregion

        #region portfolio part2
        #endregion

        #region portfolio part3
        private void SettingNewlyUsedResources(Face nextFace)
        {
            _puzzleReader.ReadDifferenceOfSets(nextFace, _puzzleReader.CurrentFace, out List<ICore> newlyCores);
            MediatorBinder.BindEvent(newlyCores, _mediatorCenter);

            _puzzleReader.ReadAllCores(nextFace, out var inUseCores);
            _mediatorCenter.SetCores(inUseCores);

            _puzzleReader.ReadDifferenceOfSets(nextFace, _puzzleReader.CurrentFace, out List<IInstance> newlyInstances);
            MediatorBinder.BindEvent(newlyInstances, _mediatorCenter);

            _puzzleReader.ReadAllInstances(nextFace, out var inUseInstances);
            _mediatorCenter.SetInstances(inUseInstances);

            newlyCores.ForEach(x => (x as IPuzzleCore)?.Init(_puzzleReaderForCore));
            newlyInstances.ForEach(x => (x as IPuzzleInstance)?.Init(_puzzleReader));
        }

        private void ClearUnusedResources(Face nextFace)
        {
            _puzzleReader.ReadDifferenceOfSets(_puzzleReader.CurrentFace, nextFace, out List<ICore> usedCores);
            MediatorBinder.UnbindEvent(usedCores, _mediatorCenter);
            usedCores.ForEach(x => (x as IDestroyable)?.Destroy());

            _puzzleReader.ReadDifferenceOfSets(_puzzleReader.CurrentFace, nextFace, out List<IInstance> usedInstances);
            MediatorBinder.UnbindEvent(usedInstances, _mediatorCenter);
            usedInstances.ForEach(x => (x as IDestroyable)?.Destroy());
        }
        #endregion

        #region portfolio part4
        #endregion

        #region portfolio part5
        #endregion
    }
}

```
### CubePuzzleComponent(4/5)
아래는 퍼즐의 진행도를 이동시키는 코드입니다. 옵저버를 통해 퍼즐 클리어 메시지를 받으면 다음 레벨로 이동시킵니다. 레벨이 넘어가면 플레이어가 준비되기 전까지 중재자는 작동하지 않습니다. 컷신에서 몬스터가 생성되는 등의 문제를 방지했습니다.

``` C#
using System;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;

namespace Puzzle
{
    public class CubePuzzleComponent : MonoBehaviour
    {
        private MessageObserverFromInstance _systemMessageFromInstance;
        private MessageObserverFromCore _systemMessageFromCore;
        private CubeMap<byte> _cubeMap;
        private CubePuzzleReader _puzzleReader;
        private CubePuzzleReaderForCore _puzzleReaderForCore;
        private readonly Mediator _mediatorCenter = new();
        private readonly UnityEvent _onReady = new();
        private readonly UnityEvent<Face> _onClearLevel = new();
        private readonly UnityEvent<Face> _onRotate = new();
        private readonly UnityEvent<Face> _onStartLevel = new();
        [SerializeField] private CubePuzzleData _cubePuzzleData;

        #region portfolio part1
        #endregion

        #region portfolio part2
        #endregion

        #region portfolio part3
        #endregion

        #region portfolio part4
        private void StartPuzzle(Face playFace)
        {
            if (!Enum.IsDefined(typeof(Face), playFace))
            {
                Debug.LogWarning($"Failed to start : {playFace} face");
                return;
            }
            ClearUnusedResources(playFace);
            SettingNewlyUsedResources(playFace);
            _puzzleReader.MoveReadWindow(playFace);
            _onStartLevel.Invoke(playFace);
        }

        private void StartNextPuzzle()
        {
            StartPuzzle(_puzzleReader.CurrentFace + 1);
        }

        private void MoveNextLevel(byte[] data)
        {
            if (!SystemReader.IsClearFace(data))
            {
                return;
            }
            _onClearLevel?.Invoke((Face)(data[0] - 1));

            if (SystemReader.CLEAR_BOTTOM_FACE.Equals(data))
            {
                ClearAllResources();
            }
        }
        #endregion

        #region portfolio part5
        #endregion
    }
}

```

### CubePuzzleComponent(5/5)
육면체가 회전했을 때 플레이어가 어느 면에 착지했는지 알아야 합니다. 아래의 코드에서는 육면체의 법선 벡터와 내적을 이용해서 현재 위쪽을 향하는 면이 어느 면인지 계산합니다. 그다음 이벤트를 발동시켜서 다른 인스턴스들도 플레이 중인 면을 알 수 있도록 합니다.

```C#
using System;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;

namespace Puzzle
{
    public class CubePuzzleComponent : MonoBehaviour
    {
        private MessageObserverFromInstance _systemMessageFromInstance;
        private MessageObserverFromCore _systemMessageFromCore;
        private CubeMap<byte> _cubeMap;
        private CubePuzzleReader _puzzleReader;
        private CubePuzzleReaderForCore _puzzleReaderForCore;
        private readonly Mediator _mediatorCenter = new();
        private readonly UnityEvent _onReady = new();
        private readonly UnityEvent<Face> _onClearLevel = new();
        private readonly UnityEvent<Face> _onRotate = new();
        private readonly UnityEvent<Face> _onStartLevel = new();
        [SerializeField] private CubePuzzleData _cubePuzzleData;

        #region portfolio part1
        #endregion

        #region portfolio part2
        #endregion

        #region portfolio part3
        #endregion

        #region portfolio part4
        #endregion

        #region portfolio part5
        private void OnRotateCube(byte[] data)
        {
            if (!data.Equals(SystemReader.ROTATE_CUBE))
            {
                return;
            }

            const float threshold = 0.98f;
            var up = _puzzleReader.BaseTransform.up;
            var right = _puzzleReader.BaseTransform.right;
            var forward = _puzzleReader.BaseTransform.forward;

            var playFace = Vector3.Dot(up, Vector3.up) > threshold ? Face.top :
                           Vector3.Dot(-up, Vector3.up) > threshold ? Face.bottom :
                           Vector3.Dot(right, Vector3.up) > threshold ? Face.right :
                           Vector3.Dot(-right, Vector3.up) > threshold ? Face.left :
                           Vector3.Dot(forward, Vector3.up) > threshold ? Face.front :
                           Vector3.Dot(-forward, Vector3.up) > threshold ? Face.back : Face.top;

            _onRotate.Invoke(playFace);
        }

        private void OnReady(byte[] data)
        {
            if (!data.Equals(SystemReader.READY_PLAYER))
            {
                return;
            }
            _onReady.Invoke();
        }

        private void OnDrawGizmosSelected()
        {
            Gizmos.color = Color.yellow;
            Gizmos.DrawWireCube(transform.position, _cubePuzzleData.BaseTransformSize.extents);
        }

        private void OnDestroy()
        {
            ClearAllResources();
            _onReady.RemoveAllListeners();
            _systemMessageFromCore.OnRecieveMessage -= MoveNextLevel;
            _systemMessageFromInstance.OnRecieveMessage -= OnReady;
            _systemMessageFromInstance.OnRecieveMessage -= OnRotateCube;
        }
        #endregion
    }
}
```



# 퍼즐
[기술 스택] Array, Hitbox, Event
> “퍼즐의 규칙은 간단합니다. 한 면의 수정의 색을 일치시키면 클리어됩니다.

![image](https://github.com/user-attachments/assets/f4877c38-4fa4-4aef-ac42-21aeec34f688)


> 여기에는 몇 가지 방해 요인이 있는데, 플레이어가 공격한 수정은 십자 범위로 색이 반전됩니다. 주기적으로 슬라임이 등장해서 수정의 색을 바꾸거나 새로운 수정을 생성합니다. 이러한 방해 요인을 제거하며 총 6개의 퍼즐을 풀면 스테이지는 클리어됩니다. 이번 파트에서는 퍼즐에 대한 내용을 설명해 보겠습니다.”

### CubeMap   
큐즐에서 퍼즐의 가장 기초가 되는 데이터입니다. 퍼즐은 정육면체에 그리드 형식으로 배치되어 있습니다. 인덱스를 계산해 일차원 배열을 가상의 육면체로 관리합니다. 초기화 시 지정된 복사기를 사용해 배열을 인스턴스로 채웁니다. 외부 클래스에서 큐브맵을 쉽게 열거할 수 있도록 인덱스를 반환하는 기능도 추가했습니다.   

메모리를 절약하기 위해 코어에서는 CubeMap<byte>를 사용하고, 인스턴스에서는 CubeMap<T>를 사용하고 있습니다.   

``` C#
public class CubeMap<T>
{
    public readonly T[] Elements;
    public readonly byte Width;

    public CubeMap(byte width, ICopyable<T> copier)
    {
        Width = width;
        Elements = new T[(int)Mathf.Pow(width, 2) * Enum.GetValues(typeof(Face)).Length];

        for (int i = 0; i < Elements.Length; i++)
        {
            Elements[i] = copier.Copy();
        }
    }

    public T GetElements(byte x, byte y, byte z)
    {
        Debug.Assert(Width * Width * z + Width * y + x < Elements.Length,
            $"Out of range : x {x} y {y} z {z} max {Elements.Length - 1}");
        return Elements[Width * Width * z + Width * y + x];
    }

    public void SetElements(byte x, byte y, byte z, T value)
    {
        Debug.Assert(Width * Width * z + Width * y + x < Elements.Length,
            $"Out of range : x {x} y {y} z {z} max {Elements.Length - 1}");
        Elements[Width * Width * z + Width * y + x] = value;
    }

    public List<byte[]> GetIndex(Face face)
    {
        List<byte[]> list = new();

        for (byte y = 0; y < Width; y++)
        {
            for (byte x = 0; x < Width; x++)
            {
                list.Add(new[] { x, y, (byte)face });
            }
        }

        return list;
    }
}

```

### FlowerPuzzleInstance
이 클래스에서는 CubeMap의 데이터를 가져와 인스턴스를 생성하고, 육면체에 배치합니다. 때문에 육면체의 크기, 퍼즐의 배치를 알아야 합니다. IPuzzleInstance를 상속받아 간단하게 퍼즐의 정보를 받아 올 수 있습니다.   
![image](https://github.com/user-attachments/assets/4c05cf14-d877-41a1-928a-1baabda3780d)

퍼즐을 공격하면 메시지를 생성해 코어로 전송합니다.   

코어에서는 메시지를 가공해 다시 인스턴스로 송출합니다.   

InstreamData로 메시지를 받으면 퍼즐의 색을 바꾸어 표현해 줍니다.    

``` C#
public class FlowerPuzzleInstance : ScriptableObject, IInstance, IPuzzleInstance, IDestroyable
{
    public DataReader DataReader { get; private set; } = FlowerReader.Instance;
    private CubeMap<Flower> _cubeMap;
    private readonly HitBoxLink _dataLink = new();
    private CubePuzzleReader _puzzleData;
    [SerializeField] private Flower _flowerPrefab;

    public void Init(CubePuzzleReader puzzleData)
    {
        _puzzleData = puzzleData;
        var instantiator = new Instantiator<Flower>(_flowerPrefab);
        _cubeMap = new CubeMap<Flower>(_puzzleData.Width, instantiator);
        _dataLink.Link(_cubeMap);
        _puzzleData.OnRotatedStage += OnRotated;
        InitFlower();
    }

    public void InstreamData(byte[] data)
    {
        var face = (Face)data[2];
        if (face is Face.bottom)
        {
            return;
        }
        var flower = _cubeMap.GetElements(data[0], data[1], data[2]);
        flower.gameObject.SetActive(true);
        switch (data[3])
        {
            case (byte)Flower.Type.Red:
                flower.Color = new Color(191f / 255f, 12f / 255f, 255f / 255f);
                break;
            case (byte)Flower.Type.Green:
                flower.Color = Color.cyan;
                break;
            default:
                flower.gameObject.SetActive(false);
                break;
        }
    }
    // 생략
}

```

### Flower Puzzle Core
코어에서는 인스턴스에서 생성된 메시지를 받으면, 메시지를 가공합니다.   
메시지 가공은 CoreFunction 클래스로 분리하여 상황에 맞게 사용할 수 있도록 했습니다. 보스 스테이지에 진행하면 바뀝니다.   

data[3] 은 퍼즐이 어떤 유형의 공격을 받았는지 나타냅니다. 유형에 따라 십자 범위로 색을 반전시키거나 한 칸 범위로 색을 반전시키는 등 유연하게 확장할 수 있도록 했습니다.   

IPuzzleCore를 상속받아 레벨이 시작하거나 클리어될 때의 이벤트에 접근할 수 있습니다.   

수신할 수 있는 메시지 종류는 하나이지만 송신할 수 있는 데이터의 종류는 여러 개입니다.    

``` C#
public class FlowerPuzzleCore : ScriptableObject, ICore, IPuzzleCore
{
    public DataReader DataReader { get; private set; } = new FlowerReader();
    private bool _isStart;
    private CubeMap<byte> _puzzle;
    private IMediatorCore _mediator;
    private CubePuzzleReaderForCore _reader;
    private readonly CoreFunction _crossAttack = FlowerCoreFuntions.AttackCross;
    private readonly CoreFunction _dotAttack = FlowerCoreFuntions.AttackDot;
    private readonly CoreFunction _createFlower = FlowerCoreFuntions.CreateFlower;
    private readonly CoreFunction _clearCheck = FlowerCoreFuntions.CheckFlowerNormalStageClear;

    public void InstreamData(byte[] data)
    {
        if (!_isStart)
        {
            return;
        }
        List<byte[]> puzzleMessages = new();
        switch (data[3])
        {
            case 1:
                _crossAttack(data, _puzzle, out puzzleMessages);
                break;
            case 2:
                _dotAttack(data, _puzzle, out puzzleMessages);
                break;
            case 3:
                _createFlower(data, _puzzle, out puzzleMessages);
                break;
            default:
                return;
        }
        foreach (var message in puzzleMessages)
        {
            _mediator.InstreamDataCore<FlowerReader>(message);
        }

        _clearCheck(data, _puzzle, out var systemMessages);
        foreach (var message in systemMessages)
        {
            _mediator.InstreamDataCore<SystemReader>(message);
        }
    }

    public void Init(CubePuzzleReaderForCore reader)
    {
        _reader = reader;
        _reader.OnStartLevel += (face) => _isStart = true;
        _reader.OnClearLevel += (face) => _isStart = false;
        _puzzle = reader.Map;
    }

    public void SetMediator(IMediatorCore mediator)
    {
        _mediator = mediator;
    }
}

```

### Attack Box
퍼즐을 공격하면 메시지를 생성합니다. 이러한 이벤트를 만들기 위해 히트 박스를 구현했습니다. 히트 박스는 Attack Box와 Hit Box로 구분되어 있습니다. 캐릭터가 공격을 하면 Attack Window가 열리고, 그 사이 Hit Box와 충돌하면 이벤트가 발동합니다.

![image](https://github.com/user-attachments/assets/c5549594-ec5d-40c5-b525-97e5c484d550)

아래의 코드에선 OnTriggerStay등의 유니티 이벤트가 아닌 CheckCollision이라는 메서드를 따로 뒀는데, 비용이 큰 트리거 이벤트에 의존하지 않도록 하기 위해서 이렇게 분리했습니다.   
부모인 CollisionBox 클래스에 있는 OnCollision 이벤트를 통해 메시지를 생성하는 기능을 주입했습니다.   

``` C#
public class AttackBox : CollisionBox
{
    public enum AttackType { None, OneHit };
    private readonly Delay _delay;
    private readonly List<Collider> _attacked = new();
    private bool _bNotWithinAttackWindow => !_delay.IsDelay() || (_attackType == AttackType.OneHit && _bHit);
    private bool _bHit;
    private AttackType _attackType;

    public AttackBox(Transform actor, float attackWindow = 0f) : base(actor)
    {
        _delay = new Delay(attackWindow);
        _delay.StartTime = -1f;
    }

    public void CheckCollision(Collider other)
    {
        if (_bNotWithinAttackWindow)
        {
            return;
        }

        if (_bNotWithinAttackWindow ||
            !other.TryGetComponent<IHitBox>(out var victim) ||
            victim.HitBox.Actor.Equals(Actor) ||
            _attacked.Contains(other))
        {
            return;
        }

        _attacked.Add(other);
        CollisionBox.InvokeCollision(this, victim.HitBox);
        _bHit = true;
    }

    public void OpenAttackWindow()
    {
        _bHit = false;
        _delay.DoStart();
        _attacked.Clear();
    }

    public void SetType(AttackType type)
    {
        _attackType = type;
    }
}

```

# 스포너
[기술 스택] AnimationCurve, Animator
> “스포너는 방해 요인인 슬라임을 생성합니다. 매력적인 슬라임을 만들기 위해 기술적으로도 기획적으로도 많이 고민했습니다. 이 파트에서는 기획에 대한 고민도 함께 정리해보고자 합니다.”

### 역동적인 움직임
슬라임이 등장할 때 피격 모션을 끼워 넣었습니다. 애니메이션을 제작할 때 역동감을 위해 의도적으로 작화 붕괴 프레임을 넣는다는 글을 보고, 참고하여 시도해 봤습니다.    

추가로 애니메이션 커브로 몬스터의 이동 궤도를 제어하고 커스텀 할 수 있도록 하여 좀 더 자연스러운 움직임을 연출했습니다.   

![image](https://github.com/user-attachments/assets/9ef556f6-c09d-4fe3-9394-64ffdf2b3abb)

### 잘 활용하면 도움이 되는 방해 요인
고인물의 플레이를 보면 몬스터를 농락하는 모습을 볼 수 있습니다. 하이 리스크를 감수하고 하이 리턴을 얻는 방식인데, 이는 적당한 긴장감을 부여합니다.   

저는 슬라임에 이런 기능을 넣자고 의견을 냈습니다. 슬라임은 퍼즐을 흐트러뜨리는 방해 요인이지만, 하나의 퍼즐만 색을 바꾼다는 특성이 있어 십자 범위의 공격밖에 하지 못하는 플레이어에게 도움이 될 수도 있습니다.   

슬라임으로만 풀 수 있는 퍼즐은 이렇게 만들어졌습니다.   

![image](https://github.com/user-attachments/assets/fe23d7c4-c972-4873-a76a-db98dff1ea63)

### 성취감을 주는 방해 요인
육면체에 대한 아이디어는 제가 제시했습니다. 때문에 육면체를 회전하는 것과, 퍼즐의 재미를 어떻게 이어줄 지 많이 고민했습니다.   
레벨을 클리어하면 육면체가 회전하며, 슬라임을 튕겨 내는 장면이 연출되는데 이것이 제 고민의 결과입니다.    
![image](https://github.com/user-attachments/assets/b41902e4-8926-48ba-b55e-17dbc0e28c4d)


### 플레이의 학습
유저는 1~5레벨의 퍼즐을 풀며 육면체의 회전과 방해 요인에 대해 학습해 왔습니다. 보스 스테이지에서는 지금까지 학습한 내용을 더 자유롭게 활용할 수 있도록 설계했습니다.   

우선 육면체를 자유롭게 회전시킬 수 있습니다. 지금까지는 퍼즐을 해결했을 때만, 주어진 방향으로 회전시킬 수밖에 없었지만 보스전에서는 육면체의 모서리로 달려가면 트리거 이벤트가 발동하며 육면체가 회전합니다.   

여기에 더해, 방해 요인의 등장 방식이 변경되었습니다. 육면체가 고공으로 상승하면서 슬라임은 자력으로 뛰어오를 수 없게 되었습니다. 이젠 보스가 슬라임을 물어서 투척합니다.   

여기에, 육면체 회전을 응용하면 재미있는 그림이 연출됩니다.   

타이밍에 맞춰 슬라임을 제거할 수 있습니다.   

![image](https://github.com/user-attachments/assets/55a9dce7-a3ae-4ba4-8e1c-af5a2f24cdc2)



# 발자국
[기술 스택] Object Pool, Caching, Particle, Singleton   
발자국은 캐릭터를 생동감 있게 만듭니다. 저는 발자국을 구현하는 여러 방법 중 파티클과 오브젝트 풀링을 활용하는 방법을 선택했습니다. 캐릭터는 육면체라는 고정된 평면 위에서만 이동하며, 파티클을 수정하여 이펙트를 만들어내기 용이하기 때문입니다.   

### Footstep Component (1/2)

왼발, 오른발, 양발을 박찰 때 이펙트를 출력해야 합니다. 발자국 컴포넌트는 메서드를 호출받으면 상황에 맞게 이펙트를 실행시킵니다. 왼발, 오른발, 착지라는 변수로 상황을 논리적으로 표현하였습니다.   
![image](https://github.com/user-attachments/assets/0aade63b-f0d3-431b-9f52-a0badb7f9b74)

### Footstep Component (1/2)
왼발, 오른발, 양발을 박찰 때 이펙트를 출력해야 합니다. 발자국 컴포넌트는 메서드를 호출받으면 상황에 맞게 이펙트를 실행시킵니다. 왼발, 오른발, 착지라는 변수로 상황을 논리적으로 표현하였습니다.

``` C#
public class FootstepComponent : MonoBehaviour
{
    private bool _bLanding;
    private bool _bLeftStep;
    private bool _bRightStep;
    [SerializeField] private Transform _leftFoot;
    [SerializeField] private Transform _rightFoot;
    [SerializeField] private ParticleSystem _footstepEffect;
    [SerializeField] private ParticleSystem _stempEffect;

    #region portfolio part1
    public void StepLeft()
    {
        _bLeftStep = true;
        PlayEffect();
    }
    public void StepRight()
    {
        _bRightStep = true;
        PlayEffect();
    }
    public void JumpUp()
    {
        _bLeftStep = true;
        _bRightStep = true;
        PlayEffect();
    }
    public void OnLand()
    {
        _bLanding = true;
        _bLeftStep = false;
        _bRightStep = false;
        PlayEffect();
    }
    #endregion

    #region portfolio part2
    private void PlayEffect()
    {
        if (_bLeftStep || _bRightStep)
        {
            if (_footstepEffect != null)
            {
                _footstepEffect.Play();
            }
        }

        if (_bLanding)
        {
            if (_stempEffect != null)
            {
                _stempEffect.Play();
            }
        }
    }
    #endregion
}

```

### Footstep Component (2/2)

몇 가지 예외 상황이 발생하면 디버깅 시 메시지를 출력하도록 했습니다. 특히 오류 메시지를 정리해 둔 DM_ERROR 정적 클래스와 name 변수를 통해 문제 상황을 해결할 수 있도록 돕고 있습니다.   

조건에 따라 상황을 파악하고 그에 맞는 행동을 지시합니다. 파티클을 실행하는 행동은 별개의 클래스에서 구현하여 재사용성을 높이고 관리가 용이하게 하고 있습니다.   

``` C#
public class FootstepComponent : MonoBehaviour
{
    private bool _bLanding;
    private bool _bLeftStep;
    private bool _bRightStep;
    [SerializeField] private Transform _leftFoot;
    [SerializeField] private Transform _rightFoot;
    [SerializeField] private ParticleSystem _footstepEffect;
    [SerializeField] private ParticleSystem _stempEffect;

    #region portfolio part1

    #endregion

    #region portfolio part2
    private void PlayEffect()
    {
        var effect = _bLanding ? _stempEffect : _footstepEffect;
        if (!effect)
        {
            Debug.LogWarning($"Effect is null. {DM_ERROR.REFERENCES_NULL} in {name}.");
            return;
        }
        if (_bLeftStep && _leftFoot == null)
        {
            Debug.LogWarning($"Left Foot is null. {DM_ERROR.REFERENCES_NULL} in {name}.");
            return;
        }
        if (_bRightStep && _rightFoot == null)
        {
            Debug.LogWarning($"Right Foot is null. {DM_ERROR.REFERENCES_NULL} in {name}.");
            return;
        }

        Vector3 effectPosition;
        if (_bLeftStep && _bRightStep)
        {
            effectPosition = Vector3.Lerp(_leftFoot.position, _rightFoot.position, 0.5f);
        }
        else if (_bLeftStep)
        {
            effectPosition = _leftFoot.position;
        }
        else
        {
            effectPosition = _rightFoot.position;
        }

        ParticlePlayer.PlayParticleAt(effect, effectPosition);
        _bLanding = _bRightStep = _bLeftStep = false;
    }
    #endregion
}

```

### Particle Player

대량으로 실행되는 파티클을 관리하기 위해 오브젝트 풀링과 캐싱을 사용하였습니다. 오브젝트 풀에 파티클을 전달하면, 파티클 프리펩이 가진 인스턴스 ID 값을 기준으로 캐싱합니다. 풀이 중복되지 않도록 관리합니다.   

풀로부터 인스턴스를 받아와 사용하고, 코루틴을 통해 파티클이 종료되면 반환하여 메모리를 관리하고 있습니다.   

발자국 컴포넌트와 분리되어 있어서, 다른 클래스에서도 필요에 따라 사용하고 있습니다.   
![image](https://github.com/user-attachments/assets/93764da0-0ceb-415c-b969-74ea74d354a7)

``` C#
public static class ParticlePlayer
{
    public static void PlayParticleAt(ParticleSystem particle, Vector3 position)
    {
        var instance = GameObjectPool<ParticleSystem>.GetPool(particle).Get();
        instance.transform.position = position;
        instance.Play();
        CoroutineRunner.instance.StartCoroutine(ReleaseEffect(particle, instance));
    }

    public static void PlayParticleAt(ParticleSystem particle, Vector3 position, Vector3 scale)
    {
        var instance = GameObjectPool<ParticleSystem>.GetPool(particle).Get();
        instance.transform.position = position;
        instance.transform.localScale = scale;
        instance.Play();
        CoroutineRunner.instance.StartCoroutine(ReleaseEffect(particle, instance));
    }

    private static IEnumerator ReleaseEffect(ParticleSystem origin, ParticleSystem particle)
    {
        yield return new WaitUntil(() => !particle.IsAlive());
        GameObjectPool<ParticleSystem>.GetPool(origin).Release(particle);
    }
}

```



# 이벤트 핸들러
[기술 스택] Unity Event, Timer, Input System

> “Unity Event를 사용하여 인스펙터에서의 작업을 용이하게 만들었습니다. 아주 기본적인 트리거 이벤트 핸들러부터 특정 상황이나, 조건에 따른 이벤트 핸들러를 만들어 팀원들의 유니티 작업을 도왔습니다.”

### IF Component
조건문을 이벤트 핸들러로 작성했습니다. 언리얼의 블루프린터에서 작업하는 것처럼 간단한 기능을 만들 수 있습니다.
![image](https://github.com/user-attachments/assets/63b1d7ce-1f8f-4160-89ee-6a62688bb5cf)

### Branch Component
IF 컴포넌트의 확장으로 조건이 거짓일 때도 이벤트를 발동시킵니다.
![image](https://github.com/user-attachments/assets/f326edcb-e541-4926-8ff9-40d264760798)

### Condition Component
주어진 조건이 참인지 거짓인지 판단하는 컴포넌트들입니다. 

![image](https://github.com/user-attachments/assets/11e2215c-4285-4fe2-9f7c-a59fb345eea2)

###Player Controller Component
입력된 키와 지정된 입력의 종류에 따라 이벤트를 실행해 줍니다. 메뉴 키를 눌러 UI 창을 켜고 끄거나, 버튼 이벤트 액션을 만들 때 유용하게 사용합니다.
![image](https://github.com/user-attachments/assets/201581f8-8c59-4e6d-aa0d-509cd7a1abeb)

### Timer Component 
타이머 클래스를 핸들링하는 타이머 컴포넌트입니다. 스톱워치와 같은 동작을 하며 타임아웃 이벤트에 타이머 스타트 이벤트를 넣어 반복 타이머로 활용하고 있습니다.   

트리거 이벤트 핸들러를 활용하여 캐릭터가 몬스터의 영역에 진입했을 때 제한 시간 타이머를 실행하는 식으로 활용하기도 합니다.   

![image](https://github.com/user-attachments/assets/b3192316-2684-4aab-b54e-43dfaa5cf071)
