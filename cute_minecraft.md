

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
