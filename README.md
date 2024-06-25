# UIManagerRefactoring
## UIManager before Refactoring
```sh
using DG.Tweening;
using System;
using UnityEngine;
using UnityEngine.UI;


public class UIManager : MonoBehaviour, IShowRewardedCallbackHandler
{
    //Bu eventleri GameState ismindeki Scriptable Object'e taşı. Herkes oradan subscribe olsun
    public static event Action OnPauseGame;
    public static event Action OnResumeGame;
    public static event Action OnStartGame;
    public static event Action OnRestartGame;
    [Header("Data Source")]
    [SerializeField] private PlayerLevelData playerLevelData;
    [SerializeField] private PlayerGameData playerGameData;
    [SerializeField] private PlayerSettingsData playerSettingsData;
    [SerializeField] private LocalSaveLoadManager localSaveLoadManager;
    [Header("Audio")]
    [SerializeField] private OneShotSFX buttonClickedSound;
    [SerializeField] private OneShotSFX playButtonSound;
    [SerializeField] private OneShotSFX endMenuCanvasSound;

    [Header("Buttons")]  
    [SerializeField] private Button pauseMenuContinueButton;
    [SerializeField] private Button pauseMenuExitButton;
    [SerializeField] private Button pauseMenuSoundButton;
    [SerializeField] private Button pauseMenuMusicButton;
    [SerializeField] private Button pauseMenuMainButton;
    [SerializeField] private Button endMenuReturnButton;
    [SerializeField] private Button endMenuRestartButton;
    [SerializeField] private Button rewardedPanelButton;
    [SerializeField] private Button startMenuStartButton;

    FoodData foodData;
    PlayerManager player;

    PlayerDataPanel playerDataPanel;
    EndMenuPanel endMenuPanel;
    StartMenuPanel startMenuPanel;
    PauseMenuPanel pauseMenuPanel;

    Tween foodGatheredTween;

    bool isAnySettingsChanged;

    private void OnEnable()
    {
        playerLevelData.OnFoodGathered += PlayerLevelData_OnFoodGathered;
    }
    private void OnDisable()
    {
        playerLevelData.OnFoodGathered -= PlayerLevelData_OnFoodGathered;
        player.OnPlayerDied -= Player_OnPlayerDied;
    }

    private void PlayerLevelData_OnFoodGathered(int currentFoodGathered)
    {
        playerDataPanel.FoodGathered(currentFoodGathered);
        endMenuPanel.FoodGathered(currentFoodGathered);

    }

    public void Initialize(PlayerManager player, FoodData foodData)
    {
        this.player = player;
        this.foodData = foodData;

        if(this.player != null)
        {
            this.player.OnPlayerDied += Player_OnPlayerDied;
        }

        playerDataPanel = GetComponentInChildren<PlayerDataPanel>();
        endMenuPanel = GetComponentInChildren<EndMenuPanel>();
        startMenuPanel = GetComponentInChildren<StartMenuPanel>();
        pauseMenuPanel = GetComponentInChildren<PauseMenuPanel>();

        SetupButtonListeners();

        LevelPanelsSetup();

        startMenuPanel.PanelAnimIn();     
    }

    private void Player_OnPlayerDied()
    {
        endMenuPanel.PanelAnimIn(playerLevelData.FoodGathered);
        endMenuCanvasSound.PlayOneShot(Vector3.zero);
        LevelPanelsAnimOut();
    }

    private void SetupButtonListeners()
    {
        
        startMenuStartButton.onClick.AddListener(StartButtonClicked);

        pauseMenuMainButton.onClick.AddListener(PauseButtonClicked);
        pauseMenuContinueButton.onClick.AddListener(ContinueButtonClicked);
        pauseMenuExitButton.onClick.AddListener(ExitButtonClicked);
        pauseMenuSoundButton.onClick.AddListener(SoundButtonClicked);
        pauseMenuMusicButton.onClick.AddListener(MusicButtonClicked);

        endMenuRestartButton.onClick.AddListener(RestartButtonClicked);
        endMenuReturnButton.onClick.AddListener(ReturnButtonClicked);
        rewardedPanelButton.onClick.AddListener(WatchRewardedButtonClicked);

    }

    private void ReturnButtonClicked()
    {
        playerGameData.AddFood(playerLevelData.FoodGathered);
        playerLevelData.ResetGatheredFood();
        buttonClickedSound.PlayOneShot(Vector3.zero);
        LevelManager.Instance.LoadScene("RanchScene");
    }

    private void RestartButtonClicked()
    {
        //TBD
        playerLevelData.ResetGatheredFood();
        endMenuPanel.PanelAnimOut();
        LevelPanelsAnimIn();
        playButtonSound.PlayOneShot(Vector3.zero);
        OnRestartGame?.Invoke();
    }

    private void MusicButtonClicked()
    {
        bool wasOn = pauseMenuPanel.ToggleMusicButton();
        playerSettingsData.SetMusicState(!wasOn);
        isAnySettingsChanged = true;     
    }

    private void SoundButtonClicked()
    {
        bool wasOn = pauseMenuPanel.ToggleSoundButton();
        playerSettingsData.SetMusicState(!wasOn);
        isAnySettingsChanged = true;  
    }

    private void ExitButtonClicked()
    {
        MakePauseMenuButtonsUninteractable();
        ReturnButtonClicked();
    }

    private void ContinueButtonClicked()
    {
        if (isAnySettingsChanged) localSaveLoadManager.StartSavingData(LocalSaveType.PlayerSettingsData);
        MakePauseMenuButtonsUninteractable();
        pauseMenuPanel.PanelSubMenuAnimOut(() =>
        {
            OnResumeGame?.Invoke();
            LevelPanelsAnimIn();
        });
        isAnySettingsChanged = false;
        buttonClickedSound.PlayOneShot(Vector3.zero);
    }

    private void PauseButtonClicked()
    {
        OnPauseGame?.Invoke();
        LevelPanelsAnimOut();
        pauseMenuPanel.PanelSubMenuAnimIn(MakePauseMenuButtonsInteractable);
        buttonClickedSound.PlayOneShot(Vector3.zero);
    }

    private void StartButtonClicked()
    {
        startMenuStartButton.interactable = false;
        startMenuPanel.PanelAnimOut(() =>
        {
            LevelPanelsAnimIn();
            OnStartGame?.Invoke();
        });
        
        playButtonSound.PlayOneShot(Vector3.zero);

        if(MusicManager.Instance != null) MusicManager.Instance.IncreaseLayerLevel(4f);

    }

    private void WatchRewardedButtonClicked()
    {
        rewardedPanelButton.interactable = false;
        float myFloat = 0f;
        DOTween.To(() => myFloat, x => myFloat = x, 1, 3).OnComplete(() =>
        {
            rewardedPanelButton.interactable = true;
        });
        AdsManager.Instance.ShowRewarded(this);
    }
    public void OnShowRewardedCompleted()
    {
        playerLevelData.SetFoodGathered(playerLevelData.FoodGathered * 2);
        endMenuPanel.ShowRewardedCompleted(playerLevelData.FoodGathered);
    }

    private void LevelPanelsSetup()
    {
        startMenuPanel.Setup(foodData);
        endMenuPanel.Setup(foodData);
        pauseMenuPanel.Setup();
        playerDataPanel.Setup(foodData);
    }
    private void LevelPanelsAnimIn()
    {
        pauseMenuPanel.PanelAnimIn();
        playerDataPanel.PanelAnimIn();
    }
    private void LevelPanelsAnimOut()
    {
        pauseMenuPanel.PanelAnimOut();
        playerDataPanel.PanelAnimOut();
    }

    private void MakePauseMenuButtonsInteractable()
    {
        pauseMenuContinueButton.interactable = true;
        pauseMenuExitButton.interactable = true;
        pauseMenuSoundButton.interactable = true;
        pauseMenuMusicButton.interactable = true;
    }
    private void MakePauseMenuButtonsUninteractable()
    {
        pauseMenuContinueButton.interactable = false;
        pauseMenuExitButton.interactable = false;
        pauseMenuSoundButton.interactable = false;
        pauseMenuMusicButton.interactable = false;
    }

   
}

```

## UIManager after Refactoring
```sh
using DG.Tweening;
using System;
using UnityEngine;
using UnityEngine.UI;


public class UIManager : MonoBehaviour, IShowRewardedCallbackHandler, ILevelPanelManager
{

    [Header("Event Manager")]
    [SerializeField] private LevelEvents eventManager;

    [Header("Data Source")]
    [SerializeField] private PlayerLevelData playerLevelData;
    [SerializeField] private PlayerGameData playerGameData;
    [SerializeField] private PlayerSettingsData playerSettingsData;
    [SerializeField] private LocalSaveLoadManager localSaveLoadManager;
    [Header("Audio")]
    [SerializeField] private OneShotSFX buttonClickedSound;
    [SerializeField] private OneShotSFX playButtonSound;
    [SerializeField] private OneShotSFX endMenuCanvasSound;

    [Header("Buttons")]  
    [SerializeField] private Button pauseMenuContinueButton;
    [SerializeField] private Button pauseMenuExitButton;
    [SerializeField] private Button pauseMenuSoundButton;
    [SerializeField] private Button pauseMenuMusicButton;
    [SerializeField] private Button pauseMenuMainButton;
    [SerializeField] private Button endMenuReturnButton;
    [SerializeField] private Button endMenuRestartButton;
    [SerializeField] private Button rewardedPanelButton;
    [SerializeField] private Button startMenuStartButton;

    FoodData foodData;
    PlayerManager player;

    IPlayerDataPanel playerDataPanel;
    IEndMenuPanel endMenuPanel;
    IStartMenuPanel startMenuPanel;
    IPauseMenuPanel pauseMenuPanel;

    Tween foodGatheredTween;

    bool isAnySettingsChanged;

    public PlayerLevelData PlayerLevelData => playerLevelData;
    public FoodData FoodData => foodData;

    private void OnEnable()
    {
        playerLevelData.OnFoodGathered += PlayerLevelData_OnFoodGathered;
    }
    private void OnDisable()
    {
        playerLevelData.OnFoodGathered -= PlayerLevelData_OnFoodGathered;
        player.OnPlayerDied -= Player_OnPlayerDied;
    }

    private void PlayerLevelData_OnFoodGathered(int currentFoodGathered)
    {
        playerDataPanel.FoodGathered(currentFoodGathered);
        endMenuPanel.FoodGathered(currentFoodGathered);

    }

    public void Initialize(PlayerManager player, FoodData foodData)
    {
        this.player = player;
        this.foodData = foodData;

        if(this.player != null)
        {
            this.player.OnPlayerDied += Player_OnPlayerDied;
        }

        playerDataPanel = GetComponentInChildren<IPlayerDataPanel>();
        endMenuPanel = GetComponentInChildren<IEndMenuPanel>();
        startMenuPanel = GetComponentInChildren<IStartMenuPanel>();
        pauseMenuPanel = GetComponentInChildren<IPauseMenuPanel>();

        SetupButtonListeners();

        LevelPanelsSetup();

        startMenuPanel.PanelAnimIn();     
    }

    private void Player_OnPlayerDied()
    {
        endMenuPanel.PanelAnimIn();
        endMenuCanvasSound.PlayOneShot(Vector3.zero);
        LevelPanelsAnimOut();
    }

    private void SetupButtonListeners()
    {
        
        startMenuStartButton.onClick.AddListener(StartButtonClicked);

        pauseMenuMainButton.onClick.AddListener(PauseButtonClicked);
        pauseMenuContinueButton.onClick.AddListener(ContinueButtonClicked);
        pauseMenuExitButton.onClick.AddListener(ExitButtonClicked);
        pauseMenuSoundButton.onClick.AddListener(SoundButtonClicked);
        pauseMenuMusicButton.onClick.AddListener(MusicButtonClicked);

        endMenuRestartButton.onClick.AddListener(RestartButtonClicked);
        endMenuReturnButton.onClick.AddListener(ReturnButtonClicked);
        rewardedPanelButton.onClick.AddListener(WatchRewardedButtonClicked);

    }

    private void ReturnButtonClicked()
    {
        playerGameData.AddFood(playerLevelData.FoodGathered);
        playerLevelData.ResetGatheredFood();
        buttonClickedSound.PlayOneShot(Vector3.zero);
        LevelManager.Instance.LoadScene("RanchScene");
    }

    private void RestartButtonClicked()
    {
        //TBD
        playerLevelData.ResetGatheredFood();
        endMenuPanel.PanelAnimOut();
        LevelPanelsAnimIn();
        playButtonSound.PlayOneShot(Vector3.zero);
        eventManager.RestartGame();
    }

    private void MusicButtonClicked()
    {
        bool wasOn = pauseMenuPanel.ToggleMusicButton();
        playerSettingsData.SetMusicState(!wasOn);
        isAnySettingsChanged = true;     
    }

    private void SoundButtonClicked()
    {
        bool wasOn = pauseMenuPanel.ToggleSoundButton();
        playerSettingsData.SetMusicState(!wasOn);
        isAnySettingsChanged = true;  
    }

    private void ExitButtonClicked()
    {
        MakePauseMenuButtonsUninteractable();
        ReturnButtonClicked();
    }

    private void ContinueButtonClicked()
    {
        if (isAnySettingsChanged) localSaveLoadManager.StartSavingData(LocalSaveType.PlayerSettingsData);
        MakePauseMenuButtonsUninteractable();
        pauseMenuPanel.PanelSubMenuAnimOut(() =>
        {
            eventManager.ResumeGame();
            LevelPanelsAnimIn();
        });
        isAnySettingsChanged = false;
        buttonClickedSound.PlayOneShot(Vector3.zero);
    }

    private void PauseButtonClicked()
    {
        eventManager.PauseGame();
        LevelPanelsAnimOut();
        pauseMenuPanel.PanelSubMenuAnimIn(MakePauseMenuButtonsInteractable);
        buttonClickedSound.PlayOneShot(Vector3.zero);
    }

    private void StartButtonClicked()
    {
        startMenuStartButton.interactable = false;
        startMenuPanel.PanelAnimOut(() =>
        {
            LevelPanelsAnimIn();
            eventManager.StartGame();
        });
        
        playButtonSound.PlayOneShot(Vector3.zero);

        if(MusicManager.Instance != null) MusicManager.Instance.IncreaseLayerLevel(4f);

    }

    private void WatchRewardedButtonClicked()
    {
        rewardedPanelButton.interactable = false;
        float myFloat = 0f;
        DOTween.To(() => myFloat, x => myFloat = x, 1, 3).OnComplete(() =>
        {
            rewardedPanelButton.interactable = true;
        });
        AdsManager.Instance.ShowRewarded(this);
    }
    public void OnShowRewardedCompleted()
    {
        playerLevelData.SetFoodGathered(playerLevelData.FoodGathered * 2);
        endMenuPanel.ShowRewardedCompleted(playerLevelData.FoodGathered);
    }

    private void LevelPanelsSetup()
    {
        startMenuPanel.Setup(this);
        endMenuPanel.Setup(this);
        pauseMenuPanel.Setup(this);
        playerDataPanel.Setup(this);
    }
    private void LevelPanelsAnimIn()
    {
        pauseMenuPanel.PanelAnimIn();
        playerDataPanel.PanelAnimIn();
    }
    private void LevelPanelsAnimOut()
    {
        pauseMenuPanel.PanelAnimOut();
        playerDataPanel.PanelAnimOut();
    }

    private void MakePauseMenuButtonsInteractable()
    {
        pauseMenuContinueButton.interactable = true;
        pauseMenuExitButton.interactable = true;
        pauseMenuSoundButton.interactable = true;
        pauseMenuMusicButton.interactable = true;
    }
    private void MakePauseMenuButtonsUninteractable()
    {
        pauseMenuContinueButton.interactable = false;
        pauseMenuExitButton.interactable = false;
        pauseMenuSoundButton.interactable = false;
        pauseMenuMusicButton.interactable = false;
    }

   
}

```
