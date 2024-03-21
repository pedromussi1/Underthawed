<h2>Code Breakdown</h2>

### <h3>Player.cs</h3>

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

<hr>

### <h3>KitchenGameManager.cs</h3>

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
<hr>
### <h3>DeliveryManager.cs</h3>

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
<hr>
### <h3>KitchenObject.cs</h3>

```csharp

```
<hr>
### <h3>Player.cs</h3>

```csharp

```
<hr>
### <h3>Player.cs</h3>

```csharp

```
<hr>
### <h3>Player.cs</h3>

```csharp

```
<hr>
### <h3>Player.cs</h3>

```csharp

```

### <h3>Player.cs</h3>

```csharp

```


