# Enemy Death Event System

## Overview
The Enemy class now emits a death event (`onDeath`) instead of handling death consequences directly. This makes enemy death a **contract** that external systems can respond to, rather than hardcoded **behavior**.

## Key Changes

### What Enemy Does Now
- Receives damage via `TakeDamage(float damage)`
- Tracks its own health
- Dies when health reaches zero
- **Emits `onDeath` event** when death occurs
- Stops AI behavior when dead (via `IsDead` property)

### What Enemy Does NOT Do
- Destroy its own GameObject
- Trigger scene loads
- Reference any flow logic
- Handle encounter management

## Usage Examples

### 1. Basic Death Handling (via EnemyManager)
```csharp
// Register enemy with a manager
EnemyManager manager = GetComponent<EnemyManager>();
manager.RegisterEnemy(enemy);

// Manager handles the death event
private void OnEnemyDeath(Enemy enemy)
{
    Debug.Log($"Enemy {enemy.name} died!");
    Destroy(enemy.gameObject, 2f); // Simple cleanup
}
```

### 2. Encounter System
```csharp
public class EncounterSystem : MonoBehaviour
{
    private int enemiesAlive;
    
    public void StartEncounter(Enemy[] enemies)
    {
        enemiesAlive = enemies.Length;
        
        foreach (Enemy enemy in enemies)
        {
            enemy.onDeath.AddListener(() => OnEnemyKilled(enemy));
        }
    }
    
    private void OnEnemyKilled(Enemy enemy)
    {
        enemiesAlive--;
        
        if (enemiesAlive == 0)
        {
            Debug.Log("Encounter complete!");
            // Trigger rewards, open doors, etc.
        }
    }
}
```

### 3. Boss Battle with Phases
```csharp
public class BossController : MonoBehaviour
{
    public Enemy bossEnemy;
    private int phase = 1;
    
    void Start()
    {
        bossEnemy.onDeath.AddListener(OnBossDeath);
    }
    
    private void OnBossDeath()
    {
        Debug.Log($"Boss died in phase {phase}");
        
        if (phase < 3)
        {
            // Resurrect for next phase
            phase++;
            StartCoroutine(ResurrectBoss());
        }
        else
        {
            // Final death
            TriggerVictory();
        }
    }
}
```

### 4. Loot and XP System
```csharp
public class LootSystem : MonoBehaviour
{
    public void RegisterEnemy(Enemy enemy, int xpValue, GameObject[] lootTable)
    {
        enemy.onDeath.AddListener(() => 
        {
            // Grant XP
            PlayerStats.AddXP(xpValue);
            
            // Spawn loot
            SpawnLoot(enemy.transform.position, lootTable);
        });
    }
}
```

### 5. Scene Progression
```csharp
public class LevelManager : MonoBehaviour
{
    public Enemy finalBoss;
    public string nextSceneName;
    
    void Start()
    {
        finalBoss.onDeath.AddListener(OnBossDefeated);
    }
    
    private void OnBossDefeated()
    {
        Debug.Log("Level complete!");
        StartCoroutine(LoadNextLevel());
    }
    
    private IEnumerator LoadNextLevel()
    {
        yield return new WaitForSeconds(3f);
        SceneManager.LoadScene(nextSceneName);
    }
}
```

## Benefits

### Reusability
- Same Enemy class works for:
  - Regular enemies in encounters
  - Boss enemies with special logic
  - Tutorial enemies
  - Testing scenarios

### Separation of Concerns
- Enemy focuses on: health, AI, damage
- External systems handle: spawning, rewards, progression
- No tight coupling between systems

### Testability
- Easy to test enemy death in isolation
- Mock event listeners for unit tests
- No dependencies on scene management

### Extensibility
- Add new death behaviors without modifying Enemy class
- Multiple systems can react to the same death event
- Future-proof for new game modes

## Migration Guide

### Before (Old Approach)
```csharp
private void Die()
{
    isDead = true;
    Destroy(gameObject, 2f); // Hardcoded behavior
}
```

### After (New Approach)
```csharp
// In Enemy.cs
private void Die()
{
    isDead = true;
    onDeath?.Invoke(); // Just signal death
}

// In separate manager/system
enemy.onDeath.AddListener(() => {
    Destroy(enemy.gameObject, 2f); // Behavior in appropriate system
});
```

## Best Practices

1. **Always register listeners in a manager or controller**
   - Don't leave death unhandled
   - Use EnemyManager as default

2. **Unsubscribe when appropriate**
   ```csharp
   void OnDisable()
   {
       enemy.onDeath.RemoveAllListeners();
   }
   ```

3. **Check IsDead property for queries**
   ```csharp
   if (!enemy.IsDead)
   {
       // Apply buff, target, etc.
   }
   ```

4. **Keep AI trivial**
   - Enemy AI is simple by design
   - Complex behavior goes in separate systems

5. **Avoid special cases in Enemy class**
   - Don't add boss-specific code to Enemy
   - Use composition and events instead
