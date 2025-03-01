

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


