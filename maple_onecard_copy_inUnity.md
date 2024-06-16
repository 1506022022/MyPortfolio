# 게임 이름

<p align="right">  
  <a href="https://youtu.be/FjVJnLojaAo">
    <img src="https://github.com/1506022022/MyPortfolio/assets/88864717/6358a717-736b-446b-90bb-528106250da7" width="80%" height="80%">
  </a>
</p>
  
<div align="right">
    <a href="https://youtu.be/FjVJnLojaAo">동영상으로 이동</a>
</div>

># 목차
- **[개요](#프로젝트-구성)**
- **[Broadcaster](#Broadcaster)**
- **[Turn System](#Turn-System)**
- **[Slot Board](#Slot-Board)**
- **[Card Button](#Card-Button)**

# 프로젝트 구성
|개요|내용|
|---|---|
|장르|턴제 카드게임|
|개발기간|2022.08.09 - 2022.08.16|
|참여인원|1인|
|개발환경|Unity|

># Broadcaster
```
원카드는 턴 이벤트를 기반으로 기능들이 구현되어야 한다고 생각했습니다.
턴 이벤트는 다양하게 존재하기 때문에 하나의 클래스에 모아두는 편이 좋겠다고 생각해
방속국 역할을 하는 클래스를 하나 만들어서 관리하도록 했습니다.

메이플 원카드를 분석해보니 턴 이벤트 하나에 다수의 개체가 접근할 수 있어야 하는 상황이었습니다.
때문에 방속국은 턴 이벤트를 채널 단위로 소지하고, 외부에서 접근해 채널을 구독, 해지할 수 있는
옵저버 패턴을 사용하는 것이 적합한 것 같아 적용시켜 보았습니다.
```
  ## 코드
``` C#
using System.Collections.Generic;

namespace ObserverPattern
{
    public static class Broadcaster
    {
        private static Channel gameStartChannel;
        private static Channel turnChangedChannel;
        private static Channel turnEndChannel;
        private static Channel reverseTurnChannel;
        private static Channel submitTimeoutChannel;
        private static Channel lastCardChannel;
        private static Channel victoryChannel;
        private static Channel gameoverChannel;
        private static Channel dropoutChannel;
        private static Channel cardSubmitChannel;
        private static Channel restartChannel;
        private static Channel gamEndChannel;

        public static Channel GameStartChannel
        {
            get
            {
                if (gameStartChannel == null) gameStartChannel = new Channel(null);
                return gameStartChannel;
            }
        }

        public static Channel TurnChangedChannel
        {
            get
            {
                if (turnChangedChannel == null) turnChangedChannel = new Channel(null);
                return turnChangedChannel;
            }
        }

        public static Channel TurnEndChannel
        {
            get
            {
                if (turnEndChannel == null) turnEndChannel = new Channel(null);
                return turnEndChannel;
            }
        }

        public static Channel ReverseTurnChannel
        {
            get
            {
                if (reverseTurnChannel == null) reverseTurnChannel = new Channel(null);
                return reverseTurnChannel;
            }
        }
        public static Channel LastcardChannel
        {
            get
            {
                if (lastCardChannel == null) lastCardChannel = new Channel(null);
                return lastCardChannel;
            }
        }
        public static Channel VictoryChannel
        {
            get
            {
                if (victoryChannel == null) victoryChannel = new Channel(null);
                return victoryChannel;
            }
        }
        public static Channel GameOverChannel
        {
            get
            {
                if (gameoverChannel == null) gameoverChannel = new Channel(null);
                return gameoverChannel;
            }

        }
        public static Channel DropoutChannel
        {
            get
            {
                if (dropoutChannel == null) dropoutChannel = new Channel(null);
                return dropoutChannel;
            }
        }
        public static Channel SubmitTimeOutChannel
        {
            get
            {
                if (submitTimeoutChannel == null) submitTimeoutChannel = new Channel(null);
                return submitTimeoutChannel;
            }
        }
        public static Channel CardSubmitChannel
        {
            get
            {
                if (cardSubmitChannel == null) cardSubmitChannel = new Channel(null);
                return cardSubmitChannel;
            }
        }
        public static Channel RestartChannel
        {
            get
            {
                if (restartChannel == null) restartChannel = new Channel(null);
                return restartChannel;
            }
        }
        public static Channel GameendChannel
        {
            get
            {
                if (gamEndChannel == null) gamEndChannel = new Channel(null);
                return gamEndChannel;
            }
        }

        public static void Reset()
        {
            gameStartChannel = null;
            turnChangedChannel = null;
            turnEndChannel = null;
            reverseTurnChannel = null;
            submitTimeoutChannel = null;
            lastCardChannel = null;
            victoryChannel = null;
            gameoverChannel = null;
            dropoutChannel = null;
            cardSubmitChannel = null;
        }
    }

    public class Channel : ISubject
    {
        private ObserverBot observer;
        private List<Observer> observers;
        public Channel(ISubject subject)
        {
            AddSuject(subject);
        }

        public void Notify()
        {
            if (observers == null) return;
            foreach (var o in observers)
            {
                o.OnNotify();
            }
        }

        public void AddObserver(Observer o)
        {
            if (observers == null) observers = new List<Observer>();
            if (o != null)
                observers.Add(o);

        }

        public void RemoveObserver(Observer o)
        {
            if (observers == null) return;
            observers.Remove(o);
        }
        public void AddSuject(ISubject subject)
        {
            if (observer == null) observer = new ObserverBot(Notify);
            if (subject != null)
            {
                subject.AddObserver(observer);
            }
        }
    }
}
```

># Turn System

```
턴에 변경사항이 생기면 동작해야 하는 여러 개체들이 있었습니다. 변경사항 확인이 필요할 때마다 개체
들이 직접 접근하면 구조가 복잡해 질 것 같아서 해당 기능의 분리가 필요하다고 생각했습니다.  

따라서 턴 시스템은 Broadcaster를 통해서 턴과 관련된 이벤트를 알리고, 이 정보가 필요한 개체는 필요한 채널을
브로드캐스터로부터 구독해서 확인할 수 있도록 의도했습니다.  
```
  ## 코드
``` C#
using ObserverPattern;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using UnityEngine;

public class TurnSystem
{
    private static TurnSystem singleton;
    public static TurnSystem Singleton
    {
        get
        {
            if (TurnSystem.singleton == null)

                singleton = new TurnSystem();


            return singleton;
        }
    }
    private TurnSystem()
    {
        restartObserver = new ObserverBot(Start);
        Broadcaster.RestartChannel.AddObserver(restartObserver);

        Start();
    }

    private List<SlotBoard> players;
    private List<SlotBoard> turns;
    public List<SlotBoard> Turns
    {
        get
        {
            if (turns == null) turns = new List<SlotBoard>();
            return turns.ToList();
        }
    }
    public SlotBoard current
    {
        get
        {
            return Turns.First();
        }
    }
    public bool isMyTurn
    {
        get
        {
            if (GameObject.Find("Me CharacterSlot")?.GetComponent<SlotBoard>() == null) return false;
            return GameObject.Find("Me CharacterSlot").GetComponent<SlotBoard>() == current;
        }
    }
    public List<SlotBoard> otherPlayers
    {
        get
        {
            if (Turns.Count > 0)
            {
                var temp = Turns;
                temp.RemoveAt(0);
                return temp;
            }
            return new List<SlotBoard>();
        }
    }
    private List<SlotBoard> outPlayers;
    private bool play;
    private bool king;

    private ObserverBot timoutObserver;
    private ObserverBot gameStartObserver;
    private ObserverBot gameOverObserver;
    private ObserverBot victoryObserver;
    private ObserverBot restartObserver;
    public SubjectAgent turnChanged;
    public SubjectAgent turnEnd;


    private void Start()
    {
        singleton = this;
        outPlayers = new List<SlotBoard>();
        turns = new List<SlotBoard>();

        players = new List<SlotBoard>();
        players.Add(MyHand.Singleton.GetComponentInParent<SlotBoard>());
        players.AddRange(Deck.SIngleton.otherPlayers);

        foreach (var player in players)
        {
            turns.Add(player);

        }
        RandomSortTurn();


        timoutObserver = new ObserverBot(TurnEnd);
        Broadcaster.SubmitTimeOutChannel.AddObserver(timoutObserver);

        gameStartObserver = new ObserverBot(() =>
        {
            outPlayers = new List<SlotBoard>();
            turnChanged.OnNotify();
            GamePlay(true);
        });
        Broadcaster.GameStartChannel.AddObserver(gameStartObserver);

        gameOverObserver = new ObserverBot(() => GamePlay(false));
        Broadcaster.GameOverChannel.AddObserver(gameOverObserver);

        victoryObserver = new ObserverBot(() => GamePlay(false));
        Broadcaster.VictoryChannel.AddObserver(victoryObserver);

        turnChanged = new SubjectAgent();
        Broadcaster.TurnChangedChannel.AddSuject(turnChanged);

        turnEnd = new SubjectAgent();
        Broadcaster.TurnEndChannel.AddSuject(turnEnd);

    }

    private void next()
    {
        if (king)
        {
            king = false;
        }
        else
        if (turns.Count > 1)
        {

            var temp = turns.First();
            turns.Remove(temp);
            if (!outPlayers.Contains(temp))
                turns.Add(temp);
        }
        if (turns.Count == 1)
            current.Win();
        else
        {
            turnChanged.OnNotify();
        }
    }

    public async void TurnEnd()
    {
        if (!play) return;
        if (!Focus.Singleton.trigger)
            if (AttackFlame.Singleton.Count == 0)
            {
                if (isMyTurn)
                {
                    Deck.SIngleton.Draw();
                }
                else
                {
                    current.Draw();
                }
            }
        SubmitTimer.Singleton.Pause();
        await Task.Delay(50);
        turnEnd.OnNotify();
        await Task.Delay(50);
        next();
    }
    public void Out(SlotBoard player)
    {
        outPlayers.Add(player);

    }
    private void GamePlay(bool play) => this.play = play;

    public void Jump()
    {
        if (turns.Count < 2) return;
        var temp = turns.First();
        turns.RemoveAt(0);
        turns.Add(temp);
    }
    public void King()
    {
        if (turns.Count < 2) return;
        king = true;

    }
    public void Reverse()
    {
        if (turns.Count < 2) return;
        turns.Reverse();
        king = true;
    }
    private void RandomSortTurn()
    {
        for (int i = 0; i < UnityEngine.Random.Range(0, 3); i++)
        {
            var temp = turns.First();
            turns.Remove(temp);
            turns.Add(temp);
        }
    }

}
```

># Slot Board
```
턴이 바뀔 때마다 이전 턴 플레이어의 선택에 의해서 상태가 변동되어야 했습니다.
턴은 슬롯 보드가 관리할 필요가 없기 때문에 단순히 Broadcaster를 통해서
턴의 흐름을 관찰하고 자신의 상태를 갱신할 수 있도록 의도했습니다.
```
  ## 코드
``` C#
public class SlotBoard : MonoBehaviour, ISubject
{
    [SerializeField]
    private GameObject myTurnLight;
    [SerializeField]
    private GameObject nextTurnLight;
    [SerializeField]
    private Animator animTurn;
    [SerializeField]
    private Animator animEmo;
    [SerializeField]
    private string nickName;
    public string NickName
    {
        get
        {
            return nickName;
        }

    }
    [SerializeField]
    private Text nameUI;


    List<Observer> observers;
    private ObserverBot turnObserver;
    private ObserverBot restartObserver;
    private SubjectAgent victorySubject;
    private SubjectAgent lastcardSubject;
    private SubjectAgent gameoverSubject;


    private void Awake()
    {
        restartObserver = new ObserverBot(reset);
        Broadcaster.RestartChannel.AddObserver(restartObserver);

        reset();
    }
    private void reset()
    {
        nameUI.text = NickName;
        animTurn.SetBool("isEnd", false);

        turnObserver = new ObserverBot(SetState);
        Broadcaster.TurnChangedChannel.AddObserver(turnObserver);

        victorySubject = new SubjectAgent();
        Broadcaster.VictoryChannel.AddSuject(victorySubject);

        lastcardSubject = new SubjectAgent();
        Broadcaster.LastcardChannel.AddSuject(lastcardSubject);

        gameoverSubject = new SubjectAgent();
        Broadcaster.GameOverChannel.AddSuject(gameoverSubject);
    }
    private void MyTurn()
    {
        if (nextTurnLight.activeSelf)
            nextTurnLight.SetActive(false);
        myTurnLight.SetActive(true);

        animTurn.Play("Anim_myTurn_Appear");
    }
    private void NextTurn()
    {
        if (myTurnLight.activeSelf)
            myTurnLight.SetActive(false);
        nextTurnLight.SetActive(true);

        animTurn.Play("Anim_Next_Appear");
    }
    private void WaitingTurn()
    {
        if (nextTurnLight.activeSelf)
            nextTurnLight.SetActive(false);
        if (myTurnLight.activeSelf)
            myTurnLight.SetActive(false);

        animTurn.Play("Anim_Waiting_Appear");
    }
    private void End()
    {
        TurnSystem.Singleton.Out(this);
        animTurn.SetBool("isEnd", true);
        myTurnLight.SetActive(false);
        nextTurnLight.SetActive(false);
    }
    public void Win()
    {
        HelpMessage.Singleton.HelpString(NickName + "님의 승리! 게임이 종료됩니다.");
        if (MyHand.Singleton.GetComponentInParent<SlotBoard>() == this)
        {
            victorySubject.Notify();

        }
        else
        {

            gameoverSubject.Notify();

        }
        End();
    }
    public void Lose()
    {
        End();
    }
    public void LastCard()
    {
        lastcardSubject.Notify();
    }
    private void SetState()
    {
        var turn = TurnSystem.Singleton.Turns;


        if (turn.Count > 1 && turn.First().Equals(this))
            MyTurn();
        else if (turn.Count > 2 && turn[1].Equals(this))
            NextTurn();
        else if (turn.Count > 1 && !turn.Any(x => x.Equals(this)))
            End();
        else WaitingTurn();

    }
    private IEnumerator Emotion(Emotion type)
    {
        animEmo.SetBool("End", false);
        animEmo.Play(type.ToString());
        animEmo.speed = 0;
        yield return new WaitForSeconds(0.6f);
        animEmo.speed = 1;
        yield return new WaitForSeconds(2.4f);
        animEmo.SetBool("End", true);

    }
    public void StartEmotion(int type)
    {
        StartCoroutine(Emotion((Emotion)type));
    }
    public void Draw()
    {
        var temp = GetComponentInChildren<HandOther>();
        if (temp != null)
        {
            temp.CardCount++;
            temp.AddCard(Deck.SIngleton.Deal());
        }
    }
    public void AddObserver(Observer o)
    {
        if (observers == null) observers = new List<Observer>();
        observers.Add(o);
    }
    public void RemoveObserver(Observer o)
    {
        if (observers == null) return;
        observers.Remove(o);
    }
    public void Notify()
    {
        if (observers == null) return;
        foreach (var o in observers)
            o.OnNotify();
    }
}

```


># Card Button
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/e8dc2d56-7eef-42c7-973b-c1035e1194ac" width="40%" height="40%"/>
<img src="https://github.com/1506022022/MyPortfolio/assets/88864717/b1adf08e-65d3-4ab1-b0bf-9e69facdecae" width="26.8%" height="26.8%"/>

```
카드는 빈번하게 변화하기 때문에 정보를 데이터 형태로 가볍게 관리하도록 의도했습니다.

따라서 턴의 변화를 관측하며 규칙에 따라 제출 가능 여부를 결정하고, 카드를 초기화 하는 기능을
데이터를 기반해서 구현했습니다.
```
  ## 코드
``` C#
using ObserverPattern;
using UnityEngine;

public class CardButton : MonoBehaviour
{
    [SerializeField]
    private GameObject disableSprite;
    [SerializeField]
    public BoxCollider2D button;
    [SerializeField]
    private Animator getAnim;

    private ObserverBot turnObserver;
    private ObserverBot gameoverObserver;


    private MyHand hand;
    public Card info { get; private set; }
    private bool active = true;

    private void Start()
    {

        hand = FindObjectOfType<MyHand>();
        Rule();

        turnObserver = new ObserverBot(Rule);
        Broadcaster.TurnChangedChannel.AddObserver(turnObserver);

        gameoverObserver = new ObserverBot(() => Active(false));
        Broadcaster.GameOverChannel.AddObserver(gameoverObserver);
    }
    public void OnMouseDown()
    {
        if (!active) return;

        Focus.Singleton.GiveTo(info);
        Broadcaster.TurnChangedChannel.RemoveObserver(turnObserver);
        Destroy(gameObject);

    }
    private void OnMouseEnter()
    {

        hand.Sort();
        GetComponent<SpriteRenderer>().sortingOrder = -32001;
        transform.GetChild(0).GetComponent<SpriteRenderer>().sortingOrder = -32000;
    }

    public void Active(bool _is)
    {
        active = _is;
        if (disableSprite != null)
            disableSprite.SetActive(!_is);
    }
    public void Init(Card info)
    {
        getAnim.Play("Anim_Me_Getani");
        this.info = info;
        var render = GetComponent<SpriteRenderer>();
        render.sprite = CardSprite.Singleton.GetSprite(info);


    }

    public void Rule()
    {
        if (!TurnSystem.Singleton.isMyTurn)
        {
            Active(false);
            return;
        }

        Active(RuleSystem.Rule(info));
    }

}

```
