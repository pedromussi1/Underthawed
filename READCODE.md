<h1>Code Breakdown</h1>

<p>The way I have decide to structure this explanation is by only analyzing the bulkier parts of the game, files that were complex enough to warrant an extended explanation. For better organization and understanding, I have separated the code files in three categories: function, UI, and counters.</p>

<h2>Function</h2>

<p>Player.cs:

Manages the player character's movement and interactions within the kitchen environment.
Handles player input, movement, and interaction events.
Communicates with various kitchen objects such as counters and plates.

KitchenGameManager.cs:

Controls the overall game state, including waiting to start, countdown to start, gameplay, and game over states.
Manages timers for game events, such as the countdown to start and gameplay duration.
Handles player input for starting the game and pausing/unpausing.

DeliveryManager.cs:

Manages the delivery of recipes within the game.
Spawns recipes at intervals during gameplay.
Tracks successful and failed recipe deliveries.

KitchenObject.cs:

Represents interactive objects within the kitchen environment.
Handles the placement and removal of kitchen objects on counters and plates.
Provides functionality for object destruction and interaction with other game elements.

SoundManager.cs:

Controls the playback of audio cues and sound effects throughout the game.
Responds to various game events such as successful recipe deliveries and object interactions to trigger appropriate sound effects.

GameInput.cs:

Manages player input bindings and interaction actions.
Allows rebinding of input actions such as movement, interaction, and pausing.
Handles input events and triggers corresponding actions in the game.
Each of these scripts plays a crucial role in different aspects of the game, such as player control, game state management, object interactions, audio feedback, and input handling. Together, they create an immersive and interactive kitchen environment where players can engage in various activities such as cooking, delivering recipes, and managing kitchen operations. By working together, these scripts provide a seamless and enjoyable gameplay experience for the player.</p>



### <h3>Player.cs</h3>

<details>
<summary>Click to expand code</summary>

```csharp

using System;
using System.Collections;
using System.Collections.Generic;
using System.Runtime.CompilerServices;
using UnityEngine;

public class Player : MonoBehaviour, IKitchenObjectParent
{
    // Singleton instance of the Player class
    public static Player Instance { get; private set; }

    // Events for when the player interacts or changes selected counter
    public event EventHandler OnPickedSomething;
    public event EventHandler<OnSelectedCounterChangedEventArgs> OnSelectedCounterChanged;
    public class OnSelectedCounterChangedEventArgs : EventArgs {
        public BaseCounter selectedCounter;
    }

    // Serialized fields accessible in the Unity Editor
    [SerializeField] private float moveSpeed = 7f;
    [SerializeField] private GameInput gameInput;
    [SerializeField] private LayerMask countersLayerMask;
    [SerializeField] private Transform kitchenObjectHoldPoint;

    // Private variables for player state and interactions
    private bool isWalking;
    private Vector3 lastInteractDir;
    private BaseCounter selectedCounter;
    private KitchenObject kitchenObject;

    // Called when the script instance is being loaded
    private void Awake()
    {
        // Ensure only one instance of Player exists
        if (Instance != null)
        {
            Debug.LogError("There is more than one player instance");
        }
        Instance = this;
    }

    // Called before the first frame update
    private void Start()
    {
        // Subscribe to input events
        gameInput.OnInteractAction += GameInput_OnInteractAction;
        gameInput.OnInteractAlternateAction += GameInput_OnInteractAlternateAction;
    }

    // Event handler for primary interaction action
    private void GameInput_OnInteractAction(object sender, System.EventArgs e)
    {
        // Check if game is playing
        if (!KitchenGameManager.Instance.IsGamePlaying()) return;

        // Perform interaction with selected counter
        if (selectedCounter != null)
        {
            selectedCounter.Interact(this);
        }
    }

    // Event handler for alternate interaction action
    private void GameInput_OnInteractAlternateAction(object sender, EventArgs e)
    {
        // Check if game is playing
        if (!KitchenGameManager.Instance.IsGamePlaying()) return;

        // Perform alternate interaction with selected counter
        if (selectedCounter != null)
        {
            selectedCounter.InteractAlternate(this);
        }
    }

    // Called once per frame
    private void Update()
    {
        // Handle player movement and interactions
        HandleMovement();
        HandleInteractions();
    }

    // Check if player is walking
    public bool IsWalking()
    {
        return isWalking;
    }

    // Handle interactions with kitchen counters
    private void HandleInteractions()
    {
        // Get normalized movement input
        Vector2 inputVector = gameInput.GetMovementVectorNormalized();
        Vector3 moveDir = new Vector3(inputVector.x, 0f, inputVector.y);

        // Check for nearby counters for selection
        float interactDistance = 2f;
        if (Physics.Raycast(transform.position, lastInteractDir, out RaycastHit raycastHit, interactDistance, countersLayerMask)){
            if (raycastHit.transform.TryGetComponent(out BaseCounter baseCounter))
            {
                if (baseCounter != selectedCounter){
                    SetSelectedCounter(baseCounter);
                }
            }else{
                SetSelectedCounter(null);
            }
        }else{
            SetSelectedCounter(null);
        }
    }

    // Handle player movement
    private void HandleMovement()
    {
        // Get normalized movement input
        Vector2 inputVector = gameInput.GetMovementVectorNormalized();
        Vector3 moveDir = new Vector3(inputVector.x, 0f, inputVector.y);

        // Calculate movement distance and check for obstacles
        float moveDistance = moveSpeed * Time.deltaTime;
        float playerRadius = .7f;
        float playerHeight = 2f;
        bool canMove =!Physics.CapsuleCast(transform.position, transform.position + Vector3.up * playerHeight, playerRadius, moveDir, moveDistance);

        // If movement is obstructed, try moving along X or Z axis
        if (!canMove)
        {
            Vector3 moveDirX = new Vector3(moveDir.x, 0, 0).normalized;
            canMove = moveDir.x != 0 && !Physics.CapsuleCast(transform.position, transform.position + Vector3.up * playerHeight, playerRadius, moveDirX, moveDistance);

            if (canMove)
            {
                moveDir = moveDirX;
            }
            else
            {
                Vector3 moveDirZ = new Vector3(0, 0, moveDir.z).normalized;
                canMove = moveDir.z != 0 && !Physics.CapsuleCast(transform.position, transform.position + Vector3.up * playerHeight, playerRadius, moveDirZ, moveDistance);

                if (canMove)
                {
                    moveDir = moveDirZ;
                }
            }
        }

        // Move the player if possible
        if (canMove)
        {
            transform.position += moveDir * moveDistance;
        }

        // Update walking state and rotation
        isWalking = moveDir != Vector3.zero;
        float rotateSpeed = 10f;
        transform.forward = Vector3.Slerp(transform.forward, moveDir, Time.deltaTime * rotateSpeed);
    }

    // Set the currently selected counter
    private void SetSelectedCounter(BaseCounter selectedCounter)
    {
        this.selectedCounter = selectedCounter;

        // Invoke event with the new selected counter
        OnSelectedCounterChanged?.Invoke(this, new OnSelectedCounterChangedEventArgs
        {
            selectedCounter = selectedCounter
        });
    }

    // Get the transform for holding kitchen objects
    public Transform GetKitchenObjectFollowTransform()
    {
        return kitchenObjectHoldPoint;
    }

    // Set the currently held kitchen object
    public void SetKitchenObject(KitchenObject kitchenObject)
    {
        this.kitchenObject = kitchenObject;

        // Invoke event when an object is picked up
        if (kitchenObject != null)
        {
            OnPickedSomething?.Invoke(this, EventArgs.Empty);
        }
    }

    // Get the currently held kitchen object
    public KitchenObject GetKitchenObject() { return kitchenObject; }

    // Clear the currently held kitchen object
    public void ClearKitchenObject()
    {
        kitchenObject = null;
    }

    // Check if the player is holding a kitchen object
    public bool HasKitchenObject()
    {
        return kitchenObject != null;
    }
}
```
</details>

<hr>

### <h3>KitchenGameManager.cs</h3>

<details>
<summary>Click to expand code</summary>
    
```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using System.Runtime.CompilerServices;
using UnityEngine;

public class KitchenGameManager : MonoBehaviour
{
    // Singleton instance of KitchenGameManager
    public static KitchenGameManager Instance { get; private set; }

    // Events for game state changes and pausing
    public event EventHandler OnStateChanged;
    public event EventHandler OnGamePaused;
    public event EventHandler OnGameUnpaused;

    // Enumeration representing different game states
    private enum State
    {
        WaitingToStart,
        CountdownToStart,
        GamePlaying,
        GameOver,
    }

    // Current game state
    private State state;

    // Timers for countdown and game duration
    private float countdownToStartTimer = 3f;
    private float gamePlayingTimer;
    private float gamePlayingTimerMax = 180; // 3 minutes
    private bool isGamePaused = false;

    // Called when the script instance is being loaded
    private void Awake()
    {
        Instance = this;
        state = State.WaitingToStart; // Initial state
    }

    // Called before the first frame update
    private void Start()
    {
        // Subscribe to input events
        GameInput.Instance.OnPauseAction += GameInput_OnPauseAction;
        GameInput.Instance.OnInteractAction += GameInput_OnInteractAction;
    }

    // Event handler for interact action
    private void GameInput_OnInteractAction(object sender, EventArgs e)
    {
        if (state == State.WaitingToStart)
        {
            // Transition to countdown state when interact action is triggered
            state = State.CountdownToStart;
            OnStateChanged?.Invoke(this, EventArgs.Empty);
        }
    }

    // Event handler for pause action
    private void GameInput_OnPauseAction(object sender, EventArgs e)
    {
        TogglePauseGame(); // Pause or unpause the game
    }

    // Update is called once per frame
    private void Update()
    {
        // Update game state based on current state
        switch (state)
        {
            case State.WaitingToStart:
                // No action required while waiting to start
                break;
            case State.CountdownToStart:
                // Countdown to start timer
                countdownToStartTimer -= Time.deltaTime;
                if (countdownToStartTimer < 0f)
                {
                    // Transition to game playing state when countdown ends
                    state = State.GamePlaying;
                    gamePlayingTimer = gamePlayingTimerMax; // Reset game timer
                    OnStateChanged?.Invoke(this, EventArgs.Empty);
                }
                break;
            case State.GamePlaying:
                // Game playing timer
                gamePlayingTimer -= Time.deltaTime;
                if (gamePlayingTimer < 0f)
                {
                    // Transition to game over state when game time runs out
                    state = State.GameOver;
                    OnStateChanged?.Invoke(this, EventArgs.Empty);
                }
                break;
            case State.GameOver:
                // No action required when game is over
                break;
        }
    }

    // Check if the game is currently playing
    public bool IsGamePlaying()
    {
        return state == State.GamePlaying;
    }

    // Check if countdown to start is active
    public bool IsCountdowntoStartActive()
    {
        return state == State.CountdownToStart;
    }

    // Get remaining time for countdown to start
    public float GetCountdownToStartTimer()
    {
        return countdownToStartTimer;
    }

    // Check if the game is over
    public bool IsGameOver()
    {
        return state == State.GameOver;
    }

    // Get normalized progress of game playing timer
    public float GetGamePlayingTimerNormalized()
    {
        return 1 - (gamePlayingTimer / gamePlayingTimerMax);
    }

    // Toggle pause state of the game
    public void TogglePauseGame()
    {
        isGamePaused = !isGamePaused;
        if (isGamePaused)
        {
            Time.timeScale = 0f; // Pause the game
            OnGamePaused?.Invoke(this, EventArgs.Empty); // Invoke pause event
        }
        else
        {
            Time.timeScale = 1f; // Unpause the game
            OnGameUnpaused?.Invoke(this, EventArgs.Empty); // Invoke unpause event
        }
    }
}
```

</details>

<hr>

### <h3>DeliveryManager.cs</h3>

<details>
<summary>Click to expand code</summary>
    
```csharp

using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class DeliveryManager : MonoBehaviour
{
    // Events for recipe spawning, completion, success, and failure
    public event EventHandler OnRecipeSpawned;
    public event EventHandler OnRecipeCompleted;
    public event EventHandler OnRecipeSuccess;
    public event EventHandler OnRecipeFailed;
    
    // Singleton instance of DeliveryManager
    public static DeliveryManager Instance { get; private set; }

    // Serialized field for recipe list scriptable object
    [SerializeField] private RecipeListSO recipeListSO;

    // List to hold waiting recipes
    private List<RecipeSO> waitingRecipeSOList;

    // Timer variables for spawning recipes
    private float spawnRecipeTimer;
    private float spawnRecipeTimerMax = 4f;
    
    // Maximum number of waiting recipes
    private int waitingRecipesMax = 4;
    
    // Count of successfully delivered recipes
    private int successfulRecipesAmount;

    // Called when the script instance is being loaded
    private void Awake()
    {
        Instance = this;
        waitingRecipeSOList = new List<RecipeSO>();
    }
    
    // Called once per frame
    private void Update()
    {
        // Decrement spawn timer
        spawnRecipeTimer -= Time.deltaTime;
        
        // Spawn a new recipe if conditions are met
        if (spawnRecipeTimer <= 0f)
        {
            spawnRecipeTimer = spawnRecipeTimerMax;

            if (KitchenGameManager.Instance.IsGamePlaying() && waitingRecipeSOList.Count < waitingRecipesMax)
            {
                // Randomly select a recipe from the list
                RecipeSO waitingRecipeSO = recipeListSO.recipeSOList[UnityEngine.Random.Range(0, recipeListSO.recipeSOList.Count)];
                
                // Add the selected recipe to the waiting list
                waitingRecipeSOList.Add(waitingRecipeSO);

                // Invoke event for spawned recipe
                OnRecipeSpawned?.Invoke(this, EventArgs.Empty);
            }
        }
    }

    // Method to deliver a recipe
    public void DeliverRecipe(PlateKitchenObject plateKitchenObject)
    {
        for (int i = 0; i < waitingRecipeSOList.Count; i++)
        {
            RecipeSO waitingRecipeSO = waitingRecipeSOList[i];

            if (waitingRecipeSO.kitchenObjectSOList.Count == plateKitchenObject.GetKitchenObjectSOList().Count)
            {
                // Check if plate contents match the recipe
                bool plateContentsMatchesRecipe = true;
                foreach (KitchenObjectSO recipeKitchenObjectSO in waitingRecipeSO.kitchenObjectSOList)
                {
                    bool ingredientFound = false;
                    foreach (KitchenObjectSO plateKitchenObjectSO in plateKitchenObject.GetKitchenObjectSOList())
                    {
                        if (plateKitchenObjectSO == recipeKitchenObjectSO)
                        {
                            ingredientFound = true;
                            break;
                        }
                    }
                    if (!ingredientFound)
                    {
                        plateContentsMatchesRecipe = false;
                    }
                }

                if (plateContentsMatchesRecipe)
                {
                    // Player delivered the correct recipe
                    successfulRecipesAmount++;

                    // Remove completed recipe from waiting list
                    waitingRecipeSOList.RemoveAt(i);
                    
                    // Invoke events for recipe completion and success
                    OnRecipeCompleted?.Invoke(this, EventArgs.Empty);
                    OnRecipeSuccess?.Invoke(this, EventArgs.Empty);
                    
                    return;
                }
            }
        }

        // No matches found, recipe delivery failed
        OnRecipeFailed?.Invoke(this, EventArgs.Empty);
    }

    // Getter for waiting recipe list
    public List<RecipeSO> GetWaitingRecipeSOList()
    {
        return waitingRecipeSOList;
    }

    // Getter for successful recipe count
    public int GetSuccessfulRecipesAmount()
    {
        return successfulRecipesAmount;
    }
}
```

</details>

<hr>

### <h3>KitchenObject.cs</h3>

<details>
<summary>Click to expand code</summary>
    
```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class KitchenObject : MonoBehaviour
{
    // Serialized field for the kitchen object scriptable object
    [SerializeField] private KitchenObjectSO kitchenObjectSO;

    // Reference to the parent object implementing the IKitchenObjectParent interface
    private IKitchenObjectParent kitchenObjectParent;

    // Getter for the kitchen object scriptable object
    public KitchenObjectSO GetKitchenObjectSO() { return kitchenObjectSO; }

    // Method to set the parent object for this kitchen object
    public void SetKitchenObjectParent(IKitchenObjectParent kitchenObjectParent)
    {
        // Clear previous parent's kitchen object
        if (this.kitchenObjectParent != null)
        {
            this.kitchenObjectParent.ClearKitchenObject();
        }
        
        // Set the new parent and associate this object with it
        this.kitchenObjectParent = kitchenObjectParent;
        
        // Check if the parent already has a kitchen object, log error if true
        if (kitchenObjectParent.HasKitchenObject())
        {
            Debug.LogError("IKitchenObjectParent already has a Kitchen object");
        }
        
        // Set this kitchen object as the child of the parent and reset position
        kitchenObjectParent.SetKitchenObject(this);
        transform.parent = kitchenObjectParent.GetKitchenObjectFollowTransform();
        transform.localPosition = Vector3.zero;
    }

    // Getter for the kitchen object parent
    public IKitchenObjectParent GetKitchenObjectParent() { return kitchenObjectParent; }

    // Method to destroy the kitchen object
    public void DestroySelf()
    {
        // Clear parent's kitchen object reference and destroy the object
        kitchenObjectParent.ClearKitchenObject();
        Destroy(gameObject);
    }

    // Method to check if this kitchen object is a plate, and if so, return it
    public bool TryGetPlate(out PlateKitchenObject plateKitchenObject)
    {
        // Check if this object is a PlateKitchenObject
        if (this is PlateKitchenObject)
        {
            // Cast this object to PlateKitchenObject and return true
            plateKitchenObject = this as PlateKitchenObject;
            return true;
        }
        else
        {
            // Not a plate, return false and null reference
            plateKitchenObject = null;
            return false;
        }
    }

    // Method to spawn a kitchen object based on a scriptable object and parent
    public static KitchenObject SpawnKitchenObject(KitchenObjectSO kitchenObjectSO, IKitchenObjectParent kitchenObjectParent)
    {
        // Instantiate the kitchen object prefab
        Transform kitchenObjectTransform = Instantiate(kitchenObjectSO.prefab);
        
        // Get the KitchenObject component from the instantiated prefab
        KitchenObject kitchenObject = kitchenObjectTransform.GetComponent<KitchenObject>();
        
        // Set the parent for the spawned kitchen object
        kitchenObject.SetKitchenObjectParent(kitchenObjectParent);
        
        // Return the spawned kitchen object
        return kitchenObject;
    }
}

```
</details>

<hr>

    
### <h3>SoundManager.cs</h3>

<details>
<summary>Click to expand code</summary>

```csharp
using System.Collections;
using System.Collections.Generic;
using Unity.VisualScripting.InputSystem;
using UnityEngine;

public class SoundManager : MonoBehaviour
{
    // Constant for player prefs sound effects volume key
    private const string PLAYER_PREFS_SOUND_EFFECTS_VOLUME = "SoundEffectsVolume";

    // Singleton instance of SoundManager
    public static SoundManager Instance { get; private set; }

    // ScriptableObject containing references to audio clips
    [SerializeField] private AudioClipRefsSO audioClipRefsSO;

    // Current volume level
    private float volume = 1f;

    // Called when the script instance is being loaded
    private void Awake()
    {
        Instance = this;

        // Load volume level from player prefs
        volume = PlayerPrefs.GetFloat(PLAYER_PREFS_SOUND_EFFECTS_VOLUME, 1f);
    }

    // Called before the first frame update
    private void Start()
    {
        // Subscribe to various game events for playing sounds
        DeliveryManager.Instance.OnRecipeSuccess += DeliveryManager_OnRecipeSuccess;
        DeliveryManager.Instance.OnRecipeFailed += DeliveryManager_OnRecipeFailed;
        CuttingCounter.OnAnyCut += CuttingCounter_OnAnyCut;
        Player.Instance.OnPickedSomething += Player_OnPickedSomething;
        BaseCounter.OnAnyObjectPlacedHere += BaseCounter_OnAnyObjectPlacedHere;
        TrashCounter.OnAnyObjectTrashed += TrashCounter_OnAnyObjectTrashed;
    }

    // Event handler for trash counter object trashed event
    private void TrashCounter_OnAnyObjectTrashed(object sender, System.EventArgs e)
    {
        TrashCounter trashCounter = sender as TrashCounter;
        PlaySound(audioClipRefsSO.trash, trashCounter.transform.position);
    }

    // Event handler for base counter object placed event
    private void BaseCounter_OnAnyObjectPlacedHere(object sender, System.EventArgs e)
    {
        BaseCounter baseCounter = sender as BaseCounter;
        PlaySound(audioClipRefsSO.objectDrop, baseCounter.transform.position);
    }

    // Event handler for player object picked something event
    private void Player_OnPickedSomething(object sender, System.EventArgs e)
    {
        PlaySound(audioClipRefsSO.objectPickup, Player.Instance.transform.position);
    }

    // Event handler for cutting counter object cut event
    private void CuttingCounter_OnAnyCut(object sender, System.EventArgs e)
    {
        CuttingCounter cuttingCounter = sender as CuttingCounter;
        PlaySound(audioClipRefsSO.chop, cuttingCounter.transform.position);
    }

    // Event handler for delivery manager recipe failed event
    private void DeliveryManager_OnRecipeFailed(object sender, System.EventArgs e)
    {
        DeliveryCounter deliveryCounter = DeliveryCounter.Instance;
        PlaySound(audioClipRefsSO.deliveryFail, deliveryCounter.transform.position);
    }

    // Event handler for delivery manager recipe success event
    private void DeliveryManager_OnRecipeSuccess(object sender, System.EventArgs e)
    {
        DeliveryCounter deliveryCounter = DeliveryCounter.Instance;
        PlaySound(audioClipRefsSO.deliverySuccess, deliveryCounter.transform.position);
    }

    // Method to play a random audio clip from an array at a given position
    private void PlaySound(AudioClip[] audioClipArray, Vector3 position, float volume = 1f)
    {
        PlaySound(audioClipArray[Random.Range(0, audioClipArray.Length)], position, volume);
    }

    // Method to play a specific audio clip at a given position with a volume multiplier
    private void PlaySound(AudioClip audioClip, Vector3 position, float volumeMultiplier = 1f)
    {
        AudioSource.PlayClipAtPoint(audioClip, position, volumeMultiplier * volume);
    }

    // Method to play footstep sound at a given position with a specific volume
    public void PlayFootstepsSound(Vector3 position, float volume)
    {
        PlaySound(audioClipRefsSO.footstep, position, volume);
    }

    // Method to play the countdown sound
    public void PlayCountdownSound()
    {
        PlaySound(audioClipRefsSO.warning, Vector3.zero);
    }

    // Method to play the warning sound at a given position
    public void PlayWarningSound(Vector3 position)
    {
        PlaySound(audioClipRefsSO.warning, position);
    }

    // Method to change the volume level
    public void ChangeVolume()
    {
        volume += .1f;
        if (volume > 1f)
        {
            volume = 0f;
        }

        // Save volume level to player prefs
        PlayerPrefs.SetFloat(PLAYER_PREFS_SOUND_EFFECTS_VOLUME, volume);
        PlayerPrefs.Save();
    }

    // Method to get the current volume level
    public float GetVolume()
    {
        return volume;
    }
}

```
</details>

<hr>

### <h3>GameInput.cs</h3>


<details>
<summary>Click to expand code</summary>
    
```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using System.Runtime.CompilerServices;
using Unity.VisualScripting;
using UnityEngine;
using UnityEngine.InputSystem;

public class GameInput : MonoBehaviour
{
    // Constant for player prefs input bindings key
    private const string PLAYER_PREFS_BINDINGS = "InputBindings";

    // Singleton instance of GameInput
    public static GameInput Instance { get; private set; }

    // Events for various input actions
    public event EventHandler OnInteractAction;
    public event EventHandler OnInteractAlternateAction;
    public event EventHandler OnPauseAction;
    public event EventHandler OnBindingRebind;

    // Enum defining input bindings
    public enum Binding
    {
        Move_Up,
        Move_Down,
        Move_Left,
        Move_Right,
        Interact,
        InteractAlternate,
        Pause,
    }

    // Player input actions reference
    private PlayerInputActions playerInputActions;

    // Called when the script instance is being loaded
    private void Awake()
    {
        Instance = this;
        playerInputActions = new PlayerInputActions();

        // Load saved binding overrides from player prefs
        if (PlayerPrefs.HasKey(PLAYER_PREFS_BINDINGS))
        {
            playerInputActions.LoadBindingOverridesFromJson(PlayerPrefs.GetString(PLAYER_PREFS_BINDINGS));
        }

        // Enable player input actions
        playerInputActions.Player.Enable();

        // Assign event handlers for input actions
        playerInputActions.Player.Interact.performed += Interact_performed;
        playerInputActions.Player.InteractAlternate.performed += InteractAlternate_performed;
        playerInputActions.Player.Pause.performed += Pause_performed;
    }

    // Called when the MonoBehaviour will be destroyed
    private void OnDestroy()
    {
        // Remove event handlers for input actions
        playerInputActions.Player.Interact.performed -= Interact_performed;
        playerInputActions.Player.InteractAlternate.performed -= InteractAlternate_performed;
        playerInputActions.Player.Pause.performed -= Pause_performed;

        // Dispose player input actions
        playerInputActions.Dispose();
    }

    // Event handler for pause action performed
    private void Pause_performed(UnityEngine.InputSystem.InputAction.CallbackContext obj)
    {
        OnPauseAction?.Invoke(this, EventArgs.Empty);
    }

    // Event handler for alternate interact action performed
    private void InteractAlternate_performed(UnityEngine.InputSystem.InputAction.CallbackContext obj)
    {
        OnInteractAlternateAction?.Invoke(this, EventArgs.Empty);
    }

    // Event handler for interact action performed
    private void Interact_performed(UnityEngine.InputSystem.InputAction.CallbackContext obj)
    {
        OnInteractAction?.Invoke(this, EventArgs.Empty);
    }

    // Method to get normalized movement vector
    public Vector2 GetMovementVectorNormalized()
    {
        Vector2 inputVector = playerInputActions.Player.Move.ReadValue<Vector2>();
        return inputVector.normalized;
    }

    // Method to get binding text for a specific input binding
    public string GetBindingText(Binding binding)
    {
        switch (binding)
        {
            default:
            case Binding.Move_Up:
                return playerInputActions.Player.Move.bindings[1].ToDisplayString();
            case Binding.Move_Down:
                return playerInputActions.Player.Move.bindings[2].ToDisplayString();
            case Binding.Move_Left:
                return playerInputActions.Player.Move.bindings[3].ToDisplayString();
            case Binding.Move_Right:
                return playerInputActions.Player.Move.bindings[4].ToDisplayString();
            case Binding.Interact:
                return playerInputActions.Player.Interact.bindings[0].ToDisplayString();
            case Binding.InteractAlternate:
                return playerInputActions.Player.InteractAlternate.bindings[0].ToDisplayString();
            case Binding.Pause:
                return playerInputActions.Player.Pause.bindings[0].ToDisplayString();
        }
    }

    // Method to rebind a specific input binding
    public void RebindBinding(Binding binding, Action onActionRebound)
    {
        // Disable player input actions during rebinding
        playerInputActions.Player.Disable();
        
        // Define input action and binding index based on the input binding
        InputAction inputAction;
        int bindingIndex;
        switch (binding)
        {
            default:
            case Binding.Move_Up:
                inputAction = playerInputActions.Player.Move;
                bindingIndex = 1;
                break;
            case Binding.Move_Down:
                inputAction = playerInputActions.Player.Move;
                bindingIndex = 2;
                break;
            case Binding.Move_Left:
                inputAction = playerInputActions.Player.Move;
                bindingIndex = 3;
                break;
            case Binding.Move_Right:
                inputAction = playerInputActions.Player.Move;
                bindingIndex = 4;
                break;
            case Binding.Interact:
                inputAction = playerInputActions.Player.Interact;
                bindingIndex = 0;
                break;
            case Binding.InteractAlternate:
                inputAction = playerInputActions.Player.InteractAlternate;
                bindingIndex = 0;
                break;
            case Binding.Pause:
                inputAction = playerInputActions.Player.Pause;
                bindingIndex = 0;
                break;
        }

        // Start interactive rebinding process
        inputAction.PerformInteractiveRebinding(bindingIndex)
            .OnComplete(callback =>
            {
                // Re-enable player input actions and invoke callback action
                callback.Dispose();
                playerInputActions.Player.Enable();
                onActionRebound();

                // Save binding overrides to player prefs
                PlayerPrefs.SetString(PLAYER_PREFS_BINDINGS, playerInputActions.SaveBindingOverridesAsJson());
                PlayerPrefs.Save();

                // Invoke event for binding rebind
                OnBindingRebind?.Invoke(this, EventArgs.Empty);
            })
            .Start();
    }
}

```

</details>

<hr>

<h2>UI</h2>

### <h3>DeliveryResultUI.cs</h3>

<details>
<summary>Click to expand code</summary>
    
```csharp
using System.Collections;
using System.Collections.Generic;
using TMPro;
using UnityEngine;
using UnityEngine.UI;

public class DeliveryResultUI : MonoBehaviour
{
    // Constant string for animation trigger
    private const string POPUP = "Popup";

    // Serialized fields accessible in the Unity Inspector
    [SerializeField] private Image backgroundImage;        // Background image for the result UI
    [SerializeField] private Image iconImage;               // Icon image representing success/failure
    [SerializeField] private TextMeshProUGUI messageText;   // Text displaying success/failure message
    [SerializeField] private Color successColor;            // Color for success background
    [SerializeField] private Color failedColor;             // Color for failure background
    [SerializeField] private Sprite successSprite;          // Sprite for success icon
    [SerializeField] private Sprite failedSprite;           // Sprite for failure icon

    private Animator animator;  // Animator component for handling animations

    // Called when the script instance is being loaded
    private void Awake()
    {
        // Get the Animator component attached to the same GameObject
        animator = GetComponent<Animator>();
    }

    // Called before the first frame update
    private void Start()
    {
        // Subscribe to events triggered by DeliveryManager for recipe success and failure
        DeliveryManager.Instance.OnRecipeSuccess += DeliveryManager_OnRecipeSuccess;
        DeliveryManager.Instance.OnRecipeFailed += DeliveryManager_OnRecipeFailed;

        // Deactivate the UI GameObject initially
        gameObject.SetActive(false);
    }

    // Event handler for recipe failure event
    private void DeliveryManager_OnRecipeFailed(object sender, System.EventArgs e)
    {
        // Activate the UI GameObject
        gameObject.SetActive(true);

        // Trigger the popup animation
        animator.SetTrigger(POPUP);

        // Set background color to indicate failure
        backgroundImage.color = failedColor;

        // Set icon image to indicate failure
        iconImage.sprite = failedSprite;

        // Set message text to indicate delivery failure
        messageText.text = "DELIVERY\nFAILED";
    }

    // Event handler for recipe success event
    private void DeliveryManager_OnRecipeSuccess(object sender, System.EventArgs e)
    {
        // Activate the UI GameObject
        gameObject.SetActive(true);

        // Trigger the popup animation
        animator.SetTrigger(POPUP);

        // Set background color to indicate success
        backgroundImage.color = successColor;

        // Set icon image to indicate success
        iconImage.sprite = successSprite;

        // Set message text to indicate delivery success
        messageText.text = "DELIVERY\nSUCCESS";
    }
}

```
</details>
<hr>

### <h3>GamePauseUI.cs</h3>

<details>
<summary>Click to expand code</summary>
    
```csharp

```
</details>
<hr>

### <h3>OptionsUI.cs</h3>

<details>
<summary>Click to expand code</summary>
    
```csharp

```
</details>
<hr>

### <h3>PlateIconsUI.cs</h3>

<details>
<summary>Click to expand code</summary>
    
```csharp

```
</details>
<hr>

### <h3>ProgressBarUI.cs</h3>

<details>
<summary>Click to expand code</summary>
    
```csharp

```
</details>
<hr>

### <h3>TutorialUI.cs</h3>

<details>
<summary>Click to expand code</summary>
    
```csharp

```
</details>
<hr>

