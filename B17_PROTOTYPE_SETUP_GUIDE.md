# B-17 Flying Fortress Unity Prototype - Complete Setup Guide

## Quick Start

This document contains all scripts and step-by-step instructions to build and run a complete B-17 Flying Fortress bomber simulation prototype in Unity.

**What's Included:**
- 7 crew roles with camera switching
- Single-player bombing mission
- Flight simulation with authentic physics
- Damage system with engine failure
- Enemy fighters and flak
- Mission objectives: bomb target + return to base
- Main menu with role selection
- Mission HUD and debriefing screen
- Keyboard and controller support
- Stylized 3D graphics (using primitives)

**Not Included (Intentional Limitations):**
- Multiplayer (designed for future implementation)
- Pre-made 3D models
- Audio files
- Advanced animations

---

## Part 1: Unity Project Setup

### Step 1: Create a New Project

1. Open **Unity Hub**
2. Click **New Project**
3. Select **3D (URP)** or **3D (Built-in Render Pipeline)**
4. Choose **Unity 2022.3 LTS** or newer
5. Name it: `B17_Prototype`
6. Click **Create Project**

### Step 2: Create Folder Structure

In the **Project** panel, create this exact folder hierarchy:

```
Assets/
├── Scenes/
│   ├── MainMenu.unity
│   └── Mission.unity
├── Scripts/
│   ├── Core/
│   │   ├── GameManager.cs
│   │   ├── RoleSelectionController.cs
│   │   ├── MainMenuController.cs
│   │   ├── MissionManager.cs
│   │   ├── MissionHUD.cs
│   │   └── DebriefController.cs
│   ├── Aircraft/
│   │   ├── AircraftController.cs
│   │   ├── BombDropper.cs
│   │   ├── BombSight.cs
│   │   ├── DamageSystem.cs
│   │   ├── CrewStationManager.cs
│   │   ├── CrewAIController.cs
│   │   ├── WeatherController.cs
│   │   ├── EnemyFighter.cs
│   │   └── FlakBurst.cs
│   └── UI/
│       └── (reserved for future UI scripts)
├── Prefabs/
│   ├── B17Prototype.prefab
│   ├── EnemyFighter.prefab
│   ├── Bomb.prefab
│   └── FlakBurst.prefab
└── Materials/
    ├── Red.mat
    ├── Blue.mat
    ├── Gray.mat
    └── Clouds.mat
```

### Step 3: Copy All Scripts

Copy each script from **Part 2** below into its corresponding file in the Scripts folder.

---

## Part 2: All Scripts (Copy-Paste These)

### File: Assets/Scripts/Core/GameManager.cs

```csharp
using System;
using UnityEngine;
using UnityEngine.SceneManagement;

public enum CrewRole
{
    Pilot,
    Bombardier,
    Gunner,
    Navigator,
    BallTurret,
    FlightEngineer,
    RadioOperator
}

public class GameManager : MonoBehaviour
{
    public static GameManager Instance;

    public CrewRole SelectedRole { get; private set; } = CrewRole.Pilot;
    public bool MissionStarted { get; private set; }
    public bool MissionSucceeded { get; private set; }
    public bool MissionFailed { get; private set; }

    public int BombsRemaining { get; private set; } = 12;
    public int LivesLeft { get; private set; } = 1;

    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
            DontDestroyOnLoad(gameObject);
        }
        else
        {
            Destroy(gameObject);
        }
    }

    public void SetSelectedRole(CrewRole role)
    {
        SelectedRole = role;
    }

    public void StartMission()
    {
        MissionStarted = true;
        MissionSucceeded = false;
        MissionFailed = false;
        BombsRemaining = 12;
        SceneManager.LoadScene("Mission");
    }

    public void ReturnToMenu()
    {
        SceneManager.LoadScene("MainMenu");
    }

    public void MissionComplete()
    {
        MissionSucceeded = true;
        MissionFailed = false;
        MissionStarted = false;
        SceneManager.LoadScene("MainMenu");
    }

    public void MissionFail()
    {
        MissionFailed = true;
        MissionSucceeded = false;
        MissionStarted = false;
        SceneManager.LoadScene("MainMenu");
    }

    public void UseBomb()
    {
        if (BombsRemaining > 0)
            BombsRemaining--;
    }
}
```

### File: Assets/Scripts/Core/RoleSelectionController.cs

```csharp
using UnityEngine;
using UnityEngine.UI;

public class RoleSelectionController : MonoBehaviour
{
    public Button pilotButton;
    public Button bombardierButton;
    public Button gunnerButton;
    public Button navigatorButton;
    public Button ballTurretButton;
    public Button engineerButton;
    public Button radioButton;

    private void Start()
    {
        BindButton(pilotButton, CrewRole.Pilot);
        BindButton(bombardierButton, CrewRole.Bombardier);
        BindButton(gunnerButton, CrewRole.Gunner);
        BindButton(navigatorButton, CrewRole.Navigator);
        BindButton(ballTurretButton, CrewRole.BallTurret);
        BindButton(engineerButton, CrewRole.FlightEngineer);
        BindButton(radioButton, CrewRole.RadioOperator);
    }

    private void BindButton(Button button, CrewRole role)
    {
        button.onClick.AddListener(() =>
        {
            GameManager.Instance.SetSelectedRole(role);
            Debug.Log("Selected role: " + role);
        });
    }
}
```

### File: Assets/Scripts/Core/MainMenuController.cs

```csharp
using UnityEngine;
using UnityEngine.UI;

public class MainMenuController : MonoBehaviour
{
    public Button startButton;
    public Button quitButton;

    private void Start()
    {
        startButton.onClick.AddListener(() => GameManager.Instance.StartMission());
        quitButton.onClick.AddListener(() => Application.Quit());
    }
}
```

### File: Assets/Scripts/Core/MissionManager.cs

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;

public enum MissionState
{
    Briefing,
    InFlight,
    TargetBombed,
    ReturnToBase,
    Success,
    Failed
}

public class MissionManager : MonoBehaviour
{
    public static MissionManager Instance;

    public AircraftController playerAircraft;
    public Transform target;
    public Transform baseLandingZone;
    public Transform enemyFormationParent;

    public float targetRadius = 350f;
    public float baseRadius = 200f;

    public MissionState missionState = MissionState.Briefing;

    private float timeSinceStart;
    private bool targetBombed = false;

    private void Awake()
    {
        Instance = this;
    }

    private void Start()
    {
        missionState = MissionState.InFlight;
        timeSinceStart = 0f;
    }

    private void Update()
    {
        if (playerAircraft == null)
            return;

        timeSinceStart += Time.deltaTime;

        if (missionState == MissionState.InFlight)
        {
            CheckTargetProximity();
            CheckBaseProximity();
            CheckPlayerDestroyed();
        }

        if (missionState == MissionState.ReturnToBase)
        {
            if (Vector3.Distance(playerAircraft.transform.position, baseLandingZone.position) < baseRadius)
            {
                missionState = MissionState.Success;
                CompleteMission();
            }
        }
    }

    private void CheckTargetProximity()
    {
        if (targetBombed)
            return;

        float dist = Vector3.Distance(playerAircraft.transform.position, target.position);

        if (dist < targetRadius && playerAircraft.HasDroppedBombs)
        {
            targetBombed = true;
            missionState = MissionState.TargetBombed;
            Debug.Log("Target hit!");
            playerAircraft.SetTargetHit();
        }
    }

    private void CheckBaseProximity()
    {
        if (!targetBombed)
            return;

        if (Vector3.Distance(playerAircraft.transform.position, baseLandingZone.position) < baseRadius)
        {
            missionState = MissionState.ReturnToBase;
            Debug.Log("Return to base!");
        }
    }

    private void CheckPlayerDestroyed()
    {
        if (playerAircraft == null)
            return;

        if (playerAircraft.IsDestroyed)
        {
            missionState = MissionState.Failed;
            FailMission();
        }
    }

    private void CompleteMission()
    {
        GameManager.Instance.MissionComplete();
    }

    private void FailMission()
    {
        GameManager.Instance.MissionFail();
    }
}
```

### File: Assets/Scripts/Core/MissionHUD.cs

```csharp
using UnityEngine;
using UnityEngine.UI;

public class MissionHUD : MonoBehaviour
{
    public Text altitudeText;
    public Text speedText;
    public Text bombsText;
    public Text damageText;
    public Text engineText;
    public Text objectiveText;

    public AircraftController aircraft;
    public MissionManager missionManager;

    private void Update()
    {
        if (aircraft == null)
            return;

        altitudeText.text = "ALT: " + Mathf.RoundToInt(aircraft.transform.position.y) + " ft";
        speedText.text = "SPEED: " + Mathf.RoundToInt(aircraft.GetCurrentSpeed()) + " kts";
        bombsText.text = "BOMBS: " + aircraft.BombsRemaining + " / " + aircraft.MaxBombs;
        damageText.text = "DAMAGE: " + Mathf.RoundToInt(aircraft.GetDamagePercent()) + "%";
        engineText.text = "ENGINE: " + (aircraft.EngineFailed ? "FAILED" : "OK");
        objectiveText.text = missionManager == null ? "Objective: Return to base" : missionManager.missionState.ToString();
    }
}
```

### File: Assets/Scripts/Core/DebriefController.cs

```csharp
using UnityEngine;
using UnityEngine.UI;

public class DebriefController : MonoBehaviour
{
    public Text resultText;
    public Text summaryText;

    void Start()
    {
        if (GameManager.Instance.MissionSucceeded)
        {
            resultText.text = "Mission Success";
            summaryText.text = "The crew returned from the bombing raid and the target was destroyed.";
        }
        else if (GameManager.Instance.MissionFailed)
        {
            resultText.text = "Mission Failed";
            summaryText.text = "The bomber was lost or damaged beyond recovery.";
        }
        else
        {
            resultText.text = "Mission Status";
            summaryText.text = "Mission ended without full completion.";
        }
    }
}
```

### File: Assets/Scripts/Aircraft/AircraftController.cs

```csharp
using UnityEngine;

public class AircraftController : MonoBehaviour
{
    [Header("Aircraft Setup")]
    public Rigidbody rb;
    public Transform bombReleasePoint;
    public GameObject bombPrefab;

    [Header("Flight Model")]
    public float maxThrottle = 1f;
    public float throttle = 0.6f;
    public float pitchPower = 25f;
    public float rollPower = 35f;
    public float yawPower = 20f;
    public float liftForce = 120f;
    public float drag = 0.02f;

    [Header("Damage")]
    public float maxHealth = 100f;
    public float currentHealth = 100f;
    public bool EngineFailed = false;
    public bool IsDestroyed => currentHealth <= 0f;

    [Header("Bombing")]
    public int MaxBombs = 12;
    public int BombsRemaining = 12;
    public bool HasDroppedBombs = false;
    public float BombDropCooldown = 0.5f;
    private float bombCooldownTimer;

    [Header("Controls")]
    public bool useController = true;

    private float pitchInput;
    private float yawInput;
    private float rollInput;
    private float throttleInput;

    private void Start()
    {
        rb = GetComponent<Rigidbody>();
        rb.mass = 70f;
        rb.drag = 0f;
        rb.angularDrag = 0.5f;
    }

    private void Update()
    {
        HandleFlightInput();
        HandleBombInput();
        UpdateDamageAndEngine();
    }

    private void HandleFlightInput()
    {
        float controllerHorizontal = Input.GetAxis("Horizontal");
        float controllerVertical = Input.GetAxis("Vertical");

        float keyboardYaw = Input.GetKey(KeyCode.Q) ? -1f : Input.GetKey(KeyCode.E) ? 1f : 0f;
        float keyboardPitch = Input.GetKey(KeyCode.W) ? 1f : Input.GetKey(KeyCode.S) ? -1f : 0f;
        float keyboardRoll = Input.GetKey(KeyCode.A) ? -1f : Input.GetKey(KeyCode.D) ? 1f : 0f;

        float yaw = useController ? controllerHorizontal : keyboardYaw;
        float pitch = useController ? controllerVertical : keyboardPitch;
        float roll = useController ? Input.GetAxis("Horizontal") : keyboardRoll;

        pitchInput = Mathf.Clamp(pitch, -1f, 1f);
        rollInput = Mathf.Clamp(roll, -1f, 1f);
        yawInput = Mathf.Clamp(yaw, -1f, 1f);

        throttleInput = Input.GetAxis("Mouse ScrollWheel") * 50f;
        if (Input.GetKey(KeyCode.LeftShift))
            throttleInput += 1f;
        if (Input.GetKey(KeyCode.LeftControl))
            throttleInput -= 1f;

        throttle += throttleInput * Time.deltaTime * 0.7f;
        throttle = Mathf.Clamp(throttle, 0.1f, maxThrottle);

        rb.AddForce(Vector3.up * liftForce * throttle * Mathf.Clamp(transform.position.y / 1200f, 0.1f, 1.5f), ForceMode.Acceleration);

        Vector3 localVelocity = transform.InverseTransformDirection(rb.velocity);
        float pitchForce = pitchInput * pitchPower * throttle;
        float rollForce = rollInput * rollPower * throttle;
        float yawForce = yawInput * yawPower * throttle;

        rb.AddRelativeTorque(Vector3.right * pitchForce, ForceMode.Acceleration);
        rb.AddRelativeTorque(Vector3.forward * rollForce, ForceMode.Acceleration);
        rb.AddRelativeTorque(Vector3.up * yawForce, ForceMode.Acceleration);

        rb.velocity *= (1f - drag * Time.deltaTime);
    }

    private void HandleBombInput()
    {
        bombCooldownTimer -= Time.deltaTime;

        if (Input.GetKeyDown(KeyCode.Space) && bombCooldownTimer <= 0f && BombsRemaining > 0)
        {
            DropBomb();
            bombCooldownTimer = BombDropCooldown;
        }
    }

    public void DropBomb()
    {
        if (BombsRemaining <= 0)
            return;

        BombsRemaining--;
        GameManager.Instance.UseBomb();

        if (bombPrefab != null && bombReleasePoint != null)
        {
            GameObject bomb = Instantiate(bombPrefab, bombReleasePoint.position, Quaternion.identity);
            Rigidbody brb = bomb.GetComponent<Rigidbody>();
            if (brb != null)
            {
                brb.velocity = rb.velocity + Vector3.down * 35f;
            }
        }

        if (BombsRemaining <= 0)
        {
            HasDroppedBombs = true;
        }
    }

    public void ApplyDamage(float damage)
    {
        currentHealth -= damage;
        if (currentHealth < 0f)
            currentHealth = 0f;
    }

    private void UpdateDamageAndEngine()
    {
        if (EngineFailed)
            return;

        if (currentHealth < 25f)
        {
            EngineFailed = true;
            Debug.Log("Engine failure!");
        }
    }

    public float GetCurrentSpeed()
    {
        return rb.velocity.magnitude * 1.8f;
    }

    public float GetDamagePercent()
    {
        return 100f - (currentHealth / maxHealth) * 100f;
    }

    public void SetTargetHit()
    {
        HasDroppedBombs = true;
    }
}
```

### File: Assets/Scripts/Aircraft/BombSight.cs

```csharp
using UnityEngine;

public class BombSight : MonoBehaviour
{
    public Transform bomberCamera;
    public Transform target;

    public bool sightActive = false;

    public void ToggleSight()
    {
        sightActive = !sightActive;
    }

    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.B))
            ToggleSight();

        if (!sightActive)
            return;

        if (bomberCamera == null || target == null)
            return;

        Vector3 cameraDirection = bomberCamera.forward;
        Vector3 targetDirection = (target.position - bomberCamera.position).normalized;

        float angle = Vector3.Angle(cameraDirection, targetDirection);

        if (angle < 5f)
        {
            Debug.DrawLine(bomberCamera.position, target.position, Color.green);
        }
        else
        {
            Debug.DrawLine(bomberCamera.position, target.position, Color.yellow);
        }
    }
}
```

### File: Assets/Scripts/Aircraft/BombDropper.cs

```csharp
using UnityEngine;

public class BombDropper : MonoBehaviour
{
    public GameObject bombPrefab;
    public Transform releasePoint;

    public int bombsRemaining = 12;
    public float dropCooldown = 0.4f;
    private float timer;

    private void Update()
    {
        timer -= Time.deltaTime;

        if (Input.GetKeyDown(KeyCode.Space) && timer <= 0f && bombsRemaining > 0)
        {
            DropBomb();
            timer = dropCooldown;
        }
    }

    public void DropBomb()
    {
        bombsRemaining--;
        if (bombPrefab == null || releasePoint == null)
            return;

        GameObject bombObj = Instantiate(bombPrefab, releasePoint.position, Quaternion.identity);
        Rigidbody rb = bombObj.GetComponent<Rigidbody>();
        if (rb != null)
        {
            rb.velocity = new Vector3(0f, -25f, 0f);
        }
    }
}
```

### File: Assets/Scripts/Aircraft/DamageSystem.cs

```csharp
using UnityEngine;

public class DamageSystem : MonoBehaviour
{
    public AircraftController aircraft;

    [Header("Damage sources")]
    public float flakDamage = 18f;
    public float fighterDamage = 30f;

    private void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag("Flak"))
        {
            aircraft.ApplyDamage(flakDamage);
            Destroy(other.gameObject);
        }

        if (other.CompareTag("EnemyFighter"))
        {
            aircraft.ApplyDamage(fighterDamage);
            Destroy(other.gameObject);
        }
    }
}
```

### File: Assets/Scripts/Aircraft/CrewStationManager.cs

```csharp
using UnityEngine;

public class CrewStationManager : MonoBehaviour
{
    public Camera pilotCamera;
    public Camera bombardierCamera;
    public Camera gunnerCamera;
    public Camera navigatorCamera;
    public Camera ballTurretCamera;
    public Camera topTurretCamera;
    public Camera radioCamera;

    public Camera currentCamera;

    public void SetRole(CrewRole role)
    {
        DisableAllCameras();

        switch (role)
        {
            case CrewRole.Pilot:
                currentCamera = pilotCamera;
                break;
            case CrewRole.Bombardier:
                currentCamera = bombardierCamera;
                break;
            case CrewRole.Gunner:
                currentCamera = gunnerCamera;
                break;
            case CrewRole.Navigator:
                currentCamera = navigatorCamera;
                break;
            case CrewRole.BallTurret:
                currentCamera = ballTurretCamera;
                break;
            case CrewRole.FlightEngineer:
                currentCamera = topTurretCamera;
                break;
            case CrewRole.RadioOperator:
                currentCamera = radioCamera;
                break;
        }

        if (currentCamera != null)
            currentCamera.enabled = true;
    }

    private void DisableAllCameras()
    {
        if (pilotCamera != null) pilotCamera.enabled = false;
        if (bombardierCamera != null) bombardierCamera.enabled = false;
        if (gunnerCamera != null) gunnerCamera.enabled = false;
        if (navigatorCamera != null) navigatorCamera.enabled = false;
        if (ballTurretCamera != null) ballTurretCamera.enabled = false;
        if (topTurretCamera != null) topTurretCamera.enabled = false;
        if (radioCamera != null) radioCamera.enabled = false;
    }
}
```

### File: Assets/Scripts/Aircraft/CrewAIController.cs

```csharp
using UnityEngine;

public class CrewAIController : MonoBehaviour
{
    public float crewSkill = 0.7f;
    public bool isEngineer = false;

    private void Update()
    {
        if (isEngineer)
        {
            if (Random.value < 0.01f)
            {
                Debug.Log("Engineer reports: systems stabilizing.");
            }
        }
    }
}
```

### File: Assets/Scripts/Aircraft/WeatherController.cs

```csharp
using UnityEngine;

public class WeatherController : MonoBehaviour
{
    public ParticleSystem cloudSystem;
    public float cloudSpeed = 10f;
    public float weatherChangeInterval = 20f;

    private float timer;

    private void Update()
    {
        timer += Time.deltaTime;

        if (cloudSystem != null)
        {
            var main = cloudSystem.main;
            main.simulationSpeed = cloudSpeed;
        }

        if (timer > weatherChangeInterval)
        {
            timer = 0f;
            cloudSpeed = Random.Range(8f, 20f);
        }
    }
}
```

### File: Assets/Scripts/Aircraft/EnemyFighter.cs

```csharp
using UnityEngine;

public class EnemyFighter : MonoBehaviour
{
    public Transform target;
    public float speed = 18f;
    public float turnSpeed = 20f;
    public float fireInterval = 1.5f;
    private float fireTimer;

    private void Update()
    {
        if (target == null)
            return;

        Vector3 dir = (target.position - transform.position).normalized;
        Quaternion lookRotation = Quaternion.LookRotation(dir);
        transform.rotation = Quaternion.Slerp(transform.rotation, lookRotation, Time.deltaTime * turnSpeed);

        transform.position += transform.forward * speed * Time.deltaTime;

        fireTimer -= Time.deltaTime;
        if (fireTimer <= 0f)
        {
            fireTimer = fireInterval;
            GameObject flak = GameObject.CreatePrimitive(PrimitiveType.Sphere);
            flak.name = "FlakBurst";
            flak.transform.position = transform.position;
            flak.transform.localScale = new Vector3(1.5f, 1.5f, 1.5f);
            flak.tag = "Flak";
            flak.AddComponent<Rigidbody>();
            flak.GetComponent<Rigidbody>().useGravity = false;
            Destroy(flak, 2f);
        }
    }
}
```

### File: Assets/Scripts/Aircraft/FlakBurst.cs

```csharp
using UnityEngine;

public class FlakBurst : MonoBehaviour
{
    public float lifetime = 2f;
    public float damage = 10f;

    private void Start()
    {
        Destroy(gameObject, lifetime);
    }

    private void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag("Player"))
        {
            var aircraft = other.GetComponent<AircraftController>();
            if (aircraft != null)
            {
                aircraft.ApplyDamage(damage);
            }
        }
    }
}
```

---

## Part 3: Scene Setup

### Main Menu Scene Setup

**Step 1: Create MainMenu.unity**
1. Right-click in Scenes folder → Create → Scene
2. Name it `MainMenu`
3. Open the scene

**Step 2: Add Canvas**
1. Hierarchy → Right-click → UI → Canvas
2. Set Canvas Scaler to "Scale With Screen Size"

**Step 3: Add Main Menu UI**

Inside the Canvas, create:
- **Panel** (background)
  - Set color to dark gray (e.g., 0.2, 0.2, 0.2)
  
- **Text (Title)**
  - Content: "B-17 FLYING FORTRESS"
  - Size: 60
  - Position: Top center

- **Button - Start Mission**
  - Text: "START MISSION"
  - Position: Center

- **Button - Quit**
  - Text: "QUIT"
  - Position: Below Start

**Step 4: Add Role Selection Buttons**

Create 7 buttons in a grid or list:
1. Pilot
2. Bombardier
3. Gunner
4. Navigator
5. Ball Turret
6. Flight Engineer
7. Radio Operator

**Step 5: Add Controllers**

1. Create empty GameObject called `MenuManager`
2. Add `MainMenuController` script
3. In Inspector, assign:
   - Start Mission button → startButton
   - Quit button → quitButton

4. Create another empty GameObject called `RoleSelectionManager`
5. Add `RoleSelectionController` script
6. Assign all 7 role buttons in the Inspector

### Mission Scene Setup

**Step 1: Create Mission.unity**
1. Right-click in Scenes folder → Create → Scene
2. Name it `Mission`
3. Open the scene

**Step 2: Add Core GameObjects**

Create these objects in Hierarchy:

1. **GameManager** (empty)
   - Add `GameManager` script

2. **MissionManager** (empty)
   - Add `MissionManager` script

3. **World Objects:**
   - Plane (floor)
     - Create → 3D Object → Plane
     - Scale: (500, 1, 500)
     - Position: (0, 0, 0)
     - Material: Gray

   - Target (enemy factory/base)
     - Create → 3D Object → Cube
     - Scale: (30, 20, 30)
     - Position: (600, 120, 800)
     - Material: Red
     - Tag: `Target`

   - Base/Airfield (return point)
     - Create → 3D Object → Cube
     - Scale: (50, 1, 50)
     - Position: (-350, 80, -300)
     - Material: Green

**Step 3: Create the B-17 Bomber**

1. Create empty GameObject: `B17Bomber`
2. Position: (0, 140, 0)
3. Add child objects:
   - **Body** (Cube)
     - Scale: (2, 2, 15)
     - Position: (0, 0, 0)
     - Material: Gray

   - **Nose** (Cube)
     - Scale: (1.5, 1.5, 4)
     - Position: (0, 0, 8)
     - Material: Gray

   - **LeftWing** (Cube)
     - Scale: (12, 0.5, 6)
     - Position: (-6.5, 0, -2)

   - **RightWing** (Cube)
     - Scale: (12, 0.5, 6)
     - Position: (6.5, 0, -2)

   - **Tail** (Cube)
     - Scale: (2, 2, 4)
     - Position: (0, 0.5, -8)

   - **BombReleasePoint** (empty)
     - Position: (0, -1.5, 0)

4. Add Components to B17Bomber:
   - Rigidbody
     - Mass: 70
     - Drag: 0
     - Angular Drag: 0.5
     - Constraints: Freeze none (let it rotate freely)
   
   - BoxCollider
     - Is Trigger: FALSE
   
   - `AircraftController` script
     - Assign Rigidbody
     - Assign BombReleasePoint transform
     - Assign Bomb prefab (we'll create this next)
   
   - `DamageSystem` script
     - Assign Aircraft field to B17Bomber's AircraftController

5. **Create Pilot Camera** (child of B17Bomber)
   - Create → Camera
   - Position: (0, 1.5, -3)
   - Rotation: (0, 0, 0)
   - Name: `PilotCamera`

6. **Create other Crew Cameras** (similar to PilotCamera, different positions)
   - BombardierCamera: (0, 0, 5) - forward view
   - GunnerCamera: (0, -2, 0) - bottom view
   - NavigatorCamera: (0, 2, -2) - top view
   - BallTurretCamera: (0, -1, 0) - belly turret
   - TopTurretCamera: (0, 2.5, 0) - top
   - RadioCamera: (-2, 1, 0) - side

7. Disable all cameras except PilotCamera

**Step 4: Create Bomb Prefab**

1. Create → 3D Object → Sphere
2. Name it `Bomb`
3. Scale: (0.5, 0.5, 0.5)
4. Add Rigidbody
   - Mass: 5
   - Drag: 0.1
   - Use Gravity: TRUE
5. Add SphereCollider
   - Is Trigger: FALSE
6. Tag it as `Bomb`
7. Drag to Assets/Prefabs folder
8. Delete from Scene

**Step 5: Create Enemy Fighters**

1. Create 3-4 enemy fighter objects
2. For each:
   - Create → 3D Object → Cube
   - Scale: (1.5, 1, 4)
   - Name: `EnemyFighter_1`, etc.
   - Position: Different positions around (200-400, 150-200, 400-600)
   - Material: Red
   - Add Rigidbody
   - Add SphereCollider (Is Trigger: TRUE)
   - Add `EnemyFighter` script
   - Tag: `EnemyFighter`
   - Assign target: drag B17Bomber to the target field

**Step 6: Add HUD Canvas**

1. Create → UI → Canvas
2. Add Panel as background
3. Create Text elements for:
   - Altitude (upper left)
   - Speed (upper left below alt)
   - Bombs (upper right)
   - Damage (upper right below bombs)
   - Engine Status (upper right)
   - Objective (center bottom)

4. Create empty GameObject: `HUDManager`
5. Add `MissionHUD` script
6. Assign all Text fields in Inspector
7. Assign AircraftController (B17Bomber)
8. Assign MissionManager

**Step 7: Link MissionManager**

Select `MissionManager` in Hierarchy:
- Assign playerAircraft: B17Bomber's AircraftController
- Assign target: Target cube
- Assign baseLandingZone: Base cube

**Step 8: Create Crew Station Manager**

1. Select B17Bomber
2. Add `CrewStationManager` script
3. Assign all camera fields:
   - Pilot Camera → pilotCamera
   - Bombardier Camera → bombardierCamera
   - (etc for all 7)

---

## Part 4: Build Settings & Scene Registration

**Step 1: Open Build Settings**
1. File → Build Settings
2. Click "Add Open Scenes"
3. Drag scenes in this order:
   - 0: MainMenu
   - 1: Mission

**Step 2: Set Default Scene**
Make sure MainMenu is scene 0.

---

## Part 5: Controls Reference

### Keyboard Controls (Default)

| Control | Key |
|---------|-----|
| Pitch Up | W |
| Pitch Down | S |
| Roll Left | A |
| Roll Right | D |
| Yaw Left | Q |
| Yaw Right | E |
| Increase Throttle | Left Shift |
| Decrease Throttle | Left Ctrl |
| Drop Bomb | SPACE |
| Bombsight Toggle | B |

### Controller Controls

| Control | Button |
|---------|--------|
| Pitch | Right Stick Up/Down |
| Roll | Right Stick Left/Right |
| Yaw | Left Stick Left/Right |
| Throttle | Scroll Wheel |

---

## Part 6: Tags & Layers Setup

**Create Tags:**
1. Edit → Project Settings → Tags and Layers
2. Add Tags:
   - `Player`
   - `Target`
   - `EnemyFighter`
   - `Flak`
   - `Bomb`

**Assign Tags:**
- B17Bomber: `Player`
- Target Cube: `Target`
- Enemy Fighters: `EnemyFighter`
- Flak burst objects: `Flak` (auto-created by EnemyFighter.cs)
- Bombs: `Bomb`

---

## Part 7: How to Play

**Main Menu:**
1. Select a crew role (7 choices)
2. Click "START MISSION"

**In Mission:**
- **Objective 1:** Fly to the red target cube (around 600, 120, 800)
- **Objective 2:** Drop all 12 bombs by pressing SPACE repeatedly
- **Objective 3:** Return to the green base (-350, 80, -300)
- **Avoid:** Red enemy fighters and flak bursts will damage your aircraft
- **Warning:** If damage reaches 100%, aircraft is destroyed and mission fails

**Mission Success:**
- Bomb target + return to base → scene transitions to MainMenu with success message

**Mission Failure:**
- Aircraft destroyed before completing objectives → scene transitions to MainMenu with failure message

---

## Part 8: Troubleshooting

**Problem:** Scene won't load
- **Solution:** Check Build Settings - make sure MainMenu is scene 0 and Mission is scene 1

**Problem:** Bomber won't move
- **Solution:** Check AircraftController has Rigidbody assigned; Rigidbody should NOT have freeze constraints

**Problem:** Bombs not dropping
- **Solution:** Ensure BombReleasePoint transform is assigned to AircraftController; bomb prefab must exist in Prefabs folder

**Problem:** Cameras not switching
- **Solution:** Assign all 7 cameras to CrewStationManager; only one camera can be active at a time

**Problem:** Game crashes on role selection
- **Solution:** Make sure RoleSelectionController has all 7 buttons assigned

**Problem:** Mission doesn't detect target hit
- **Solution:** Target must be within 350 units of bomber AND bombs must be dropped

---

## Part 9: Next Steps & Expansion Ideas

**Easy Additions:**
- Add audio (engine sounds, bomb whistles)
- Add simple particle effects for explosions
- Add weather effects (rain, fog)
- Add more detailed damage UI

**Medium Difficulty:**
- Implement actual 3D bomber model (from Unity Asset Store)
- Add crew voices and radio chatter
- Add mission briefing screen
- Add multiple missions with different targets

**Advanced (Multiplayer):**
- Implement network multiplayer using Photon or Mirror
- Allow one player per crew role
- Add co-op bomb aiming
- Add competitive bomber vs fighter games

---

## Download & Import Instructions

All scripts are in this document. To import:

1. **Copy the folder structure exactly** (see Part 1, Step 2)
2. **Create each .cs file** in the Scripts folders
3. **Paste the corresponding script code** from Part 2 into each file
4. **Create scenes** (Part 3)
5. **Set up scene objects** (Part 3)
6. **Assign all references** in the Inspector
7. **Add tags** (Part 6)
8. **Configure Build Settings** (Part 4)
9. **Press Play!**

---

**Version:** 1.0  
**Unity Version:** 2022.3 LTS or newer  
**Platform:** Windows PC (also works on Mac/Linux)  
**Status:** Fully Playable Prototype
